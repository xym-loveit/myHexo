---
title: Docker 基本操作之容器
date: 2018-04-13 19:42:56
categories: Docker系列
tags: [docker容器,create命令,start命令,run命令,stop命令,restart命令,attach命令,exec命令,rm命令,export命令,import命令]
description: Docker入门指南，容器各种操作命令。
---
容器是Docker的另一个核心概念。简单来说，容器时镜像的一个运行实例。所不同的是，镜像是静态只读文件，而容器带有运行时需要的可写文件层。如果认为虚拟机是模拟运行的一整套操作系统（包括内核、应用运行环境和其他系统环境）和跑在上面的应用，那么Docker容器就是独立运行的一个（或一组）应用，以及它们必须的运行环境。

## 创建容器

### 1、新建容器

命令格式：`docker create [OPTIONS] IMAGE [COMMAND] [ARG...]`

```
Options:
      --add-host list                  Add a custom host-to-IP mapping (host:ip)
  -a, --attach list                    Attach to STDIN, STDOUT or STDERR
      --blkio-weight uint16            Block IO (relative weight), between 10 and 1000, or 0 to disable (default 0)
      --blkio-weight-device list       Block IO weight (relative device weight) (default [])
      --cap-add list                   Add Linux capabilities
      ...
      
```
Create命令和后续的run命令支持的选项都十分复杂，主要包括如下几大类：与容器运行模式相关、与容器和环境配置相关、与容器资源限制和安全保护相关。

Create命令与容器运行模式相关的选项见下图：

