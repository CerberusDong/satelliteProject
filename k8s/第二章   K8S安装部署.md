# 第二章   K8S安装部署

## 2.1 部署K8S服务介绍

| 可选方案     | 介绍                                                         |
| ------------ | ------------------------------------------------------------ |
| kubeadm      | 官方推荐 Kubeadm 是一个提供了 `kubeadm init` 和 `kubeadm join` 的工具， 作为创建 Kubernetes 集群的 “快捷途径” 的最佳实践。 |
| kubespray    | 是基于kubeadm +Ansbile 批量集群化部署、如果集群规模大，想要快速体验k8s服务，还是推荐这种。 |
| The Hard Way | 利用二进制文件 一步步配置和部署一套高可用的 Kubernetes 集群，非常困难而繁琐，准备深入研究K8s的可以考虑，生产和运维不建议。 |

## 2.2  使用kubeadm 部署k8s 集群

- 系统镜像： ubuntu-16.04.7
- Docker 版本：18.09.7  
- 虚拟机：vmware 12 pro
- k8s版本：v1.14.6
- 虚拟机配置如下：

| 节点类型 | IP地址          | 配置                  | 主机名      |
| -------- | --------------- | --------------------- | ----------- |
| master   | 192.168.24.130  | 2G内存/2核CPU/60G磁盘 | k8sMaster-1 |
| node1    | 192.168.163.140 | 1G内存/2核CPU/60G磁盘 | k8sNode-1   |
| node2    | 192.168.164.141 | 1G内存/2核CPU/60G磁盘 | k8sNode-2   |
| harbor   | 192.168.164.142 | 1G内存/2核CPU/60G磁盘 | harbor-ztl  |

注意： 如果电脑没有那么大内存，master 可分为2G  node分为1G 。把乌班图设置为无图形界面。

**乌班图16.04 设置无图形界面**

```shell
# 关闭图形界面
> systemctl disable lightdm.service
> reboot # 重启生效

# 开启图形界面
ln -s  /lib/systemd/system/lightdm.service  /etc/systemd/system/display-manager.service
```



**配置集群免SSH密码登陆**

```shell
> apt-get install expect
> mkdir /root/script
> vim /etc/ssh/ssh_config    #　StrictHostKeyChecking no

# 将 auto_ssh 上传到master节点上
1 在ip.txt中输入各节点ip地址，一行一个ip
2 修改scp_to_cluster.sh和copy_id.sh的服务器用户名和密码
3 运行如下命令
# chmod 777 ./*
#./distribute_file.sh ../auto_ssh /root/script
```



**修改主机名和hosts**

```shell
> vim /etc/hostname # k8sMaster-1、k8sNode-1、k8sNode-2、harbor-ztl
> vim /etc/hosts # harbor 需配域名形式：harbor.cerberus.com
> scp /etc/hosts root@192.168.24.131:/etc/hosts  #从本地copy hosts到其它机器
```



**具体安装过程**

####  1. 关闭swap开启ipv4转发

```shell
# 关闭系统swap 和 开启内核ipv4转发开启
sudo swapoff -a
sudo sed -i '/swap/ s/^/#/' /etc/fstab
sudo vim /etc/sysctl.conf
net.ipv4.ip_forward = 1   #开启ipv4转发，允许内置路由
sudo sysctl -p  # 立即生效

```

#### 2、安装指定版本docker

```shell
sudo apt-cache madison docker.io  # 获取在现仓库docker版本缓存列表
apt-get install docker.io=18.09.7-0ubuntu1~16.04.7  # 从列表中选择对应版本
docker --version  # 获取Docker 版本
#验证docker状态
systemctl status docker  
# 配置镜像加速器，登陆阿里云个人账号，获取加速镜像地址
sudo mkdir -p /etc/docker
sudo tee /etc/docker/daemon.json <<-'EOF'
{"registry-mirrors": ["https://lq2tji6h.mirror.aliyuncs.com"]
}
EOF
# 重启Docker服务
sudo systemctl daemon-reload 
sudo systemctl restart docker
```



#### 3、安装K8S 所需工具包

您将在所有机器上安装这些软件包：

