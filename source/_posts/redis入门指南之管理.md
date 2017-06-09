---
title: redis入门指南之管理
date: 2017-06-09 17:22:22
categories: redis系列
tags: [redis安全,redis数据库密码,redis命令重命名,redis通信协议,redis管理工具] 
description: Redis管理知识，包括协议和安全等内容，及其第三方管理工具。
---

###### 重要星级 ★★★

---
## 安全  
Redis的作者Salvatore Sanfilippo曾经发表过Redis宣言，其中提到了Redis以简洁为美。同样在安全层面Redis也没有做太多的工作。  
### 1、可信的环境  
Redis的安全设计是在“Redis运行在可信环境”这个前提下做出的。在生产环境运行时不能允许外界直接连接到Redis服务器上，而应该通过应用程序进行中转，运行在可信的环境中是保证Redis安全的最重要方法。  
Redis的默认配置会接受来自任何地址发送来的请求，即在任何一个拥有公网IP的服务器上启动Redis服务器，都可以被外界直接访问到。要更改这一设置，在配置文件中修改bind参数，如只允许本机应用连接Redis，可以将bind参数改成：  

    bind 127.0.0.1

如果想要绑定多个地址，中间采用空格隔开，配置多个即可。  

    bind 127.0.0.1 192.168.100.238

### 2、数据库密码  
除此之外，还可以通过配置文件中的` requirepass `参数为Redis设置一个密码。例如：  

    requirepass 12345678

客户端每次连接到Redis时都需要发送密码，否则Redis会拒绝执行客户端发来的命令。  
例如：  

    127.0.0.1:6379> info default
    NOAUTH Authentication required.
    127.0.0.1:6379> keys *
    (error) NOAUTH Authentication required.

发送密码需要使用AUTH命令，就像这样：  

    127.0.0.1:6379> auth 123456
    OK
    127.0.0.1:6379> client list
    id=145 addr=127.0.0.1:55057 fd=5 name= age=9 idle=0 flags=N db=0 sub=0 psub=0 multi=-1 qbuf=0 qbuf-free=32768 obl=0 oll=0 omem=0 events=r cmd=client

由于Redis的性能极高，并且输入错误密码后Redis并不会进行主动延迟（考虑到Redis的单线程模型），所以攻击者可以通过穷举法破解Redis的密码，因此设置密码时一定要选择复杂的密码。  
**提示：配置Redis复制的时候如果主数据库设置了密码，需要在从数据库的配置文件中通过masterauth参数设置主数据库的密码，以使从数据库连接主数据库时自动使用AUTH命令认证。**  

### 3、重命名命令  
Redis支持在配置文件中将命令重命名，比如将FLUSHALL 命令重命名成一个比较复杂的名字，以保证只有自己的应用可以使用该命令。就像下面这样：  

    rename-command FLUSHALL QKSYSJK

如果希望直接禁用某个命令可以将命令重命名成空字符串：  

    rename-command FLUSHALL ""

无论设置密码还是重命名命令，都需要保证配置文件的安全性，否则就没有任何意义了。  

## 通信协议  
Redis通信协议是Redis客户端与Redis之间交流的语言，通信协议规定了命令和返回值的格式。了解Redis通信协议后不仅可以理解AOF文件的格式和主从复制时主数据库向从数据库发送的内容等，还可以开发自己的客户端。  
Redis支持两种通信协议，一种是二进制安全的统一请求协议（unified request protocol），另一种是比较直观的便于在telnet程序中输入的简单协议。这两种协议只是命令格式有区别，命令返回值的格式是一样的。  

### 1、简单协议  
简单协议适合在telnet程序中和Redis通信。简单协议的命令格式就是将命令和各个参数使用空格分隔开，如“EXISTS foo ”、“SET foo bar”等。由于Redis解析简单协议时只是简单地以空格分隔参数，所以无法输入二进制字符。我们可以通过telnet程序测试：  

    [admin@KFCS2 redis-stable]$ telnet  127.0.0.1 6379
    Trying 127.0.0.1...
    Connected to 127.0.0.1.
    Escape character is '^]'.
    set foo bar
    -NOAUTH Authentication required.
    auth 123456
    +OK
    set foo bar 
    +OK
    get foo
    $3
    bar
    lpush plist 1 2 3          
    :3
    lrange plist 0 -1
    *3
    $1
    3
    $1
    2
    $1
    1
    errorcommand
    -ERR unknown command 'errorcommand'
    
我们在telnet程序中输入的5条命令恰好展示了Redis5种返回类型的格式，上面章节介绍了这5种返回值类型在redis-cli中的展现形式，这些展现形式是经过了redis-cli封装的，而上面的内容才是Redis真正返回的格式。下面分别介绍。  
1、错误回复  
错误回复（error reply）以-开头，并在后面跟上错误信息，最后以\r\n结尾：  
-ERR unknown command 'errorcommand'\r\n

