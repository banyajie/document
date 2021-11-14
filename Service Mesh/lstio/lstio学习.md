```shell
1：添加命名空间的标签，当应用部署时，告诉lstio去自动的注入 Envoy Sidecar 代理

kubectl label namespace dev istio-injection=enabled



在没有 istio-injection 标记的命名空间中，在部署前可以使用 istioctl kube-inject 命令将 Envoy 容器手动注入到应用的 pod 中：
2： istioctl kube-inject -f <your-app-spec>.yaml | kubectl apply -f -





```





>bookInfo

```yaml

```

