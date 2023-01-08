# 第七章   Docker 网络

Docker 容器和服务如此强大的原因之一是您可以将它们连接在一起，或者将它们连接到非 Docker 工作负载。Docker 容器和服务甚至不需要知道它们部署在 Docker 上，或者它们的对等点是否也是 Docker 工作负载。无论您的 Docker 主机运行 Linux、Windows 还是两者的混合，您都可以使用 Docker 以与平台无关的方式管理它们。

本主题定义了一些基本的 Docker 网络概念，并让您准备好设计和部署应用程序以充分利用这些功能

## 7.1  docker网络初探

```shell
#　docker安装时会创建一个名为 docker0 的Linux bridge，新建的容器会自动桥接到这个接口
root@ztloo:~# ifconfig | grep docker -A2
docker0   Link encap:以太网  硬件地址 02:42:a0:bc:85:b8  
          inet 地址:172.17.0.1  广播:172.17.255.255  掩码:255.255.0.0
          UP BROADCAST MULTICAST  MTU:1500  跃点数:1
          
root@ztloo:~# docker ps
CONTAINER ID    IMAGE         STATUS      　　　　PORTS                    NAMES
118fee577613   hello-flask      Up 30 seconds       0.0.0.0:8081->5000/tcp   flask_test1
966b5ccd8d1a   hello-flask      Up 3 seconds        0.0.0.0:8082->5000/tcp   flask_test2  

root@ztloo:~# docker inspect --format='{{.NetworkSettings.IPAddress}}' flask_test1
172.17.0.2 
root@ztloo:~# docker inspect --format='{{.NetworkSettings.IPAddress}}' flask_test2
172.17.0.3

```

![docker网络-1](C:\Users\Administrator\Desktop\Docker与K8s\img\docker网络-1.png)





## 7.2 查看docker内置网络驱动

```shell
root@ztloo:~# docker network ls
NETWORK ID          NAME                DRIVER              SCOPE
eb68f0a71dce        bridge              bridge              local
15f5f353615c        host                host                local
59d9aa1f4fc5        none                null                local

# bridge：默认网络驱动程序。如果您未指定驱动程序，则这是您正在创建的网络类型
# host：对于独立容器，去掉容器与 Docker 主机之间的网络隔离，直接使用主机的网络
# none：对于这个容器，禁用所有网络
```



  **网络驱动总结**

- 当您需要多个容器在同一个 Docker 主机上进行通信时，**用户自定义的桥接网络是最佳选择。**
- 当网络堆栈不应该与 Docker 主机隔离时，**主机网络是最好的，但您希望容器的其他方面被隔离。**
- 当您需要在不同 Docker 主机上运行的容器进行通信时，或者当多个应用程序使用 swarm 服务一起工作时，**覆盖网络是最佳选择。**
- 当您从 VM 设置迁移或需要容器看起来像网络上的物理主机时，**Macvlan 网络是最佳选择，每个主机都有唯一的 MAC 地址。**
- 第三方网络插件**允许您将 Docker 与专门的网络堆栈集成。



## 7.3 查看默认桥接网络

```shell
# 显示默认桥接网络详情。
root@ztloo:~# docker network inspect bridge


# 通过ping 来确认容器之间网络通信是否正常
------------------------------------------------
# 安装工具
# 安装ping 工具
root@118fee577613:/src# apt update 
root@118fee577613:/src# apt install iputils-ping
root@118fee577613:/src# apt install net-tools


 # 在flask_test1 容器上ping 自己和flask_test2容器
root@118fee577613:/src# ping 172.17.0.2 
root@118fee577613:/src# ping 172.17.0.3


 # 在flask_test2 容器上ping 自己和flask_test1容器
root@966b5ccd8d1a:/src# ping 172.17.0.2
root@966b5ccd8d1a:/src# ping 172.17.0.3


# 因为ip会变化，如果通过容器名能ping通就好了。下面我们自定义桥接网络来实现。
root@966b5ccd8d1a:/src# ping flask_test1
ping: flask_test1: Name or service not known
root@966b5ccd8d1a:/src# ping flask_test2
ping: flask_test2: Name or service not known

	
```

