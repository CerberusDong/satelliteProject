# 第三章   安装helm与镜像仓库

Helm 是一种简化安装和管理 Kubernetes 应用程序的工具。把它想象成 Kubernetes 的 apt/yum/homebrew。

- Helm 呈现您的模板并与 Kubernetes API 通信

- Helm 在您的笔记本电脑、CI/CD 或您希望它运行的任何地方运行。

- Charts 是 Helm 包，至少包含两件事：

  - 包装说明 ( `Chart.yaml`)
  - 一个或多个模板，其中包含 Kubernetes 清单文件

- Charts包可以存储在磁盘上，或从远程图表存储库（如 Debian 或 RedHat 软件包）中获取。

### 3.1 版本对应 

版本对应表

| helm 版本 | k8s 版本        |
| --------- | --------------- |
| 2.15.x    | 1.15.x - 1.14.x |
| 2.14.x    | 1.14.x - 1.13.x |

### 3.2  安装helm二进制包

**在主节点上安装**

```shell
# 下载helm 包并发到程序可执行目录
wget https://get.helm.sh/helm-v2.14.3-linux-amd64.tar.gz
tar -zxvf helm-v2.14.3-linux-amd64.tar.gz
cd linux-amd64/
mv helm /usr/local/bin/
chmod +x /usr/local/bin/helm

echo 'source <(helm completion bash)' >> /etc/profile   # 配置命令自动补全
. /etc/profile
```

### 3.3 安装Tiller server

```shell
# 编辑和新建tiller-rbac.yaml 文件
> root@k8sMaster-1:~# cat tiller-rbac.yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: tiller
  namespace: kube-system
---
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRoleBinding
metadata:
  name: tiller
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
  - kind: ServiceAccount
    name: tiller
    namespace: kube-system
    
# 创建tiller 对象 
root@k8sMaster-1:~# kubectl apply -f tiller-rbac.yaml
serviceaccount/tiller created
clusterrolebinding.rbac.authorization.k8s.io/tiller created

# Tiller server的环境初始化
> helm init --service-account tiller  --tiller-image registry.cn-hangzhou.aliyuncs.com/google_containers/tiller:v2.14.3  --stable-repo-url https://kubernetes.oss-cn-hangzhou.aliyuncs.com/charts --skip-refresh  

# 验证 Tiller服pod是否异常
root@k8sMaster-1:~# kubectl get pod -n kube-system | grep tiller
tiller-deploy-7f4d76c4b6-h86kd        0/1     ImagePullBackOff   0          20d

# 如果出错可以通过容器名称查看详细异常信息
> kubectl describe pod tiller-deploy-7f4d76c4b6-h86kd  -n kube-system
# 异常解决方法，删除tiller deployment，顺带会一起删除tiller pod
root@k8sMaster-1:~# kubectl delete -n kube-system deployment tiller-deploy
deployment.extensions "tiller-deploy" deleted

# Tiller server的环境重新初始化
> helm init --upgrade --service-account tiller --tiller-image registry.cn-hangzhou.aliyuncs.com/google_containers/tiller:v2.14.3  --stable-repo-url https://kubernetes.oss-cn-hangzhou.aliyuncs.com/charts --skip-refresh  
```

### 3.4  配置helm仓库

```shell
# 列出仓库
root@k8sMaster-1:~# helm repo list
NAME      URL                                             
stable    https://kubernetes-charts.storage.googleapis.com
local     http://127.0.0.1:8879/charts

# 如果初始化没有指定--stable-repo-url，默认是Google，在国外，速度特别慢，所以需要更换为国内源，
root@k8sMaster-1:~#  helm repo add stable https://kubernetes.oss-cn-hangzhou.aliyuncs.com/charts
"stable" has been added to your repositories
root@k8sMaster-1:~#  helm repo list
NAME      URL                                                   
stable    https://kubernetes.oss-cn-hangzhou.aliyuncs.com/charts
local     http://127.0.0.1:8879/charts

# 更新仓库和查看版本
root@k8sMaster-1:~# helm repo update
Hang tight while we grab the latest from your chart repositories...
...Skip local chart repository
...Successfully got an update from the "stable" chart repository
Update Complete.
root@k8sMaster-1:~# helm version
Client: &version.Version{SemVer:"v2.14.3", GitCommit:"0e7f3b6637f7af8fcfddb3d2941fcc7cbebb0085", GitTreeState:"clean"}
Server: &version.Version{SemVer:"v2.14.3", GitCommit:"0e7f3b6637f7af8fcfddb3d2941fcc7cbebb0085", GitTreeState:"clean"}

# 搜索MySQL
root@k8sMaster-1:~# helm search mysql
```

