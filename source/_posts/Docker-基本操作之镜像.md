---
title: Docker 基本操作之镜像
date: 2018-04-13 09:24:54
categories: Docker系列
tags: [docker镜像,pull命令,images命令,tag命令,inspect命令,history命令,search命令,rmi命令,rm命令,commit命令,import命令,save命令,load命令,push命令]
description: Docker入门指南，各镜像操作命令，上传镜像到镜像服务器（阿里云、网易蜂巢等云平台）详细步骤。
---

## 拉取镜像
命令格式：`docker pull [OPTIONS] NAME[:TAG|@DIGEST]`
```
Options:
  -a, --all-tags               是否获取仓库中所有镜像，默认为否
      --disable-content-trust  是否跳过镜像校验，默认为是
```

* 如果不显示指定TAG，默认拉取latest
* 镜像文件是由若干层（layer）组成，层的唯一标识是以一个256比特的64个十六进程字符构成。使用docker pull下载时会获取各层的信息。当不同的镜像包括相同的层时，本地仅会存储层的一份内容，减小了需要的存储空间。
* 当仓库地址（registry，注册服务器）省略不写时，默认使用`docker hub`服务器，如果从非官方仓库下载，则需要在仓库名称前指定完整的镜像注册服务器地址（e.g. hub.c.163.com/library/）。

```
//去网易蜂巢拉取镜像仓库

[root@xxx ~]# docker pull hub.c.163.com/library/memcached:latest 
latest: Pulling from library/memcached
810fd2d89f8f: Pull complete 
0f1e2d8abe76: Pull complete 
b9608bffd4d0: Pull complete 
a6554c2d9f43: Pull complete 
40661d641679: Pull complete 
Digest: sha256:537918e564521a6aa1d4da202e33af500ecfcb4ab9be78d5a6f222ef919b3ba9
Status: Downloaded newer image for hub.c.163.com/library/memcached:latest

``` 

## 列出镜像
命令格式：`docker images [OPTIONS] [REPOSITORY[:TAG]]`

```
Options:
  -a, --all             显示所有镜像（包括临时文件），默认为否
      --digests         列出镜像的数字摘要值，默认为否
  -f, --filter filter   过滤列出的镜像
      --format string   控制输出格式
      --no-trunc        对输出结果中太长部分是否进行截断，如镜像ID信息，默认为是
  -q, --quiet           仅输出ID信息，默认为否

```

```
[root@xxx ~]# docker images
REPOSITORY                        TAG                 IMAGE ID            CREATED             SIZE
centos                            7                   0b99289e40ee        About an hour ago   435MB
test                              0.1                 ff67a67177d8        About an hour ago   222MB
hello-world                       latest              e38bc07ac18e        32 hours ago        1.85kB
ubuntu                            14.04               a35e70164dfb        5 weeks ago         222MB
hub.c.163.com/library/memcached   latest              8b057b9de580        7 months ago        58.6MB
hub.c.163.com/public/ubuntu       14.04               2fe5c4bba1f9        2 years ago         237MB
163ubuntu                         14.04               2fe5c4bba1f9        2 years ago         237MB

```
* REPOSITORY:来源于哪个仓库，比如Ubuntu仓库用来保存Ubuntu系列的基础镜像。
* TAG：镜像标签信息，用来标注不同的版本信息。如：14.04、latest等。
* IMAGE ID：镜像的ID（唯一标识镜像），如hub.c.163.com/public/ubuntu：14.04和163ubuntu：14.04镜像的ID都是2fe5c4bba1f9，说明目前他们指向同一个镜像
* CREATED：说明镜像的更新时间
* 镜像大小，优秀的镜像往往体积都较小

其中镜像的ID信息十分重要，它唯一标识了镜像。在使用镜像ID的时候，一般可以使用该ID的前若干个字符组成的可区分串来替代完整的ID。

TAG信息用来标识来自同一个仓库的不同镜像。例如ubuntu仓库中有多个镜像，通过TAG信息来区分发行版本，包括10.04、12.04、12.10、14.04等标签。

