###CloudStack创建VM源码分析
本文主要目的是梳理CS创建VM的代码逻辑（也就是调用deployVmCmd这个API）。我们忽略前面关于create方法部门而将分析的入口定于VirtualMachineManagerImpl类的orchestrateStart方法，粗略的时序图如下：
![image](pic/cs_deploy_vm)
可以看出大部分工作都在方法planDeployment中完成，概况起来就是确定该VM应该在哪个集群（Cluster）的哪个主机(Host)中运行和系统卷放在哪个存储池(StoragePool)中.

我们先来看方法2`orderClusters`