### 3.5  搭建Harbor 仓库

harbor是VMWare公司提供的一个docker私有仓库构建程序，功能非常强大.

```shell
# 安装compose 编排服务，因为habor仓库的安装需要compose 编排服务
apt install  docker-compose=1.8.0-2~16.04.1

# 下载harbor 1.7.1 版本的二进制包
wget https://storage.googleapis.com/harbor-releases/release-1.7.0/harbor-online-installer-v1.7.1.tgz
# 解压缩
tar -zxvf harbor-online-installer-v1.7.1.tgz


# 在主节点和各个子节点配置，镜像仓库地址，insecure-registries把自己的ip 和harbor 的地址添加进去
# 可以用主机名代替，防止IP变化, 还有k8sNode-1、k8sNode-2
sudo tee /etc/docker/daemon.json <<-'EOF'
{
  "registry-mirrors": ["https://877qt60u.mirror.aliyuncs.com"],
  "insecure-registries":["k8sNode-2","harbor.cerberus.com"]
}
EOF

#重新加载配置
sudo systemctl daemon-reload
sudo systemctl restart docker

# 切换到对应目录修改配置，例如密码等。
cd harbor
vim harbor.cfg  # 可以修改harbor 的配置,主机名：harbor.ztloo.com
./install.sh  # 开始安装，不支持helm
./install.sh   --with-clair --with-chartmuseum # 支持helm

# 账号：admin 密码默认是Harbor12345
# 如果harbor仓库使用账户和密码登陆不上，报502，重新执行执行部署脚

# 如果镜像拉取失败，解决方案
ERROR: Get https://registry-1.docker.io/v2/: dial tcp: lookup registry-1.docker.io on 127.0.1.1:53: read udp 127.0.0.1:48920->127.0.1.1:53: i/o timeout

1.运行命令，修改文件：
vim /etc/docker/daemon.json
文件中加入：
{ "registry-mirrors":["https://docker.mirrors.ustc.edu.cn"] }
vi /etc/resolv.conf
# 增加或修改为下面一行地方内容
nameserver 8.8.8.8
#重启docker服务
sudo systemctl daemon-reload

# 超时,因为组件特别多，性能差所以调高时间
vim /etc/profile
export DOCKER_CLIENT_TIMEOUT=240
export COMPOSE_HTTP_TIMEOUT=240
source /etc/profile

```

**插播问题**

```shell
 # 部署好的harbor 服务异常
 docker start $(docker ps -qf status=exited) # 启动未运行容器
 
 # k8s异常 
 root@k8sMaster-1:~# kubectl get node
The connection to the server 192.168.163.151:6443 was refused - did you specify the right host or port?

#　原因：因为docker的/etc/docker/daemon.json 格式有问题

# 检查docker状态
 systemctl status docker

# 修改etc/docker/daemon.json

# 重新启动各个节点docker
sudo systemctl daemon-reload 
sudo systemctl restart docker
```



### 3. 6  验证 harbor 功能

```shell
# 验证镜像的push 和pull 
－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－－
# 登陆harbor仓库
root@k8sMaster-1:~# docker login harbor.cerberus.com
Username: admin
Password: Harbor12345

#拉取镜像
root@k8sMaster-1:~# docker pull hello-world
Using default tag: latest
latest: Pulling from library/hello-world
2db29710123e: Pull complete
Digest: sha256:2498fce14358aa50ead0cc6c19990fc6ff866ce72aeb5546e1d59caac3d0d60f
Status: Downloaded newer image for hello-world:latest

#给镜像打tag（镜像的格式为，镜像仓库IP：端口/镜像名称）
root@k8sMaster-1:~# docker tag hello-world harbor.cerberus.com/mytest/hello-world:v2


#PUSH到仓库
root@k8sMaster-1:~# docker push harbor.cerberus.com/mytest/hello-world:v2
The push refers to repository [harbor.ztloo.com/mytest/hello-world]
e07ee1baac5f: Layer already exists 
v2: digest: sha256:f54a58bc1aac5ea1a25d796ae155dc228b3f0e11d046ae276b39c4bf2f13d8c4 size: 525


# 在nod1 从harbor 仓库中拉取镜像
root@k8sNode-1:~# docker pull harbor.cerberus.com/mytest/hello-world:v2
v2: Pulling from mytest/hello-world
2db29710123e: Pull complete 
Digest: sha256:f54a58bc1aac5ea1a25d796ae155dc228b3f0e11d046ae276b39c4bf2f13d8c4
Status: Downloaded newer image for harbor.ztloo.com/mytest/hello-world:v
```



