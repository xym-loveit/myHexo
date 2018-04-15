---
title: Docker 基本操作之仓库
date: 2018-04-15 15:43:29
categories: Docker系列
tags: [docker仓库,login命令,search命令,pull命令,本地搭建私有仓库配置,时速云镜像仓库]
description: Docker入门指南，仓库概念及基本操作。
---
仓库（Repository）是集中存放镜像的地方，分为公共仓库和私有仓库。一个容易与之混淆的概念是注册服务器（Registry）。实际上注册服务器是存放仓库的具体服务器，一个注册服务器上可以有多个仓库，而每个仓库下面可以有多个镜像。从这方面来说，可将仓库看做一个具体的项目或目录。例如对于仓库地址`private-docker.com/ubuntu`来说,`private-docker.com`是注册服务器地址，`ubuntu`是仓库名。

## Docker Hub公共镜像
访问地址：[https://hub.docker.com/](https://hub.docker.com/)

### 1、登录仓库
命令格式：`docker login [OPTIONS] [SERVER]`

```

Options:
  -p, --password string   Password
      --password-stdin    Take the password from stdin
  -u, --username string   Username
  
//通过存储的登录信息，直接确认登录  
[root@xxx ~]# docker login
Authenticating with existing credentials...
WARNING! Your password will be stored unencrypted in /root/.docker/config.json.
Configure a credential helper to remove this warning. See
https://docs.docker.com/engine/reference/commandline/login/#credentials-store

Are you sure you want to proceed? [y/N] y
Login Succeeded  

```

### 2、基本操作

```

//查找官方仓库中的镜像，搜索关键词centos
[root@xxx ~]# docker search centos
NAME                               DESCRIPTION                                     STARS               OFFICIAL            AUTOMATED
centos                             The official build of CentOS.                   4188                [OK]                
ansible/centos7-ansible            Ansible on Centos7                              108                                     [OK]
jdeathe/centos-ssh                 CentOS-6 6.9 x86_64 / CentOS-7 7.4.1708 x86_…   94                                      [OK]
consol/centos-xfce-vnc             Centos container with "headless" VNC session…   52                                      [OK]
imagine10255/centos6-lnmp-php56    centos6-lnmp-php56                              40                                      [OK]
tutum/centos                       Simple CentOS docker image with SSH access      38                                      
gluster/gluster-centos             Official GlusterFS Image [ CentOS-7 +  Glust…   26                                      [OK]

...

//下载centos镜像
[root@xxx ~]# docker pull centos
Using default tag: latest
latest: Pulling from library/centos
469cfcc7a4b3: Pull complete 
Digest: sha256:989b936d56b1ace20ddf855a301741e52abca38286382cba7f44443210e96d16
Status: Downloaded newer image for centos:latest

```

根据search结果，可将镜像资源分为两类。一种是类似centos这样的镜像，称为基础或根镜像。这些镜像是由docker公司创建、验证、支持、提供。这样的镜像往往使用单个单词作为名字。还有一种类型，比如`ansible/centos7-ansible`镜像，它是由docker用户ansible创建并维护的，带有用户名称前缀，表明是某用户下的某仓库。可以通过用户名称前缀`user_name/镜像名`来指定使用某个用户提供的镜像。另外，在查找的时候通过-s N参数可以指定仅显示评价为N星级以上的镜像。

### 3、自动创建
自动创建（Automated Builds）功能对于需要经常升级镜像内程序来说，十分方便。有时候，用户创建了镜像，安装了某个软件，如果软件发布新版本则需要手动更新镜像。而自动创建允许用户通过Docker Hub指定跟踪一个目标网站（目前支持Github或Bitbucket）上的项目，一旦项目发生新的提交，则自动执行创建。

要配置自动创建，包括如下步骤：
* 1)创建并登陆Docker Hub，以及目标网站；在目标网站中连接账户到Docker Hub。
* 2)在Docker Hub中配置一个“自定创建”
* 3)选取一个目标网站中的项目（需要含Dockerfile）和分支；
* 4)指定Dockerfile位置，并提交创建

之后，可以在Docker Hub的“自动创建”页面中跟踪每次创建的状态。

