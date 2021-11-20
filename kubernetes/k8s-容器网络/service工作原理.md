#### service工作原理

1.   iptalbes工作模式

     >   Service 是由 kube-proxy 组件，加上 iptables 来共同实现的
     >
     >   service提交后，一旦它被提交给 Kubernetes，那么 kube-proxy 就可以通过 Service 的 Informer 感知到这样一个 Service 对象的添加。而作为对这个事件的响应，它就会在宿主机上创建这样一条 iptables 规则（你可以通过 iptables-save 看到它）
     >
     >   
     >
     >   -A KUBE-SERVICES -d 10.0.1.175/32 -p tcp -m comment --comment "default/hostnames: cluster IP" -m tcp --dport 80 -j KUBE-SVC-NWV5X2332I4OT4T3
     >
     >   ：凡是目的地址是 10.0.1.175、目的端口是 80 的 IP 包，都应该跳转到另外一条名叫 KUBE-SVC-NWV5X2332I4OT4T3 的 iptables 链进行处理。
     >
     >   
     >
     >   ```shell
     >    KUBE-SVC-NWV5X2332I4OT4T3 规则集合
     >    
     >   -A KUBE-SVC-NWV5X2332I4OT4T3 -m comment --comment "default/hostnames:" -m statistic --mode random --probability 0.33332999982 -j KUBE-SEP-WNBA2IHDGP2BOBGZ
     >   -A KUBE-SVC-NWV5X2332I4OT4T3 -m comment --comment "default/hostnames:" -m statistic --mode random --probability 0.50000000000 -j KUBE-SEP-X3P2623AGDH6CDF3
     >   -A KUBE-SVC-NWV5X2332I4OT4T3 -m comment --comment "default/hostnames:" -j KUBE-SEP-57KPRZ3JQVENLNBR
     >   
     >   
     >   所以。service对应了三条NAT规则
     >   
     >   -A KUBE-SEP-57KPRZ3JQVENLNBR -s 10.244.3.6/32 -m comment --comment "default/hostnames:" -j MARK --set-xmark 0x00004000/0x00004000
     >   -A KUBE-SEP-57KPRZ3JQVENLNBR -p tcp -m comment --comment "default/hostnames:" -m tcp -j DNAT --to-destination 10.244.3.6:9376
     >   
     >   -A KUBE-SEP-WNBA2IHDGP2BOBGZ -s 10.244.1.7/32 -m comment --comment "default/hostnames:" -j MARK --set-xmark 0x00004000/0x00004000
     >   -A KUBE-SEP-WNBA2IHDGP2BOBGZ -p tcp -m comment --comment "default/hostnames:" -m tcp -j DNAT --to-destination 10.244.1.7:9376
     >   
     >   -A KUBE-SEP-X3P2623AGDH6CDF3 -s 10.244.2.3/32 -m comment --comment "default/hostnames:" -j MARK --set-xmark 0x00004000/0x00004000
     >   -A KUBE-SEP-X3P2623AGDH6CDF3 -p tcp -m comment --comment "default/hostnames:" -m tcp -j DNAT --to-destination 10.244.2.3:9376
     >   
     >   
     >   而 DNAT 规则的作用，就是在 PREROUTING 检查点之前，也就是在路由之前，将流入 IP 包的目的地址和端口，改成–to-destination 所指定的新的目的地址和端口。可以看到，这个目的地址和端口，正是被代理 Pod 的 IP 地址和端口。
     >   
     >   
     >   ```

2.   IPVS模式

>   >   IPVS 模式的工作原理，其实跟 iptables 模式类似。当我们创建了前面的 Service 之后，kube-proxy 首先会在宿主机上创建一个虚拟网卡（叫作：kube-ipvs0），并为它分配 Service VIP 作为 IP 地址
>   >
>   >   ```
>   >   
>   >   # ip addr
>   >     ...
>   >     73：kube-ipvs0：<BROADCAST,NOARP>  mtu 1500 qdisc noop state DOWN qlen 1000
>   >     link/ether  1a:ce:f5:5f:c1:4d brd ff:ff:ff:ff:ff:ff
>   >     inet 10.0.1.175/32  scope global kube-ipvs0
>   >     valid_lft forever  preferred_lft forever
>   >     
>   >     
>   >   # kube-proxy 就会通过 Linux 的 IPVS 模块，为这个 IP 地址设置三个 IPVS 虚拟主机，并设置这三个虚拟主机之间使用轮询模式 (rr) 来作为负载均衡策略
>   >   
>   >   
>   >   # ipvsadm -ln
>   >    IP Virtual Server version 1.2.1 (size=4096)
>   >     Prot LocalAddress:Port Scheduler Flags
>   >       ->  RemoteAddress:Port           Forward  Weight ActiveConn InActConn     
>   >     TCP  10.102.128.4:80 rr
>   >       ->  10.244.3.6:9376    Masq    1       0          0         
>   >       ->  10.244.1.7:9376    Masq    1       0          0
>   >       ->  10.244.2.3:9376    Masq    1       0          0
>   >       
>   >   ```
>   >
>   >   IPVS 在内核中的实现其实也是基于 Netfilter 的 NAT 模式，所以在转发这一层上，理论上 IPVS 并没有显著的性能提升。但是，IPVS 并不需要在宿主机上为每个 Pod 设置 iptables 规则，而是把对这些“规则”的处理放到了内核态，从而极大地降低了维护这些规则的代价。这也正印证了我在前面提到过的，“将重要操作放入内核态”是提高性能的重要手段。



3.   Service 与 DNS 的关系

     >   