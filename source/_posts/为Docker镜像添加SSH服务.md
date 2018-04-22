---
title: 为Docker镜像添加SSH服务
date: 2018-04-20 16:01:19
categories: Docker系列
tags: [Dockerfile创建SSH服务,Commit容器创建SSH服务]
description: Docker镜像添加SSH服务支持。
---
一些进入容器的办法，比如`attach`、`exec`等命令，都无法解决远程管理容器的问题。因此，当我们需要远程登录到容器内进行一些操作的时候，就需要`SSH`的支持了。

有两种创建带有SSH服务的镜像：基于`Docker commit`命令创建和基于`Dockerfile`创建。

## 基于commit命令创建
Docker提供了`docker commit`命令，支持用户提交自己对制定容器的修改，并生产新的镜像。  
命令格式：`docker commit CONTAINER[REPOSITORY:[:TAG]]`

### 1、准备工作
首先，使用`ubuntu:14.04`镜像来创建容器：
`docker run -it ubuntu:14.04 /bin/bash`
更新`apt`缓存，并安装`openssh-server`
`apt-get update;apt-get install -y openssh-server`

### 2、安装和配置SSH服务
如果需要正常启动SSH服务，则目录`/var/run/sshd`必须存在，手动创建它，并启动SSH服务。
```
mkdir -p /var/run/sshd
/usr/sbin/sshd -D &

```

此时查看容器的22端口（ssh服务默认监听端口），可见此端口已经处于监听状态：
```
netstat -tlnp
//修改SSH服务的安全登录配置，取消pam登录限制
sed -ri 's/session required pam_loginuid.so/#session required pam_loginuid.so/g` /etc/pam.d/sshd

```
在root用户目录下创建`.ssh`目录，并复制需要登录的公钥信息（一般为本地主机目录下的`.ssh/id_rsa.pub`文件，可由`ssh-keygen -t rsa`命令生成）到`authorized_keys`文件中。

创建自动启动SSH服务的可执行文件`run.sh`。并添加可执行权限：
```
vi /run.sh
chmod +x run.sh

```
其中，`run.sh`脚本内容如下：
```
#!/bin/bash
/usr/sbin/sshd -D

```
最后，退出容器执行，`exit`命令。

### 3、保存镜像
`docker commit [OPTIONS]  CONTAINER [REPOSITORY[:TAG]]`

### 4、使用镜像
启动容器，并添加端口映射`10022-->22`。其中10022是宿主主机的端口，22是容器的SSH服务端口：
```
//启动容器
docker run -p 10022:22 -d sshd:ubuntu /run.sh 
//查看运行进程
docker ps

```
在宿主主机（`192.168.1.200`）或其他主机上，可以通过`SSH`访问10022端口来登录容器：
```
ssh 192.168.1.200 -p 10022

```

## 使用Dockerfile创建

### 1、创建工作目录
创建一个`ssh_ubuntu`目录,`mkdir ssh_ubuntu`,并在其中创建`Dockerfile`和`run.sh`：
```
touch Dockerfile run.sh

```
### 2、编写`run.sh`脚本和`authorized_keys`文件
```
#!/bin/bash
/usr/sbin/sshd -D

```
在宿主主机上生成SSH秘钥对，并创建`authorized_keys`文件：
```
ssh-keygen -t rsa
cat ~/.ssh/id_rsa.pub > authorized_keys

```
### 3、编写Dockerfile
```
#设置基础镜像
FROM ubuntu:14.04
#提供作者信息
LABEL author="xym" author_email="xym@126.com"
#更新命令
RUN apt-get update
#安装SSH服务
RUN apt-get install -y openssh-server
RUN mkdir -p /var/run/sshd
RUN mkdir -p /root/.ssh
#取消pam限制
RUN sed -ri 's/session required pam_loginuid.so/#session required pam_loginuid.so/g' /etc/pam.d/sshd 
#复制配置文件到相应位置，并赋予脚本可执行权限
ADD authorized_keys /root/.ssh/authorized_keys
ADD run.sh /run.sh
RUN chmod 755 run.sh
#开放端口
EXPOSE 22
#设置自启动命令
CMD ["/run.sh"]

```
### 4、创建镜像xd
在ssh_ubuntu目录下，使用`docker build`命令来创建镜像，这里注意还有一个"."，表示使用当前目录中的`Dockerfile`：
```
cd ssh_ubuntu
docker build -t sshd:Dockerfile .

```
### 5、测试镜像，运行容器
```
//启动镜像，映射容器22端口到本地的10122端口
docker run -d -p 10122:22 sshd:Dockerfile
//在宿主主机新打开一个终端，连接到新建的容器
ssh 192.168.1.200 -p 10122

```

