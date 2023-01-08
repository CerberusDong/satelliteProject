# 第四章   Docker Dockerfile详解

## 4.1 Dockerfile是什么

Dockerfile 是一个用来构建镜像的文本文件

例如：部署你的SpringBoot项目

- 基于JDK1.8的镜像
- 需要将jar等文件放到镜像目录
- 运行镜像后需要启动SpringBoot项目

这时候如果直接从仓库获取镜像，然后运行成容器，并进入容器部署自己的项目，这一列操作就非常麻烦。而如果有的Dockerfile，则可以一次性生成满足你需求的自有镜像。

## 4.2 Dockerfile构建镜像示例

```shell
# 1、添加一个目录
# 如:/usr/local/dockerfile

# 2、添加Dockerfile文件
# 在上面的目录下创建一个Dockerfile文件，文件名最好使用Dockerfile，这样生成镜像的命令就不用指定文件了，文件内容如下
FROM openjdk:8-jdk-alpine
VOLUME /tmp
COPY *.jar demo.jar
ENTRYPOINT ["java","-jar","/demo.jar"]

# 3、将Jar包放到目录下

# 4、生成镜像
# docker build -t 镜像名称:标签名 .
docker build -t demo:v1 .

# 生成完成之后,查看本地镜像
docker ps
```

## 4.3 Dockerfile构建命令详解

```shell
# 1、FROM
# FROM命令是Dockerfile文件的开始，标识创建的镜像是基于哪个镜像
# 格式：FROM 镜像名[:标签名]
# 示例1：基于Nginx
FROM nginx
# 示例2：基于openjdk:8-jdk-alpine
FROM openjdk:8-jdk-alpine

# 2、copy
# 复制指令，从上下文目录中复制文件或者目录到容器里指定路径
# 格式
# COPY [--chown=<user>:<group>] <源路径1>...  <目标路径>
# COPY [--chown=<user>:<group>] ["<源路径1>",...  "<目标路径>"]
# 将以hom开头的文件和目录拷贝到容器目录下
COPY hom* /mydir/
# 将一些文件拷贝到容器目录下
COPY hom?.txt /mydir/
# 将一个文件拷贝到容器，并重新命名
COPY xxxx.jar abc.jar


```

