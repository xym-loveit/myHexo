---
title: Docker 数据管理
date: 2018-04-15 19:04:53
categories: Docker系列
tags: [docker数据管理,-v选项,--volumes-from选项,数据卷备份,数据卷恢复]
description: Docker入门指南，通过数据卷（-v参数）和数据卷容器（--volumes-from参数）来做数据管理（使用数据卷容器做数据卷的备份和恢复）。
---
在生产环境中使用Docker过程中，往往需要对数据进行持久化，或者需要在多个容器之间进行数据共享，这必然涉及到容器的数据管理操作。

容器中管理数据主要有两种方式：

数据卷（Data volumes）：容器内数据直接映射到本地主机环境；
数据卷容器（Data Volume Containers）：使用特定容器维护数据卷；

本文首先介绍如果在容器内创建数据卷，并且把本地的目录或文件挂载到容器内的数据卷中。接下来，会介绍如何使用数据卷容器，在容器和主机、容器和容器之间共享数据，并实现数据的备份和恢复。

## 数据卷
数据卷是一个可供容器使用的特殊目录，它将主机操作系统目录直接映射进容器，类似于linux中的mount操作。

数据卷可以提供很多有用的特性：  
* 数据卷可以在容器之间共享和重用，容器间传递数据将变得高效方便
* 对数据卷内数据的修改会立马生效，无论是容器内操作还是本机操作
* 对数据卷的更新不会影响镜像，解耦了应用和数据
* 卷会一直存在，直到没有容器使用，可以安全的卸载它

### 1、在容器内创建一个数据卷
在用`docker run`命令的时候，使用-v标记可以在容器内创建一个数据卷。多次重复使用-v标记可以创建多个数据卷。

```

//使用training/webapp镜像创建一个web（name参数指定容器名称）容器，并创建一个数据卷挂载到容器的/webapp目录（-v 指定创建的数据卷）
// -P参数是将容器服务暴露的端口自动映射到本地主机的临时端口上，python app.py(为执行的命令COMMAND和其对应的参数ARG)
[root@xxx ~]# docker run -d -P --name web -v /webapp training/webapp python app.py
Unable to find image 'training/webapp:latest' locally
latest: Pulling from training/webapp
e190868d63f8: Pull complete 
909cd34c6fd7: Pull complete 
0b9bfabab7c1: Pull complete 
a3ed95caeb02: Pull complete 
10bbbc0fc0ff: Pull complete 
fca59b508e9f: Pull complete 
e7ae2541b15b: Pull complete 
9dd97ef58ce9: Pull complete 
a4c1b0cb7af7: Pull complete 
Digest: sha256:06e9c1983bd6d5db5fba376ccd63bfa529e8d02f23d5079b8f74a616308fb11d
Status: Downloaded newer image for training/webapp:latest
e7816725b3d075b56410c9d64543a4febfe965e9a5d7cc1c8ea82c92c966f030

//查看镜像
[root@xxx ~]# docker images
REPOSITORY                                        TAG                 IMAGE ID            CREATED             SIZE
hub.c.163.com/public/ubuntu                       14.04               2fe5c4bba1f9        2 years ago         237MB
training/webapp                                   latest              6fae60ef3446        2 years ago         349MB  

//查看运行态的容器
[root@xxx ~]# docker ps
CONTAINER ID        IMAGE               COMMAND             CREATED             STATUS              PORTS                     NAMES
e7816725b3d0        training/webapp     "python app.py"     21 seconds ago      Up 20 seconds       0.0.0.0:32768->5000/tcp   web


```

### 2、挂载一个主机目录作为数据卷
使用-v标记也可以指定挂载一个本地的已有目录到容器中去作为数据卷（推荐方式）。

```

//使用-v /src/webapp:/opt/webapp 加载主机/src/webapp目录到容器的/opt/webapp目录，python为command命令，app.py为运行参数
[root@xxx webapp]# docker run -d -P --name web -v /src/webapp:/opt/webapp training/webapp python app.py
1e0d2372472f8697a65c7da879a8807cf048bc4601b7136e354e4b0c87a7126f

//查看运行的容器
[root@xym webapp]# docker ps
CONTAINER ID        IMAGE               COMMAND             CREATED             STATUS              PORTS                     NAMES
1e0d2372472f        training/webapp     "python app.py"     4 seconds ago       Up 3 seconds        0.0.0.0:32781->5000/tcp   web

```
这个功能在进行测试的时候非常方便，比如用户可以将一些程序或数据放到本地目录中，然后在容器中运行和使用。另外，本地目录的路径必须是绝对路径，如果目录不存在,Docker会自动创建

Docker挂载数据卷的默认权限是读写（rw），用户也可以通过`ro`指定为只读。加了`ro`之后，容器内对所挂载数据卷内的数据就无法修改了。

```

[root@xym ~]# docker run -d -P --name web -v /src/webapp:/opt/webapp:ro training/webapp python app.py
f5d7d04aa46c50db6696efa8554a9d344bbd3f13eb077be3c680a1ac89d509a0

```

### 3、挂载一个本地主机文件作为数据卷

`-v`参数，也可以从主机挂载单个文件到容器中作为数据卷（不推荐）。
  
