---
title: Docker镜像构建文件Dockerfile及相关命令介绍
date: 2018-04-19 15:22:20
categories: Docker系列
tags: [Dockerfile命令,Dockerfile命令详解,FROM命令,RUN命令,CMD命令,ADD命令,COPY命令,ENTRYPOINT命令,ENV命令,ONBUILD命令,ARG命令,WORKDIR命令，LABEL命令,EXPOSE命令,VOLUME命令,USER命令]
description: Dockerfile镜像构建文件命令详解，方便查阅。
---

使用`docker build`命令或使用`Docker Hub`的自动构建功能构建Docker镜像时，都需要一个`Dockerfile`文件。`Dockerfile`文件是一个由一系列构建指令组成的文本文件，`docker build`命令会根据这些构建指令完成`Docker`镜像的构建。本文将会介绍`Dockerfile`文件，及其中使用的构建指令。

## Dockerfile文件使用

`docker build`命令会根据`Dockerfile`文件及上下文构建新`Docker镜像`。构建上下文是指Dockerfile所在的本地路径或一个URL（Git仓库地址）。构建上下文环境会被递归处理，所以，构建所指定的路径还包括了子目录，而URL还包括了其中指定的子模块。

### 构建镜像
将当前目录做为构建上下文时，可以像下面这样使用docker build命令构建镜像：
```
$ docker build .
Sending build context to Docker daemon  6.51 MB
...

```

说明：构建会在Docker后台守护进程（daemon）中执行，而不是`CLI`中。构建前，构建进程会将全部内容（递归）发送到守护进程。大多情况下，应该将一个空目录作为构建上下文环境，并将`Dockerfile`文件放在该目录下。

在构建上下文中使用的`Dockerfile`文件，是一个构建指令文件。为了提高构建性能，可以通过`.dockerignore`文件排除上下文目录下，不需要的文件和目录。

`Dockerfile`一般位于构建上下文的根目录下，也可以通过`-f`指定该文件的位置：

```
$ docker build -f /path/to/a/Dockerfile .

```

构建时，还可以通过-t参数指定构建成后，镜像的仓库、标签等：

### 镜像标签
```
$ docker build -t shykes/myapp .

```

如果存在多个仓库下，或使用多个镜像标签，就可以使用多个`-t`参数：

```
$ docker build -t shykes/myapp:1.0.2 -t shykes/myapp:latest .

```

在Docker守护进程执行`Dockerfile`中的指令前，首先会对`Dockerfile`进行语法检查，有语法错误时会返回：

```
$ docker build -t test/myapp .
Sending build context to Docker daemon 2.048 kB
Error response from daemon: Unknown instruction: RUNCMD

```

### 缓存

Docker 守护进程会一条一条的执行`Dockerfile`中的指令，而且会在每一步提交并生成一个新镜像，最后会输出最终镜像的ID。生成完成后，Docker 守护进程会自动清理你发送的上下文。

`Dockerfile`文件中的每条指令会被独立执行，并会创建一个新镜像，`RUN cd /tmp`等命令不会对下条指令产生影响。

Docker 会重用已生成的中间镜像，以加速`docker build`的构建速度。以下是一个使用了缓存镜像的执行过程：

```
$ docker build -t svendowideit/ambassador .
Sending build context to Docker daemon 15.36 kB
Step 1/4 : FROM alpine:3.2
 ---> 31f630c65071
Step 2/4 : MAINTAINER SvenDowideit@home.org.au
 ---> Using cache
 ---> 2a1c91448f5f
Step 3/4 : RUN apk update &&      apk add socat &&        rm -r /var/cache/
 ---> Using cache
 ---> 21ed6e7fbb73
Step 4/4 : CMD env | grep _TCP= | (sed 's/.*_PORT_\([0-9]*\)_TCP=tcp:\/\/\(.*\):\(.*\)/socat -t 100000000 TCP4-LISTEN:\1,fork,reuseaddr TCP4:\2:\3 \&/' && echo wait) | sh
 ---> Using cache
 ---> 7ea8aef582cc
Successfully built 7ea8aef582cc

```

