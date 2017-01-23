##CloudStack High Availability源码分析
关于CloudStack HA的设计原理和思路，在官方文档中已经给出了比较清晰的解释
<https://cwiki.apache.org/confluence/display/CLOUDSTACK/High+Availability+Developer%27s+Guide>，这里不再赘述。

本文的主要目的是从代码的实现方面来梳理CS关于HA的逻辑。
重点是理清下图的细节：
![image](pic/cs_ha_host)
***
我们先来看DirectAgentAttache的内部类PingTask,首先我们要知道每一个注册到CS中的主机都有一个对应的DirectAgentAttache,这也就意味着每一个HOST都有一个PingTask线程在后台循环运行，时间间隔是由全局变量ping.interval来指定的，默认是60s.

我们来看PingTask的代码
      
    ServerResource resource = _resource;
    if (resource != null) {
          PingCommand cmd = resource.getCurrentStatus(_id);
          int retried = 0;
          while (cmd == null && ++retried <= _HostPingRetryCount.value()) {
                Thread.sleep(1000*_HostPingRetryTimer.value());
                cmd = resource.getCurrentStatus(_id);
          }
          if (cmd == null) {
                s_logger.warn("Unable to get current status on " + _id + "(" + _name + ")");
                return;
          }
          _agentMgr.handleCommands(DirectAgentAttache.this, seq, new Command[] {cmd});
    }
_id代表host_id,当getCurrentStatus能返回正确的cmd就说明能够Ping通该host，那接下来就是执行_agentMgr.handleCommands

    public void handleCommands(final AgentAttache attache, final long sequence, final Command[] cmds) {
        for (final Pair<Integer, Listener> listener : _cmdMonitors) {
            final boolean processed = listener.second().processCommands(attache.getId(), sequence, cmds);
        }
    }    
其中我们关心BehindOnPingListener，我们来看它的processCommands方法

    @Override
    public boolean processCommands(final long agentId, final long seq, final Command[] commands) {
        final boolean processed = false;
        for (final Command cmd : commands) {
            if (cmd instanceof PingCommand) {
                pingBy(agentId);
            }
        }
        return processed;
    }
接下来是pingBy方法

    public void pingBy(final long agentId) {
        // Update PingMap with the latest time if agent entry exists in the PingMap
        if (_pingMap.replace(agentId, InaccurateClock.getTimeInSeconds()) == null) {
            s_logger.info("PingMap for agent: " + agentId + " will not be updated because agent is no longer in the PingMap");
        }
    }
这里重点就是这个_pingMap，我们看到它其实是一个ConcurrentHashMap,key是agentId(比如hostId),value是一个时间戳，就是当我们这一次如果Ping通之后会把当前时间作为value插入到_pingMap 中。我们回顾一下上面说过PingTask是每ping.interval时间间隔执行一次，所以如果我们的主机是在正常运行的话那么_pingMap就会几乎每ping.interval更新一次。（当然执行getCurrentStatus方法会有一定的延迟）那如果主机出现突然的故障导致网络无法连接的情况下，那_pingMap中的时间就会一直停留在上一次Ping通的那个时间戳。

所以我们来总结一下PingTask的逻辑：就是每隔ping.interval(默认60s)去Ping我们的主机，如果能够Ping通就更新_pingMap中的value为当前时间戳，否则什么都不做。
***
接下来我们要看的另一个后台线程是MonitorTask,同样是每隔ping.interval执行一次，先是方法findAgentsBehindOnPing

        protected List<Long> findAgentsBehindOnPing() {
            final List<Long> agentsBehind = new ArrayList<Long>();
            final long cutoffTime = InaccurateClock.getTimeInSeconds() - getTimeout();
            for (final Map.Entry<Long, Long> entry : _pingMap.entrySet()) {
                if (entry.getValue() < cutoffTime) {
                    agentsBehind.add(entry.getKey());
                }
            }
            return agentsBehind;
        }     
        
        protected long getTimeout() {
            return (long) (PingTimeout.value() * PingInterval.value());
        } 
