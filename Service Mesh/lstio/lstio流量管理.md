#### 流量管理（路由、超时、重试、负载均衡）

>   Istio 能够让我们很容易地控制服务之间的网络流量，这是通过两个概念来实现的，即`DestinationRule`和`VirtualService`。
>
>   `DestinationRule`定义了在路由发生之后如何为网络流量提供服务的策略。在 destination rule 中我们可以配置的内容如下所示：
>
>   
>
>   -   网络流量策略
>   -   负载均衡策略
>   -   连接池设置
>   -   mTLS
>   -   回弹性
>   -   使用标签（label）指定服务的子集（subset），这些子集会在`VirtualService`中用到。



kubectl port-forward service/istio-ingressgateway 30883:80 -n istio-system



服务架构

```shell
kubectl get pods -n default

NAME                                READY   STATUS    RESTARTS   AGE
book-service-5cc59cdcfd-5qhb2       2/2     Running   0          79m
rating-service-v1-64b67cd8d-5bfpf   2/2     Running   0          63m
rating-service-v2-66b55746d-f4hpl   2/2     Running   0          63m

kubectl get services -n default
NAME           TYPE           CLUSTER-IP       EXTERNAL-IP   PORT(S)          AGE
book-service   LoadBalancer   10.106.237.42    &lt;pending&gt;     8080:31304/TCP   111m
kubernetes     ClusterIP      10.96.0.1        &lt;none&gt;        443/TCP          132m
rating         LoadBalancer   10.109.106.128   &lt;pending&gt;     8080:31216/TCP   95m

默认情况下，Istio 会使用 round-robin 方式平衡对服务的调用。在本例中，请求会在rating:v1（返回固定的评分值 1）和rating:v2（在启动的时候进行随机的评分计算，在本例中，会对 ID 为 1 的图书返回 3）之间进行平衡。


```



1.   DestinationRule

     >   `DestinationRule`定义了在路由发生之后如何为网络流量提供服务的策略

     >   ```yaml
     >   apiVersion: networking.istio.io/v1alpha3
     >   kind: DestinationRule
     >   metadata:
     >    	name: rating
     >   spec:
     >    	host: rating    # kubernetes service 的dns域名
     >    subsets:        	# 创建两个子集
     >    - labels:
     >        app.kubernetes.io/version: v1.0.0    # 选择器，带标签的pod
     >      name: version-v1
     >    - labels:
     >        app.kubernetes.io/version: v2.0.0    # 
     >      name: version-v2
     >   
     >   ```
     >
     >   

2.   VirtualService

     >   `VirtualService`能够让我们配置请求该如何路由至 Istio 服务网格的服务中。借助 virtual service，实现像 A/B 测试、蓝/绿部署、金丝雀发布或灰度发布这样的策略就会变得非常简单

     >   ```yaml
     >   apiVersion: networking.istio.io/v1alpha3
     >   kind: VirtualService
     >   metadata:
     >    name: rating
     >   spec:
     >    hosts:
     >    - rating
     >    http:
     >    - route:
     >      - destination:     # destination 路由
     >          host: rating
     >          subset: version-v1    # destinationRule中定义的路由，所有流量都到v1版本
     >        weight: 100    # 权重100
     >   
     >   ```
     >
     >   通过修改weight字段实现金丝雀发布
     >
     >   ```yaml
     >   apiVersion: networking.istio.io/v1alpha3
     >   kind: VirtualService
     >   metadata:
     >    name: rating
     >   spec:
     >    hosts:
     >    - rating
     >    http:
     >    - route:
     >      - destination:
     >          host: rating
     >          subset: version-v1
     >        weight: 75    # v1 版本权重
     >      - destination:
     >          host: rating
     >          subset: version-v2  # v2 版本权重
     >        weight: 25
     >   
     >   ```
     >
     >   基于身份的路由
     >
     >   ```yaml
     >   apiVersion: networking.istio.io/v1alpha3
     >   kind: VirtualService
     >   metadata:
     >    name: rating
     >   spec:
     >     hosts:
     >     - rating
     >     http:
     >     - match:            	# 匹配规则
     >       - headers:			# 请求头
     >           end-user:		# end-user=jason用户
     >             exact: jason  
     >       route:
     >       - destination:
     >           host: rating
     >           subset: version-v1
     >     - route:
     >       - destination:
     >           host: rating
     >           subset: version-v2
     >   ```
     >
     >   