构建缓存仅会使用本地父生成链上的镜像。如果不想使用本地缓存的镜像，也可以通过`--cache-from`指定缓存。指定后将再不使用本地生成的镜像链，而是从镜像仓库中下载。

### 寻找缓存的逻辑

Docker 寻找缓存的逻辑其实就是树型结构根据 `Dockerfile` 指令遍历子节点的过程。下图可以说明这个逻辑。

```

 FROM base_image:version           Dockerfile:
           +----------+                FROM base_image:version
           |base image|                RUN cmd1  --> use cache because we found base image
           +-----X----+                RUN cmd11 --> use cache because we found cmd1
                / \
               /   \
       RUN cmd1     RUN cmd2           Dockerfile:
       +------+     +------+           FROM base_image:version
       |image1|     |image2|           RUN cmd2  --> use cache because we found base image
       +---X--+     +------+           RUN cmd21 --> not use cache because there's no child node
          / \                                        running cmd21, so we build a new image here
         /   \
RUN cmd11     RUN cmd12
+-------+     +-------+
|image11|     |image12|
+-------+     +-------+

```

大部分指令可以根据上述逻辑去寻找缓存，除了 `ADD` 和 `COPY` 。这两个指令会复制文件内容到镜像内，除了指令相同以外，Docker 还会检查每个文件内容校验(不包括最后修改时间和最后访问时间)，如果校验不一致，则不会使用缓存。

除了这两个命令，Docker 并不会去检查容器内的文件内容，比如 `RUN apt-get -y update`，每次执行时文件可能都不一样，但是 Docker 认为命令一致，会继续使用缓存。这样一来，以后构建时都不会再重新运行`apt-get -y update`。

如果 Docker 没有找到当前指令的缓存，则会构建一个新的镜像，并且之后的所有指令都不会再去寻找缓存。

## Dockerfile文件格式

```
# Comment
INSTRUCTION arguments

```

```

# 注释
指令 参数

```

`Dockerfile`文件中指令不区分大小写，但为了更易区分，约定使用**大写**形式。

`Docker` 会依次执行`Dockerfile`中的指令，**文件中的第一条指令必须是FROM，FROM指令用于指定一个基础镜像**。

以#开头的行，Docker会认为是注释。但#出现在指令参数中时，则不是注释。如：

```

# Comment
RUN echo 'we are running some # of cool things'

```

## Dockerfile中使用指令

### FROM

`FROM`指令用于指定其后构建新镜像所使用的基础镜像。FROM指令必是`Dockerfile`文件中的首条命令，启动构建流程后，Docker将会基于该镜像构建新镜像，`FROM`后的命令也会基于这个基础镜像。

`FROM`语法格式为：
```
FROM <image>

```
或
```
FROM <image>:<tag>

```
或
```
FROM <image>:<digest>

```

通过`FROM`指定的镜像，可以是任何有效的基础镜像。FROM有以下限制：

* FROM必须是Dockerfile中第一条非注释命令
* 在一个Dockerfile文件中创建多个镜像时，FROM可以多次出现。只需在每个新命令FROM之前，记录提交上次的镜像ID。
* tag或digest是可选的，如果不使用这两个值时，会使用latest版本的基础镜像

### RUN

`RUN`用于在镜像容器中执行命令，其有以下两种命令执行方式：

#### shell执行
在这种方式会在`shell`中执行命令，Linux下默认使用`/bin/sh -c`，Windows下使用`cmd /S /C`。

注意：通过`SHELL`命令修改RUN所使用的默认shell
```
RUN <command>

```
#### exec执行
```
RUN ["executable", "param1", "param2"]

```
`RUN`可以执行任何命令，然后在当前镜像上创建一个新层并提交。提交后的结果镜像将会用在`Dockerfile`文件的下一步。

通过`RUN`执行多条命令时，可以通过`\`换行执行：

```
RUN /bin/bash -c 'source $HOME/.bashrc; \
echo $HOME'

```

也可以在同一行中，通过分号分隔命令：
```
RUN /bin/bash -c 'source $HOME/.bashrc; echo $HOME'