镜像大小信息只是表示该镜像的逻辑体积大小，实际上由于相同的镜像本地只会存储一份，物理占用的存储空间会小于各镜像的逻辑体积之和。

更多子命令选项还可以通过`man docker-images`帮助命令来查看。

## 使用tag命令添加镜像标签
命令格式：`docker tag SOURCE_IMAGE[:TAG] TARGET_IMAGE[:TAG]`

```
[root@xxx ~]# docker tag hub.c.163.com/public/ubuntu:14.04 163ubuntu:14.04 
[root@xxx ~]# docker images
REPOSITORY                        TAG                 IMAGE ID            CREATED             SIZE
centos                            7                   0b99289e40ee        About an hour ago   435MB
test                              0.1                 ff67a67177d8        2 hours ago         222MB
hello-world                       latest              e38bc07ac18e        32 hours ago        1.85kB
ubuntu                            14.04               a35e70164dfb        5 weeks ago         222MB
hub.c.163.com/library/memcached   latest              8b057b9de580        7 months ago        58.6MB
163ubuntu                         14.04               2fe5c4bba1f9        2 years ago         237MB
hub.c.163.com/public/ubuntu       14.04               2fe5c4bba1f9        2 years ago         237MB
[root@xym ~]# 

```
为了方便后续工作中使用特定镜像，还可以使用docker tag命令来为本地镜像任意添加新的标签。例如添加一个新的`163ubuntu:14.04`镜像标签。之后用户就可以直接使用`163ubuntu:14.04`来表示这个镜像了。观察`163ubuntu:14.04`的ID跟源镜像`hub.c.163.com/public/ubuntu:14.04`完全一致。他们实际上指向同一个镜像文件，只是别名不同而已。`docker tag`命令添加的标签实际上起到了类似连接的作用。

## 使用inspect命令查看详细信息
命令格式：`docker inspect [OPTIONS] NAME|ID [NAME|ID...]`

```
Options:
  -f, --format string   使用指定的模板，格式化输出
  -s, --size            如果是容器类型，表示其总大小
      --type string     返回指定类型的json格式
      
```
```
[root@xxx ~]# docker inspect 163ubuntu:14.04 
[
    {
        "Id": "sha256:2fe5c4bba1f935f179e83cd5354403d1231ffc9df9c1621967194410eaf8d942",
        "RepoTags": [
            "163ubuntu:14.04",
            "hub.c.163.com/public/ubuntu:14.04"
        ],
        "RepoDigests": [
            "hub.c.163.com/public/ubuntu@sha256:ffc2fc66f8e0bfa4b417b817054d3ebec130c8db44342b8fa394e25779633257"
        ],
        "Parent": "",
        "Comment": "",
        "Created": "2016-03-16T03:29:48.276492132Z",
        "Container": "f807d2c2c41cc21db9201605a962047278719a09cb945d0a3d5a2a587d978769",
        "ContainerConfig": {
            "Hostname": "f807d2c2c41c",
            "Domainname": "",
            "User": "",
            "AttachStdin": false,
            "AttachStdout": false,
            "AttachStderr": false,
            "Tty": false,
            "OpenStdin": false,
            "StdinOnce": false,
            "Env": [],
            "Cmd": [
                "/bin/sh",
                "-c",
                "#(nop) ENTRYPOINT &{[\"/bin/sh\" \"-c\" \"/usr/sbin/sshd -D\"]}"
            ],
            "Image": "7c5c7629089e80dea49161f10c36678cc8934601a730f8f8eb2a58d2e14c6610",
            "Volumes": null,
            "WorkingDir": "",
            "Entrypoint": [
                "/bin/sh",
                "-c",
                "/usr/sbin/sshd -D"
            ],
            "OnBuild": null,
            "Labels": {}
        },
        "DockerVersion": "1.9.1",
        "Author": "",
        "Config": {
            "Hostname": "f807d2c2c41c",
            "Domainname": "",
            "User": "",
            "AttachStdin": false,
            "AttachStdout": false,
            "AttachStderr": false,
            "Tty": false,
            "OpenStdin": false,
            "StdinOnce": false,
            "Env": [],
            "Cmd": null,
            "Image": "7c5c7629089e80dea49161f10c36678cc8934601a730f8f8eb2a58d2e14c6610",
            "Volumes": null,
            "WorkingDir": "",
            "Entrypoint": [
                "/bin/sh",
                "-c",
                "/usr/sbin/sshd -D"
            ],
            "OnBuild": null,
            "Labels": {}
        },
        "Architecture": "amd64",
        "Os": "linux",
        "Size": 237059566,
        "VirtualSize": 237059566,
        "GraphDriver": {
            "Data": {
                "DeviceId": "27",
                "DeviceName": "docker-253:0-102127602-c30e1cbfce6f98d947b8df2100734cddf113980a3ccdb356a1b84f27825a3dbe",
                "DeviceSize": "10737418240"
            },
            "Name": "devicemapper"
        },
        "RootFS": {
            "Type": "layers",
            "Layers": [
                "sha256:89688d062a0607fb50d0955de8964659e66f1bb41164b2d2b473d1edd7d8af90",
                "sha256:704e51eef17861bc3a2a7355709a7ce0b11ab720cc1b0e00235f984b33494b0e",
                "sha256:98b4fca781e7eab1cfb4d6427b60c4490b4c7d71a0bca622c7dd03cecb657a6d",
                "sha256:a695e8d298aaf8ee68638151e6068518475130eccdd224ba0591981f212e5ea2",
                "sha256:836a329bec9925c8fc76232885344a0053d19534ae108e5cbc111490481b5778",
                "sha256:5f70bf18a086007016e948b04aed3b82103a36bea41755b6cddfaf10ace3c6ef"
            ]
        },
        "Metadata": {
            "LastTagTime": "2018-04-13T10:03:07.660866339+08:00"
        }
    }
]

```
更多用法请使用`man docker-inspect`帮助命令。

