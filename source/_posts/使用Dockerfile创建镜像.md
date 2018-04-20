---
title: 使用Dockerfile创建镜像
date: 2018-04-20 10:18:05
categories: Docker系列
tags: [Dockerfile命令]
description: Dockerfile基础介绍。
---

`Dockerfile`是一个文本格式的配置文件，用户可以使用`Dockerfile`来快速创建自定义的镜像。

## 基本结构
`Dockerfile`由一行行命令语句组成，并且支持以`#`开头的注释行。
一般而言，`Dockerfile`分为四部分：基础镜像信息、维护者信息、镜像操作指令和容器启动时执行指令。
```
#一个nginx dockerfile例子
#基础镜像
FROM ubuntu
#维护者信息
MAINTAINER xym xym@126.com
#镜像操作指令
RUN apt-get update && apt-get install -y nginx
RUN echo "daemon off;" > /etc/nginx/nginx.conf
#容器启动时执行指令
CMD /usr/sbin/nginx

```
其中一开始必须指明所基于的镜像名称，接下来一般是说明维护信息。后面则是镜像操作指令，例如RUN指令，RUN指令将对镜像执行跟随的命令。每运行一条RUN指令，镜像就添加新的一层，并提交。最后是CMD指令，用来指定运行容器时的操作命令。

## 指令说明
指令的一般格式为`INSTRUCTION arguments`，指令包括`FROM`、`MAINTAINER`、`RUN`等。

