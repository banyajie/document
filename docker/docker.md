#### docker 学习

-   docker 安装

    >   

-   docker命令

    -   镜像命令

        ``` shell
        docker images
        
        docker image COMMEND
        docker image build -f dockerfile xx xx
        docker image history 
        docker image import 		    # docker image import [OPTIONS] file|URL|- [REPOSITORY[:TAG]]
        docker image inspect xxxx
        docker image load
        docker image ls
        docker image pull
        docker image push
        docker image rm
        docker image save
        docker image tag
        docker image prune 			# Remove unused images
        ```

        

    -   容器命令

        ```shell
        docker run
        docker ps
        docker info 
        docker inspect
        docker logs
        docker rm
        docker rmi
        docker start
        docker stop
        
        ```

        

    -   操作命令

        ```shell
        docker exec
        docker attach
        
        ```

        

-   docker镜像原理

    -   联合挂载系统

        ```yaml
        
        ```

        

-   容器数据卷

    -   匿名挂载
    -   声明挂载
    -   数据卷容器
    -   

-   DockerFile

    -   常用命令

        ```dockerfile
        MAINTAINER <name:email>
        FROM ImageName
        RUN <commend>		# RUN /bin/bash -c 'source $HOME/.bashrc; echo $HOME'   # RUN ["/bin/bash", "-c", "echo hello"]
        CMD	<commend>		# CMD ["/usr/bin/wc","--help"]
        LABEL <key>=<value> <key>=<value> <key>=<value> ...
        EXPOSE 90/udp 
        EXPOSE 80/tcp
        ENV <key>=<value> ... 	# ENV MY_NAME="John Doe" 	#ENV MY_DOG=Rex\ The\ Dog
        ADD [--chown=<user>:<group>] <src>... <dest>
        COPY [--chown=<user>:<group>] <src>... <dest>\
        ENTRYPOINT ["executable", "param1", "param2"] # 
        VOLUME ["/data"]
        USER <user>[:<group>]
        WORKDIR /path/to/workdir
        ```

        

-   Docker网络原理

    -   Veth peer

    -   网桥

    -   --link

    -   覆盖网络

    -   自定义网络

        -   网络模式 --net
            -   1：bridge  桥接模式（默认）
            -   2：none 
            -   3：hosts
            -   4：container
        -   docker network

    -   网络联通

        -   ```shell
            docker network connect
            
            ```

        -   

-   Docker Compose

    >   定义和运行docker多容器的工具

    >   ```yml
    >   version: "3.9"  # optional since v1.27.0
    >   services:
    >     web:
    >       build: .
    >       ports:
    >         - "5000:5000"
    >       volumes:
    >         - .:/code
    >         - logvolume01:/var/log
    >       links:
    >         - redis
    >     redis:
    >       image: redis
    >   volumes:
    >     logvolume01: {}
    >     
    >   ```
    >
    >   

-   Docker Swarm（集群管理和编排）

    >   1：集群搭建操作
    >
    >   ```shell
    >   docker node ls
    >   
    >   docker swarm --help
    >   Commands:
    >     ca          Display and rotate the root CA
    >     init        Initialize a swarm
    >     join        Join a swarm as a node and/or manager
    >     join-token  Manage join tokens
    >     leave       Leave the swarm
    >     unlock      Unlock swarm
    >     unlock-key  Manage the unlock key
    >     update      Update the swarm
    >   
    >   
    >   ```
    >
    >   2：集群服务操作
    >
    >   ```shell
    >   docker service
    >   Commands:
    >     create      Create a new service
    >     inspect     Display detailed information on one or more services
    >     logs        Fetch the logs of a service or task
    >     ls          List services
    >     ps          List the tasks of one or more services
    >     rm          Remove one or more services
    >     rollback    Revert changes to a service's configuration
    >     scale       Scale one or multiple replicated services
    >     update      Update a service
    >   
    >   ```
    >
    >   3：Task
    >
    >   4：Docker Stack（集群部署）
    >
    >   ```shell
    >   docker stack
    >   Commands:
    >     deploy      Deploy a new stack or update an existing stack
    >     ls          List stacks
    >     ps          List the tasks in the stack
    >     rm          Remove one or more stacks
    >     services    List the services in the stack
    >   
    >   
    >   ```
    >
    >   

-   CI/CD 

-   可视化

    -   Portainer
    -   Rancher

-   







阿里云镜像加速器地址：

https://a8kp3oms.mirror.aliyuncs.com