## 使用history命令查看镜像历史
命令格式：`docker history [OPTIONS] IMAGE`

```
Options:
      --format string   指定格式化模板输出
  -H, --human           Print sizes and dates in human readable format (default true)
      --no-trunc        Don't truncate output
  -q, --quiet           Only show numeric IDs

```
```
[root@xxx ~]# docker history 163ubuntu:14.04 
IMAGE               CREATED             CREATED BY                                      SIZE                COMMENT
2fe5c4bba1f9        2 years ago         /bin/sh -c #(nop) ENTRYPOINT &{["/bin/sh" "-…   0B                  
<missing>           2 years ago         /bin/sh -c sed -i 's/#PasswordAuthentication…   2.54kB              
<missing>           2 years ago         /bin/sh -c mkdir /var/run/sshd                  0B                  
<missing>           2 years ago         /bin/sh -c apt-get update && apt-get install…   69.3MB              
<missing>           2 years ago         /bin/sh -c #(nop) ADD file:b7283a2724cc73e4c…   872B                
<missing>           2 years ago         /bin/sh -c #(nop) ADD file:9f6d0ad171ede3597…   168MB 

```

## 镜像搜寻
命令格式：`docker search [OPTIONS] TERM`
```
Options:
  -f, --filter filter   Filter output based on conditions provided
      --format string   Pretty-print search using a Go template
      --limit int       Max number of search results (default 25)
      --no-trunc        Don't truncate output
      
      Filter
             Filter output based on these conditions:
                - stars=<numberOfStar> 指定仅显示评价为指定星级以上的镜像，默认为0，即输出所有镜像
                - is-automated=(true|false) 仅显示自动创建的镜像，默认为否
                - is-official=(true|false) 仅显示官方的镜像，默认为否
```