- `kubeadm`：引导集群的命令。
- `kubelet`：在集群中的所有机器上运行的组件，并执行诸如启动 pod 和容器之类的操作。
- `kubectl`：与您的集群对话的命令行工具。

```shell
# 安装依赖
sudo apt-get update && apt-get install -y apt-transport-https
sudo apt-get install curl

# 安装GPG证书
sudo curl https://mirrors.aliyun.com/kubernetes/apt/doc/apt-key.gpg | apt-key add -

# 使用国内kubernetes的源（阿里源）
sudo tee /etc/apt/sources.list.d/kubernetes.list <<-'EOF'
deb https://mirrors.aliyun.com/kubernetes/apt kubernetes-xenial main
EOF

# 更新软件源
sudo apt-get update

# 安装指定版本，版本一致防止出错
sudo apt-get install -y kubelet=1.14.6-00 kubeadm=1.14.6-00 kubectl=1.14.6-00

# 保持版本，取消自动更新 
sudo apt-mark hold kubelet=1.14.6-00 kubeadm=1.14.6-00 kubectl=1.14.6-00

# 设置开启自启
sudo systemctl enable kubelet && sudo systemctl start kubelet
```

#### 4.修改k8s与docker驱动一致

```shell
docker info # 查看驱动
sudo vim /etc/systemd/system/kubelet.service.d/10-kubeadm.conf  
# 在KUBELET_KUBECONFIG_ARGS 后面追加 --cgroup-driver=cgroupfs

//重启kubelet
sudo systemctl daemon-reload
sudo systemctl restart kubelet
```



> k8s注册节点提示Docker SystemdCheck: detected cgroupfs" as the Docker cgroup driver. The recommended dr fiver is" systemd 时在进行修改。我这里直接把k8s 的驱动修改成跟Docker 驱动一致。

如果提示上面的内容，可以把Docker 的驱动修改成跟K8s 驱动一致来解决问题

```SHELL
#  修改或者创建etc/docker/daemon.json，加入下面的内容
{
  "registry-mirrors": ["https://registry.aliyuncs.com"],  #镜像源要保持一致
  "exec-opts": ["native.cgroupdriver=systemd"],        #docker与k8s 中cgroupdriver保持一致
}
```



#### 5、查看k8s需要哪些镜像

```shell
# 通过制定config 命令来查看需要哪些镜像。
> kubeadm config images list --kubernetes-version=v1.14.6
k8s.gcr.io/kube-apiserver:v1.14.6
k8s.gcr.io/kube-controller-manager:v1.14.6
k8s.gcr.io/kube-scheduler:v1.14.6
k8s.gcr.io/kube-proxy:v1.14.6
k8s.gcr.io/pause:3.1
k8s.gcr.io/etcd:3.3.10
k8s.gcr.io/coredns:1.3.1
```

#### 6、初始化主节点

```shell
# 初始化k8s集群
kubeadm init \
   --image-repository registry.aliyuncs.com/google_containers \
   --kubernetes-version v1.14.6 \ 
   --apiserver-advertise-address=192.168.163.139 \  
   --service-cidr=10.1.0.0/16 \
   --pod-network-cidr=10.244.0.0/16 \
```

- image-repository ： 配置为国内镜像仓库，防止部署失败
- kubernetes-version ： 制定k8s 版本
- apiserver-advertise-address ： 制定api server  地址。一般为主机物理地址
- service-cidr： Service层的网络IP范围, 这个是虚拟IP不会体现在路由表上, 与前面的IP区分开就行。
- --pod-network-cidr  ：Pod层的网络IP范围, 需要与后面要配置的kube-flannel.yml里的设置一致

**注意：在root 用户下执行。**

```shell
kubeadm init --image-repository registry.aliyuncs.com/google_containers --kubernetes-version v1.14.6 --apiserver-advertise-address=192.168.24.130 --service-cidr=10.1.0.0/16 --pod-network-cidr=10.244.0.0/16
```

**成功之后在最后会有提示：**

```shell
kubeadm join 192.168.24.130:6443 --token msct39.omssf2xxlol6qbsb \
    --discovery-token-ca-cert-hash sha256:ef2b8306983f50bc53774eb9c1ad2e00fedfc17f16764d95b4f637c2d65f78b9
```