### 3.7  配置k8s与harbor通信

需要把对应docker 物理节点登陆过harbor仓库的依据，配置到k8s 中，作为token密钥。k8s通过这个token密钥跟harbor仓库交互。

```shell
# 1、保证在节点中的daemon.jon文件中加入harbor仓库的IP地址或域名，上面已配置。
# 2 、在node节点登陆过harbor仓库后需要查看，harbor的登录凭据。一定是登陆过的harbor仓库的。
> cat .docker/config.json |base64 -w 0

# 3、 在master节点上创建secret镜像拉取资源服务
> vim registry-pull-secret.yaml # dockerconfigjson 对应的秘钥是由cat获取出来的结果
apiVersion: v1
kind: Secret
metadata:
  name: registry-pull-secret
data:
  .dockerconfigjson: ewoJImF1dGhzIjogewoJCSJoYXJib3IuY2VyYmVydXMuY29tIjogewoJCQkiYXV0aCI6ICJZV1J0YVc0NlNHRnlZbTl5TVRJek5EVT0iCgkJfQoJfSwKCSJIdHRwSGVhZGVycyI6IHsKCQkiVXNlci1BZ2VudCI6ICJEb2NrZXItQ2xpZW50LzE4LjA5LjcgKGxpbnV4KSIKCX0KfQ==
type: kubernetes.io/dockerconfigjson



# 4、创建秘钥对象资源服务
#创建资源 成功
root@k8sMaster-1:~# kubectl create -f registry-pull-secret.yaml
secret/registry-pull-secret created

# 报错，解决
root@k8sMaster-1:~# kubectl create -f registry-pull-secret.yaml
Error from server (AlreadyExists): error when creating "registry-pull-secret.yaml": secrets "registry-pull-secret" already exists

# 删除对应资源服务
root@k8sMaster-1:~# kubectl delete -f registry-pull-secret.yaml
secret "registry-pull-secret" delete

#重新创建资源 
root@k8sMaster-1:~# kubectl create -f registry-pull-secret.yaml
secret/registry-pull-secret created

# 5、查看生成的资源
root@k8sMaster-1:~# kubectl get secret
NAME                   TYPE                                  DATA   AGE
default-token-kt7j9    kubernetes.io/service-account-token   3      11d
registry-pull-secret   kubernetes.io/dockerconfigjson        1      38s

# 参考文档
参考：https://blog.51cto.com/u_14227204/2534349
https://blog.csdn.net/qq_35886593/article/details/105327208
安装离线安装包
https://www.cnblogs.com/sunyllove/p/9888955.html
https://blog.csdn.net/qq_40378034/article/details/90752212
https://www.cnblogs.com/majiang/p/11218792.html
https://goharbor.io/docs/2.4.0/install-config/installation-prereqs/
https://blog.csdn.net/QwQNightmare/article/details/106084642
```



### 3.8 helm 配置Harbor 支持

```shell
# 安装push 插件
root@k8sMaster-1:~# helm plugin install https://github.com/chartmuseum/helm-push
Downloading and installing helm-push v0.10.2 ...
https://github.com/chartmuseum/helm-push/releases/download/v0.10.2/helm-push_0.10.2_linux_amd64.tar.gz
Installed plugin: cm-push

# 查看安装的的插件
root@k8sMaster-1:~# helm plugin list
NAME   	VERSION	DESCRIPTION                      
cm-push	0.10.2 	Push chart package to ChartMuseum

# 添加私有仓库
# 添加harbor helm 私服
# 首先需要创建项目myrepo(当前设计的模式为public)
# chartrepo是必备的,不可缺少，不然就会推送到默认的library上面去了

root@k8sMaster-1:~# helm repo add --username=admin --password=Harbor12345 myrepo http://harbor.cerberus.com/chartrepo/myrepo
"myrepo" has been added to your repositories
root@k8sMaster-1:~# helm repo list
NAME  	URL                                                   
stable	https://kubernetes.oss-cn-hangzhou.aliyuncs.com/charts
local 	http://127.0.0.1:8879/charts                          
myrepo	http://harbor.ztloo.com/chartrepo/myrepo  

# 创建demo验证
root@k8sMaster-1:~/helmtest# helm create app
root@k8sMaster-1:~/helmtest# ls
app

# 注意cm-push 插件名由helm plugin list 获取。
# 只有在create 项目的上层目录执行push 才会识别。
root@k8sMaster-1:~/helmtest# helm cm-push app myrepo
Pushing app-0.1.0.tgz to myrepo...
Done.
```



![harbor-helm](https://raw.githubusercontent.com/CerberusDong/pictures/main/202302142338238.png)