```
//搜索所有自动创建且评价为20+的带nginx关键字的镜像
[root@xxx ~]# docker search --filter=is-automated=true --filter=stars=20 nginx
NAME                                                   DESCRIPTION                                     STARS               OFFICIAL            AUTOMATED
jwilder/nginx-proxy                                    Automated Nginx reverse proxy for docker con…   1315                                    [OK]
richarvey/nginx-php-fpm                                Container running Nginx + PHP-FPM capable of…   544                                     [OK]
jrcs/letsencrypt-nginx-proxy-companion                 LetsEncrypt container to use with nginx as p…   348                                     [OK]
webdevops/php-nginx                                    Nginx with PHP-FPM                              99                                      [OK]
zabbix/zabbix-web-nginx-mysql                          Zabbix frontend based on Nginx web-server wi…   49                                      [OK]
bitnami/nginx                                          Bitnami nginx Docker Image                      48                                      [OK]
1and1internet/ubuntu-16-nginx-php-phpmyadmin-mysql-5   ubuntu-16-nginx-php-phpmyadmin-mysql-5          33                                      [OK]

```

* NAME：镜像名字
* DESCRIPTION：描述
* STARS：星级（表示该镜像受欢迎程度）
* OFFICIAL：是否官方创建
* AUTOMATED：是否自动创建

默认结果按照星级评价进行排序。

## 删除镜像
命令格式：`docker rmi [OPTIONS] IMAGE [IMAGE...]`
```
Options:
  -f, --force      强制删除
      --no-prune   Do not delete untagged parents

```

### 使用标签删除镜像

```
//删除前查看镜像
[root@xxx ~]# docker images
REPOSITORY                        TAG                 IMAGE ID            CREATED             SIZE
centos                            7                   0b99289e40ee        2 hours ago         435MB
test                              0.1                 ff67a67177d8        3 hours ago         222MB
hello-world                       latest              e38bc07ac18e        33 hours ago        1.85kB
ubuntu                            14.04               a35e70164dfb        5 weeks ago         222MB
hub.c.163.com/library/memcached   latest              8b057b9de580        7 months ago        58.6MB
163ubuntu                         14.04               2fe5c4bba1f9        2 years ago         237MB
hub.c.163.com/public/ubuntu       14.04               2fe5c4bba1f9        2 years ago         237MB
//删除镜像操作
[root@xxx ~]# docker rmi 163ubuntu:14.04 
Untagged: 163ubuntu:14.04
//删除后查看镜像
[root@xxx ~]# docker images
REPOSITORY                        TAG                 IMAGE ID            CREATED             SIZE
centos                            7                   0b99289e40ee        2 hours ago         435MB
test                              0.1                 ff67a67177d8        3 hours ago         222MB
hello-world                       latest              e38bc07ac18e        33 hours ago        1.85kB
ubuntu                            14.04               a35e70164dfb        5 weeks ago         222MB
hub.c.163.com/library/memcached   latest              8b057b9de580        7 months ago        58.6MB
hub.c.163.com/public/ubuntu       14.04               2fe5c4bba1f9        2 years ago         237MB

```
    
**注意**：当同一个镜像拥有多个标签的时候，docker rmi命令只是删除该镜像多个标签中的指定标签而已，并不影响镜像文件。因此上述操作相当于只是删除了镜像`2fe5c4bba1f9`的一个标签而已。但当镜像只剩下一个标签的时候就要小心了,此时再使用`docker rmi`命令会彻底删除镜像。

```
[root@xxx ~]# docker rmi hub.c.163.com/library/memcached:latest 
Untagged: hub.c.163.com/library/memcached:latest
Untagged: hub.c.163.com/library/memcached@sha256:537918e564521a6aa1d4da202e33af500ecfcb4ab9be78d5a6f222ef919b3ba9
Deleted: sha256:8b057b9de580ce01fdce47c7ca1632ce03b925f9464afc0d91821b066f32204d
Deleted: sha256:0382f79fb93ba9f862822a9c594019a32a39f5d46321e242a388c2e455d369e6
Deleted: sha256:7f7bc5c738d847e3e2e647b467d3bb363bbbc99f53649cba7df71c2326fc183d
Deleted: sha256:1c3bf91649b76e0956ab416404cb81bb30f1be5377316af42ecf680cb50c2d34
Deleted: sha256:bbebfa4c46fd5f64fc0e0d71fc5c94448fa04fa0b20b74dc97ad0ec07cd8ff45
Deleted: sha256:eb78099fbf7fdc70c65f286f4edc6659fcda510b3d1cfe1caa6452cc671427bf

```
例如删除标签为`hub.c.163.com/library/memcached:latest`的镜像，由于该镜像没有额外的标签指向它，执行`docker rmi`命令，可以看出它会删除这个镜像文件的所有层。