```
//使用-v ~/.bash_history:/.bash_history，这样就可以记录在容器中输入过的命令历史了

[root@xxx ~]# docker run -it -v ~/.bash_history:/.bash_history ubuntu /bin/bash
root@43e50ea02e35:/# ls
bin  boot  dev  etc  home  lib  lib64  media  mnt  opt  proc  root  run  sbin  srv  sys  tmp  usr  var
root@43e50ea02e35:/# history 
    1  ls
    2  history 

```
挂载文件引起的问题：使用文件编辑工具，包括`vi`或者`sed --in-place`的时候，可能会造成文件`inode`的改变，从Docker1.1.0起，这会导致报错误信息。所以推荐的方式是直接挂载文件所在目录。

## 数据卷容器
如果用户需要在多个容器之间共享一些持续更新的数据，最简单的方式是使用数据卷容器。数据卷容器也是一个容器，但是它的目的是专门用来提供数据卷供其他容器挂载。

首先创建一个数据卷容器`dbdata`,并在其中创建一个数据卷挂载到`/dbdata`。
```
//创建一个数据卷容器
[root@xxx ~]# docker run -it -v /dbdata --name dbdata ubuntu

//查看目录
root@d4bb57243d45:/# ls
bin  boot  dbdata  dev  etc  home  lib  lib64  media  mnt  opt  proc  root  run  sbin  srv  sys  tmp  usr  var

```

然后，可以在其他容器中使用`--volumes-from`来挂载`dbdata`容器中的数据卷，例如创建`db1`和`db2`两个容器，并从`dbdata`容器挂载数据卷：

```
// 创建2个容器挂载dbdata容器中的数据卷
[root@xxx ~]# docker run -it --volumes-from dbdata --name db1 ubuntu
[root@xxx ~]# docker run -it --volumes-from dbdata --name db2 ubuntu

```
此时，容器db1和容器db2都挂载同一个数据卷到相同的`dbdata`目录。三个容器任何一方在该目录下的写入，其他容器都可以看到。例如，在`dbdata`容器中创建一个`test`文件,在`db1`容器中可能查看到它：  

```

//在dbdata容器的数据卷中创建文件a
root@9e90f695bcb8:/dbdata# touch a
root@9e90f695bcb8:/dbdata# ll
total 4
drwxr-xr-x.  2 root root   14 Apr 14 10:02 ./
drwxr-xr-x. 22 root root 4096 Apr 14 10:01 ../
-rw-r--r--.  1 root root    0 Apr 14 10:02 a

//在db1容器中查看数据卷目录dbdata，也发现了文件a
root@ab4426a23cb4:/dbdata# ll
total 4
drwxr-xr-x.  2 root root    6 Apr 14 10:01 ./
drwxr-xr-x. 22 root root 4096 Apr 14 10:02 ../
root@ab4426a23cb4:/dbdata# ls     
a

//db2结果和db1一样
root@b4bea0f56613:/# cd dbdata/
root@b4bea0f56613:/dbdata# ll
total 4
drwxr-xr-x.  2 root root   14 Apr 14 10:02 ./
drwxr-xr-x. 22 root root 4096 Apr 14 10:03 ../
-rw-r--r--.  1 root root    0 Apr 14 10:02 a


```
可以多次使用`--volumes-from`参数来从多个容器挂载多个数据卷。还可以从其他已经挂载了容器卷的容器来挂载数据卷。  

使用`--volumes-from`参数所挂载数据卷的容器自身并不需要保持在运行状态。如果删除了挂载容器（包括dbdata、db1和bd2），数据卷并不会被自动删除。如果要删除一个数据卷，必须在删除最后一个还挂载着它的容器时显示使用`docker rm -v`命令来指定同时删除关联的数据卷。

## 利用数据卷容器迁移数据

可以使用数据卷容器对其中的数据卷进行备份、恢复、以实现数据的迁移。

### 1、备份
使用如下命令来备份`dbdata`数据卷容器内的数据卷：

```

[root@xym ~]# docker run --volumes-from dbdata -v $(pwd):/backup --name worker ubuntu tar -zcvf /backup/backup.tar /dbdata
tar: Removing leading `/' from member names
/dbdata/
/dbdata/a

[root@xym ~]# 

```
分析命令：  

* 利用ubuntu镜像创建一个容器worker。对应命令参数：`docker run --name worker ubuntu`
* 使用`--volumes-from dbdata`参数来让`worker`容器挂载`dbdata`容器的数据卷(即dbdata)。对应命令参数：`--volumes-from dbdata`
* 使用 `-v $(pwd):/backup`参数来挂载本地的当前目录到`woker`容器的`/backup`目录。对应命令参数：`-v $(pwd):/backup`
* `worker`容器启动后，使用`tar -zcvf /backup/backup.tar /dbdata`命令来将`/dbdata`下内容备份为容器内的`/backup/backup.tar`,即宿主主机当前目录下的`backup.tar`。

### 2、恢复
如果要将数据恢复到一个容器，可以按照下面步骤操作。首先创建一个带有数据卷的容器`dbdata2`。

```
docker run -v /dbdata --name dbdata2 ubuntu /bin/bash

```

然后创建另一个新的容器，挂载`dbdata2`的容器，并使用`untar`解压备份文件到所挂载的容器中：

```

docker run --volumes-from dbdata2 -v $(pwd):/backup busybox untar /backup/backup.tar
 
```






