3.   回弹性

     >   1：错误（在节点出错情况下可以自动返回）
     >
     >   2：重试配置

     ```yaml
     apiVersion: networking.istio.io/v1alpha3
     kind: VirtualService
     metadata:
      name: rating
     spec:
      hosts:
      - rating
      http:
      - route:
        - destination:
            host: rating
        retries:      		# 重试
          attempts: 2		# 重试次数
          perTryTimeout: 5s	# 超时时间
          retryOn: 5xx     	# 在出现5xx错误的时候重试
     
     
     ```

     >   3：断路器
     >
     >   问题描述：
     >
     >   	1. 如果服务已经处于过载状态的话，发送更多的请求对它的恢复就会导致恢复更慢
     >   	2. 服务出现故障，重试不会解决问题
     >   	3. 每次重试，都会增加负载（网络流量、socket等等）

     ```yaml
     apiVersion: networking.istio.io/v1alpha3
     kind: DestinationRule
     metadata:
      name: rating
     spec:
      host: rating
      subsets:
      - labels:
          version: v1
        name: version-v1
      - labels:
          version: v2
        name: version-v2
      trafficPolicy:        # 流量策略
        connectionPool:
          http:
            http1MaxPendingRequests: 3
            maxRequestsPerConnection: 3
          tcp:
            maxConnections: 3
        outlierDetection:    		# 断路器 - 异常检测
          baseEjectionTime: 3m   	# 触发后跳闸3分钟，在这个时间之后，断路器会保持半开状态，如果再次失败就会一直打开
          consecutive5xxErrors: 1	# 连续错误次数 1次，出现1次就打开
          interval: 1s        		# 时间窗口大小。1秒钟内出现错误1次
          maxEjectionPercent: 100
     
     ```

     

4.   认证

     >   lstio 会自动将代理和工作负载之间的所有网络流量升级为 mTLS，这个过程不需要修改任何的服务代码。与此同时，作为开发人员，我们会使用 HTTP 协议实现服务。当服务被“Istio 化”的时候，服务之间的通信会采用 HTTPS。Istio 会负责管理证书，担任证书颁发机构并撤销/更新证书。

     ```shell
     
     # 校验 mTLS 是否已启用
     istioctl experimental authz check book-service-5cc59cdcfd-5qhb2 -a
     
     LISTENER[FilterChain]     HTTP ROUTE          ALPN        mTLS (MODE)          AuthZ (RULES)
     
     ...
     virtualInbound[5] inbound|8080|http|book-service.default.svc.cluster.local             istio,istio-http/1.0,istio-http/1.1,istio-h2 noneSDS: default     yes (PERMISSIVE)     no (none)
     
     …
     book-service 托管在了 8080 端口，并且以 permissive 策略配置了 mTLS
     ```

     >   授权
     >
     >   1：基于token授权管理
     >
     >   RequestAuthentication 资源对象
     >
     >   应用RequestAuthentication资源对象，这个策略能够确保如果`Authorization`头信息包含 JWT token 的话，它必须是合法的、没有过期的、由正确的用户颁发的并且没有被篡改。
     >
     >   ```yaml
     >   apiVersion: "security.istio.io/v1beta1"
     >   kind: "RequestAuthentication"
     >   metadata:
     >    name: "bookjwt"
     >    namespace: default
     >   spec:
     >    selector:
     >      matchLabels:
     >        app.kubernetes.io/name: book-service
     >    jwtRules:
     >    - issuer: "testing@secure.istio.io"  
     >      jwksUri: "https://gist.githubusercontent.com/lordofthejars/7dad589384612d7a6e18398ac0f10065/raw/ea0f8e7b729fb1df25d4dc60bf17dee409aad204/jwks.json"
     >   
     >   # issuer：token 的合法颁发者。如果所提供的 token 没有在iss JWT 字段指定该颁发者，那么这个 token 就是非法的。
     >   # jwksUri：jwks文件的 URL，它指定了公钥注册的地址，用来校验 token 的签名。
     >   ```
     >
     >   2：基于角色访问控制（role-based access control，RBAC）
     >
     >   ```yaml
     >   apiVersion: security.istio.io/v1beta1
     >   kind: AuthorizationPolicy    # 认证策略
     >   metadata:
     >    name: require-jwt
     >    namespace: default
     >   spec:
     >    selector:
     >      matchLabels:
     >        app.kubernetes.io/name: book-service     # 选择器匹配service
     >    action: ALLOW
     >    rules:   # 规则
     >    - from:
     >      - source:
     >         requestPrincipals: ["testing@secure.istio.io/testing@secure.istio.io"]
     >      when:
     >      - key: request.auth.claims[role]
     >        values: ["customer"]
     >   
     >   # 允许合法token下的 role:customer 
     >   ```
     >
     >   

5.   故障注入

     >   ```yaml
     >   apiVersion: networking.istio.io/v1alpha3
     >   kind: VirtualService
     >   metadata:
     >     name: ratings
     >   spec:
     >     hosts:
     >     - ratings
     >     http:
     >     - match:
     >       - headers:
     >           end-user:
     >             exact: jason
     >       fault:   # 故障注入
     >         delay: # 延迟 针对用户end-user=jason的用户请求。延迟7s
     >           percentage:
     >             value: 100.0
     >           fixedDelay: 7s
     >       route:
     >       - destination:
     >           host: ratings
     >           subset: v1
     >     - route:
     >       - destination:
     >           host: ratings
     >           subset: v1
     >   ```

6.   镜像功能

     >   流量镜像，也称为影子流量，是一个以尽可能低的风险为生产带来变化的强大的功能。镜像会将实时流量的副本发送到镜像服务。镜像流量发生在主服务的关键请求路径之外
     >
     >   
     >
     >   