### 使用镜像ID删除镜像
当使用docker rmi命令，并且后面跟上镜像的ID（也可能是能进行区分的部分ID串前缀）时，先会尝试删除所有指向该镜像的标签，然后删除该镜像本身。
```
//删除前查看镜像
[root@xxx ~]# docker images
REPOSITORY                        TAG                 IMAGE ID            CREATED             SIZE
centos                            7                   0b99289e40ee        3 hours ago         435MB
test                              0.1                 ff67a67177d8        3 hours ago         222MB
hello-world                       latest              e38bc07ac18e        33 hours ago        1.85kB
ubuntu                            14.04               a35e70164dfb        5 weeks ago         222MB
hub.c.163.com/library/memcached   latest              8b057b9de580        7 months ago        58.6MB
163mem                            latest              8b057b9de580        7 months ago        58.6MB
hub.c.163.com/public/ubuntu       14.04               2fe5c4bba1f9        2 years ago         237MB

//通过镜像ID删除镜像，提示多个镜像引用此镜像ID，必须使用-f 参数强制删除
[root@xxx ~]# docker rmi 8b057b9de580
Error response from daemon: conflict: unable to delete 8b057b9de580 (must be forced) - image is referenced in multiple repositories
[root@xym ~]# docker rmi -f 8b057b9de580
Untagged: 163mem:latest
Untagged: hub.c.163.com/library/memcached:latest
Untagged: hub.c.163.com/library/memcached@sha256:537918e564521a6aa1d4da202e33af500ecfcb4ab9be78d5a6f222ef919b3ba9
Deleted: sha256:8b057b9de580ce01fdce47c7ca1632ce03b925f9464afc0d91821b066f32204d
Deleted: sha256:0382f79fb93ba9f862822a9c594019a32a39f5d46321e242a388c2e455d369e6
Deleted: sha256:7f7bc5c738d847e3e2e647b467d3bb363bbbc99f53649cba7df71c2326fc183d
Deleted: sha256:1c3bf91649b76e0956ab416404cb81bb30f1be5377316af42ecf680cb50c2d34
Deleted: sha256:bbebfa4c46fd5f64fc0e0d71fc5c94448fa04fa0b20b74dc97ad0ec07cd8ff45
Deleted: sha256:eb78099fbf7fdc70c65f286f4edc6659fcda510b3d1cfe1caa6452cc671427bf

//删除后查看镜像列表，发现与此ID关联的镜像都已删除成功
[root@xxx ~]# docker images
REPOSITORY                    TAG                 IMAGE ID            CREATED             SIZE
centos                        7                   0b99289e40ee        3 hours ago         435MB
test                          0.1                 ff67a67177d8        3 hours ago         222MB
hello-world                   latest              e38bc07ac18e        33 hours ago        1.85kB
ubuntu                        14.04               a35e70164dfb        5 weeks ago         222MB
hub.c.163.com/public/ubuntu   14.04               2fe5c4bba1f9        2 years ago         237MB

```

**注意**：当有该镜像创建的容器存在时，镜像文件是无法被删除的，如要删除该镜像请先删除该容器，然后再删除镜像。

```
//使用镜像运行容器
[root@xym ~]# docker run ubuntu:14.04 echo "Hello"
Hello
//查看所有状态容器
[root@xym ~]# docker ps -a
CONTAINER ID        IMAGE                               COMMAND                  CREATED             STATUS                     PORTS               NAMES
8bdb6f330ec9        ubuntu:14.04                        "echo Hello"             7 seconds ago       Exited (0) 6 seconds ago                       determined_varahamihira

```