官方建议：要开始使用您的集群，您需要以普通用户身份运行以下命令，调整我们的配置文件：

我这里验证、最终还是在root 用户下成功执行：

```shell
su zhangtl # 切换成普通用户，如果不成功，就sudo -i 切换成root 用户
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```



#### 7、添加flannel网络插件

flannel 它实现了容器之间可以跨主机网络通信。

> Flannel是CoreOS团队针对Kubernetes设计的一个网络服务，它的功能是让集群中的不同节点创建的Docker容器都具有全集群唯一的虚拟IP地址，Flannel的设计目的就是为集群中的所有节点重新规划IP地址，从而使得不同节点上的容器能够获得“同属一个内网”且”不重复的”IP地址，并让属于不同节点上的容器能够直接通过内网IP通信，

```shell
wget https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
root@k8sMaster-1:~# kubectl apply -f kube-flannel.yml
podsecuritypolicy.policy/psp.flannel.unprivileged created
clusterrole.rbac.authorization.k8s.io/flannel created
clusterrolebinding.rbac.authorization.k8s.io/flannel created
serviceaccount/flannel created
configmap/kube-flannel-cfg created
daemonset.apps/kube-flannel-ds created
```

安转插件参考官方文档：https://kubernetes.io/docs/concepts/cluster-administration/addons/

**报错解决方式如下**：

unable to recognize "kube-flannel.yml": Get http://localhost:8080/api?timeout=32s: dial tcp 127.0.0.1:8080: connect: connection refused

原因之一：kubectl启动需要依赖 /etc/kubernetes/下的admin.conf文件，但只有master主节点经过初始化才能生成该文件。解决方法：将master节点的admin.conf复制到node节点对应目录下，

```shell
mkdir -p $HOME/.kube  # 分别在子节点执行
scp /etc/kubernetes/admin.conf  root@k8sNode-1:$HOME/.kube/config
scp /etc/kubernetes/admin.conf  root@k8sNode-2:$HOME/.kube/config

sudo chown $(id -u):$(id -g) $HOME/.kube/config # 分别在节点上执行
```





#### 8、验证master节点状态·

```shell
root@k8sMaster-1:~# kubectl get nodes
NAME          STATUS     ROLES    AGE   VERSION
k8smaster-1   NotReady   master   34m   v1.14.6

#  过几分钟 
root@k8sMaster-1:~# kubectl get nodes
NAME          STATUS   ROLES    AGE   VERSION
k8smaster-1   Ready    master   36m   v1.14.6
```

#### 9、验证kube-system

```shell
kubectl get pods -n kube-system -owide
```

