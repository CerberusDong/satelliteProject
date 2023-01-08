# 第五章   Docker-file进阶

## 5.1 升级版Docker-file

```shell
FROM ubuntu:21.04
RUN apt-get update && \
    apt-get install -y wget && \
    wget https://github.com/ipinfo/cli/releases/download/ipinfo-2.0.1/ipinfo_2.0.1_linux_amd64.tar.gz && \
    tar zxf ipinfo_2.0.1_linux_amd64.tar.gz && \
    mv ipinfo_2.0.1_linux_amd64 /usr/bin/ipinfo && \
    rm -rf ipinfo_2.0.1_linux_amd64.tar.gz

```

**注意：在编写Docker-file的时候，尽量让指令减少。尤其是run 指令。要不然会增大镜像的大小，会让其变得臃肿。**

## 5.2 基础镜像选择

- 官方镜像优于非官方的镜像，如果没有官方镜像，则尽量选择Dockerfile开源的
- 固定版本tag而不是每次都使用latest
- 尽量选择体积小的镜像

## 5.3 文件的复制和目录操作	

`COPY` 和 `ADD` 都可以把local的一个文件复制到镜像里，如果目标目录不存在，则会自动创建

```shell
# 复制普通文件
FROM python:3.9.5-alpine3.13
COPY hello.py /app/hello.py

# 复制压缩文件
# ADD 比 COPY高级一点的地方就是，如果复制的是一个gzip等压缩文件时，ADD会帮助我们自动去解压缩文件。
FROM python:3.9.5-alpine3.13
ADD hello.tar.gz /app/

```

## 5.4 构建参数和环境变量 

`ARG` 和 `ENV` 是经常容易被混淆的两个Dockerfile的语法，都可以用来设置一个“变量”。 但实际上两者有很多的不同</br> 通过ARG和 ENV可以轻松替换下载文件的版本号。

```shell
# ENV 设置变量
FROM ubuntu:21.04
ENV VERSION=2.0.1
RUN apt-get update && \
    apt-get install -y wget && \
    wget https://github.com/ipinfo/cli/releases/download/ipinfo-${VERSION}/ipinfo_${VERSION}_linux_amd64.tar.gz && \
    tar zxf ipinfo_${VERSION}_linux_amd64.tar.gz && \
    mv ipinfo_${VERSION}_linux_amd64 /usr/bin/ipinfo && \
    rm -rf ipinfo_${VERSION}_linux_amd64.tar.gz


# ARG 设置变量
FROM ubuntu:21.04
ARG VERSION=2.0.1
```

