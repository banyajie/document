#### 服务可观测

1.   Primetheus - 用于监控（发送与网络流量和服务相关的各种信息）

2.   Grafana - 用户可视化

     >   ```shell
     >   kubectl port-forward -n istio-system grafana-b54bb57b9-k5qbm 3000:3000
     >   Forwarding from 127.0.0.1:3000 -> 3000
     >   Forwarding from [::1]:3000 -> 3000
     >   ```
     >
     >   

3.   Jaeger + zipkin - 用户跟踪

4.   Kiali - 服务流量概览（它能够管理 Istio 并观察服务网格参数，比如服务是如何连接的、它们是如何执行的以及 Istio 资源是如何注册的。）

     >   ```shell
     >   kubectl port-forward -n istio-system kiali-d45468dc4-fl8j4 20001:20001
     >   Forwarding from 127.0.0.1:20001 -> 20001
     >   Forwarding from [::1]:20001 -> 20001
     >   ```



