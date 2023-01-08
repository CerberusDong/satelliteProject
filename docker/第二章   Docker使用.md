# 第二章   Docker使用

## 3.1  Docker CLI 命令简单使用 

```shell
# 1、获取Dokcer相关信息
> docker -v  # 只是简要版本
> docker version  #详细版本信息
> docker info # 显示系统信息，包括硬件状态、容器信息等。
> docker  # 直接回车，显示帮助信息

# 2、拉取（下载）nginx镜像
> docker pull nginx  # 下载nginx镜像
# pull 默认从官方hub 仓库中拉取：https://hub.docker.com/_/nginx

# 3、载入容器
root@ztloo:~# docker run --name nginx-test -p 8081:80 -d nginx  
d06c2ab7da8d35dff78fddcf6abf23cacd86d7d5122597eea39f794cc43e6953

参数说明：
--name nginx-test：容器名称。
-p 8081:80： 端口进行映射，将本地 8081 端口映射到容器内部的 80 端口。
-d ： 设置容器在在后台一直运行。

run命令执行流程：

1、 在本地查找nginx镜像，如有有基于它创建容器、
   分配虚拟IP、并把虚拟IP的80的端口映射到主机8081端口上、启动容器运行指定的命令

2、在本地查找nginx镜像，如果没有，去远程仓库中获取（默认的registry是Docker Hub）
   下载最新版本的nginx镜像 （nginx:latest 默认) 
   分配虚拟IP、并把虚拟IP的80的端口映射到主机8081端口上、启动容器运行指定的命令

# 4、验证服务
root@ztloo:~# curl http://127.0.0.1:8081
<!DOCTYPE html>
<title>Welcome to nginx!</title>


# 5、查看容器和镜像信息
> docker ps | grep nginx   # 查看已运行容器，可结合grep 进行过滤
> docker ps -a   # 查看所有包括运行和停止的的容器信息
> docker inspect --format ＂{{.State.Running}}＂ nginx-test  # 检查容器是否运行
> docker images | grep nginx # 查看镜像信息

# 6、给镜像重新命名和换标签
root@ztloo:~# docker pull nginx:stable-alpine
# 验证
root@ztloo:~# docker images
REPOSITORY          TAG                 IMAGE ID            CREATED             SIZE
nginx               latest              605c77e624dd        3 months ago        141MB
nginx               stable-alpine       373f8d4d4c60        5 months ago        23.2MB
# 重命名
root@ztloo:~# docker tag nginx:stable-alpine my-nginx:v1
root@ztloo:~# docker images
REPOSITORY          TAG                 IMAGE ID            CREATED             SIZE
nginx               latest              605c77e624dd        3 months ago        141MB
my-nginx            v1                  373f8d4d4c60        5 months ago        23.2MB
nginx               stable-alpine       373f8d4d4c60        5 months ago        23.2MB
```

## 2.2  容器的启停和资源的清理

