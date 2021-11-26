###### 介绍

###### 常用操作命令

```shell

kubectl patch
#kubectl patch (-f FILENAME | TYPE NAME) [-p PATCH|--patch-file FILE] [options]
kubectl patch -n ns svc svc_name -p '{"spec":{"type":"NodePort"}}'


kubectl delete -f - 

```

