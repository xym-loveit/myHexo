---
title: redis入门指南-准备阶段
date: 2017-05-23 10:26:22
categories: redis系列
tags: [redis配置,redis多数据库]
description: 介绍运行redis及其redis的基础知识；学习redis的最好的办法就是动手尝试她。
---
## 一、启动和停止

redis 可执行文件说明  

文件名|说明
---|---
redis-server|Redis服务器
redis-cli|Redis命令行客户端
redis-benchmark|Redis性能测试工具
redis-check-aof|AOF文件修复工具

### 启动redis

---

#### 直接启动

```
redis-server --port 6379
```
redis 默认使用6379,通过port参数可自定义端口号

#### 通过初始化脚本启动redis
在redis的源代码目录utils下有一个名为redis_init_script的初始化脚本文件，我们需要配置redis的运行方式和持久化文件、日志文件的存储位置等，步骤如下：
1. 配置初始化脚本。将初始化文件复制到/etc/init.d目录中，并更名为redis_端口号，其中端口号表示redis监听的端口号，客户端通过该端口号链接redis。然后修改脚本第6行REDISPORT变量值为同样的端口号。

2. 建立需要的文件夹。

需要建立目录及说明

目录名|说明
---|---
/etc/redis|存放redis配置文件
/var/redis/端口号|存放redis持久化文件

3. 修改配置文件。将配置文件模板复制到/etc/redis目录中，依端口号命名(如：“6379.conf”),然后按下表对文件参数进行编辑。

参数|值|说明
---|---|---
daemonize|yes|使redis依守护进程模式启动
pidfile|/var/run/redis_端口号.pid|设置redis的pid文件位置
port|端口号|设置redis监听的端口号
dir|/var/redis/端口号|设置持久化文件存储位置

现在就可以使用/etc/init.d/redis_端口号 start来启动redis了，然后执行下面命令让redis随系统自启动。


```
sudo update-rc.d redis_端口号 defaults
```

---

### 停止redis

考虑到redis有可能正在将内存中的数据同步到磁盘中，强制终止redis进程可能导致丢失数据。正确停止redis的方式应该是向redis发送shutdown命令。


```
redis-cli shutdown
```

当redis接受到shutdown命令后，会先断开所有的客户端连接，然后根据配置执行持久化，最后完成退出。

redis可以妥善处理sigterm信号，所以使用**kill redis 的pid**也可以正常完成退出，效果和使用shutdown命令一样。

---

## 二、Redis的客户端命令

redis-cli（Redis Command Line Interface）是redis自带的基于命令行的客户端。使用Redis-cli向redis发送命令可以观察不同类型的命令。

### 发送命令
1. 将命令作为redis-cli 的参数执行，如：redis-cli shutdown。redis执行的时候会按照默认配置（服务器地址：127.0.0.1，端口号：6379，链接redis），通过-h和-p参数可以自定义地址和端口号。


```
redis-cli -h 127.0.0.1 -p 6379
```
可以通过redis-cli ping在测试redis是否和客户端连接正常，如果正常会受到pong。如：

```
$ redis-cli ping
pong
```

2. 不带参数运行redis-cli，进入交互模式

```
$ redis-cli
127.0.0.1:6379>ping
PONG
127.0.0.1:6379>echo hi
"hi"
```
这种方式可以输入多次命令，比较方便

### 命令返回值

1. 状态回复 

最简单的回复，例如向redis执行set命令redis回复OK，ping命令返回pong，状态回复直接显示状态信息。例如：


```
127.0.0.1:6379> PING
PONG
```


2. 错误回复

当出现命令不存在或命令格式有错误等情况时redis会返回错误回复（error reply），错误回复以（error）开否并在后面跟上错误信息。如执行一个不存在的命令：


```
127.0.0.1:6379> ERRORCOMMAND
(error) ERR unknown command 'ERRORCOMMAND'
```


3. 整数回复

redis虽然没有整数类型，但却提供了一些用于操作整数的命令，如递增键值的incr命令会以整数形式返回递增后的键值。除此之外还有一些其他命令也会返回整数，如可以获取当前数据库中键的数据量的DBSIZE命令等。整数回复（integer reply）以（integer）开头，后面跟上整数数据：


```
127.0.0.1:6379> incr test
(integer) 1
127.0.0.1:6379> dbsize
(integer) 23
```


4. 字符串回复

字符串回复（bulk reply），为最常见的回复类型，当请求一个字符串类型键的键值或一个其他类型键中某个元素的时候就会得到字符串回复。字符串回复以双引号包裹：


```
127.0.0.1:6379> get test
"1"
127.0.0.1:6379> hget car:1 speed
"200km/h"
```

特殊情况时当请求的键值不存在时，会得到一个空结果，显示为（nil）。如：

```
127.0.0.1:6379> get noexists
(nil)

```


5. 多行字符串回复

多行字符串回复（multi-bulk reply）也比较常见，当请求一个非字符串类型键的元素列表时就会收到多行字符串回复。多行字符串中每行都以一个序号开头，如：


```
127.0.0.1:6379> keys *
 1) "homepageZhdll_20175rank2"
 2) "clueUserKey:363:4:321102"
 3) "car:1"
 4) "homepageCyph_20175rank2"

```
特殊情况,当数据库中为空时


```
127.0.0.1:6379> flushdb
OK
127.0.0.1:6379> keys *
(empty list or set)

```

## 三、配置

 redis支持通过配置文件来设置参数选项。启用配置文件的方法是在启动时将配置文件路径作为参数传递给redis-server，如：
 

```
redis-server /path/to/redis.conf
```

通过启动参数传递同名的配置选项会覆盖配置文件中的参数，像：


```
redis-server /path/to/redis.conf --loglevel warning
```
redis提供了配置文件模板位于源代码的根目录中。
除此之外还可以在redis运行时通过 config set命令在不重新启动redis服务的情况下动态修改部分redis配置，就像
：

```
127.0.0.1:6379> config set loglevel warning
OK
```

并不是所有的命令都可以使用config set命令动态修改，同样在运行的时候也可以使用config get 命令获得redis当前的配置情况，如：


```
127.0.0.1:6379> config get loglevel
1) "loglevel"
2) "warning"

```

第一行为选项名，第二行选项值


## 四、多数据库
每个数据库对外都是以0开始的递增命名，redis默认支持16个数据库（0,1,2,3...15）可以通过配置参数databases来修改。客户端与redis建立连接后会自动选择0号数据库，不过可以通过select 命令更换数据库如：选择一号数据库


```
127.0.0.1:6379> select 1
OK
127.0.0.1:6379[1]> get test
(nil)
```

然而这些以数字命名的数据库又与我们理解的数据库有所区别。首选redis不支持自定义数据库名字，每个数据库都以编号命名，开发者必须自己记录哪些数据库存储了哪些数据。另外redis也不支持为每个数据库设置不同的访问密码，所以一个客户端要么可以访问全部数据库，要么连一个数据库也没有权限访问。最重要的一点是多个数据库之间并不是完全隔离的，比如flushall 命令可以清空一个redis实例中所有数据库中的数据。综上所述，这些数据库更像是一种命名空间，而不适宜存储不同应用程序的数据。比如可以使用0号数据库存储某个应用生产环境中的数据，使用1号数据库存储测试环境中的数据，但不适宜使用0号数据库存储A应用的数据，而使用1号数据库存储B应用的数据，不同的应用应该使用不同的redis实例存储数据。由于redis非常轻量级，一个空的redis实例占用的内存只有1MB左右，所以不用担心多个redis实例会额外占用很多内存。