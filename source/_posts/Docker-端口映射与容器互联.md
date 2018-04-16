---
title: Docker 端口映射与容器互联
date: 2018-04-16 16:11:59
categories: Docker系列
tags: [docker容器端口映射,P/p命,link选项,容器互联原理]
description: Docker入门指南,端口映射和容器互联。
---

在实践中会经常碰到需要多个服务组件容器共同协作的情况，这往往需要多个容器之间能够互相访问到对方的服务。除了通过网络访问外，Docker还提供了两个很方便的功能来满足服务访问的基本需求：一是允许映射容器内应用的服务端口到本地宿主主机；另一个是互联机制实现多个容器间通过容器名来快速访问。

## 端口映射实现访问容器

### 1、从外部访问容器应用

在启动容器的时候，如果不指定对应的参数，在容器外部是无法通过网络来访问容器内的网络应用和服务的。当容器中运行一些网络应用，要让外部访问这些应用时，可以通过-p或-P参数来指定端口映射。当使用-P（大写的）标记时，Docker会随机映射一个端口到内部容器开放的网络端口：

```

[root@xxx ~]# docker run -d -P training/webapp python app.py
f2c1a06b94b49de281b403fa339d5975be5dd6fae662c664b300540c851c3565

//ps命令后发现本地主机的32783被映射到了容器的5000端口。访问宿主主机的32783端口即可访问容器内的web应用
[root@xym ~]# docker ps -a
CONTAINER ID        IMAGE               COMMAND                  CREATED             STATUS                          PORTS                     NAMES
f2c1a06b94b4        training/webapp     "python app.py"          3 seconds ago       Up 2 seconds                    0.0.0.0:32783->5000/tcp   inspiring_hawking

//同样使用docker logs命令来查看应用信息
[root@xxx ~]# docker logs -f  f2c
 * Running on http://0.0.0.0:5000/ (Press CTRL+C to quit)
172.17.0.1 - - [14/Apr/2018 16:13:50] "GET / HTTP/1.1" 200 -

```

`-p`（小写的）可以指定要映射的端口，并且，在一个指定端口上只可以绑定一个容器。支持的格式有：`IP:HostPort:ContainerPort|IP::ContainerPort|HostPort:ContainerPort`

### 2、映射所有接口地址
```
//使用HostPort:ContainerPort格式将本地的5000端口映射到容器的5000端口
[root@xxx ~]# docker run -d -p 5000:5000 training/webapp python app.py
22c0bc271d46033c16f9ebdbc60ebf78938a1a0710bcede98afd15f887a92968

//查看容器，注意端口映射栏
[root@xxx ~]# docker ps -a
CONTAINER ID        IMAGE               COMMAND                  CREATED             STATUS                      PORTS                    NAMES
22c0bc271d46        training/webapp     "python app.py"          8 seconds ago       Up 7 seconds                0.0.0.0:5000->5000/tcp   confident_saha

//多次使用-p标记可以绑定多个端口
[root@xxx ~]# docker run -d -p 5000:5000 -p 8080:80  training/webapp python app.py
c18177c0e1ec3a40c175cec0b7c165d8e0ff9576087c1734293e890357152919

//查看容器，注意端口映射栏
[root@xxx ~]# docker ps -a
CONTAINER ID        IMAGE               COMMAND                  CREATED             STATUS                      PORTS                                          NAMES
c18177c0e1ec        training/webapp     "python app.py"          3 seconds ago       Up 2 seconds                0.0.0.0:5000->5000/tcp, 0.0.0.0:8080->80/tcp   suspicious_swartz

```

### 3、映射到指定地址的指定端口

```

//只有自己访问自己
[root@xxx ~]# docker run -d -p 127.0.0.1:5000:5000 training/webapp python app.py 
4da1f3ed9ee27026b9929e2b96ebd422e2bf6ab212b07ab8fbb8339c322fef70

[root@xxx ~]# docker ps -a
CONTAINER ID        IMAGE               COMMAND                  CREATED             STATUS                      PORTS                      NAMES
4da1f3ed9ee2        training/webapp     "python app.py"          4 seconds ago       Up 4 seconds                127.0.0.1:5000->5000/tcp   affectionate_elion

```