```
//使用标签删除镜像，提示被容器引用
[root@xxx ~]# docker rmi ubuntu:14.04 
Error response from daemon: conflict: unable to remove repository reference "ubuntu:14.04" (must force) - container 8bdb6f330ec9 is using its referenced image a35e70164dfb

//无法强制删除
[root@xxx ~]# docker rmi -f a35e70164dfb
Error response from daemon: conflict: unable to delete a35e70164dfb (cannot be forced) - image has dependent child images

//删除容器
[root@xxx ~]# docker rm 8bdb6
8bdb6

//成功删除镜像
[root@xxx ~]# docker rmi ubuntu:14.04 
Untagged: ubuntu:14.04
Untagged: ubuntu@sha256:ed49036f63459d6e5ed6c0f238f5e94c3a0c70d24727c793c48fded60f70aa96

```

## 创建镜像
创建镜像的方法有三种：基于已有镜像的容器创建、基于本地模板导入、基于Dockerfile创建。
先介绍前二种创建方式

### 基于已有镜像的容器创建
命令格式：`docker commit [OPTIONS] CONTAINER [REPOSITORY[:TAG]]`

```
Options:
  -a, --author string    作者信息
  -c, --change list      提交的时候指定Dockerfile指令，包括CMD/ENTRYPOINT/ENV/EXPOSE/LABEL/ONBUILD/USER/VOLUME/WORKDIR等
  -m, --message string   提交信息
  -p, --pause            提交时暂停容器运行

```

下面将演示如何使用该命令创建一个新镜像。首先，启动一个镜像，并在其中进行修改操作，例如创建一个test文件，之后推出：
```
[root@xxx ~]# docker run -it ubuntu:latest /bin/bash
root@72ed7e58bc28:/# touch test
root@72ed7e58bc28:/# exit
exit

```

记住此时的容器ID为：`72ed7e58bc28`,此时容器跟原`ubuntu:latest`镜像相比，已经发生了变化，可以使用docker commit命令来提交为一个新的镜像。提交的时候可以使用ID或名称来指定容器：
```
[root@xxx ~]# docker commit -m "提交一个新镜像" -a "xym" 72ed7e58bc28 xymtest:0.1
sha256:8a758d16a99b414b738bee50b485c3d99f6093c0c4002efd9dd5dd740efd2ee9
[root@xxx ~]# docker images
REPOSITORY                    TAG                 IMAGE ID            CREATED             SIZE
xymtest                       0.1                 8a758d16a99b        26 seconds ago      113MB
centos                        7                   0b99289e40ee        4 hours ago         435MB
test                          0.1                 ff67a67177d8        4 hours ago         222MB
...

```
### 基于本地模板导入
用户也可以直接从一个操作系统模板文件导入一个镜像。

命令格式：`docker import [OPTIONS] file|URL|- [REPOSITORY[:TAG]]`
```
Options:
  -c, --change list      导入时候指定Dockerfile指令，包括CMD/ENTRYPOINT/ENV/EXPOSE/LABEL/ONBUILD/USER/VOLUME/WORKDIR等
  -m, --message string   提交信息

```