```shell
# 预备容器服务的启停和删除的演示资源
root@ztloo:~# docker pull tomcat:jdk8-corretto
root@ztloo:~# docker images
REPOSITORY          TAG                 IMAGE ID            CREATED             SIZE
nginx               latest              605c77e624dd        3 months ago        141MB
tomcat              jdk8-corretto       2414658c61d1        4 months ago        379MB
my-nginx            v1                  373f8d4d4c60        5 months ago        23.2MB
nginx               stable-alpine       373f8d4d4c60        5 months ago        23.2MB
root@ztloo:~# docker run --name tomcat8 -p 8088:8080 -d  tomcat:jdk8-corretto


# 1、容器服务的启停和删除
> docker stop nginx-test  # 通过名字停止容器
> docker start nginx-test  # 通过名字启动容器
> docker restart nginx-test   # 通过名字或容器id 、重新启动
> docker rm nginx-test   #  删除容器，删除之前一定要执行docker stop nginx-test 确保容器关闭状态 
> docker rmi -f hello：tag  # 删除名字为hello 的镜像
> docker rmi -f 镜像id # 删除镜像id相同的镜像  
 
 
# 2.1 预备资源的批量处理的演示资源
root@ztloo:~# docker pull hello-world
root@ztloo:~# docker run --name nginx-1 -p 8081:80 -d nginx 
root@ztloo:~# docker run --name nginx-2 -p 8082:80 -d nginx 
root@ztloo:~# docker run --name nginx-3 -p 8083:80 -d nginx 

# 2.2 资源的批量处理
> docker stop $(docker ps -qa) # 停止所有容器
> docker start $(docker ps -qa) # 开启所有容器
> docker start $(docker ps -qf status=exited) # 启动未运行容器，需先停止nginx-1、nginx-2看演示效果
> docker container prune  # 删除所有停止运行的容器，需先停止nginx-1、nginx-2看演示效果
> docker image prune # 删除 所有未被 tag 标记和未被容器使用的镜像
> docker image prune -a #删除 所有未被容器使用的镜像，hello 镜像没有被使用。
> docker system prune   #清理磁盘，删除关闭的容器、无用的数据卷和网络，以及无tag的镜像
> docker system prune -a  # 没有被使用过的docker虚拟对象，都会收到无情的清理，清理的更加彻底，慎用



# 3.1  预备-开启nginx镜像相关的所有容器-演示资源
root@ztloo:~# docker pull hello-world
root@ztloo:~# docker run --name nginx-1 -p 8081:80 -d nginx 
root@ztloo:~# docker run --name nginx-2 -p 8082:80 -d nginx 
root@ztloo:~# docker stop nginx-1
root@ztloo:~# docker stop nginx-2

# 3.2 开启nginx镜像相关的所有容器
> docker start $(docker ps -a | grep nginx |awk '{print $1}') 

# 4.1 预备-删除所有未运行的容器-演示资源
root@ztloo:~# docker run --name nginx-4 -p 8084:80 -d nginx 
root@ztloo:~# docker stop nginx-4
# 4.2 删除所有未运行的容器 
root@ztloo:~#docker rm -fv $(docker ps -qf status=exited) 
# 参数说明：f是强制删除，v 删除相关联的卷

-------------------------------------------------------------------------------------

# 5、案例：docker删除某个镜像创建的所有容器，和最终删除nginx镜像。
-------
资源准备
-------
root@ztloo:~# docker run --name tomcat8 -p 8088:8080 -d  tomcat:jdk8-corretto


# step1:# 停止nginx 镜像创建的所有容器
> docker stop $(docker ps -a | grep nginx |awk '{print $1}') 

# step2: 删除nginx 镜像创建的所有容器
# 加-f可以直接删除正在运行的容器，容器无需关闭。
> docker rm -fv $(docker ps -a | grep nginx |awk '{print $1}') 

# step3:删除镜像名字包含nginx 的
# 加-f 可以直接删除镜像，即使有引用的容器，容器需关闭状态下。
> docker rmi -f $(docker images | grep "nginx" | awk '{print $3}') 

# 注意 普通删除容器、和镜像资源的时候，要保证是相关容器关闭状态。
```

## 2.3  运行容器的交互处理

```shell
-------
资源准备
-------
root@ztloo:~# docker pull nginx 
root@ztloo:~# docker run --name nginx-test -p 8081:80 -d nginx  

# 查看实时日志
> docker logs -f nginx-test # 查看容器名字为 nginx-test的实时日志

# 进入容器交互模式
> docker exec -it nginx-test bash  # 注意有的容器是bash ,有的容器需要/bin/bash
  i:即使没有附加也保持STDIN（标准输入流） 打开
  t:分配一个伪终端

# 从容器里面拷贝文件到宿主机
# docker cp 容器名：拷贝的文件在容器里面的路径  要拷贝到宿主机的相应路径
# 案例1： 把nginx-test 容器中的nginx 配置文件copy 到 宿主机的/opt/ 下面 
root@ztloo:/opt# docker cp nginx-test:/etc/nginx/nginx.conf /opt/
root@ztloo:/opt# docker cp nginx-test:/etc/nginx/nginx.conf . # 先切换到对应宿主机目录。

# 案例2： 把nginx-test 容器中的nginx 配置文件所在目录  整个复制到 物理主机的/opt/ 下面
root@ztloo:/opt# docker cp nginx-test:/etc/nginx/ /opt/  
root@ztloo:/opt# ls

------------------------------------------------------------------------------------------------
# 从宿主机中拷贝文件到容器
# docker cp 要拷贝的文件或路径 容器名：要拷贝到容器里面对应的路径
# 案例1： 把物理主机的/opt/ 下面的hello 文件copy 到nginx-test 容器/etc/nginx/目录下面
root@ztloo:/opt# vim hello
root@ztloo:/opt# ls
containerd  hello  nginx
root@ztloo:/opt# cat hello 
dfsdfsdfsf

sdfsdfsdfsdfs
root@ztloo:/opt# docker cp ./hello nginx-test:/etc/nginx/

# 案例2： 把物理主机的/opt/ 下面的ztl目录整个copy 到nginx-test 容器/etc/nginx/目录下面
root@ztloo:/opt# mkdir ztl
root@ztloo:/opt# cd ztl/
root@ztloo:/opt/ztl# ls
hello-word-11  word
root@ztloo:/opt/ztl# cd ..
root@ztloo:/opt# docker cp ./ztl/ nginx-test:/etc/nginx/
```