全局变量ping.timeout默认值是2.5，这段代码的意思就是找出上一次Ping通的时间距离现在超过ping.interval的2.5倍的主机，简单讲就是Ping不通或者Ping通的延时超过我们认为的不合理时间的主机。
正常情况下该方法返回的会是一个空的List，这个时候MonitorTask就结束当前任务。但是如果出现网络延时或者主机故障的时候，就要执行接下来的代码。

    final List<Long> behindAgents = findAgentsBehindOnPing();
    for (final Long agentId : behindAgents) {
        final QueryBuilder<HostVO> sc = QueryBuilder.create(HostVO.class);
        sc.and(sc.entity().getId(), Op.EQ, agentId);
        final HostVO h = sc.find();
        if (h != null) {
            final ResourceState resourceState = h.getResourceState();
            if (resourceState == ResourceState.Disabled || resourceState == ResourceState.Maintenance || resourceState == ResourceState.ErrorInMaintenance) {
                disconnectWithoutInvestigation(agentId, Event.ShutdownRequested);
            } else {
                final HostVO host = _hostDao.findById(agentId);
                if (host != null && (host.getType() == Host.Type.ConsoleProxy || host.getType() == Host.Type.SecondaryStorageVM
                                || host.getType() == Host.Type.SecondaryStorageCmdExecutor)) {
                    disconnectWithoutInvestigation(agentId, Event.ShutdownRequested);
                } else {
                    disconnectWithInvestigation(agentId, Event.PingTimeout);
                }
            }
        }
    }
我们假设出问题的是一台计算节点，那么一路往下将要执行的将是AgentManagerImpl的handleDisconnectWithInvestigation方法

    protected boolean handleDisconnectWithInvestigation(final AgentAttache attache, Status.Event event) {
        final long hostId = attache.getId();
        HostVO host = _hostDao.findById(hostId);
        if (host != null) {
            Status nextStatus = null;
            nextStatus = host.getStatus().getNextStatus(event);
            if (nextStatus == Status.Alert) {
                Status determinedState = investigate(attache);
                if (determinedState == null) {
                    if ((System.currentTimeMillis() >> 10) - host.getLastPinged() > AlertWait.value()) {
                        determinedState = Status.Alert;
                    } else {
                        return false;
                    }
                }
                final Status currentStatus = host.getStatus();
                if (determinedState == Status.Down) {
                    event = Status.Event.HostDown;
                } else if (determinedState == Status.Up) {
                    agentStatusTransitTo(host, Status.Event.Ping, _nodeId);
                    return false;
                } else if (determinedState == Status.Disconnected) {
                    if (currentStatus == Status.Disconnected) {
                        if ((System.currentTimeMillis() >> 10) - host.getLastPinged() > AlertWait.value()) {
                            event = Status.Event.WaitedTooLong;
                        } else {
                            return false;
                        }
                    } else if (currentStatus == Status.Up) {
                        event = Status.Event.AgentDisconnected;
                    }
                }
            } 
        }
        handleDisconnectWithoutInvestigation(attache, event, true, true);
        host = _hostDao.findById(hostId); // Maybe the host magically reappeared?
        if (host != null && host.getStatus() == Status.Down) {
            _haMgr.scheduleRestartForVmsOnHost(host, true);
        }
        return true;
    }
