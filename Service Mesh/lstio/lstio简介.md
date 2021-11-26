

###  服务网格（ServiceMesh）

>   服务网格是一个专用的基础设施层，目的在于使得服务与服务之间的通信变得安全、快速和可靠。
>
>   服务网格通常以轻量级网络代理的形式实现并且会与服务代码部署在一起，它会拦截服务所有进站/出站的网络流量。
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



####  lstio核心功能

>-   流量管理：超时、重试、负载均衡、熔断、限流
>    -   Fault Tolerance - 使用响应状态码。istio识别错误码来进行重试时各种定义操作
>    -   金丝雀发布、蓝绿发布
>-   安全性：最终用户身份验证和授权
>-   可观测性：跟踪、监控、记录
>    -   跟踪：在请求头中添加标记
>-   平台

#### 核心组件

> Pilot：服务发现、可观测
>
> Citadel：服务之间的TSL证书
>
> Mixer：制定策略、遥测

#### lstio 架构

>   从架构上来讲，Istio 服务网格是由数据平面（data plane）和控制平面（control plane）组成的。
>
>   Istio拦截所有网络流量，并通过在每个pod中注入智能代理作为sidecar来应用一组规则。启用所有功能的代理包括 **数据平面**， 并且这些代理可由**控制平面** 动态配置。
>
>   **数据平面** 是由以 sidecar 形式部署的 Envoy 代理组成的。这个代理会拦截所有网络之间的通信。它还会收集和报告所有网格流量的遥测数据。
>
>   **控制平面** 负责管理和配置 Envoy 代理。
>
>   ![图4.与数据平面相关的控制平面](https://raw.githubusercontent.com/servicemesher/website/master/content/blog/back-to-microservices-with-istio-p1/61411417ly1g0exu2b2mqj20m80eogn9.jpg)

##### 数据平面

>   

##### 控制平面

>   



#### istio操作

##### 服务说明

>   ![图6情感分析微服务](https://raw.githubusercontent.com/servicemesher/website/master/content/blog/back-to-microservices-with-istio-p1/61411417ly1g0exu2q2ppj20m80apmxy.jpg)

##### istio 注入

>   安装：
>
>   ```shell
>   istioctl manifest apply --set profile=demo
>   ```
>
>   
>
>   自动注入：
>
>   ```shell
>   # 自动注入：1：添加命名空间的标签，当应用部署时，告诉lstio去自动的注入 Envoy Sidecar 代理
>   kubectl label namespace dev istio-injection=enabled
>   
>   # 手动注入：在没有 istio-injection 标记的命名空间中，在部署前可以使用 istioctl kube-inject 命令将 Envoy 容器手动注入到应用的 pod 中：
>   2： istioctl kube-inject -f <your-app-spec>.yaml | kubectl apply -f -
>   ```
>
>   手动注入：

##### 入口网关

>   ingressgateway
>
>   允许流量进入集群的最佳做法是通过Istio的 **入口网关** 将其自身置于集群的边缘，并在传入流量上实现Istio的功能，如路由，负载均衡，安全性和监控。
>
>   **默认情况下Istio将阻止任何传入流量**， 直到我们定义网关。
>
>   
>
>   **定义网关资源**
>
>   ```yaml
>   apiVersion: networking.istio.io/v1beta1
>   kind: Gateway
>   metadata:
>     namespace: dev
>     name: http-gateway
>   spec:
>     selector:
>       istio: ingressgateway
>     servers:
>     - port:
>         number: 80
>         name: http
>         protocol: HTTP
>       hosts:
>       - "*"
>   ```
>
>   网关现在允许在端口80中进行访问，但它不知道在何处路由请求。这需要使用**Virtual Service**来实现。

##### 虚拟服务（Virtural Service）

>   **定义虚拟服务**
>
>   ```yaml
>   apiVersion: networking.istio.io/v1beta1
>   kind: VirtualService
>   metadata:
>     name: sa-external-services
>   spec:
>     hosts:
>     - "*"
>     gateways:						# 选择网关
>     - http-gateway                      
>     http:
>     - match:
>       - uri:
>           exact: /
>       - uri:
>           prefix: /callback
>       - uri:
>           exact: /logout
>       - uri:
>           prefix: /static
>       - uri:
>           regex: '^.*\.(ico|png|jpg)$'
>       route:
>       - destination:
>           host: sa-frontend             # 2
>           port:
>             number: 80
>     - match:
>       - uri:
>           prefix: /sentiment
>       route:
>       - destination:
>           host: sa-web-app
>           port:
>             number: 80
>     - match:
>       - uri:
>           prefix: /feedback
>       route:
>       - destination:
>           host: sa-feedback
>           port:
>             number: 80%             
>   ```
>
>   VirtualService适用于通过**http网关** 发出的请求
>
>   Destination定义请求路由到的服务

##### Destionation (路由规则)

>   Ab-TEST
>
>   ```yaml
>   apiVersion: networking.istio.io/v1alpha3
>   kind: DestinationRule
>   metadata:
>     name: sa-frontend
>   spec:
>     host: sa-frontend			
>     trafficPolicy:			# 策略
>       loadBalancer:			# 
>         consistentHash:
>           httpHeaderName: version   # 根据“version”标头的内容生成一致的哈希
>   ```
>
>   目标规则
>
>   ```yaml
>   apiVersion: networking.istio.io/v1alpha3
>   kind: DestinationRule
>   metadata:
>     name: sa-logic
>   spec:
>     host: sa-logic    # 1
>     subsets:
>     - name: v1        # 2
>       labels:
>         version: v1   # 3
>     - name: v2
>       labels:
>         version: v2  
>   ```
>
>   
>
>   金丝雀发布
>
>   ```yaml
>   apiVersion: networking.istio.io/v1alpha3
>   kind: VirtualService
>   metadata:
>     name: sa-logic
>   spec:
>     hosts:
>       - sa-logic    
>     http:
>     - route: 
>       - destination: 
>           host: sa-logic
>           subset: v1
>         weight: 80
>       - destination: 
>           host: sa-logic
>           subset: v2
>         weight: 20%
>        
>   ```
>
>   镜像服务 - 实践中的虚拟服务
>
>   ```yaml
>   apiVersion: networking.istio.io/v1alpha3
>   kind: VirtualService
>   metadata:
>     name: sa-logic
>   spec:
>     hosts:
>       - sa-logic           # 1
>     http:
>     - route:
>       - destination:
>           host: sa-logic   # 2
>           subset: v1       # 3
>       mirror:              # 4
>         host: sa-logic     
>         subset: v2
>         
>   ```
>
>   



>   
