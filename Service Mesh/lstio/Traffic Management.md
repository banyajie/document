### Trafic Management（流量管理）

>   官方文档：https://preliminary.istio.io/latest/docs/concepts/traffic-management/
>
>   Istio’s traffic routing rules let you easily control the flow of traffic and API calls between services. Istio simplifies configuration of service-level properties like circuit breakers, timeouts, and retries, and makes it easy to set up important tasks like A/B testing, canary rollouts, and staged rollouts with percentage-based traffic splits. It also provides out-of-box failure recovery features that help make your application more robust against failures of dependent services or the network.
>
>   

#### Virtual services

#####  虚拟服务的作用

>   -   通过一个虚拟服务解决指向多个服务，例如指向kubernetes下一个namesapce下的所有服务
>   -   结合Gateway配置流量规则，控制入口流量和出口流量

##### 例子

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: reviews
spec:
  hosts:			# 主机，可寻址的主机列表 ip地址、dns、k8s短名称、通配
  - reviews
  http:
  - match:
    - headers:
        end-user:
          exact: jason
    route:
    - destination:
        host: reviews
        subset: v2
  - route:
    - destination:
        host: reviews
        subset: v3
```

##### 说明

-   hosts

    -   虚拟服务作用的下游主机列表（ip、主机、）

-   routing rules（路由规则）

    >match condition
    >
    >```yaml
    >- match:
    >   - headers:
    >       end-user:
    >         exact: jason
    >```
    >
    >Destination
    >
    >```yaml
    >route:
    >- destination:
    >    host: reviews
    >    subset: v2
    >```

-   Routing rule 优先级 

    >   ```yaml
    >    - match:
    >       - headers:
    >           end-user:
    >             exact: jason
    >       route:
    >       - destination:
    >           host: reviews
    >           subset: v2
    >     - route:
    >       - destination:
    >           host: reviews
    >           subset: v3
    >           
    >    # http header 中 end-user=jason 的路由到 reviews v2版本
    >    # 其余默认v3版本
    >   ```
    >
    >   路由规则权限自上而下顺序决定。

-   其他的匹配规则

    >   关键字：
    >
    >   -   exact
    >   -   prefix
    >   -   reg正则
    >
    >   ```yaml
    >   apiVersion: networking.istio.io/v1alpha3
    >   kind: VirtualService
    >   metadata:
    >     name: bookinfo
    >   spec:
    >     hosts:
    >       - bookinfo.com
    >     http:
    >     - corsPolicy:
    >     	allowCredentials:
    >     	allowHeaders:
    >     	allowMethods:
    >     - match:
    >       - uri:					# uri 匹配
    >           prefix: /reviews	# uri 前缀匹配 
    >       route:
    >       - destination:
    >           host: reviews
    >     - match:
    >       - uri:
    >           prefix: /ratings
    >       route:
    >       - destination:
    >           host: ratings
    >   
    >   ---  # AB TEST
    >   spec:
    >     hosts:
    >     - reviews
    >     http:
    >     - route:
    >       - destination:
    >           host: reviews
    >           subset: v1
    >         weight: 75
    >       - destination:
    >           host: reviews
    >           subset: v2
    >         weight: 25
    >         
    >         
    >   ```
    >
    >   

#### Destination rules

##### 规则说明

###### host

###### trafficPolicy

-   loadBalancer

    -   simple

        -   ROUND_ROBIN： 默认规则 。轮询
        -   LEAST_CONN：最少请求负载（随机两个健康主机。选取其中请求负载最少的）
        -   RANDOM：随机一个健康的主机
        -   PASSTHROUGH：

    -   consistentHash

        -   httpHeaderName
        -   httpCookie
        -   useSourceIp
        -   httpQueryParameterName
        -   minimumRingSize

    -   localityLbSetting

    -   

    -   ```yaml
        apiVersion: networking.istio.io/v1alpha3
        kind: DestinationRule
        metadata:
          name: bookinfo-ratings
        spec:
          host: ratings.prod.svc.cluster.local
          trafficPolicy:
            loadBalancer:
              simple: ROUND_ROBIN   # 
              
        ---
        apiVersion: networking.istio.io/v1alpha3
        kind: DestinationRule
        metadata:
          name: bookinfo-ratings
        spec:
          host: ratings.prod.svc.cluster.local
          trafficPolicy:
            loadBalancer:
              consistentHash:		# hash
                httpCookie:
                  name: user
                  ttl: 0s
        ```

        

-   connectionPool

-   outlierDetection

-   tls

-   portLevelSettings

    ```yaml
    
    
    ```

###### subsets

###### exportTo

###### 例子

>   ```yaml
>   apiVersion: networking.istio.io/v1alpha3
>   kind: DestinationRule
>   metadata:
>     name: my-destination-rule
>   spec:
>     host: my-svc
>     trafficPolicy:
>       loadBalancer:		# 负载均衡策略 Random、Weighted、Least equests
>         simple: RANDOM
>     subsets:				# 子集
>     - name: v1
>       labels:
>         version: v1
>     - name: v2
>       labels:
>         version: v2
>       trafficPolicy:
>         loadBalancer:
>           simple: ROUND_ROBIN
>     - name: v3
>       labels:
>         version: v3
>   ```
>
>   

#### Gateway

###### 说明

#### Service Entries

###### 说明

#### Sidecars

###### 说明

#### 网络弹性和测试

###### 超时

>   ```yaml
>   apiVersion: networking.istio.io/v1alpha3
>   kind: VirtualService
>   metadata:
>     name: ratings
>   spec:
>     hosts:
>     - ratings
>     http:
>     - route:
>       - destination:
>           host: ratings
>           subset: v1
>       timeout: 10s
>   
>   ```
>
>   

###### 重试

>   ```yaml
>   apiVersion: networking.istio.io/v1alpha3
>   kind: VirtualService
>   metadata:
>     name: ratings
>   spec:
>     hosts:
>     - ratings
>     http:
>     - route:
>       - destination:
>           host: ratings
>           subset: v1
>       retries:
>         attempts: 3
>         perTryTimeout: 2s
>   
>   ```
>
>   

###### 断路器

>   ```yaml
>   apiVersion: networking.istio.io/v1alpha3
>   kind: DestinationRule
>   metadata:
>     name: reviews
>   spec:
>     host: reviews
>     subsets:
>     - name: v1
>       labels:
>         version: v1
>       trafficPolicy:
>         connectionPool:
>           tcp:
>             maxConnections: 100
>   
>   ```
>
>   

###### 故障注入

>   ```yaml
>   apiVersion: networking.istio.io/v1alpha3
>   kind: VirtualService
>   metadata:
>     name: ratings
>   spec:
>     hosts:
>     - ratings
>     http:
>     - fault:  		# 故障注入
>         delay:
>           percentage:
>             value: 0.1
>           fixedDelay: 5s      
>       route:
>       - destination:
>           host: ratings
>           subset: v1
>   
>   ```
>
>   