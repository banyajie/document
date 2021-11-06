```
1：访问minikube中运行的服务


nodePort Access

minikube service --url <service-name>



kubectl patch service istio-ingressgateway -n istio-system -p '{"spec":{"type":"NodePort"}}' 




kiali无法访问的问题：

kubectl get deployments.apps -n istio-system kiali -o yaml

创建secret资源
kubectl create secret generic kiali -n istio-system --from-literal "username=admin" --from-literal "passphrase=admin"

重启kaili

kubectl delete pod -n istio-system kiali-569c9f8b6c-p6d99 --force


端口转发

kubectl port-forward --address=0.0.0.0 -n istio-system kiali-569c9f8b6c-lgf8x 10080:20001














```