## 国内时速云镜像
访问地址：[https://hub.tenxcloud.com/](https://hub.tenxcloud.com/)

### 下载镜像

```

//下载镜像
[root@xxx ~]# docker pull index.tenxcloud.com/tenxcloud/centos
Using default tag: latest
latest: Pulling from tenxcloud/centos
a3ed95caeb02: Pull complete 
5989106db7fb: Pull complete 
c9d12ea9fc45: Pull complete 
68317fcc0aa1: Pull complete 
83ef48200e63: Pull complete 
c6eb26bf54de: Pull complete 
1bcf3170bbc2: Pull complete 
Digest: sha256:190cbd5234c4aad993b852d5f118ecfba5499adc6f752026938bce0eca754b0c
Status: Downloaded newer image for index.tenxcloud.com/tenxcloud/centos:latest

//查看镜像
[root@xxx ~]# docker images
REPOSITORY                                        TAG                 IMAGE ID            CREATED             SIZE
xymtest                                           0.1                 8a758d16a99b        22 hours ago        113MB
ubuntu                                            latest              c9d990395902        32 hours ago        113MB
centos                                            latest              e934aafc2206        7 days ago          199MB
registry                                          latest              d1fd7d86a825        3 months ago        33.3MB
index.tenxcloud.com/tenxcloud/centos              latest              6e7516266d96        23 months ago       310MB


```

## 搭建本地私有仓库

### 1、使用registry镜像创建私有仓库

安装Docker后，可以通过官方提供的`registry`镜像来简单搭建一套本地私有仓库环境：

```

//在本地启动registry镜像，作为私有服务器，监听5000端口
[root@xxx ~]# docker run -d -p 5000:5000 registry
Unable to find image 'registry:latest' locally
latest: Pulling from library/registry
81033e7c1d6a: Pull complete 
b235084c2315: Pull complete 
c692f3a6894b: Pull complete 
ba2177f3a70e: Pull complete 
a8d793620947: Pull complete 
Digest: sha256:672d519d7fd7bbc7a448d17956ebeefe225d5eb27509d8dc5ce67ecb4a0bce54
Status: Downloaded newer image for registry:latest
67083c200be2f6a043377a9b4d69af24d0ba9c58a140b753634f5be4ede67464

```
这将自动下载并启动一个`registry`容器，创建本地的私有仓库服务。默认情况下，会将仓库创建在容器的`/tmp/registry`目录下。可以通过-v参数来将镜像文件存放在本地的指定路径。例如以下实例将上传的镜像放到`/opt/data/registry`目录：

```

//在本地启动registry镜像，作为私有服务器，监听5000端口,并指定本地目录数据卷/opt/data/registry，映射容器内/tmp/registry目录
[root@xxx ~]# docker run -d -p 5000:5000 -v /opt/data/registry:/tmp/registry registry
5b9a1ad53352c4e3a6f5bf7d70ef9a6d573cb4da064fb24f8f406444d125555c

```

### 2、管理私有仓库
```

//标记镜像
[root@xxx ~]# docker tag xymtest:0.1 192.168.206.128:5000/test

//上传镜像
[root@xxx ~]# docker push 192.168.206.128:5000/test
The push refers to repository [192.168.206.128:5000/test]
Get https://192.168.206.128:5000/v2/: http: server gave HTTP response to HTTPS client

//编辑daemon.json,配置，"insecure-registries":["192.168.206.128:5000"]，表示信任这个私有仓库，不进行安全证书检查
[root@xxx docker]# pwd
/etc/docker
[root@xxx docker]# cat daemon.json
{
  //表示信任这个私有仓库，不进行安全证书检查
  "insecure-registries":["192.168.206.128:5000"],
  "registry-mirrors": ["https://4vehewku.mirror.aliyuncs.com"]
}

//重启docker 服务
[root@xxx docker]# systemctl restart docker

//推送本地镜像到私有仓库
[root@xxx docker]# docker push 192.168.206.128:5000/test
The push refers to repository [192.168.206.128:5000/test]
6d7697f5e458: Pushed 
a8de0e025d94: Pushed 
a5e66470b281: Pushed 
ac7299292f8b: Pushed 
e1a9a6284d0d: Pushed 
fccbfa2912f0: Pushed 
latest: digest: sha256:46d25028e0eb194348b8b1256b1375238b44116a018de67f3318a1bb9954ee9d size: 1564

//下载刚刚上传的，私有仓库镜像
[root@xxx docker]# docker pull 192.168.206.128:5000/test
Using default tag: latest
latest: Pulling from test
Digest: sha256:46d25028e0eb194348b8b1256b1375238b44116a018de67f3318a1bb9954ee9d
Status: Downloaded newer image for 192.168.206.128:5000/test:latest

```

如果要使用安全证书，我们也可以从较知名的CA服务商（如`verisign`）申请公开的`SSL/TLS`证书，或者使用`openssl`等软件自行生成。











