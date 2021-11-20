#### kubelet 工作原理

<img src="https://static001.geekbang.org/resource/image/91/03/914e097aed10b9ff39b509759f8b1d03.png" alt="img" style="zoom:50%;" />



SyncLoop：kubelet的工作核心（一个控制循环）

驱动事件：

-   Pod更新事件
-   Pod生命周期变化
-   kubelet本身设置的执行周期
-   定时的清理事件

​        跟其他控制器类似，kubelet 启动的时候，要做的第一件事情，就是设置 Listers，也就是注册它所关心的各种事件的 Informer。这些 Informer，就是 SyncLoop 需要处理的数据的来源。

​		此外，kubelet 还负责维护着很多很多其他的子控制循环（也就是图中的小圆圈）。这些控制循环的名字，一般被称作某某 Manager，比如 Volume Manager、Image Manager、Node Status Manager 等等。

比如：

-   Node Status Manager，就负责响应 Node 的状态变化，然后将 Node 的状态收集起来，并通过 Heartbeat 的方式上报给 APIServer。
-   CPU Manager，就负责维护该 Node 的 CPU 核的信息，以便在 Pod 通过 cpuset 的方式请求 CPU 核的时候，能够正确地管理 CPU 核的使用量和可用量。