![kubesystem](https://raw.githubusercontent.com/CerberusDong/pictures/main/202302142335257.png)

#### 10、coreDNS排错步骤

如果出现coreDNS 报错CrashLoopBackOff**

```shell
# step 1 查看pod 对应日志
> kubectl logs -n kube-system coredns-f7567f89b-2zjxl  #  后面接着pod名称
参数解释 
logs： 可以查看容器的运行logs日志
n： 为命名空间
coredns-f7567f89b-2zjxl ： 为容器名字

# 解决方案1：修改dns 配置文件。
> vim /etc/resolv.conf #  配置文件nameserver 8.8.8.8
root@k8sMaster-1:~# /etc/init.d/resolvconf restart 
systemctl daemon-reload
systemctl restart kubelet

# 验证coredns 状态是否修复
kubectl get pods -n kube-system -owide | grep coredns


# 解决方案2：直接修改容器资源文件 （这里我通过此方法解决的问题）
kubectl edit cm coredns -n kube-system # 注释loop后（:wq保存退出）,问题解决
systemctl daemon-reload
systemctl restart kubelet

# 验证coredns 状态是否修复
  kubectl get pods -n kube-system -owide | grep coredns
```

> 出错问题原因： 当 CoreDNS 日志包含 messageLoop ... detected ...时，这意味着loop检测插件在上游 DNS 服务器之一中检测到无限转发循环。这是一个致命错误，因为使用无限循环操作会消耗内存和 CPU，直到主机最终内存不足而死亡。



#### 11、添加节点

###### 打通ssh 和修改hosts

```shell
# 1、修改各个节点主机名
> vim /etc/hostname  
# 如果你是克隆虚拟机，虚拟机的主机名还是之前的， 要修改成现在节点名。

# 2、打通ssh 免秘钥
> vim /etc/ssh/ssh_config  #  StrictHostKeyChecking no（取消注释并改为no）

# 3、创建脚本执行目录，这里有自用集群打通ssh 自动化脚本工具
> mkdir /root/script # 创建脚本执行目录，把配置好的auto_ssh脚本文件夹上传到主节点 

# 4、每台节点安装expect
> apt install expect

# 5、最后在主节点上执行
> chmod 777 ./*
> ./distribute_file.sh ../auto_ssh /root/script

# 6、修改hosts 解析
192.168.163.139 k8sMaster-1
192.168.163.140 k8sNode-1
192.168.163.141 k8sNode-2
```



###### 添加节点到主节点

```shell
# 7、把主节点的网络插件yml 复制到各个节点
root@k8sMaster-1:~# scp kube-flannel.yml root@k8sNode-1:/root/
root@k8sMaster-1:~# scp kube-flannel.yml root@k8sNode-2:/root/

# 8、添加节点到k8s主节点，需要添加集群的都需要执行这个命令
kubeadm join 192.168.163.139:6443 --token tio97f.wdi7nwn4jmq4vbck \
    --discovery-token-ca-cert-hash sha256:14a2be879abb55c34fd85ca44053b36d5793d6522e88fb178f6a8cc5d4d0375d
```

![addNode](https://raw.githubusercontent.com/CerberusDong/pictures/main/202302142335269.png)

###### 子节点安装网络插件

```shell
> vi kube-flannel.yml# 修改其中net-conf.json的Network参数使其与kubeadm init时指定的 --pod-network-cidr一致, 因为文件里面的默认地址已经跟init时一致，不需要再改
> kubectl apply -f kube-flannel.yml
```



###### 子节点网络报错解决

原因之一：kubectl启动需要依赖 /etc/kubernetes/下的admin.conf文件，但只有master主节点经过初始化才能生成该文件。**解决方法：将master节点的admin.conf复制到node节点对应目录下**，

```shell
> mkdir -p $HOME/.kube  # 分别在子节点执行
> scp /etc/kubernetes/admin.conf  root@k8sNode-1:$HOME/.kube/config  # 在主节点上执行
> scp /etc/kubernetes/admin.conf  rootk8sNode-2:$HOME/.kube/config # 在主节点上执行
> sudo chown $(id -u):$(id -g) $HOME/.kube/config  # 分别在节点上执行
```



#### 12、集群验证

```shell
> kubectl get nodes # 获取集群节点信息
> kubectl get pods -n kube-system -owide # 获取集群基础核心容器信息
```



#### 13、问题解决和参考文档

提示WARNING: No swap limit support

```shell
docker info提示WARNING: No swap limit support


When users run Docker, they may see these messages when working with an image:
    WARNING: Your kernel does not support cgroup swap limit. WARNING: Your
    kernel does not support swap limit capabilities. Limitation discarded.
    To prevent these messages, enable memory and swap accounting on your system. To enable these on system using GNU GRUB (GNU GRand Unified Bootloader),


   步骤如下：
    Log into Ubuntu as a user with sudo privileges.
    Edit the /etc/default/grub file.
    Set the GRUB_CMDLINE_LINUX value as follows:
    GRUB_CMDLINE_LINUX="cgroup_enable=memory swapaccount=1"
    Save and close the file.
    Update GRUB.
    $ sudo update-grub
    Reboot your system.

参考文档：
https://blog.csdn.net/xiaoxiao133/article/details/87908161
https://www.cnblogs.com/hanshanxiaoheshang/p/10942550.html
```

**参考链接：**

```shell
https://www.cnblogs.com/flafly/articles/14324094.html
https://www.iteye.com/blog/sunbin-2512460
https://www.jianshu.com/p/36a1d06c2f6e
https://blog.csdn.net/longlong6682/article/details/107147405
https://www.cnblogs.com/lshan/p/14482436.html</br>
https://blog.51cto.com/liuzhengwei521/2427495
https://www.orchome.com/10431
https://www.wenjiangs.com/doc/n4ul6dt8j
```