```
`RUN`指令创建的中间镜像会被缓存，并会在下次构建中使用。如果不想使用这些缓存镜像，可以在构建时指定`--no-cache`参数，如：`docker build --no-cache`。

### CMD
`CMD`用于指定在容器启动时所要执行的命令。`CMD`有以下三种格式：
```
CMD ["executable","param1","param2"]
CMD ["param1","param2"]
CMD command param1 param2

```
`CMD`不同于`RUN`，`CMD`用于指定在容器启动时所要执行的命令，而`RUN`用于指定镜像构建时所要执行的命令。

`CMD`与`RUN`在功能实现上也有相似之处。如：
```

docker run -t -i itbilu/static_web_server /bin/true

```
等价于：

cmd ["/bin/true"]

CMD在Dockerfile文件中仅可指定一次，指定多次时，会覆盖前的指令。

另外，`docker run`命令也会覆盖`Dockerfile`中CMD命令。如果`docker run`运行容器时，使用了`Dockerfile`中CMD相同的命令，就会覆盖`Dockerfile`中的CMD命令。

如，我们在构建镜像的`Dockerfile`文件中使用了如下指令：
```
CMD ["/bin/bash"]

```
使用`docker build`构建一个新镜像，镜像名为`itbilu/test`。构建完成后，使用这个镜像运行一个新容器，运行效果如下：

```
$ sudo docker run -i -t itbilu/test
root@e3597c81aef4:/# 

```
在使用`docker run`运行容器时，我们并没有在命令结尾指定会在容器中执行的命令，这时Docker就会执行在`Dockerfile`的CMD中指定的命令。

如果不想使用CMD中指定的命令，就可以在`docker run`命令的结尾指定所要运行的命令：
```
$ sudo docker run -i -t itbilu/test /bin/ps
  PID TTY          TIME CMD
    1 ?        00:00:00 ps

```

这时，docker run结尾指定的`/bin/ps`命令覆盖了`Dockerfile`的CMD中指定的命令。

### ENTRYPOINT
`ENTRYPOINT`用于给容器配置一个可执行程序。也就是说，每次使用镜像创建容器时，通过`ENTRYPOINT`指定的程序都会被设置为默认程序。`ENTRYPOINT`有以下两种形式：

```
ENTRYPOINT ["executable", "param1", "param2"]
ENTRYPOINT command param1 param2

```
`ENTRYPOINT`与`CMD`非常类似，不同的是通过`docker run`执行的命令不会覆盖`ENTRYPOINT`，而`docker run`命令中指定的任何参数，都会被当做参数再次传递给`ENTRYPOINT`。`Dockerfile`中只允许有一个`ENTRYPOINT`命令，多指定时会覆盖前面的设置，而只执行最后的`ENTRYPOINT`指令。

`docker run`运行容器时指定的参数都会被传递给`ENTRYPOINT`，且会覆盖CMD命令指定的参数。如，执行`docker run <image> -d`时，`-d`参数将被传递给入口点。

也可以通过`docker run --entrypoint`重写`ENTRYPOINT`入口点。

如：可以像下面这样指定一个容器执行程序：
```
ENTRYPOINT ["/usr/bin/nginx"]
```
完整构建代码：

```
# Version: 0.0.3
FROM ubuntu:16.04
MAINTAINER 何民三 "cn.liuht@gmail.com"
RUN apt-get update
RUN apt-get install -y nginx
RUN echo 'Hello World, 我是个容器' \ 
   > /var/www/html/index.html
ENTRYPOINT ["/usr/sbin/nginx"]
EXPOSE 8

```
使用`docker build`构建镜像，并将镜像指定为`itbilu/test`：
```
$ sudo docker build -t="itbilu/test" .

```
构建完成后，使用`itbilu/test`启动一个容器：
```
$ sudo docker run -i -t  itbilu/test -g "daemon off;"

```
在运行容器时，我们使用了`-g "daemon off;"`，这个参数将会被传递给`ENTRYPOINT`，最终在容器中执行的命令为`/usr/sbin/nginx -g "daemon off;"`。

### LABEL

`LABEL`用于为镜像添加元数据，元数以键值对的形式指定：
```
LABEL <key>=<value> <key>=<value> <key>=<value> ...