我们先看一下该方法最后的那个if,就是在特定的条件下我们最终的目的就是重启该主机上的所有虚拟机，这才是 HA的真正目的。但是我们要记住我们进入这个handleDisconnectWithInvestigation方法的前提其实是很简单的，就是只要我们发现距离上一次Ping通该主机的时间超过比如说2分半钟就会进入该方法，而我们要真正执行HA应该是要非常确定该主机确实是挂掉了的情况下才发生的。所以该方法前面一大堆都是在反复的确认主机的状态，就如方法名所示Inverstigation(调查)。
我们假设该主机的currentStatus是UP,event我们知道是PingTimeout,所以nextStatus就是Alert。接下来就是执行investigate方法

    protected Status investigate(final AgentAttache agent) {
        final Long hostId = agent.getId();
        final HostVO host = _hostDao.findById(hostId);
        if (host != null && host.getType() != null && !host.getType().isVirtual()) {
            final Answer answer = easySend(hostId, new CheckHealthCommand());
            if (answer != null && answer.getResult()) {
                final Status status = Status.Up;
                return status;
            }
            return _haMgr.investigate(hostId);
        }
        return Status.Alert;
    }
该方法先会向该hostId发送一个CheckHealthCommand，这个时候会有两种可能：
>1、如果能够接受到应答说明此时该主机是正常的就直接返回UP状态，我们在回到handleDisconnectWithInvestigation就会发现此时该任务也就基本结束了意思就是触发该方法的仅仅是临时的网络不通或者什么情况现在主机已经恢复正常

>2.那另一种情况就是CheckHealthCommand没有得到应答，也就是说我直接从management-server去请求你主机你没有反应，那也不代表你就真的挂了，接下来怎么办呢，我们去找各种侦探（investigators）去调查你是否alive

    @Override
    public Status investigate(final long hostId) {
        final HostVO host = _hostDao.findById(hostId);
        if (host == null) {
            return Status.Alert;
        }
        Status hostState = null;
        for (Investigator investigator : investigators) {
            hostState = investigator.isAgentAlive(host);
            if (hostState != null) {
                return hostState;
            }
        }
        return hostState;
    }
那假如我们的主机是一台XenServer的主机的话，最重要的当然是XenServerInvestigator，我们来看它的isAgentAlive方法

    public Status isAgentAlive(Host agent) {
        CheckOnHostCommand cmd = new CheckOnHostCommand(agent);
        List<HostVO> neighbors = _resourceMgr.listAllHostsInCluster(agent.getClusterId());
        for (HostVO neighbor : neighbors) {
            Answer answer = _agentMgr.easySend(neighbor.getId(), cmd);
            if (answer != null && answer.getResult()) {
                CheckOnHostAnswer ans = (CheckOnHostAnswer)answer;
                if (!ans.isDetermined()) {
                    continue;
                }
                return ans.isAlive() ? null : Status.Down;
            }
        }
        return null;
    }
逻辑也很简单就是我直接找不到你我就去找你同一个Cluster中的邻居，我向你的每一个邻居主机发送一个CheckOnHostCommand命令，看它们能不能知道你到底怎么了。关于CheckOnHostCommand命令的具体实现在开头那篇官网的文章里有详细的说明

If the network ping investigation returns that it cannot detect the status of the host, CloudStack HA then relies on the hypervisor specific investigation.  For VmWare, there is no such investigation as the hypervisor host handles its own HA.  For XenServer and KVM, CloudStack HA deploys a monitoring script that writes the current timestamp on to a heartbeat file on shared storage.  If the timestamp cannot be written, the hypervisor host self-fences by rebooting itself.  For these two hypervisors, CloudStack HA sends a CheckOnHostCommand to a neighboring hypervisor host that shares the same storage.  The neighbor then checks on the heartbeat file on shared storage and see if the heartbeat is no longer being written.  If the heartbeat is still being written, the host reports that the host in question is still alive.  If the heartbeat file’s timestamp is lagging behind, after an acceptable timeout value, the host reports that the host in question is down and HA is started on the VMs on that host.