### 4、映射到指定地址的任意端口
```

//使用IP::ContainerPort绑定localhost的任意端口到容器的5000端口，本地主机会自动分配一个端口
[root@xxx ~]# docker run -d -p 127.0.0.1::5000 training/webapp python app.py
ed9497ef017e2e90fac7e783c92ccde2f59c14f62d429190cb24c0dfa43eeefb

[root@xxx ~]# docker ps -a
CONTAINER ID        IMAGE               COMMAND                  CREATED             STATUS                         PORTS                       NAMES
ed9497ef017e        training/webapp     "python app.py"          5 seconds ago       Up 4 seconds                   127.0.0.1:32768->5000/tcp   wonderful_yonath

//还可以使用udp标记来指定udp端口
[root@xxx ~]# docker run -d -p 5000:5000/udp training/webapp python app.py
4880a7d119f226591ad1b99ad0324d55d8e2caa98a399c9f426e6757fc7491c5
[root@xym ~]# docker ps -a
CONTAINER ID        IMAGE               COMMAND                  CREATED             STATUS                         PORTS                              NAMES
4880a7d119f2        training/webapp     "python app.py"          3 seconds ago       Up 3 seconds                   5000/tcp, 0.0.0.0:5000->5000/udp   gracious_williams
b5257d2e

``` 

### 5、查看映射端口配置
使用`docker port`命令来查看当前映射的端口配置，也可以查看到绑定的地址： 
```

[root@xxx ~]# docker port priceless_franklin 5000
0.0.0.0:5000

```

## 互联机制实现便捷互访
容器的互联（linking）是一种让多个容器中应用进行快速交互的方式。它会在源和接受容器之间创建连接关系，接受容器可以通过容器名快速访问到源容器，而不用指定具体的IP地址。

### 1、自定义容器命名
连接系统依据容器名称来执行。因此，首先需要定义好一个好记的容器名字。虽然当创建容器的时候，系统默认会分配一个名字，但自定义容器名字有两个好处：

* 自定义的名字比较好记，比如一个web应用容器，我们可以给它起个名字叫web ，一目了然。
* 当要连接其他容器时，即便重启，也可以使用容器名而不用改变，比如连接web容器到db容器。

```

//使用--name参数自定义容器名称
[root@xxx ~]# docker run -d -p 5000:5000 --name web training/webapp python app.py
757d2ee95be01e2c509426c52bf5b4176ff7199eb654b5854ddf0e9b8412c044

//查看运行容器，注意NAMES栏
[root@xxx ~]# docker ps -l
CONTAINER ID        IMAGE               COMMAND             CREATED             STATUS              PORTS                    NAMES
757d2ee95be0        training/webapp     "python app.py"     5 seconds ago       Up 4 seconds        0.0.0.0:5000->5000/tcp   web

//还可使用inspect --format "{{.Name}}"获取容器名字
[root@xxx ~]# docker inspect --format "{{.Name}}" 757d2ee95be
/web

```

在执行`docker run`的时候如果添加了`--rm`标记，则容器在终止后会立即删除。注意`--rm`和`-d`参数不能同时使用。

