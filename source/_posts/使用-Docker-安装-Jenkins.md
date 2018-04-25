---
title: 使用 Docker 安装 Jenkins
date: 2018-04-25 11:14:57
categories: Docker系列
tags: [Docker安装Jenkins,Jenkins]
description: Docker安装Jenkins简单步骤。
---
## 介绍
>jenkins是一个用Java编写的开源自动化服务器,它是Hudson的一个分支project ; 它是一个持续集成软件(continuous integration),它以节点为单位,连接整个工作流, 通过各种类型插件支持构成具有个性化要求的项目持续集成, 通过各种各样的插件(plugin)来实现各个节点的功能, 它们共同完成持续集成(自动部署)/自动测试或者持续交付等工作.

## 安装

1、直接从 DockerHub 上pull 镜像：
```
docker pull jenkins

```

2、由于 Jenkins 容器运行后，会自动在宿主计算机中挂在一个数据卷 `var/jenkins_home`，我们在主机中可以新建一个数据卷的文件夹，这里注意的是，有权限问题，不然会启动失败，有点坑这里，卡了半天，给宿主的这个挂载卷目录中加上下面的权限的就好了，改成为uid 1000的用户，具体参考阿里云[谈谈 Docker Volume 之权限管理（一）](https://yq.aliyun.com/articles/53990)：
```
chown -R 1000 /var/jenkins_home

```
3、启动容器
```
docker run -p 7322:8080 -p 50000:50000 -v /var/jenkins_home/:/var/jenkins_home/  --name my_jenkins -d jenkins

```

* 8080 端口是访问 jenkins 网页的端口，如果你想在 80 端口访问，就改成 -p 80:8080
* 50000 端口与 slave 有关，主要作为master的jenkins用来连接slave的。

可以更改挂载卷的目录，不过记得也要设置目录权限的问题。

使用 `docker ps` 查看运行的容器。


## 配置和使用

1、使用 host + port 访问 jenkins，会进入第一个页面：

![Jenkins安装](http://op7wplti1.bkt.clouddn.com/install_Jenkins.png)
    
因为我们将目录`/var/jenkins_home`已经挂载在宿主主机，可以直接去这个目录查看密码：
```
 cat /var/jenkins_home/secrets/initialAdminPassword
c49d4a7883e1410685c45092a6fabeac

```

2、进入后就开始安装插件的过程，然后等待安装完成。

3、然后跳出一个页面设置账号和密码，这样就安装完成，后面学习使用 jenkins 运用到工作中。