![桥接网络](C:\Users\Administrator\Desktop\Docker与K8s\img\桥接网络.png)



```shell
# 通过容器名字查看桥接网络的ip地址，flask_test1为容器名字
docker container inspect --format '{{.NetworkSettings.IPAddress}}' flask_test1
```



## 7.4  使用自定义桥接网络

升级flask 项目增加数据库功能，使用redis 容器提供功能服务。

```shell
# step1 :修改Dockerfie
FROM python:3.9.5-slim
COPY app.py /src/app.py
RUN pip install flask redis
WORKDIR /src
ENV FLASK_APP=app.py 
ENV FLASK_ENV=development
ENV REDIS_HOST=redis
EXPOSE 5000
CMD ["flask", "run", "-h", "0.0.0.0"]

# step2  修改app.py升级我们的python代码，加入redis服务
import os
import socket
from flask import Flask,render_template
from redis import Redis

app = Flask(__name__)
redis = Redis(host=os.environ.get('REDIS_HOST', '127.0.0.1'), port=6379)

@app.route('/')
def hello_world():
    return 'Hello, World!'

@app.route('/info') #页面链接该路由名称
def f_info():
    return render_template('example.html')


@app.route('/redis')
def hello():
    redis.incr('hits')
    return f"Hello Container World! I have been seen {redis.get('hits').decode('utf-8')} times and my hostname is {socket.gethostname()}.\n"


# step3: 准备redis镜像，和build 新flask 服务
root@ztloo:~/flaskPro# docker pull redis

root@ztloo:~/flaskPro# ls
app.py  Dockerfile  Dockerfile-cmd  Dockerfile-redis  templates
root@ztloo:~/flaskPro# docker build -f Dockerfile-redis -t flask-redis .

# step4：创建一个自定义的docker bridge
root@ztloo:~/flaskPro# docker network create -d bridge fr-network
569d2965ddd5db58f2a1c51f8a7e64fc5a262da92e5a62eb0e406a78e64f6017

# step5:创建redis 容器和flask 容器
root@ztloo:~/flaskPro# docker run -d --name redis-server --network fr-network redis
b7982d98ddac9c85f946b75b9a7bb5c1fcbbca4ec1602526ff4d5f768d6fe69e

root@ztloo:~/flaskPro# docker run -d --network fr-network --name fr --env REDIS_HOST=redis-server -p 5000:5000 flask-redis
31ed7144825084d7919dd6a5d42d7a70d9ab5e16606fbbfa2fb94c7e0a2a6c2c

# step6:服务验证
root@ztloo:~/flaskPro# curl http://localhost:5000/redis
Hello Container World! I have been seen 1 times and my hostname is 31ed71448250.
root@ztloo:~/flaskPro# curl http://localhost:5000/redis
Hello Container World! I have been seen 2 times and my hostname is 31ed71448250.
root@ztloo:~/flaskPro# curl http://localhost:5000/redis
Hello Container World! I have been seen 3 times and my hostname is 31ed71448250.
root@ztloo:~/flaskPro# curl http://localhost:5000/redis

# 显示网络信息
> docker network inspect fr-network 

```

在用户定义的网络上`fr-network `，容器不仅可以通过 IP 地址进行通信，还可以将容器名称解析为 IP 地址。这种能力称为**自动服务发现**。让我们连接`fr-network`并测试一下。



在前面例子中容器与容器之间的通信都是通过网络中的IP地址来完成的，这种方式显然是不合理的，因为这个IP地址可能会在启动容器时发生变化，而且也比较难记。

那么解决这一问题的方法就是使用网络别名，容器在网络是是允许有别名的，且这个别名在所在网络中都可以直接访问，这就类似局域网在各物理机的主机名。

```shell
# 网络别名fr-alias，防止ip 变化
> docker  run -d --network fr-network --network-alias fr-alias --name fr --env REDIS_HOST=redis-server -p 5000:5000 flask-redis
```



更具体的网络教程请参考官方：https://docs.docker.com/network/network-tutorial-standalone/