### 2、容器互联
使用`--link`参数可以让容器之间安全地进行交互。
```
//创建一个db容器
[root@xxx ~]# docker run -d --name db training/postgres
Unable to find image 'training/postgres:latest' locally
latest: Pulling from training/postgres
a3ed95caeb02: Pull complete 
6e71c809542e: Pull complete 
2978d9af87ba: Pull complete 
e1bca35b062f: Pull complete 
500b6decf741: Pull complete 
74b14ef2151f: Pull complete 
7afd5ed3826e: Pull complete 
3c69bb244f5e: Pull complete 
d86f9ec5aedf: Pull complete 
010fabf20157: Pull complete 
Digest: sha256:a945dc6dcfbc8d009c3d972931608344b76c2870ce796da00a827bd50791907e
Status: Downloaded newer image for training/postgres:latest
3b48a3a82a86a52244527112a4a03e98e951c8edcdaedb3b63bc1a0775ac0315

//删除原来的web容器
[root@xxx ~]# docker rm -f web
web

//重建web容器，并让它连接到db容器,--link参数的格式为--link name:alias，其中name是要连接的容器名称，alias是这个连接的别名
[root@xxx ~]# docker run -d -P --name web --link db:db training/webapp python app.py
ca82ea2a2e5ad9b407d5c80fcfd6cd01f7e03be46864e5058b539075e858c626

//查看运行容器
[root@xxx ~]# docker ps
CONTAINER ID        IMAGE               COMMAND                  CREATED              STATUS              PORTS                     NAMES
ca82ea2a2e5a        training/webapp     "python app.py"          22 seconds ago       Up 21 seconds       0.0.0.0:32784->5000/tcp   web
3b48a3a82a86        training/postgres   "su postgres -c '/us…"   About a minute ago   Up About a minute   5432/tcp                  db

//查看接受容器连接信息
[root@xxx ~]# docaker inspect --format "{{.HostConfig.Links}}" web
[/db:/web/db]

```
Docker相当于在两个互联的容器之间创建了一个虚机通道，而且不用映射他们的端口在宿主主机上。在启动db容器的时候并没有使用-p和-P标记，从而避免了暴露数据库服务端口到外部网路上。

Docker通过两种方式为容器公开连接信息：

* 更新环境变量
* 更新`/etc/hosts`文件

使用env命令来查看web容器的环境变量：
```

[root@xxx ~]# docker run --rm --name web2 --link db:db training/webapp env
PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
HOSTNAME=6f8232ea8d36
DB_PORT=tcp://172.17.0.3:5432
DB_PORT_5432_TCP=tcp://172.17.0.3:5432
DB_PORT_5432_TCP_ADDR=172.17.0.3
DB_PORT_5432_TCP_PORT=5432
DB_PORT_5432_TCP_PROTO=tcp
DB_NAME=/web2/db
DB_ENV_PG_VERSION=9.3
HOME=/root

```

其中`DB_`开头的环境变量时提供web容器连接db容器使用的，前缀采用大写的连接别名。除了环境变量之外，Docker还添加host信息到子容器的`/etc/hosts`文件，下面是父容器web的hosts文件：

```

//创建容器，并进入bash
[root@xxx ~]# docker run -it --rm --link db:db training/webapp /bin/bash

//查看hosts配置
root@d64fd0fa99f0:/opt/webapp# cat /etc/hosts
172.17.0.3	db 3b48a3a82a86
172.17.0.4	d64fd0fa99f0

//查看db容器，发现其将容器id作为主机名
root@3b48a3a82a86:/# cat /etc/hosts
127.0.0.1	localhost
::1	localhost ip6-localhost ip6-loopback
172.17.0.3	3b48a3a82a86

```

这里有两个hosts信息，第一个是db容器的IP、主机名和容器id，第二个是web容器，web容器使用自己的id作为默认主机名。
```

//安装ping命令
root@d64fd0fa99f0:/opt/webapp# apt-get install inetutils-ping
Unpacking inetutils-ping (2:1.9.2-1) ...
Setting up inetutils-ping (2:1.9.2-1) ...

//执行ping命令，测试与db容器的连通性
root@d64fd0fa99f0:/opt/webapp# ping db
PING db (172.17.0.3): 56 data bytes
64 bytes from 172.17.0.3: icmp_seq=0 ttl=64 time=0.267 ms
64 bytes from 172.17.0.3: icmp_seq=1 ttl=64 time=0.314 ms
64 bytes from 172.17.0.3: icmp_seq=2 ttl=64 time=0.260 ms
64 bytes from 172.17.0.3: icmp_seq=3 ttl=64 time=0.131 ms

```