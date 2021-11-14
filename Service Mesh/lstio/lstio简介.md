####  服务网格

>   服务网格是一个专用的基础设施层，目的在于使得服务与服务之间的通信变得安全、快速和可靠。
>
>   服务网格通常以轻量级网络代理的形式实现并且会与服务代码部署在一起，它会拦截服务所有进站/出站的网络流量。
>
>   

###  lstio

>   [Istio](https://istio.io/) 是一个适用于 Kubernetes 的开源服务网格实现。Istio 采用的策略是集成一个网络流量代理到 Kubernetes Pod 中，而这个过程是借助[sidecar容器](https://istio.io/latest/docs/setup/additional-setup/sidecar-injection/)实现的。sidecar 容器与服务容器运行在同一个 Pod 中。因为它们运行在系统的 Pod 之中，所以两个容器会共享 IP、生命周期、资源、网络和存储。
>
>   
>
>   Istio 使用[Envoy Proxy](https://www.envoyproxy.io/)作为 sidecar 容器中的网络代理，并且会配置 Pod 通过 Envoy 代理（sidecar 容器）发送所有的入站/出站流量。
>
>   
>
>   Envoy 代理的 sidecar 容器实现了如下的特性：
>
>   -   智能路由和跨服务的负载均衡。
>   -   故障注入。
>   -   回弹性：重试和断路器。
>   -   可观察性和遥测：指标与跟踪。
>   -   安全性：加密和授权。
>   -   全局范围（fleet-wide）的策略执行。



#### lstio 架构

>   从架构上来讲，Istio 服务网格是由数据平面（data plane）和控制平面（control plane）组成的。
>
>   **数据平面** 是由以 sidecar 形式部署的 Envoy 代理组成的。这个代理会拦截所有网络之间的通信。它还会收集和报告所有网格流量的遥测数据。
>
>   **控制平面** 负责管理和配置 Envoy 代理。
>
>   ![img](https://imgopt.infoq.com/fit-in/1200x2400/filters:quality(80)/filters:no_upscale()/articles/microservicilities-istio/en/resources/8image016-1624026324115.jpg)

###  lstio核心功能

>-   流量管理
>-   安全
>-   可观测性
>-   平台