2、状态回复  
状态回复（status reply）以+开头，并在后面跟上状态信息，最后以\r\n结尾：  
+OK\r\n

3、整数回复  
整数回复（integer reply）以：开头，并在后面跟上数字，最后以\r\n结尾：  
:3\r\n

4、字符串回复  
字符串回复（bulk reply）以$开头，并在后面跟上字符串的长度，并以以\r\n分隔，接着是字符串的内容和\r\n：  
$3\r\nbar\r\n  
如果返回值是空结果nil，则会返回$-1以和空字符串相区别。


5、多行字符串回复  
多行字符串回复（multi-bulk reply）以*开头，并在后面跟上字符串回复的组数，并以\r\n分隔。接着后面跟的就是字符串回复的具体内容了：  
*3\r\n$1\r\n3\r\n$1\r\n2\r\n$1\r\n1\r\n

### 2、统一请求协议  
统一请求协议是从Redis1.2开始加入的，其命令格式和多行字符串回复的格式很类似，如 SET foo bar 的统一请求协议写法是*3\r\n$3\r\nSET\r\n$3\r\nfoo\r\n$3\r\nbar\r\n。还是使用telnet进行演示：  

    [admin@KFCS2 redis-stable]$ telnet  127.0.0.1 6379
    Trying 127.0.0.1...
    Connected to 127.0.0.1.
    Escape character is '^]'.
    *3
    $3  
    SET
    $3    
    foo
    $3
    bar
    -NOAUTH Authentication required.
    auth 123456
    +OK

同样发送命令时指定了后面字符串的长度，所以命令的每个参数都可以包含二进制的字符。统一请求协议的返回值格式和简单协议一样。  
Redis的AOF文件和主从复制时主数据库向从数据库发送的内容都是用了统一请求协议。如果要开发一个和Redis直接通信的客户端，推荐使用此协议。如果只是想通过telnet向Redis服务器发送命令则使用简单协议就可以了。  
## 管理工具  
工欲善其事必先利其器。在使用Redis的时候如果能够有效利用Redis的各种管理工具，将会大大方便开发和管理。  
### 1、redis-cli  
作为Redis自带的命令行客户端，你可以从任何安装有Redis的服务器中找到它，所以对于管理Redis而言redis-cli是最简单实用的工具。  
redis-cli可以执行大部分的Redis命令，包括查看数据库信息的INFO命令，更改数据库设置的CONFIG命令和强制进行RDB快照的SAVE命令等。下面介绍几个管理Redis时非常有用的命令。  
1、耗时命令日志  
当一条命令执行时间超时限制时，Redis会将该命令的执行时间等信息加入耗时命令日志（slow log）以供开发者查看。可以通过配置文件`slowlog-log-slower-than` 参数设置这一限制，要注意单位是微妙（1000 000微妙相当于1秒），默认值是10 000。耗时命令日志存储在内存中，可以通过配置文件的`slowlog-max-len`参数来限制记录的条数。为了产生一些耗时命令日志作为演示，这里将`slowlog-log-slower-than`参数值设置为0，即记录所有命令。如果设置为负数则会关闭耗时命令日志。如：  

    127.0.0.1:6379> slowlog get
     1) 1) (integer) 16
        2) (integer) 1497011605
        3) (integer) 52
        4) 1) "set"
           2) "foo"
           3) "bar"
     2) 1) (integer) 15
        2) (integer) 1497011599
        3) (integer) 6
        4) 1) "get"
           2) "foo"
           
每条日志都由以下4个部分组成：  
（1）改日志唯一ID  
（2）改命令执行的UNIX时间  
（3）改命令的耗时时间，单位是微妙  
（4）命令及其参数  

2、命令监控  
Redis提供了MONITOR命令来监控Redis执行的所有命令，redis-cli同样支持这个命令，如在redis-cli中执行MONITOR： 

    127.0.0.1:6379> monitor
    OK
    
这时Redis执行的任何命令都会在redis-cli中打印出来，如我们打开另一个redis-cli执行SET foo bar命令，在之前的redis-cli中会输出如下内容：  

    1497012041.878429 [0 127.0.0.1:55062] "auth" "123456"
    1497012053.311366 [0 127.0.0.1:55062] "set" "foo" "bar"

MONITOR命令非常影响Redis的性能，一个客户端使用MONITOR命令会降低Redis将近一半的负载能力。所以MONITOR命令只适合用来调试和纠错。

### 2、采用phpRedisAdmin  
Redis有一款使用PHP开发的网页管理工具phpRedisAdmin。phpRedisAdmin支持以树结构查看键列表，编辑键值，导入/导出数据库数据，查看数据库信息和查看键信息等功能。  
### 3、Rdbtools  
Rdbtools是一个Redis快照文件解析器，它可以根据快照文件导出JSON数据文件、分析Redis中每个键的占用空间情况等。