![env_arg](https://raw.githubusercontent.com/CerberusDong/pictures/main/202301082154961.png)

**ARG 可以在镜像build的时候动态修改value, 通过 `--build-arg`**

```shell
docker image build -f .\Dockerfile-arg -t ipinfo-arg-2.0.0 --build-arg VERSION=2.0.0 .
```

## 5.5  容器启动命令

 **CMD** 

`CMD` 设置的命令，可以在docker container run 时传入其它命令，覆盖掉 `CMD` 的命令

```shell
root@ztloo:~/flaskPro# vim Dockerfile-cmd 
FROM ubuntu:16.04
CMD ["echo", "hello docker"]

# 指定dockerfile 构建镜像
root@ztloo:~/flaskPro# docker build -f Dockerfile-cmd -t u:v1 .

# 启动容器，--rm 选项 容器退出时就能够自动清理容器，一般用于调试容器时添加。
root@ztloo:~/flaskPro# docker run --rm --name ubt2 u:v1  
hello docker

# 会覆盖CMD命令
root@ztloo:~/flaskPro# docker run --rm --name ubt u:v1 ls /
bin   dev  home  lib64	mnt  proc  run	 srv  tmp  var
boot  etc  lib	 media	opt  root  sbin  sys  usr

```

 **ENTRYPOINT**

ENTRYPOINT 所设置的命令是一定会被执行的。不会向CMD那样被覆盖。

ENTRYPOINT和CMD可以联合使用，ENTRYPOINT` 设置执行的命令，CMD传递参数

## 5.6 Shell 格式和 Exec 格式

**SHELL 格式**

```
CMD echo "hello docker"
ENTRYPOINT echo "hello docker"
```

**Exec格式**

```shell
ENTRYPOINT ["echo", "hello docker"]
CMD ["echo", "hello docker"]
```

**注意**

在变量传值的时候，exec 格式的需要加sh -c 。

```shell
FROM ubuntu:21.04
ENV NAME=docker
CMD echo "hello $NAME"

FROM ubuntu:21.04
ENV NAME=docker
CMD ["sh", "-c", "echo hello $NAME"]
```

## 5.7  镜像缓存概念

**使用镜像缓存：**构建新镜像如果某镜像层已经存在，就直接使用，无需重新创建

**不适用镜像缓存**：如果我们希望在构建镜像时不使用缓存，可以在 docker build 命令中加上 --no-cache 参数

Docker 会缓存已有镜像的镜像层，构建新镜像时，如果某镜像层已经存在，就直接使用，无需重新创建。

**缓存失效**：Dockerfile 中每一个指令都会创建一个镜像层，下层是依赖于上层的。无论什么时候，只要某一层发生变化，其下面所有层的缓存都会失效。

**使用缓存场景**：除了构建时使用缓存，Docker 在下载镜像时也会使用。例如我们下载 httpd 镜像。
由 Dockerfile 可知 httpd 的 base 镜像为 debian，正好之前已经下载过 debian 镜像，所以有缓存可用。通过 docker history 可以进一步验证。

## 5.8  构建忽略设置

.Dockerignore 文件使您可以提及在构建镜像时可能要忽略的文件和/或目录的列表。这肯定会减小镜像的大小，总之，始终建议您在构建上下文中创建一个.dockerignore文件，以使Docker镜像小而安全，并加快构建过程。

**这是一个示例.dockerignore文件：**

```shell
# 忽略

# 在根的任何直接子目录中排除、名称以 temp 开头的文件和目录。 
# 例如 /somedir/temporary.txt  /somedir/temp 都会被排除
*/temp* 

# 从根以下两级的任何子目录中排除以 temp 开头的文件和目录。 
# 例如，/somedir/subdir/temporary.txt 被排除
*/*/temp*  

# 排除根目录中名称为 temp 的单字符扩展名的文件和目录。
# 例如 /tempa 和 /tempb 被排除在外。
temp?

#  排除所有以md 结尾的，但是README.md除外
*.md
!README.md
```

## 5.9 多阶段构建

[多阶段构建](https://docs.docker.com/develop/develop-images/multistage-build/)允许您大幅减小最终镜像的大小，而无需努力减少中间层和文件的数量。

由于镜像是在构建过程的最后阶段构建的，因此您可以通过[利用构建缓存](https://docs.docker.com/develop/develop-images/dockerfile_best-practices/#leverage-build-cache)来最小化镜像层。

例如，如果您的构建包含多个层，您可以将它们从不经常更改（以确保构建缓存可重用）排序到更频繁更改：

- 安装构建应用程序所需的工具
- 安装或更新库依赖项
- 生成您的应用程序

Go 应用程序的 Dockerfile如下所示：

```shell
# syntax=docker/dockerfile:1
FROM golang:1.16-alpine AS build

# 安装项目所需的工具
# 运行 `docker build --no-cache .` 来更新依赖
RUN apk add --no-cache git
RUN go get github.com/golang/dep/cmd/dep


# 使用 Gopkg.toml 和 Gopkg.lock 列出项目依赖项
# 这些层仅在 Gopkg 文件更新时重新构建
COPY Gopkg.lock Gopkg.toml /go/src/project/
WORKDIR /go/src/project/
# 安装库依赖
RUN dep ensure -vendor-only
# 复制整个项目并构建它
# 当项目目录中的文件发生变化时，会重建该层
COPY . /go/src/project/
RUN go build -o /bin/project


# 这会产生单层图像
FROM scratch
#可以直接使用第一阶段生成的文件，相当于接管了第一阶段的上下文
COPY --from=build /bin/project /bin/project 
ENTRYPOINT ["/bin/project"]
CMD ["--help"]


# 总结
上面的from 的作用是用来产生我们想要的文件，例如/bin/project 被编译好的可以执行程序。 然后我们在最后一个from 中直接使用它所产生的文件资源，通过COPY --from。

因为多阶段构建，最终构建的镜像以最后一个from 的构建的。所以导致它会比较小。因为我们这里只需要可以运行go 就可以了，基础镜像可以是空镜像scratch，导致之前几百兆，几个G 的变成了几十兆，就可以了，这就是多阶段构建的魅力。

```

**要记住**：多阶段构建，就是前面多个from 的产生的资源，供最后一个from使用，以生产者模式，来构建镜像的。

最终只保留最后一个from构建我们的镜像。 会舍弃之前的from ，之前的from 只保留了最后一个from 需要使用到的资源而已。

## 5.10  镜像优化注意事项

**不要安装不必要的包**

为了减少复杂性、依赖关系、文件大小和构建时间，请避免安装额外或不必要的软件包，因为它们可能“很高兴拥有”。例如，您不需要在数据库图像中包含文本编辑器。

**解耦应用程序**

每个容器应该只有一个关注点。将应用程序解耦到多个容器中，可以更轻松地水平扩展和重用容器。例如，一个 Web 应用程序堆栈可能由三个独立的容器组成，每个容器都有自己独特的图像，以分离的方式管理 Web 应用程序、数据库和内存缓存。

将每个容器限制为一个进程是一个很好的经验法则，但这不是一个硬性规定。例如，不仅可以 [使用 init 进程生成](https://docs.docker.com/engine/reference/run/#specify-an-init-process)容器，某些程序可能会自行生成其他进程。例如，[Celery](https://docs.celeryproject.org/)可以产生多个工作进程，而[Apache](https://httpd.apache.org/)可以为每个请求创建一个进程。

使用您的最佳判断来保持容器尽可能清洁和模块化。如果容器相互依赖，您可以使用[Docker 容器网络](https://docs.docker.com/network/) 来确保这些容器可以通信。

**尽量减少层数**

在旧版本的 Docker 中，尽量减少镜像中的层数以确保它们的性能非常重要。添加了以下功能以减少此限制：

- 只有说明`RUN`, `COPY`,`ADD`创建图层。其他指令创建临时中间图像，并且不增加构建的大小。
- 在可能的情况下，使用[多阶段构建](https://docs.docker.com/develop/develop-images/multistage-build/)，并且只将您需要的工件复制到最终图像中。这允许您在中间构建阶段包含工具和调试信息，而不会增加最终映像的大小。

**对多行参数进行排序**

只要有可能，通过按字母数字排序多行参数来简化以后的更改。这有助于避免重复包并使列表更容易更新。这也使 PR 更容易阅读和审查。在反斜杠 ( ) 前添加一个空格`\`也有帮助。

[`buildpack-deps`这是图像](https://github.com/docker-library/buildpack-deps)中的一个示例：

```
RUN apt-get update && apt-get install -y \
  bzr \
  cvs \
  git \
  mercurial \
  subversion \
  && rm -rf /var/lib/apt/lists/*
```



## 5.11 系统多架构构建 (扩展)

Docker 19.03及以上的版本支持`docker buildx build`命令使用 BuildKit 来构建镜像。通过`--platform`参数可以支持构建多架构的Docker镜像。

```shell
# step1 ：登陆hub 
> docker login
#  列出构建器实例
root@ztloo:~/flaskPro# docker buildx ls
NAME/NODE DRIVER/ENDPOINT STATUS  PLATFORMS
default * docker                  
  default default         running linux/amd64, linux/386
# create 创建一个新的构建器实例
root@ztloo:~/flaskPro#  docker buildx create --use --name mybuild 
mybuild
root@ztloo:~/flaskPro# docker buildx ls
NAME/NODE  DRIVER/ENDPOINT             STATUS   PLATFORMS
mybuild *  docker-container                     
  mybuild0 unix:///var/run/docker.sock inactive 
  

#构建镜像
root@ztloo:~/flaskPro# docker buildx build --push --platform linux/arm64  -t ztloo/flaskmany:v1 .
```