大致的意思是CS会在每一个XenServer和KVM的主机上运行一段监控脚本，这个脚本会将当前时间戳写入一个在共享存储的文件中。如果某一台主机发现自己无法往文件中写入数据将会强制自己重启。
那上面那段代码的逻辑就是向与该被调查的主机共享存储的其他主机发送CheckOnHostCommand命令，邻居主机接受到命令就去查看文件中被调查主机有没有持续的更新时间戳，如果有它就返回相应说该主机is still alive，否则就返回说该主机is down.
这样只有主机确实出了故障无法连接的情况下，handleDisconnectWithInvestigation方法中的determinedState才会是Status.Down，那么event就变成了Status.Event.HostDown，接下来就执行HighAvailabilityManagerImpl的scheduleRestartForVmsOnHost方法重起该主机上的所以虚拟机
，然后是在数据库中出入一个HaWorkVO，然后唤醒CS启动的时候初始化好的WorkerThread,到很重要的同样是HighAvailabilityManagerImpl的restart方法

    protected Long restart(final HaWorkVO work) {
        boolean isHostRemoved = false;
        Boolean alive = null;
        if (work.getStep() == Step.Investigating) {
            if (!isHostRemoved) {
                Investigator investigator = null;
                for (Investigator it : investigators) {
                    investigator = it;
                    try
                    {
    （1）                alive = investigator.isVmAlive(vm, host);
                        break;
                    } catch (UnknownVM e) {
                        s_logger.info(investigator.getName() + " could not find " + vm);
                    }
                }
                boolean fenced = false;
                if (alive == null) {
                    for (FenceBuilder fb : fenceBuilders) {
    （2）            Boolean result = fb.fenceOff(vm, host);
                        if (result != null && result) {
                            fenced = true;
                            break;
                        }
                    }
                }
    （3）        _itMgr.advanceStop(vm.getUuid(), true);
            }
        }
        vm = _itMgr.findById(vm.getId());
    （4）if (!_forceHA && !vm.isHaEnabled()) {
            return null; // VM doesn't require HA
        }
        try {
            HashMap<VirtualMachineProfile.Param, Object> params = new HashMap<VirtualMachineProfile.Param, Object>();
    （5）    if (_haTag != null) {
                params.put(VirtualMachineProfile.Param.HaTag, _haTag);
            }
            WorkType wt = work.getWorkType();
            if (wt.equals(WorkType.HA)) {
                params.put(VirtualMachineProfile.Param.HaOperation, true);
            }
    （6）   try{
                _itMgr.advanceStart(vm.getUuid(), params, null);
            }catch (InsufficientCapacityException e){
                s_logger.warn("Failed to deploy vm " + vmId + " with original planner, sending HAPlanner");
                _itMgr.advanceStart(vm.getUuid(), params, _haPlanners.get(0));
            }
        }
        return (System.currentTimeMillis() >> 10) + _restartRetryInterval;
    }
如上代码我所标记的有5个重点需要关注的。
大致的流程如下：
>（1）调用各个`investigator.isVmAlive`方法，如果isAlive则什么都不做，否则往下走

>（2）调用`fb.fenceOff`方法

>（3）执行`_itMgr.advanceStop`方法

>（4）关于_forceHA变量，因为我在全局变量和数据库的configuration表中都没有找到，所以初始化的值为 FALSE，那么也就是说只有`vm.isHaEnabled`为ture的VM才会继续执行下去，否则直接return了

>（5）`_haTag`的值是由全局变量ha.tag来指定的，默认为空，如果指定了这个值对后面确定VM分配主机很重要，记住这行代码`params.put(VirtualMachineProfile.Param.HaTag, _haTag);`

>（6）这里有没有很熟悉，是的，凡是读过CS创建VM实例的过程代码的人都知道这个方法就是去分配一个VM，那么到这里整个CS的HA执行代码就完成大部分了，接下来就是重启VM,至于该VM能否重启就要依赖各种条件了，比如该Cluster中有没有合适的主机、主机的物理资源是否充足、有没有设置ha.tag、VM有没有使用标签等等这里就不再详述了，后续专门写文章来解释`_itMgr.advanceStart`方法。