```
使用`LABEL`指定元数据时，一条`LABEL`指定可以指定一或多条元数据，指定多条元数据时不同元数据之间通过空格分隔。推荐将所有的元数据通过一条`LABEL`指令指定，以免生成过多的中间镜像。

如，通过`LABEL`指定一些元数据：
```
LABEL version="1.0" description="这是一个Web服务器" by="IT笔录"

```
指定后可以通过`docker inspect`查看：
```
$sudo docker inspect itbilu/test
"Labels": {
    "version": "1.0",
    "description": "这是一个Web服务器",
    "by": "IT笔录"
},

```
注意：`Dockerfile`中还有个`MAINTAINER`命令，该命令用于指定镜像作者。但`MAINTAINER`并不推荐使用，更推荐使用`LABEL`来指定镜像作者。如：
```
LABEL maintainer="itbilu.com"

```

### EXPOSE
`EXPOSE`用于指定容器在运行时监听的端口：
```
EXPOSE <port> [<port>...]

```
`EXPOSE`并不会让容器的端口访问到主机。要使其可访问，需要在`docker run`运行容器时通过`-p`来发布这些端口，或通过`-P`参数来发布`EXPOSE`导出的所有端口。

### ENV
`ENV`用于设置环境变量，其有以下两种设置形式：
```
ENV <key> <value>
ENV <key>=<value> ...

```
如，通过`ENV`设置一个环境变量：
```
ENV ITBILU_PATH /home/itbilu/

```
设置后，这个环境变量在`ENV`命令后都可以使用。如：
```
WORKERDIR $ITBILU_PATH

```
这些环境变量不仅可以构建镜像过程使用，使用该镜像创建的容器中也可以使用。如：
```
$ docker run -i -t  itbilu/test 
root@196ca123c0c3:/# cd $ITBILU_PATH
root@196ca123c0c3:/home/itbilu# 

```
### ADD
`ADD`用于复制构建环境中的文件或目录到镜像中。其有以下两种使用方式：
```
ADD <src>... <dest>
ADD ["<src>",... "<dest>"]

```
通过`ADD`复制文件时，需要通过`<src>`指定源文件位置，并通过`<dest>`来指定目标位置。`<src>`可以是一个构建上下文中的文件或目录，也可以是一个`URL`，但不能访问构建上下文之外的文件或目录。

如，通过`ADD`复制一个网络文件：
```
ADD http://wordpress.org/latest.zip $ITBILU_PATH

```
在上例中，`$ITBILU_PATH`是我们使用`ENV`指定的一个环境变量。

另外，如果使用的是本地归档文件（`gzip、bzip2、xz`）时，Docker会自动进行解包操作，类似使用`tar -x`。

### COPY
`COPY`同样用于复制构建环境中的文件或目录到镜像中。其有以下两种使用方式：
```
COPY <src>... <dest>
COPY ["<src>",... "<dest>"]

```
`COPY`指令非常类似于`ADD`，不同点在于`COPY`只会复制构建目录下的文件，不能使用`URL`也不会进行解压操作。

### VOLUME
`VOLUME`用于创建挂载点，即向基于所构建镜像创始的容器添加卷：
```
VOLUME ["/data"]

```
一个卷可以存在于一个或多个容器的指定目录，该目录可以绕过联合文件系统，并具有以下功能：
* 卷可以容器间共享和重用
* 容器并不一定要和其它容器共享卷
* 修改卷后会立即生效
* 对卷的修改不会对镜像产生影响
* 卷会一直存在，直到没有任何容器在使用它

`VOLUME`让我们可以将源代码、数据或其它内容添加到镜像中，而又不并提交到镜像中，并使我们可以多个容器间共享这些内容。

如，通过`VOLUME`创建一个挂载点：
```
ENV ITBILU_PATH /home/itbilu/
VOLUME [$ITBILU_PATH]