![http://op7wplti1.bkt.clouddn.com/1900654235_8b1-i_epub.jpg](http://op7wplti1.bkt.clouddn.com/1900654235_8b1-i_epub.jpg)

![http://op7wplti1.bkt.clouddn.com/1900654235_8b1x-i_epub.jpg](http://op7wplti1.bkt.clouddn.com/1900654235_8b1x-i_epub.jpg)

### FROM
指定所创建镜像的基础镜像，如果本地不存在，则默认会去`Docker Hub`下载指定镜像。

命令格式：`FROM <image>` or `FROM <image>:<tag>` or `FROM <image>@<digest>`

任何`Dockerfile`中的第一条指令必须为FROM指令。并且，如果在同一个`Dockerfile`中创建多个镜像，可以使用多个FROM指令（每个镜像一次）。 

### MAINTAINER
指定维护者信息，格式为 `MAINTAINER <name>`。例如：
```
MAINTAINER  image_creator@docker.com

```
该信息会写入生成镜像的Author属性域中，可以使用 `docker inspect imageName/imageId`查看。 

### RUN
运行指定命令，格式为：`RUN <command>` 或 `RUN ["executable","param1","param2"]`。注意，后一个指令会被解析为`json`数组，因此必须使用双引号。

前者默认将在`shell`终端中运行命令，即`/bin/sh-c`;后者则使用`exec`执行，不会启动shell环境。
指定使用其他终端类型可以使用第二种方式实现，例如：`RUN ["bin/bash","-c","echo hello"]`。

每条RUN指令将在当前镜像的基础上执行指定指令，并提交为新的镜像。当命令较长时可以使用"\"来换行。

### CMD
CMD指令用来指定启动容器时默认执行的命令。它支持三种格式：
```
//使用exec执行，是推荐的使用方式
CMD ["executable","param1","param2"]

//在/bin/sh中执行，提供给需要交互的应用
CMD command param1 param2

//提供给ENTRYPOINT的默认参数
CMD ["param1","param2"]

```
每个`Dockerfile`只能有一条CMD命令，如果指定了多条命令，只有最后一条会被执行。
如果用户启动容器时手动指定了运行的命令（作为run的参数），则会覆盖掉CMD指定的命令。

### LABEL
LABEL指令用来指定生成镜像的元数据标签信息。
命令格式：`LABEL <key>=<value> <key>=<value>...`
```
LABEL author="xym" author_email=xxx@126.com

```

### EXPOSE
声明镜像内的服务所监听的端口。

命令格式：`EXPOSE <port>[<port>...]`
```
EXPOSE 22 80 8443

```
该命令只是起到声明作用，并不会自动完成端口映射。在启动容器时需要使用`-P`，Docker主机会自动分配一个宿主机的临时端口转发到指定的端口；使用`-p`，则可以具体指定哪个宿主机的本地端口会映射过来。

### ENV
指定环境变量，在镜像生成过程中会被后续RUN指令使用，在镜像启动的容器中也会存在。

命令格式：`ENV <key> <value>` or `ENV <key>=<value>`

指令指定的环境变量在运行中可以被覆盖掉，如`docker run --env <key>=<value> built_image`

### ADD
该命令将复制指定的`<src>`路径下的内容到容器中的`<dest>`路径下。

命令格式：`ADD <src> <dest>`

其中`<src>`可以是`Dockerfile`所在目录的一个相对路径（文件或目录），也可以是一个URL，还可以是一个tar文件（如果为`tar`文件，会自动解压到`<dest>`路径下）。`<dest>`可以是镜像内的路径，或者相对于工作目录（`WORKDIR`）的相对路径。

路径支持正则格式：`ADD *.c /code/`

### COPY
格式为`COPY <src> <dest>`

复制本地主机的`<src>`（为`Dockerfile`所在目录的相对路径、文件或者目录）下的内容到镜像中的`<dest>`下。目标路径不存在时，会自动创建。路径同样支持正则表达式。当使用本地目录为源目录时，推荐使用`COPY`。

### ENTRYPOINT
指定镜像的默认入口命令，该入口命令会在启动容器时作为根命令执行，所有传入值作为该命令的参数。
支持两种格式：
```
ENTRYPOINT ["executable","param1","param2"]（exec调用执行）
ENTRYPOINT command param1 param2（shell中执行）

```
此时，`CMD`指令指定值将作为根命令的参数。
每个Dockerfile中只能有一个`ENTRYPOINT`，当指定多个时，只有最后一个有效。在运行时，可以被`--entrypoint`参数覆盖掉，如`docker run --entrypoint`

### VOLUME
创建一个数据卷挂载点。格式为`VOLUME ["/data"]`，可以从本地主机或其他容器挂载数据卷，一般用来存放数据库和需要保存的数据等。

### USER
指定运行容器时的用户名和UID，后续的RUN等指令也会使用指定的用户身份。
格式为：`USER daemon`

当服务不需要管理员权限时，可以通过该命令指定运行用户，并且可以在之前创建所需要的用户。
```
RUN groupadd -r postgres && useradd -r -g postgres postgres

```
要临时获取管理员权限可以使用`gosu`或`sudo`

### WORKDIR
为后续的`RUN/CMD/ENTRYPOINT`指令配置工作目录。

格式为：`WORKDIR /path/to/workdir`
可以使用多个`WORKDIR`指令，后续命令如果参数是相对路径，后续命令如果是相对路径，则会基于之前命令指定的路径。例如：
```
WORKDIR /a
WORKDIR b
WORKDIR c
RUN pwd

```
则最终路径为`/a/b/c`

### ARG
指定一些镜像内使用的的参数（例如版本号信息等），这些参数在执行`docker build`命令时以`--build-arg<varname>=<value>`格式传入。格式为`ARG <name>=[<default value>]`，则可以用`docker build --build-arg <name>=<value> .`来指定参数值。

### ONBUILD

配置当所创建的镜像作为其他镜像的基础镜像时，所执行的创建操作指令。格式为：`ONBUILD [INSTRUCTION]`

例如：Dockerfile使用了如下的内容创建了镜像`image-A`：

```

[...]
ONBUILD ADD . /app/src
ONBUILD RUN /usr/local/bin/python-build --dir /app/src
[...]

```

如果基于`image-A`创建新的镜像时，新的Dockerfile中使用`FROM image-A`指定基础镜像，会自动执行`ONBUILD`指令的内容，等价于在后面添加了两条指令：

```

FROM image-A
#自动执行onbuild指定的命令
ADD . /app/src
RUN /usr/local/bin/python-build --dir /app/src

```

使用`ONBUILD`指令的镜像，推荐在标签中注明，例如：`ruby:1.9-onbuild`

### STOPSIGNAL
指定所创建镜像启动的容器接受退出的信号值，例如：  
`STOPSIGNAL signal`

### HEALTHCHECK
配置所启动容器如何进行健康检查（如何判断健康与否），自Docker1.12开始支持。

格式有两种：

`HEALTHCHECK [OPTIONS] CMD command`

根据所执行命令返回值是否为0来判断；
`HEALTHCHECK NONE`禁止基础镜像中的健康检查。

OPTIONS支持：

* --interval=DURATION（默认为30s）：过多久检查一次；
* --timeout=DURATION（默认为30s）：每次检查等待结果的超时；
* --retries=N（默认为3）：如果失败了，重试几次才最终确定失败。

### SHELL
指定其他命令使用`shell`时的默认shell类型。默认值为`["/bin/sh","-c"]`

注意：对于`Windows`系统，建议在Dockerfile开头添加`#escape=`来指定转义信息。

## 创建镜像
编写完成`Dockerfile`之后，可以通过`docker build`命令来创建镜像。基本格式为`docker build [选项] 内容路径`，该命令将读取指定路径下（包括子目录）的Dockerfile，并将该路径下的所有内容发送给Docker服务端，由服务端来创建镜像。因此除非生成镜像需要，否则一般建议放置Dockerfile的目录为空目录。有两点经验：

* 如果使用非内容路径下的Dockerfile，可以通过-f选项来指定其路径
* 要指定生成镜像的标签信息，可以使用-t选项

例如，指定Dockerfile所在路径为`/tmp/docker_builder/`,并且希望生成镜像标签为`build_repo/first_image`,可以使用下面命令：  
```
docker build -t build_repo/first_image /tmp/docker_builder/

```
## 使用.dockerignore文件
可以通过`.dockerignore`文件（每一行添加一条匹配模式）来让`Docker`忽略匹配模式路径下的目录和文件。例如：  
```
#comment
*/temp*
*/*/temp*
tmp?
~*

```

## 编写Dockerfile的指导原则
* 精简镜像用途：尽量让每个镜像的用途都比较集中、单一，避免构造大而复杂，多功能的镜像；
* 选用合适的基础镜像：过大的基础镜像会造成生成臃肿的镜像，一般推荐较为小巧的`debian`镜像；
* 提供足够清晰的命令注释和维护者信息：Dockerfile也是一种代码，需要考虑方便后续扩展和他人使用；
* 正确使用版本：使用明确的版本号信息，如：1.0，2.0而非latest，将避免内容不一致可能引发的惨案；
* 减少镜像层数：如果希望所生成镜像的层数尽量少，则要尽量合并指令，例如多个RUN指令可以合并为一条；
* 及时删除临时文件和缓存文件：特别是在执行apt-get指令后，/var/cache/apt/下面会缓存一些安装包；
* 提高生成速度：如合理使用缓存，减少内容目录下的文件，或使用`.dockerignore`文件指定等。
* 调整合理的指令顺序：在开启缓存的情况下，内容不变的指令尽量放在前面，这样可以尽量复用
* 减少外部源的干扰：如果确实要从外部引入数据，需要指定持久地址，并带有版本信息，让他人可以重复而不出错


## 优良的镜像

### BUSYBOX
BusyBox是一个集成了100多个最常见的`Liunx`命令和工具（如：cat/echo/grep/mount/telnet等）的精简工具箱。它只有几MB的大小，很方便进行各种快速验证，被誉为“Linux系统的瑞士军刀”。BusyBox可以运行于多款POSIX环境的操作系统中，如Liunx（包括Andrid）、Hurd、FreeBSD等。

```
//查询busybox镜像
docker search busybox

//下载busybox镜像
docker pull busybox

//使用busybox创建容器
docker run -it --name my_busybox busybox

```

### Alpine
Alpine操作系统是一个面向安全的轻型Linux发行版。Alpine是由非商业组织维护的支持广泛场景的Linux发行版，它特别为资深/重度Liunx用户而优化，关注安全、性能和资源效能。Alpine镜像适用于更多常用场景，并且是一个优秀的可以适用于生产的基础环境。

```
//查询busybox镜像
docker search alpine

//下载busybox镜像
docker pull alpine

//使用busybox创建容器
docker run -it --name my_alpine alpine

```

### Debian/Ubuntu

### CentOS/Fedora

