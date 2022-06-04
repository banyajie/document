#### 原理

##### CRD

> Custom Resource Definitionition 自定义资源类型
>
> K8s 通过 ApiServer 在 Etcd中注册一个自定义的资源类型。通过实现 Custom  Controller 来监听资源对象的事件变化

##### Operator

> 感知应用状态的控制器



#### 开发流程

##### 环境准备

###### 安装 operator-sdk 搭建 docker registry

```shell
# mac 安装 

brew install operator-sdk


# 搭建docker  registry

docker run -d -p 5000:5000 --restart always --name registry -V ~/docker-data/docker_registry:/var/lib/registry registry:2

# docker 配置
# "insecure-registries":["banyajie.com:5000"]

# 查看镜像

http://banyajie.com:5000/v2/_catalog

```



###### 开发operator mod

> 流程
>
> - operator-sdk  -- > add CRD -- > add controller --> build and run 

```shell
# operator 
# 初始化目录
operator-sdk init --domain=example.com --repo=github.com/example-inc/memcached-operator


# 创建api
operator-sdk create api --group cache --version v1 --kind Memcached --resource=true --controller=true


# 创建控制器

# 构建镜像
make docker-build IMG=liumiaocn/memcache:v1

# build、启动
make install && make deploy IMG=liumiaocn/memcache:v1


# 创建CRD



```



