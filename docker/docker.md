#### Docker

##### Docker 架构图

![img](https://gimg2.baidu.com/image_search/src=http%3A%2F%2Ftva1.sinaimg.cn%2Flarge%2F00831rSTly1gd9pnl499jj30tg0gkwj1.jpg&refer=http%3A%2F%2Ftva1.sinaimg.cn&app=2002&size=f9999,10000&q=a80&n=0&g=0n&fmt=jpeg?sec=1640489356&t=1844a0934fbab1649377fb704fe56f19)

-   镜像：容器模板，通过image创建容器
-   容器：
-   仓库：

##### Docker 安装

>   安装：https://docs.docker.com/engine/install/
>
>   配置
>
>   加速器

##### Docker 命令

###### 常用命令

>   ```shell
>   
>   ```
>
>   

###### 镜像命令

>   ```shell
>   docker images 		# 查看主机上的所有镜像
>   docker images -a 	# 所有
>   docker images -q 	# 只显示镜像ID
>   
>   docker search imageName 	# 搜索镜像
>   # 过滤
>   docker search imageName --fileter=STARTS=3000	# 搜索starts > 3000
>   
>   docker pull
>   
>   # 删除
>   docker rmi -f $(docker images -qa)
>   docker images -qa | docker rm -f -
>   
>   
>   docker image COMMEND
>   docker image build -f dockerfile xx xx
>   docker image history 
>   docker image import 		    # docker image import [OPTIONS] file|URL|- [REPOSITORY[:TAG]]
>   docker image inspect xxxx		# inspect
>   docker image load
>   
>   docker image ls
>   docker image pull		
>   docker image push
>   docker image rm
>   docker image save
>   docker image tag
>   docker image prune 			 	# Remove unused images
>   
>   
>   docker commit 当前运行的容器名 新镜像名:版本号
>   
>   
>   ```
>
>   

###### 容器命令

>   ```shell
>   docker run -d				# 后台启动
>   
>   docker run -d iamgeName --name newName -c "while true; do echo xxx; "
>   docker run --name my-nginx -d -p 8080:80 nginx 
>   
>   
>   docker logs -tf -tail 10 imagesID		# 日志
>   
>   docker top containerID		# 查看容器内部的进程信息
>   
>   docker inspect containerID # 查看容器信息
>   
>   # 进入容器、新启动一个交互命令行
>   docker exec -it containerID /bin/bash 
>   
>   # 进入容器（当前正在运行的命令行）
>   docker attach containerID 
>   
>   
>   # 从容器里面cp东西
>   docker cp containerID:/dir /
>   
>   
>   ```
>
>   

###### 操作命令

>   ![img](https://img-blog.csdn.net/2018071915491757?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzE2MjkwNzkx/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)

###### 操作实例

>   Nginx 部署
>
>   ```shell
>   docker search
>   docker pull nginx
>   
>   dokcer run -d --name ngingx01 -p 3344:80 nginx
>   
>   # 配置文件
>   $ docker run --name my-custom-nginx-container -v /host/path/nginx.conf:/etc/nginx/nginx.conf:ro -d nginx
>   
>   docker start、stop
>   ```

##### 可视化工具

>   -   portainer
>   -   Rancher
>   -   

##### Docker 镜像

>   UnionFS - 联合文件系统
>
>   
>
>   ```shell
>   
>   docker commit -m "" -a=作者 容器ID 目标镜像名:版本
>   
>   docker commit -a="byj" -m="do something" containerID target:v1
>   
>   
>   ```
>
>   

##### Docker 数据卷

>   为什么需要数据卷？
>
>   -   数据持久化
>   -   数据共享



>   -   匿名挂载
>
>       -   ```shell
>           docker run -d -P --name nginx01 -v /etc/nginx nginx
>           
>           
>           docker volume ls
>           
>           
>           docker volume inspect volumeID
>           
>           
>           
>           
>           ```
>        
>       -   
>
>   -   具名挂载
>
>   -   数据卷容器
>
>       -   --volumes-from

##### DockerFIle

>   用来构建image
>
>   ```dockerfile
>   FROM  	centos		# 基础镜像
>   MAINTAINER 			# user:email
>   RUN					# 镜像构建过程中需要运行的命令
>   ADD					# 像镜像中添加的内容
>   WORKDIR				# 工作目录
>   VOLUME				# 挂载卷
>   EXPOSE				# 暴露端口
>   CMD					# 容器启动的时候执行的命令，只有最后一个会生效，可被替代
>   ENTRYPOINT			# 容器启动的时候执行的命令，可以追加命令
>   ONBUILD				# 继承
>   COPY				# 类似ADD
>   ENV					# 环境变量
>   
>   ```
>
>   ```shell
>   MAINTAINER <name:email>
>   FROM ImageName
>   RUN <commend>		# RUN /bin/bash -c 'source $HOME/.bashrc; echo $HOME'   # RUN ["/bin/bash", "-c", "echo hello"]
>   CMD	<commend>		# CMD ["/usr/bin/wc","--help"]
>   LABEL <key>=<value> <key>=<value> <key>=<value> ...
>   EXPOSE 90/udp 
>   EXPOSE 80/tcp
>   ENV <key>=<value> ... 	# ENV MY_NAME="John Doe" 	#ENV MY_DOG=Rex\ The\ Dog
>   ADD [--chown=<user>:<group>] <src>... <dest>
>   COPY [--chown=<user>:<group>] <src>... <dest>\
>   ENTRYPOINT ["executable", "param1", "param2"] # 
>   VOLUME ["/data"]
>   USER <user>[:<group>]
>   WORKDIR /path/to/workdir
>   
>   
>   
>   
>   ```
>
>   例子
>
>   ```dockerfile
>   FROM centos
>   MAINTAINER byj<yajieban@sina.cn>
>   
>   ENV MYPATH /usr/local
>   WORKDIR $MYPATH
>   
>   RUN YUM -y install vim
>   RUN YUM -y install net-tools
>   
>   EXPOSE 80
>   
>   CMD echo $MYPATH
>   CMD echo "build end"
>   
>   CMD /bin/bash
>   ```
>
>   docker history 镜像ID    查看构建过程



##### Docker 网络原理

-   Veth peer

-   网桥

    -   Docker0

-   --link

    -   ```shell
        docker exec -it ngx01 ping ngx02
        
        # 通过容器名字是无法联通的
        
        
        docker run -d -P --name ngx03 --link ngx02 nginx
        
        # --link 后
        # docker exec -it ngx03 ping ngx02   是可以联通的
        
        但是
        # docker exec -it ngx02 ping ngx03    是不通的 （反向不可以）
        
        
        
        ```

    -   

-   https://mvwebfs.ali.kugou.com/202204181810/cbfa3635f0e796a4b381be5100c61e2f/KGTX/CLTX002/8dde145f97a335ce44915708bebab40e.mp4覆盖网络

-   自定义网络

    -   网络模式 --net

        -   1：bridge  桥接模式（默认）
        -   2：none  - 不配置网络
        -   3：hosts - 使用主机网络
        -   4：container

    -   docker network

    -   ```shell
        
        Usage:  docker network COMMAND
        
        Manage networks
        
        Commands:
          connect     Connect a container to a network
          create      Create a network
          disconnect  Disconnect a container from a network
          inspect     Display detailed information on one or more networks
          ls          List networks
          prune       Remove all unused networks
          rm          Remove one or more networks
        
        Run 'docker network COMMAND --help' for more information on a command
        ```

    -   ```shell
        # 创建网络
        docker network create --driver bridge --subnet 192.168.0.0/16 --gateway 192.168.0.1 mynet
        
        # 查看网络
        docker network ls
        
        # 使用自定义网络
        docker run -d -P --name ngx01 --net mynet nginx
        
        docker run -d -P --name ngx02 --net mynet nginx
        
        
        # ping
        docker exec -it ngx01 ping ngx02-ip    # 可以联通
        docker exec -it ngx02 ping ngx02 	   # 也可以联通
        ```

    -   

-   网络联通

    -   ```shell
        docker network 
        connect     Connect a container to a network
        
        # 
        docker network connect mynet ngx03
        
        # 联通后查看
        
        docker network inspect mynet  （一个容器多个IP）
        
        
        
        ```

##### redis cluster 搭建

>   ```shell
>   # 1：创建 redis 网络
>   
>   docker network create redis --subset 172.168.0.1/16
>   
>   # 2: 通过脚本创建redis 的配置文件
>   
>   for port in $(seq 1 6); \
>   do \
>   mkdir -p /mydata/redis/node-${port}/conf
>   touch /mydata/redis/node-${port}/conf/redis.conf
>   cat << EOF >/mydata/redis/node-${port}/conf/redis.conf
>   port 6379
>   bind 0.0.0.0
>   cluster-enabled yes
>   cluster-config-file nodes.conf
>   cluster-node-timeout 5000
>   cluster-announce-ip 172.168.0.1${port}
>   cluster-announce-port 6379
>   cluster-announce-bus-port 16379
>   appendonly yes
>   EOF
>   done
>   
>   # 3：启动6个redis实例节点
>   
>   # 4：创建集群
>   redis-cli --cluster create ip1:port ip2:port ip3:port ip4:port ip5:port ip6:port --cluster-replicas 1
>   
>   
>   ```
>
>   

##### Docker Compose

>   定义、运行多个容器，批量容器编排
>
>   Yaml 文件（配置文件）
>
>   步骤
>
>   -   Dockerfile
>
>       -   定义项目的运行
>
>   -   docker-compose.yml
>
>       -   定义service、什么是service？
>
>           -   应用
>
>           -   ```shell
>               docker service ls
>               
>               
>               ```
>        
>           -   
>        
>       -   docker-compose.yml 文件编写
>
>   -   dokcer compose up
>
>       -   启动项目
>
>   
>
>   yaml规则
>
>   ```yaml
>   # 官方文档示例
>   
>   version: "3.9"
>   services:
>   
>     redis:	# 服务
>       image: redis:alpine
>       ports:
>         - "6379"
>       networks:
>         - frontend
>       deploy:		# 集群相关配置
>         replicas: 2
>         update_config:
>           parallelism: 2
>           delay: 10s
>         restart_policy:
>           condition: on-failure
>   
>     db:
>       image: postgres:9.4
>       volumes:
>         - db-data:/var/lib/postgresql/data
>       networks:
>         - backend
>       deploy:
>         placement:
>           max_replicas_per_node: 1
>           constraints:
>             - "node.role==manager"
>   
>     vote:
>       image: dockersamples/examplevotingapp_vote:before
>       ports:
>         - "5000:80"
>       networks:
>         - frontend
>       depends_on:
>         - redis
>       deploy:
>         replicas: 2
>         update_config:
>           parallelism: 2
>         restart_policy:
>           condition: on-failure
>   
>     result:
>       image: dockersamples/examplevotingapp_result:before
>       ports:
>         - "5001:80"
>       networks:
>         - backend
>       depends_on:
>         - db
>       deploy:
>         replicas: 1
>         update_config:
>           parallelism: 2
>           delay: 10s
>         restart_policy:
>           condition: on-failure
>   
>     worker:
>       image: dockersamples/examplevotingapp_worker
>       networks:
>         - frontend
>         - backend
>       deploy:
>         mode: replicated
>         replicas: 1
>         labels: [APP=VOTING]
>         restart_policy:
>           condition: on-failure
>           delay: 10s
>           max_attempts: 3
>           window: 120s
>         placement:
>           constraints:
>             - "node.role==manager"
>   
>     visualizer:
>       image: dockersamples/visualizer:stable
>       ports:
>         - "8080:8080"
>       stop_grace_period: 1m30s
>       volumes:
>         - "/var/run/docker.sock:/var/run/docker.sock"
>       deploy:
>         placement:
>           constraints:
>             - "node.role==manager"
>   
>   networks:
>     frontend:
>     backend:
>   
>   volumes:
>     db-data:
>   ```
>
>   **一键部署项目**
>
>   官方示例
>
>   ```yaml
>   
>   version: "3.9"
>       
>   services:
>     db:
>       image: mysql:5.7
>       volumes:
>         - db_data:/var/lib/mysql
>       restart: always
>       environment:
>         MYSQL_ROOT_PASSWORD: somewordpress
>         MYSQL_DATABASE: wordpress
>         MYSQL_USER: wordpress
>         MYSQL_PASSWORD: wordpress
>       
>     wordpress:
>       depends_on:
>         - db
>       image: wordpress:latest
>       volumes:
>         - wordpress_data:/var/www/html
>       ports:
>         - "8000:80"
>       restart: always
>       environment:
>         WORDPRESS_DB_HOST: db:3306
>         WORDPRESS_DB_USER: wordpress
>         WORDPRESS_DB_PASSWORD: wordpress
>         WORDPRESS_DB_NAME: wordpress
>   volumes:
>     db_data: {}
>     wordpress_data: {}
>   
>   
>   
>   ```
>
>   

##### Docker Swarm

Docker Swarm（集群管理和编排）

>   ![Swarm mode cluster](https://docs.docker.com/engine/swarm/images/swarm-diagram.png)
>
>   集群组成：
>
>   -   master 节点（Manager）
>
>   -   工作节点（Worker）
>
>       
>
>   1：集群搭建操作
>
>   ```shell
>   docker node ls
>   
>   docker swarm --help
>   Commands:
>    ca          Display and rotate the root CA
>    init        Initialize a swarm							# 初始化集群
>    join        Join a swarm as a node and/or manager		# 加入
>    join-token  Manage join tokens
>    leave       Leave the swarm
>    unlock      Unlock swarm
>    unlock-key  Manage the unlock key
>    update      Update the swarm
>   
>   
>   # 获取令牌
>   docker swarm join-token manager   # 管理节点
>   docker swarm join-tokne workder	  # 工作节点
>   
>   
>   ```
>
>   2：集群服务操作
>
>   ```shell
>   docker service 
>   
>   Commands:
>    create      Create a new service     # 同理docker run 启动一个服务
>    inspect     Display detailed information on one or more services
>    logs        Fetch the logs of a service or task
>    ls          List services
>    ps          List the tasks of one or more services
>    rm          Remove one or more services
>    rollback    Revert changes to a service's configuration
>    scale       Scale one or multiple replicated services
>    update      Update a service       				# 扩缩容
>   
>   ```
>
>   3：Task
>
>   ```shell
>   ```
>
>   
>
>   4：Docker Stack（集群部署）
>
>   ```shell
>   docker stack
>   Commands:
>    deploy      Deploy a new stack or update an existing stack
>    ls          List stacks
>    ps          List the tasks in the stack
>    rm          Remove one or more stacks
>    services    List the services in the stack
>   
>   
>   ```
>



##### Docker Stack

>   集群部署

##### docker secret

##### docker config



##### CI/CD

##### 源码



阿里云镜像加速器地址：

https://a8kp3oms.mirror.aliyuncs.com