要直接导入一个镜像，可以使用OpenVZ提供的模板来创建，或者用其他已导出的镜像模板来创建。OpenVZ模板的下载地址为：[https://openvz.org/Download/template/precreated](https://openvz.org/Download/template/precreated)

例如：下载了`centos-7-x86_64-minimal.tar.gz`模板压缩包，之后使用以下命令导入：
```
[root@xym ~]# cat centos-7-x86_64-minimal.tar.gz | docker import - centos-7:import
sha256:be5e039acd03e1c3489841f6edd244954a4c1eb534de7fff605d807136b7e735
[root@xym ~]# docker images
REPOSITORY                    TAG                 IMAGE ID            CREATED              SIZE
centos-7                      import              be5e039acd03        About a minute ago   435MB
xymtest                       0.1                 8a758d16a99b        18 minutes ago       113MB
centos                        7                   0b99289e40ee        4 hours ago          435MB
test                          0.1                 ff67a67177d8        4 hours ago          222MB
...

```

## 存出和载入镜像
可以使用`docker save` 和`docker load`命令来存出和载入镜像。

### 存出镜像
命令格式：`docker save [OPTIONS] IMAGE [IMAGE...]`
```
Options:
  -o, --output string   写出目标文件
  
```


如果要导出镜像到本地文件，可以使用`docker save`命令。例如，导出本地的`163ubuntu:14.04`镜像为文件`163ubuntu_14.04.tar`
```
[root@xxx ~]# docker save -o 163ubuntu_14.04.tar 163ubuntu:14.04 
[root@xxx ~]# ls -t
163ubuntu_14.04.tar ...

```
之后，用户就可以通过复制该文件（163ubuntu_14.04.tar）将镜像分享给其他人。

### 载入镜像
命令格式：`docker load [OPTIONS]`
```
Options:
  -i, --input string   指定要导入的tar归档镜像文件
  -q, --quiet          Suppress the load output

```

可以通过`docker load` 将导出的tar文件再导入到本地镜像库，例如从文件`163ubuntu_14.04.tar`导入镜像到本地镜像列表：

```
[root@xxx ~]# docker load --input 163ubuntu_14.04.tar 
Loaded image: 163ubuntu:14.04

或者可以使用

[root@xym ~]# docker load < 163ubuntu_14.04.tar 
Loaded image: 163ubuntu:14.04

```
这将导入镜像及其相关元数据信息（包括标签等）。

## 上传镜像
命令格式：`docker push [OPTIONS] NAME[:TAG]|[REGISTRY_HOST[:REGISTRY_PORT]/]NAME[:TAG]`
```
Options:
      --disable-content-trust   Skip image signing (default true)

```
可以使用`docker push`命令上传镜像到仓库，默认上传到`Docker Hub`官方仓库（需要登录）。
用户在`Docker Hub网站`注册后可以上传自制镜像。**例如用户user上传本地的test:latest镜像，可以先添加新的标签user/test:latest,然后用docker push命令上传镜像**：
 
#### 上传镜像到网易蜂巢docker

参见官网操作文档：[https://www.163yun.com/help/documents/15587826830438400](https://www.163yun.com/help/documents/15587826830438400)

1、登录网易云镜像仓库
docker login -u {你的网易云邮箱账号或手机号码} -p {你的网易云密码} hub.c.163.com

返回「Login Succeded」即为登录成功。

```
[root@xxx /]# docker login hub.c.163.com
Username: xxx@126.com（你的账号）	        
Password: 
WARNING! Your password will be stored unencrypted in /root/.docker/config.json.
Configure a credential helper to remove this warning. See
https://docs.docker.com/engine/reference/commandline/login/#credentials-store

Are you sure you want to proceed? [y/N] y
Login Succeeded

```

* 2、标记本地镜像
docker tag {镜像名或ID} hub.c.163.com/{你的用户名}/{标签名}

你的网易云镜像仓库推送地址为 hub.c.163.com/{你的用户名}/{标签名}

Attention: 此处为你的用户名，不是你的邮箱帐号或者手机号码 登录网易云控制台，页面右上角头像右侧即为「用户名」

推送至不存在的镜像仓库时，自动创建镜像仓库并保存新推送的镜像版本；
推送至已存在的镜像仓库时，在该镜像仓库中保存新推送的版本，当版本号相同时覆盖原有镜像。

```
[root@xxx /]# docker tag hub.c.163.com/public/ubuntu:14.04 hub.c.163.com/xxx/163ubuntu:14.04
[root@xxx /]# docker images
REPOSITORY                          TAG                 IMAGE ID            CREATED             SIZE
163ubuntu                           14.04               2fe5c4bba1f9        2 years ago         237MB
hub.c.163.com/public/ubuntu         14.04               2fe5c4bba1f9        2 years ago         237MB
hub.c.163.com/xxx/163ubuntu   14.04               2fe5c4bba1f9        2 years ago         237MB

```

* 3. 推送至网易云镜像仓库
docker push hub.c.163.com/{你的用户名}/{标签名}

默认为私有镜像仓库，推送成功后即可在控制台的「镜像仓库」查看。

```
[root@xxx /]# docker push hub.c.163.com/xxx/163ubuntu:14.04 
The push refers to repository [hub.c.163.com/xxx/163ubuntu]
5f70bf18a086: Pushed 
836a329bec99: Pushed 
a695e8d298aa: Pushed 
98b4fca781e7: Pushed 
704e51eef178: Pushed 
89688d062a06: Pushed 
14.04: digest: sha256:c6740481ffab5f07e8785e6f07d5e2bec8ba9436d67f6e686d0fe1bf65c651be size: 4031

```

#### 上传镜像到阿里云

>Docker的镜像存储中心通常被称为Registry。
> 当您需要获取Docker镜像的时候，首先需要登录Registry，然后拉取镜像。在您修改过镜像之后，您可以再次将镜像推送到Registry中去。
> 
> Docker的镜像地址是什么？我们来看一个完整的例子。（以容器服务的公共镜像为例）
> registry.cn-hangzhou.aliyuncs.com/acs/agent:0.8
> 
> registry.cn-hangzhou.aliyuncs.com 叫做 "Registry域名"。
> acs 叫做 "命名空间"。
> agent 叫做 "仓库名称"。
> 0.8 叫做 "Tag"、"镜像标签"（非必须，默认latest）。
> 将这个几个完全独立的概念组合一下，还有几个术语。
> registry.cn-hangzhou.aliyuncs.com/acs/agent 称为 "仓库坐标"。
> acs/agent 称为 "仓库全名"（通常在API中使用）。

参见文档：[https://yq.aliyun.com/articles/70756](https://yq.aliyun.com/articles/70756)

了解相关说明后发现，阿里云有一个"命名空间"的概念，所以**要想上传自己的镜像，请先去个人中心创建命名空间**。

比如：当前创建的命名空间为`xym`,则通过以下命令即可将自己的镜像上传到命名空间


* 1、docker login 以阿里云杭州公网Registry为例：登陆时必须指明登陆的 "Registry域名"

```
[root@xxx /]# docker login registry.cn-hangzhou.aliyuncs.com
Username: xxx
Password: 
WARNING! Your password will be stored unencrypted in /root/.docker/config.json.
Configure a credential helper to remove this warning. See
https://docs.docker.com/engine/reference/commandline/login/#credentials-store

Are you sure you want to proceed? [y/N] y
Login Succeeded

```
* 2、标记本地镜像
```
//注意这里的xym为：命名空间
[root@xxx /]# docker tag hub.c.163.com/public/ubuntu:14.04 registry.cn-hangzhou.aliyuncs.com/xym/163ubuntu:14.04
[root@xxx /]# docker images
REPOSITORY                                        TAG                 IMAGE ID            CREATED             SIZE
centos                                            7                   0b99289e40ee        6 hours ago         435MB
ubuntu                                            latest              c9d990395902        12 hours ago        113MB
hub.c.163.com/public/ubuntu                       14.04               2fe5c4bba1f9        2 years ago         237MB
registry.cn-hangzhou.aliyuncs.com/xym/163ubuntu   14.04               2fe5c4bba1f9        2 years ago         237MB

```

* 3. 推送至阿里云
```
[root@xxx /]# docker push registry.cn-hangzhou.aliyuncs.com/xym/163ubuntu
The push refers to repository [registry.cn-hangzhou.aliyuncs.com/xym/163ubuntu]
5f70bf18a086: Pushed 
836a329bec99: Pushed 
a695e8d298aa: Pushed 
98b4fca781e7: Pushed 
704e51eef178: Pushed 
89688d062a06: Pushed 
14.04: digest: sha256:ef619091bc47f4b82ce4e984668ee96a2447f244bbd73fbaffe103605c454c28 size: 1569

```

登录控制台查看推送结果。

## 镜像加速

参见阿里控制台，CentOS 镜像加速器帮助说明:
> 针对Docker客户端版本大于1.10.0的用户

> 您可以通过修改daemon配置文件/etc/docker/daemon.json来使用加速器：
> sudo mkdir -p /etc/docker
> sudo tee /etc/docker/daemon.json <<-'EOF'
> {
>   "registry-mirrors": ["https://4vehewku.mirror.aliyuncs.com"]
> }
> EOF
> sudo systemctl daemon-reload
> sudo systemctl restart docker

其他情况请自行 [google](https://www.google.com)