![Create命令与容器运行模式相关的选项](http://op7wplti1.bkt.clouddn.com/1900654235_4b1-i_epub.jpg)

Create命令与容器环境和配置相关选项如下图：

![Create命令与容器环境和配置相关选项1](http://op7wplti1.bkt.clouddn.com/1900654235_4b2-i_epub.jpg)

![Create命令与容器环境和配置相关选项2](http://op7wplti1.bkt.clouddn.com/1900654235_4b2x-i_epub.jpg)

Create命令与容器资源限制和安全保护相关选项如下图：

![Create命令与容器资源限制和安全保护相关选项1](http://op7wplti1.bkt.clouddn.com/1900654235_4b3-i_epub.jpg)

![Create命令与容器资源限制和安全保护相关选项2](http://op7wplti1.bkt.clouddn.com/1900654235_4b3x-i_epub.jpg)

```

//根据镜像创建一个容器
[root@xxx /]# docker create -it registry.cn-hangzhou.aliyuncs.com/xym/163ubuntu:14.04 
d26cbeecf22d92fdff515a9bb8146521c8e4c6ea76cf7baab5abebb4b31dfc52

//查看docker本地所有容器,注意状态为Created
[root@xxx /]# docker ps -a
CONTAINER ID        IMAGE                                                   COMMAND                  CREATED             STATUS              PORTS               NAMES
d26cbeecf22d        registry.cn-hangzhou.aliyuncs.com/xym/163ubuntu:14.04   "/bin/sh -c '/usr/sb…"   8 seconds ago       Created                                 practical_wright

//使用start启动docker容器
[root@xxx /]# docker start d26
d26

//查看docker本地所有容器,注意状态为Up
[root@xxx /]# docker ps -a
CONTAINER ID        IMAGE                                                   COMMAND                  CREATED             STATUS              PORTS               NAMES
d26cbeecf22d        registry.cn-hangzhou.aliyuncs.com/xym/163ubuntu:14.04   "/bin/sh -c '/usr/sb…"   41 seconds ago      Up 2 seconds                            practical_wright

```
其他比较重要的选项还包括：

```
-l，--label=[]：以键值对方式指定容器的标签信息;

--label-file=[]：从文件读取标签信息。

```

### 2、启动容器
命令格式：`docker start [OPTIONS] CONTAINER [CONTAINER...]`

```

//查看本地所有Docker容器，注意状态为Exited
[root@xxx /]# docker ps -a
CONTAINER ID        IMAGE                                                   COMMAND                  CREATED             STATUS                       PORTS               NAMES
d26cbeecf22d        registry.cn-hangzhou.aliyuncs.com/xym/163ubuntu:14.04   "/bin/sh -c '/usr/sb…"   14 minutes ago      Exited (137) 5 seconds ago                       practical_wright

//使用start启动容器
[root@xxx /]# docker start d26cb
d26cb

//查看本地所有Docker容器，注意状态为Up
[root@xxx /]# docker ps -a
CONTAINER ID        IMAGE                                                   COMMAND                  CREATED             STATUS              PORTS               NAMES
d26cbeecf22d        registry.cn-hangzhou.aliyuncs.com/xym/163ubuntu:14.04   "/bin/sh -c '/usr/sb…"   14 minutes ago      Up 2 seconds  

```

### 3、新建并启动容器

除了创建容器后通过start命令来启动，也可以直接新建并启动容器。所需要的命令主要为`docker run`,等价于先执行`docker create`命令，再执行`docker start`命令。  
例如，下面的命令输出一个“Hello World”，之后容器自动终止：

```

//创建容器并执行一个输出命令
[root@xxx /]# docker run ubuntu:latest /bin/echo "Hello World"
Hello World

//查看本地所有Docker容器，注意状态为Exited
[root@xxx /]# docker ps -a
CONTAINER ID        IMAGE               COMMAND                  CREATED             STATUS                     PORTS               NAMES
3fa47db4c105        ubuntu:latest       "/bin/echo 'Hello Wo…"   8 seconds ago       Exited (0) 7 seconds ago                       elated_goldwasser

```

这跟在本地直接执行`/bin/echo "Hello World"`几乎感觉不出任何区别。当利用`docker run`来创建并启动容器时，Docker在后台运行的标准操作：  

* 检查本地是否存在指定的镜像，不存在就从共有仓库下载；
* 利用镜像创建容器，并启动该容器；
* 分配一个文件系统给容器，并在只读的镜像层外面挂载一层可读写层；
* 从宿主主机配置的网桥接口中桥接一个虚拟接口到容器中；
* 从网桥的地址池配置一个IP地址给容器；
* 执行用户指定的应用程序
* 执行完毕后容器被自动终止

启动一个终端，并允许用户进行交互：  

```

//其中-t选项当Docker分配一个伪终端（pseudo-tty）并绑定到容器的标准输入上，-i则让容器的标准输入保持打开。
[root@xxx /]# docker run -it ubuntu:latest /bin/bash
root@21536661367e:/# pwd
/

//用户可以在交互模式下输入系统命令进行操作
root@21536661367e:/# ls
bin  boot  dev  etc  home  lib  lib64  media  mnt  opt  proc  root  run  sbin  srv  sys  tmp  usr  var

//用户可以在交互模式下输入系统命令进行操作，使用ps可以看到系统只运行了bash应用，并没有运行其他无关的进程
root@21536661367e:/# ps
   PID TTY          TIME CMD
     1 pts/0    00:00:00 bash
    10 pts/0    00:00:00 ps
    
//用户可以输入exit或ctrl+d来退出容器
root@21536661367e:/# exit
exit
[root@xxx /]# 

```
对于所创建的bash容器，当使用exit命令退出之后，容器就自动处于退出（Exited）状态了。这是因为对Docker容器来说，当运行的应用退出后，容器也就没有必要继续运行了。  
某些时候，执行`docker run`会出错，因为命令无法正常执行容器会直接退出，此时可以查看退出的错误代码。  

* 125：Docker daemon执行出错，例如指定了不支持的Docker命令参数；
* 126：所指定的命令无法执行，例如权限出错。
* 127：容器内命令无法找到；

命令执行出错后，会默认返回错误码。

### 4、守护态运行

更多的时候，需要让Docker容器在后台以守护态（Daemonized）形式运行。此时，可以通过添加-d参数来实现。

```

//-d参数以后台模式启动Docker，返回容器id
[root@xxx ~]# docker run -d ubuntu:latest /bin/sh -c "while true;do echo hello world;sleep 1;done"
5d4cfe5b4b6109fd8df8717fcd87790e7692f70d7e8e8d7590ba4c269f6dd717

//通过docker ps查看运行的容器
[root@xxx ~]# docker ps
CONTAINER ID        IMAGE               COMMAND                  CREATED             STATUS              PORTS               NAMES
5d4cfe5b4b61        ubuntu:latest       "/bin/sh -c 'while t…"   13 seconds ago      Up 12 seconds                           thirsty_lewin

//使用docker logs命令获取容器输出信息
[root@xxx ~]# docker logs 5d4c
hello world
hello world
hello world
hello world
hello world
hello world

```
## 终止容器
命令格式：`docker stop [OPTIONS] CONTAINER [CONTAINER...]`
```
Options:
  -t, --time int   Seconds to wait for stop before killing it (default 10)
  
```
原理：首先向容器发送SIGTERM信号,等待一段超时时间（默认10秒）后，再发送SIGKILL信号来终止容器：
```
//终止前
[root@xxx ~]# docker ps
CONTAINER ID        IMAGE               COMMAND                  CREATED             STATUS              PORTS               NAMES
5d4cfe5b4b61        ubuntu:latest       "/bin/sh -c 'while t…"   16 minutes ago      Up 16 minutes                           thirsty_lewin

//发送终止命令
[root@xxx ~]# docker stop 5d4c
5d4c

//终止后状态为Exited
[root@xxx ~]# docker ps -a
CONTAINER ID        IMAGE               COMMAND                  CREATED             STATUS                        PORTS               NAMES
5d4cfe5b4b61        ubuntu:latest       "/bin/sh -c 'while t…"   18 minutes ago      Exited (137) 56 seconds ago                       thirsty_lewin

```

* 使用`docker kill`命令会直接发送SIGKILL信号来强行终止容器。
* 当容器中指定的应用终结时，容器也会自动终止。
* 使用`docker start`可以重新启动处于终止状态的容器。
* 使用`docker restart`命令会将一个运行态的容器先终止，然后再重新启动

```

//重新启动已终止的容器
[root@xxx ~]# docker start 5d4c
5d4c

//查看运行的容器
[root@xxx ~]# docker ps
CONTAINER ID        IMAGE               COMMAND                  CREATED             STATUS              PORTS               NAMES
5d4cfe5b4b61        ubuntu:latest       "/bin/sh -c 'while t…"   25 minutes ago      Up 6 seconds                            thirsty_lewin

//发送SIGKILL信号来强行终止容器
[root@xxx ~]# docker kill 5d4c
5d4c

//查看运行容器
[root@xxx ~]# docker ps
CONTAINER ID        IMAGE               COMMAND             CREATED             STATUS              PORTS               NAMES

//重启容器
[root@xxx ~]# docker restart 5d4c
5d4c

//查看运行容器
[root@xxx ~]# docker ps
CONTAINER ID        IMAGE               COMMAND                  CREATED             STATUS              PORTS               NAMES
5d4cfe5b4b61        ubuntu:latest       "/bin/sh -c 'while t…"   25 minutes ago      Up 1 second                             thirsty_lewin

```
## 进入容器

在使用-d参数时，容器启动后会进入后台，用户无法看到容器中的信息，也无法操作。这个时候如果要进入容器进行操作，有三种方法：

### 1、使用attach命令
命令格式：`docker attach [OPTIONS] CONTAINER`
```
Options:
      --detach-keys string   指定退出attach模式的快捷键序列，默认是ctrl+q ctrl+p;
      --no-stdin             是否关闭标准输入，默认是保持打开
      --sig-proxy            是否代理收到的系统信号给应用进程，默认为true


[root@xxx ~]# docker run -itd ubuntu /bin/bash
3535fd6c5ccab3c4d4737c1385ac18c36bd190c129c70c664fbf8c62a1707e67
[root@xxx ~]# docker attach 3535
root@3535fd6c5cca:/# 
root@3535fd6c5cca:/# ls
bin  boot  dev  etc  home  lib  lib64  media  mnt  opt  proc  root  run  sbin  srv  sys  tmp  usr  var

```
attach命令缺点：当多个窗口同时用attach命令连到同一个容器的时候，所有窗口都会同步显示。当某个窗口因命令阻塞时，其他窗口也无法执行操作了。

### 2、使用exec命令（exec命令用于从外部运行容器内部的命令）

命令格式：`docker exec [OPTIONS] CONTAINER COMMAND [ARG...]`
```
Options:
  -d, --detach               以后台模式运行命令
      --detach-keys string   Override the key sequence for detaching a container
  -e, --env list             设置环境变量
  -i, --interactive          打开标准输入接受用户输入命令，默认为false
      --privileged           Give extended privileges to the command
  -t, --tty                  分配一个伪终端，默认为false
  -u, --user string          执行命令的用户名或ID
  -w, --workdir string       Working directory inside the container


[root@xxx ~]# docker ps
CONTAINER ID        IMAGE               COMMAND                  CREATED             STATUS              PORTS               NAMES
5d4cfe5b4b61        ubuntu:latest       "/bin/sh -c 'while t…"   About an hour ago   Up About a minute                       thirsty_lewin

//使用exec进入后台运行的镜像中
[root@xxx ~]# docker exec -it thirsty_lewin /bin/bash
root@5d4cfe5b4b61:/# 

```
通过以上可以看出，一个bash终端被打开了，在不影响容器内其他应用的前提下，用户可以很容易与容器进行交互。

注意：通过指定`-it`参数来保持标准输入打开，并且分配一个伪终端。通过`exec`命令对容器执行操作是最为推荐的方式。

### 3、使用第三方`nsenter`工具

在`util-linux`软件包版本2.23+中包含`nsenter`工具，如果系统中的`util-linux`包没有该命令，可以按照下面方式从源码安装：

```

$ cd /emp;curl https://mirrors.edge.kernel.org/pub/linux/utils/util-linux/v2.32/util-linux-2.32.tar.gz | tar -zxvf;
cd util-linux-2.32;

$ ./configure --without-ncurses

$ make nsenter && cp nsenter /usr/local/bin

```
为了使用`nsenter` 连接到容器，还需要找到容器PID，可以通过下面的命令获取：

```

//获取容器运行PID
PID=$(docker inspect --format"{% raw %}{{.State.Pid}}{% endraw %}" <container>)

//通过PID，连接到容器：
`nsenter --target $PID --mount --uts --ipc --net --pid`

```

下面使用完整的命令执行该操作：

```

//查看运行容器
[root@xxx ~]# docker ps
CONTAINER ID        IMAGE               COMMAND                  CREATED             STATUS              PORTS               NAMES
5d4cfe5b4b61        ubuntu:latest       "/bin/sh -c 'while t…"   About an hour ago   Up 4 minutes                            thirsty_lewin

//查看运行容器进程PID
[root@xxx ~]# docker inspect --format "{% raw %}{{.State.Pid}}{% endraw %}" thirsty_lewin
18637

//进入容器
[root@xxx ~]# nsenter --target 18637 --mount --uts --ipc --net --pid
mesg: ttyname failed: No such file or directory

//查看用户
root@5d4cfe5b4b61:~# w
 15:22:55 up 15:59,  0 users,  load average: 0.03, 0.04, 0.05
USER     TTY      FROM             LOGIN@   IDLE   JCPU   PCPU WHAT

```

## 删除容器
可以使用docker rm命令来删除处于终止或退出状态的容器。

命令格式：`docker rm [OPTIONS] CONTAINER [CONTAINER...]`

```

Options:
  -f, --force     强制终止并删除一个运行中的容器
  -l, --link      删除容器的连接但保留容器
  -v, --volumes   删除容器挂载的数据卷

//查看运行中的容器
[root@xxx ~]# docker ps
CONTAINER ID        IMAGE               COMMAND                  CREATED             STATUS              PORTS               NAMES
5d4cfe5b4b61        ubuntu:latest       "/bin/sh -c 'while t…"   About an hour ago   Up 14 minutes                           thirsty_lewin

//删除运行状态的容器
[root@xxx ~]# docker rm 5d4c
Error response from daemon: You cannot remove a running container 5d4cfe5b4b6109fd8df8717fcd87790e7692f70d7e8e8d7590ba4c269f6dd717. Stop the container before attempting removal or force remove

//强制删除运行状态的容器
[root@xxx ~]# docker rm -f 5d4c
5d4c

```

## 导入和导出容器
某些时候，需要将容器从一个系统迁移到另外一个系统，此时可以使用docker的导入和导出功能。

### 1、导出容器
导出容器是指导出一个已经创建的容器到一个文件，不管此时这个容器是否处于运行状态。
命令格式：`docker export [OPTIONS] CONTAINER`

```
Options:
  -o, --output string  指定导出的tar归档文件，也可直接通过重定向来实现。
  

//显示所有容器
[root@xxx ~]# docker ps -a
CONTAINER ID        IMAGE               COMMAND                  CREATED             STATUS                    PORTS               NAMES
684d1d0dc403        ubuntu:latest       "/bin/sh -c 'while t…"   19 seconds ago      Up 18 seconds                                 vigorous_brahmagupta
3fa47db4c105        ubuntu:latest       "/bin/echo 'Hello Wo…"   11 hours ago        Exited (0) 11 hours ago                       elated_goldwasser

//将运行中的容器导出tar文件
[root@xx ~]# docker export -o test_for_run.tar 684d

//将已经退出的容器导出tar文件
[root@xxx ~]# docker export 3fa4 > test_for_stop.tar
  
```

之后，可将导出的tar文件传输到其他机器上，然后再通过导入命令导入到系统中，从而实现容器的迁移。

### 2、导入容器
导出的文件又可以使用`docker import`命令导入变成镜像。
命令格式：`docker import [OPTIONS] file|URL|- [REPOSITORY[:TAG]]`

```
Options:
  -c, --change list      在导入的同时执行对容器进行修改的Dockerfile指令
  -m, --message string   Set commit message for imported image

//将tar文件导入系统中
[root@xxx ~]# docker import test_for_run.tar xym/ubuntu:1.0
sha256:f916030e78e9046defa752bfc32a99b96460e098d3ee1cab1a5048150255d27e

[root@xxx ~]# docker images
REPOSITORY                                        TAG                 IMAGE ID            CREATED             SIZE
xym/ubuntu                                        1.0                 f916030e78e9        2 seconds ago       85.8MB
centos-7                                          import              be5e039acd03        19 hours ago        435MB

```
之前镜像章节中介绍过使用`docker load`命令来导入一个镜像文件，与`docker import`命令十分类似。

实际上，既可以使用`docker load`命令来导入镜像存储文件到本地镜像库，也可以使用`docker import`命令来导入一个容器快照到本地镜像库。

这二者的区别在于容器快照文件将丢弃所有历史记录和元数据信息（即仅保存容器当时的快照状态），而镜像存储文件将保存完整记录，体积也更大。此外，从容器快照文件导入时可以重新指定标签。

