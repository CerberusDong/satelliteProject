# 第一章   K8S 概述

Kubernetes是谷歌以Borg为前身，基于谷歌15年生产环境经验的基础上开源的一个项目，Kubernetes致力于提供跨主机集群的自动部署、扩展、高可用以及运行应用程序容器的平台。

Kubernetes 这个名字，起源于古希腊，是舵手的意思，所以它的 logo 即像一张渔网又像一个罗盘，谷歌选择这个名字还有一个深意：既然docker把自己比作一只鲸鱼，驮着集装箱，在大海上遨游，google 就要用Kubernetes去掌握大航海时代的话语权，去捕获和指引着这条鲸鱼按照主人设定的路线去巡游。

得益于 docker 的特性，服务的创建和销毁变得非常快速、简单。Kubernetes 正是以此为基础，实现了集群规模的管理、编排方案，使应用的发布、重启、扩缩容能够自动化。

![image-20230214232808626](https://raw.githubusercontent.com/CerberusDong/pictures/main/202302142328717.png)



**K8sMaster** : 管理K8sNode的。

 **K8sNode**:具有docker环境 和k8s组件（kubelet、k-proxy） ，载有容器服务的工作节点。

**Controller-manager**:  k8s 的大脑，它通过 API Server监控和管理整个集群的状态，并确保集群处于预期的工作状态。

**API Server：**k8s API Server提供了k8s各类资源对象（pod,RC,Service等）的增删改查及watch等HTTP Rest接口，是整个系统的数据总线和数据中心。

**etcd：**高可用强一致性的服务发现存储仓库，kubernetes集群中，etcd主要用于配置共享和服务发现

**Scheduler**： 主要是为新创建的pod在集群中寻找最合适的node，并将pod调度到K8sNode上。

**kubelet**: 作为连接Kubernetes Master和各Node之间的桥梁，用于处理Master下发到本节点的任务，管理 Pod及Pod中的容器

**k-proxy** 是 kubernetes 工作节点上的一个网络代理组件，运行在每个节点上，维护节点上的网络规则。这些网络规则允许从集群内部或外部的网络会话与 Pod 进行网络通信。监听 API server 中 资源对象的变化情况，代理后端来为服务配置负载均衡。

**Pod**： 一组容器的打包环境。在Kubernetes集群中，Pod是所有业务类型的基础，也是K8S管理的最小单位级，它是一个或多个容器的组合。这些容器共享存储、网络和命名空间，以及如何运行的规范。（k8s =学校、pod = 班级、容器= 学生）



## 1.1  为什么要使用K8S

当服务越来越多，容器越来越多， 单台物理主机无法满足你的性能需要时。

需要实现，多台物理机进行分布式系统设计来实现服务的负载均衡，多台物理机的服务与服务之间的通信、并且易扩容，以伸缩、易部署、高可用等等的实现，想想就很头疼。

这个时候谷歌开源了他们的Borg 公开版本，并命名叫Kubernetes , 简称k8s，我们就可以利用现成的K8s 的分布式容器管理平台，来管理我们的复杂多变的服务、就可以避免我们耗费很大的人力、财力、时间成本来实现这种分布式系统架构所带来的开销，只需要学会k8s 如何使用和运维即可。 

互联网公司有超强的横向扩容能力可以让我们的竞争力大大提升，我们利用k8s 提供的工具，不用修改代码，就能将包含几个node 的小集群，扩展到可以媲美百度，阿里，腾讯的超大集群规模，甚至可以在线完成集群的扩容，只要微服务架构设计的合理，就能轻松的进行弹性伸缩，系统就能够承受住大量用户并发访问带来的巨大压力。



## 1.2 K8S 提供的功能

1. **服务发现和负载均衡**： Kubernetes 可以使用 DNS 名称或自己的 IP 地址公开容器，如果进入容器的流量很大， Kubernetes 可以负载均衡并分配网络流量，从而使部署稳定。
2. **存储编排** : Kubernetes 允许你自动挂载你选择的存储系统，例如本地存储、公共云提供商等。
3. **自动部署和回滚**： 你可以使用 Kubernetes 描述已部署容器的所需状态，它可以以受控的速率将实际状态 更改为期望状态。例如，你可以自动化 Kubernetes 来为你的部署创建新容器， 删除现有容器并将它们的所有资源用于新容器.
4. **自动完成装箱计算** ： Kubernetes 允许你指定每个容器所需 CPU 和内存（RAM）。 当容器指定了资源请求时，Kubernetes 可以做出更好的决策来管理容器的资源。
5. **自我修复**： Kubernetes 重新启动失败的容器、替换容器、杀死不响应用户定义的 运行状况检查的容器，并且在准备好服务之前不将其通告给客户端
6. **密钥与配置管理** ：Kubernetes 允许你存储和管理敏感信息，例如密码、OAuth 令牌和 ssh 密钥。 你可以在不重建容器镜像的情况下部署和更新密钥和应用程序配置，也无需在堆栈配置中暴露密钥。



##  1.3  K8S入门

#### **集群设计**

Kubernetes 可以管理大规模的集群，使集群中的每一个节点彼此连接，能够像控制一台单一的计算机一样控制整个集群。

集群有两种角色，一种是 **master** ，一种是 **Node**（也叫worker）。

- **master** 是集群的"大脑"，负责管理整个集群：像应用的调度、更新、扩缩容等。
- **Node** 就是具体"干活"的，一个Node一般是一个虚拟机或物理机，它上面事先运行着 docker 服务和 kubelet 服务（ Kubernetes 的一个组件），当接收到 master 下发的"任务"后，Node 就要去完成任务（用 docker 运行一个指定的应用）

![k8s1](https://raw.githubusercontent.com/CerberusDong/pictures/main/202302142332808.png)

#### Deployment 

**Deployment - 应用管理者**

当我们拥有一个 Kubernetes 集群后，就可以在上面跑我们的应用了，前提是我们的应用必须支持在 docker 中运行，也就是我们要事先准备好docker镜像。

有了镜像之后，一般我们会通过Kubernetes的 **Deployment** 的配置文件去描述应用，比如应用叫什么名字、使用的镜像名字、要运行几个实例、需要多少的内存资源、cpu 资源等等。

有了配置文件就可以通过Kubernetes提供的命令行客户端 - **kubectl** 去管理这个应用了。kubectl 会跟 Kubernetes 的 master 通过RestAPI通信，最终完成应用的管理。
比如我们刚才配置好的 Deployment 配置文件叫 app.yaml，我们就可以通过
"kubectl create -f app.yaml" 来创建这个应用啦，之后就由 Kubernetes 来保证我们的应用处于运行状态，当某个实例运行失败了或者运行着应用的 Node 突然宕机了，Kubernetes 会自动发现并在新的 Node 上调度一个新的实例，保证我们的应用始终达到我们预期的结果。

![02_first_app](https://raw.githubusercontent.com/CerberusDong/pictures/main/202302142332839.png)

#### Pod

**Pod - Kubernetes最小调度单位**

其实在上一步创建完 Deployment 之后，Kubernetes 的 Node 做的事情并不是简单的docker run 一个容器。出于像易用性、灵活性、稳定性等的考虑，Kubernetes 提出了一个叫做 Pod 的东西，作为 Kubernetes 的最小调度单位。所以我们的应用在每个 Node 上运行的其实是一个 Pod。Pod 也只能运行在 Node 上。如下图：

![image-20230214233406128](https://raw.githubusercontent.com/CerberusDong/pictures/main/202302142334285.png)



那么什么是 Pod 呢？Pod 是一组容器（当然也可以只有一个）。容器本身就是一个小盒子了，Pod 相当于在容器上又包了一层小盒子。这个盒子里面的容器有什么特点呢？

- 可以直接通过 volume 共享存储。
- 有相同的网络空间，通俗点说就是有一样的ip地址，有一样的网卡和网络设置。
- 多个容器之间可以“了解”对方，比如知道其他人的镜像，知道别人定义的端口等。

至于这样设计的好处呢，还是要大家深入学习后慢慢体会啦~

![03_pods](https://raw.githubusercontent.com/CerberusDong/pictures/main/202302142332852.png)

#### Service 

**Service - 服务发现 - 找到每个Pod**

上面的 Deployment 创建了，Pod 也运行起来了。如何才能访问到我们的应用呢？

最直接想到的方法就是直接通过 Pod-ip+port 去访问，但如果实例数很多呢？好，拿到所有的 Pod-ip 列表，配置到负载均衡器中，轮询访问。但上面我们说过，Pod 可能会死掉，甚至 Pod 所在的 Node 也可能宕机，Kubernetes 会自动帮我们重新创建新的Pod。再者每次更新服务的时候也会重建 Pod。而每个 Pod 都有自己的 ip。所以 Pod 的ip 是不稳定的，会经常变化的。

面对这种变化我们就要借助另一个概念：Service。它就是来专门解决这个问题的。不管Deployment的Pod有多少个，不管它是更新、销毁还是重建，Service总是能发现并维护好它的ip列表。Service对外也提供了多种入口：

1. ClusterIP：Service 在集群内的唯一 ip 地址，我们可以通过这个 ip，均衡的访问到后端的 Pod，而无须关心具体的 Pod。
2. NodePort：Service 会在集群的每个 Node 上都启动一个端口，我们可以通过任意Node 的这个端口来访问到 Pod。
3. LoadBalancer：在 NodePort 的基础上，借助公有云环境创建一个外部的负载均衡器，并将请求转发到 NodeIP:NodePort。
4. ExternalName：将服务通过 DNS CNAME 记录方式转发到指定的域名（通过 spec.externlName 设定）。

![04_services](https://raw.githubusercontent.com/CerberusDong/pictures/main/202302142332815.png)

#### RollingUpdate

**RollingUpdate - 滚动升级**

滚动升级是Kubernetes中最典型的服务升级方案，主要思路是一边增加新版本应用的实例数，一边减少旧版本应用的实例数，直到新版本的实例数达到预期，旧版本的实例数减少为0，滚动升级结束。在整个升级过程中，服务一直处于可用状态。并且可以在任意时刻回滚到旧版本。

![05_rollingupdate](https://raw.githubusercontent.com/CerberusDong/pictures/main/202302142332849.gif)