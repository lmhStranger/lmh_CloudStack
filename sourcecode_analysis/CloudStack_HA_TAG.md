###CloudStack中全局变量ha.tag的用法说明
本文主要解释CS中全局变量ha.tag的用法，默认情况下该变量为空，我们先来看一下官方对它的解释
    
    HA tag defining that the host marked with this tag can be used for HA purposes only
假如我们设置该全局变量的值为`test_ha`，那么所有标签为`test_ha`的主机就只能用作HA主机，意思就是说当我们创建常规的客户VM的时候会排除掉这些主机。我们来看FirstFitAllocator类中的allocatorTo方法

    //只有High Availability当中重启虚拟机时该haVmTag才不为空并等于全局变量ha.tag的值
    String haVmTag = (String)vmProfile.getParameter(VirtualMachineProfile.Param.HaTag);
    if (haVmTag != null) {
        clusterHosts = _hostDao.listByHostTag(type, clusterId, podId, dcId, haVmTag);
    } else {
        //如果VM所选计算方案和模板都没有主机标签则查找不带ha.tag的主机
        if (hostTagOnOffering == null && hostTagOnTemplate == null) {
            clusterHosts = _resourceMgr.listAllUpAndEnabledNonHAHosts(type, clusterId, podId, dcId);
        } else {
            List<HostVO> hostsMatchingOfferingTag = new ArrayList<HostVO>();
            List<HostVO> hostsMatchingTemplateTag = new ArrayList<HostVO>();
            //如果计算方案有主机标签则查找带计算方案标签的主机
            //(**ha.tag**)
            if (hasSvcOfferingTag) {
                hostsMatchingOfferingTag = _hostDao.listByHostTag(type, clusterId, podId, dcId, hostTagOnOffering);
            }
            //如果模板有主机标签则查找带模板标签的主机
            if (hasTemplateTag) {
                hostsMatchingTemplateTag = _hostDao.listByHostTag(type, clusterId, podId, dcId, hostTagOnTemplate);
            }
        }
    }
从以上代码可以看出如果是创建常规VM带有ha.tag的主机会被排除，而如果是通过HA虚拟机重启则会忽略计算方案和模板的标签直接查找带有ha.tag标签的主机。

这里有两点需要注意：
>1、If you set ha.tag, be sure to actually use that tag on at least one host in your cloud. If the tag specified in ha.tag is not set for any host in the cloud, the HA-enabled VMs will fail to restart after a crash.就是说如果设置了全局变量ha.tag那么一定要确保至少有一台主机将标签设置为ha.tag的值，否则HA将会失效。这一点从以上代码也可看出。

>2、If a VM is running on a dedicated HA host, then it must be an HA-enabled VM whose original host failed. (With one exception: It is possible for an administrator to manually migrate any VM to a dedicated HA host.)这段话是从官方文档<http://docs.cloudstack.apache.org/projects/cloudstack-administration/en/4.8/reliability.html>摘录的。意思是如果有VM运行在了一台设置了ha.tag标签的主机上，那么该VM一定是HA-enabled并且是原来的主机挂掉导致的（有一个例外那就是管理员主动将VM在线迁移到该主机）。我这里有一点疑问就是说我全局变量设置了并且主机也设置成了ha.tag的标签，我创建一个带有ha.tag主机标签的计算方案，那么在创建VM的时候该VM是不是会分配到该主机呢，经过测试是会分配到的.所以不知道这是一个bug还是说我对 ha.tag的理解跟开发者的设计有误。

关于High Availability的更详细的说明可参考另一篇文章<https://github.com/lmhStranger/lmh_CloudStack/blob/master/sourcecode_analysis/CloudStack_HA.md>

                        

