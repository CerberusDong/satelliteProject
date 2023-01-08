# 第一章   Docker概述与安装

Docker包括一个命令行程序、一个后台守护进程，以及一组远程服务。它是运行在操作系统之上的一个软件。

解决了常见的软件问题并简化了安装、运行、发布和删除软件。这一切能够实现是通过使用一项UNIX技术，称为容器。

## 1.1 容器概念

容器是您机器上的**沙盒进程**，与主机上的所有其他进程隔离。这种隔离利用了[内核命名空间和 cgroups](https://medium.com/@saschagrunert/demystifying-containers-part-i-kernel-space-2c53d6979504)，这些特性在 Linux 中已经存在了很长时间。 你可以理解容器是一种快速打包技术。

**内核命名空间（Name Space）**: 用于资源的隔离，比如多个容器之间的隔离。

 **c groups**：负责资源的管理和控制。比如限制某个容器使用多少CPU、内存等。

> 容器是与系统其他部分隔离开的一个或一组进程。运行这些进程所需的所有文件都由另一个镜像提供，这意味着从开发到测试再到生产的整个过程中，Linux 容器都具有可移植性和一致性。
>
> 假设您在开发一个应用。您使用的是一台笔记本电脑，而且您的开发环境具有特定的配置。其他开发人员身处的环境配置可能稍有不同。您正在开发的应用不止依赖于您当前的配置，还需要某些特定的库、依赖项和文件。
>
> 与此同时，您的企业还拥有标准化的开发和生产环境，有着自己的配置和一系列支持文件。您希望尽可能多在本地模拟这些环境，而不产生重新创建服务器环境的开销。因此，您要如何确保应用能够在这些环境中运行和通过质量检测，并且在部署过程中不出现令人头疼的问题，也无需重新编写代码和进行故障修复？答案就是使用容器。

##  1.2 为什么使用容器技术

**容器技术出现之前：**

![image-20230108211134271](https://raw.githubusercontent.com/CerberusDong/pictures/main/202301082111439.png)

**容器技术出现之后：**

![image-20230108211154318](https://raw.githubusercontent.com/CerberusDong/pictures/main/202301082111429.png)

## 1.3 Docker 三要素

- [x] 容器
- [x] 镜像
- [x] 仓库

在某一个服务器或者笔记本上的win10系统上，使用docker 上创建使用了nginx的tomcat项目的容器。并把他们打包成镜像上传到镜像仓库中。 这个时候，只要能链接到镜像仓库带有Docker引擎的机器，都能进行安装和部署。不论你是windows、centos 还是乌班图等系统，都能一键部署，不需要更改任何配置文件，不需要安装某些特殊依赖。

![image-20230108211214996](https://raw.githubusercontent.com/CerberusDong/pictures/main/202301082112092.png)

## 1.4  容器镜像

运行容器时，它使用隔离的文件系统。此自定义文件系统由**镜像**提供。

镜像包含容器的文件系统，它必须包含运行应用程序所需的一切——所有依赖项、配置、脚本、二进制文件等。 配置：例如环境变量、运行的默认命令、和其他元数据。

## 1.5 Docker安装与配置

```shell
sudo apt-cache madison docker.io
apt-get install docker.io=18.09.7-0ubuntu1~16.04.7  # 从列表中选择对应版本
docker --version  # 获取Docker 版本

systemctl status docker  #验证docker状态


# 配置镜像加速器，登陆阿里云个人账号，获取加速镜像地址
sudo mkdir -p /etc/docker
sudo tee /etc/docker/daemon.json <<-'EOF'
 {
   "registry-mirrors": ["https://877qt60u.mirror.aliyuncs.com"]
 }
EOF

# 重新加载配置文件和重启docker
sudo systemctl daemon-reload 
sudo systemctl restart docker


# 出现apt 占用
# E: Could not get lock /var/lib/dpkg/lock - open (11: Resource temporarily unavailable
root@ztloo:~# ps -e | grep apt
1488 ?        00:00:00 apt.systemd.dai
1522 ?        00:00:00 apt.systemd.dai
root@ztloo:~# sudo kill -9 1488
root@ztloo:~# sudo kill -9 1522

```