```
构建的镜像，并指定镜像名为`itbilu/test`。构建镜像后，使用新构建的运行一个容器。运行容器时，需`-v`参将能本地目录绑定到容器的卷（挂载点）上，以使容器可以访问宿主机的数据。
```
$ sudo docker run -i -t -v ~/code/itbilu:/home/itbilu/  itbilu/test 
root@31b0fac536c4:/# cd /home/itbilu/
root@31b0fac536c4:/home/itbilu# ls
README.md  app.js  bin  config.js  controller  db  demo  document  lib  minify.js  node_modules  package.json  public  routes  test  views

```
如上所示，我们已经可以容器的`/home/itbilu/`目录下访问到宿主机`~/code/itbilu`目录下的数据了。

### USER
`USER`用于指定运行镜像所使用的用户：
```
USER daemon

```
使用`USER`指定用户时，可以使用用户名、UID或GID，或是两者的组合。以下都是合法的指定试：
```
USER user
USER user:group
USER uid
USER uid:gid
USER user:gid
USER uid:group

```
使用`USER`指定用户后，`Dockerfile`中其后的命令`RUN`、`CMD`、`ENTRYPOINT`都将使用该用户。镜像构建完成后，通过`docker run`运行容器时，可以通过`-u`参数来覆盖所指定的用户。

### WORKDIR
`WORKDIR`用于在容器内设置一个工作目录：
```
WORKDIR /path/to/workdir

```
通过`WORKDIR`设置工作目录后，`Dockerfile`中其后的命令`RUN`、`CMD`、`ENTRYPOINT`、`ADD`、`COPY`等命令都会在该目录下执行。

如，使用`WORKDIR`设置工作目录：
```
WORKDIR /a
WORKDIR b
WORKDIR c
RUN pwd

```
在以上示例中，`pwd`最终将会在`/a/b/c`目录中执行。

在使用`docker run`运行容器时，可以通过`-w`参数覆盖构建时所设置的工作目录。

### ARG
`ARG`用于指定传递给构建运行时的变量：
```
ARG <name>[=<default value>]

```
如，通过`ARG`指定两个变量：
```
ARG site
ARG build_user=IT笔录

```
以上我们指定了`site`和`build_user`两个变量，其中`build_user`指定了默认值。在使用`docker build`构建镜像时，可以通过`--build-arg <varname>=<value>`参数来指定或重设置这些变量的值。
```
$ sudo docker build --build-arg site=itiblu.com -t itbilu/test .


```
这样我们构建了`itbilu/test`镜像，其中`site`会被设置为`itbilu.com`，由于没有指定`build_user`，其值将是默认值IT笔录。

### ONBUILD
`ONBUILD`用于设置镜像触发器：
```
ONBUILD [INSTRUCTION]

```
当所构建的镜像被用做其它镜像的基础镜像，该镜像中的触发器将会被钥触发。

如，当镜像被使用时，可能需要做一些处理：
```
[...]
ONBUILD ADD . /app/src
ONBUILD RUN /usr/local/bin/python-build --dir /app/src
[...]

```

### STOPSIGNAL
`STOPSIGNAL`用于设置停止容器所要发送的系统调用信号：
```
STOPSIGNAL signal

```
所使用的信号必须是内核系统调用表中的合法的值，如：`9`、`SIGKILL`。

### SHELL
`SHELL`用于设置执行命令（`shell`式）所使用的的默认shell类型：
```
SHELL ["executable", "parameters"]

```
`SHELL`在Windows环境下比较有用，Windows下通常会有cmd和powershell两种shell，可能还会有sh。这时就可以通过SHELL来指定所使用的shell类型：

```
FROM microsoft/windowsservercore

# Executed as cmd /S /C echo default
RUN echo default

# Executed as cmd /S /C powershell -command Write-Host default
RUN powershell -command Write-Host default

# Executed as powershell -command Write-Host hello
SHELL ["powershell", "-command"]
RUN Write-Host hello

# Executed as cmd /S /C echo hello
SHELL ["cmd", "/S"", "/C"]
RUN echo hello

```

> 原文链接：[https://itbilu.com/linux/docker/VyhM5wPuz.html](https://itbilu.com/linux/docker/VyhM5wPuz.html)