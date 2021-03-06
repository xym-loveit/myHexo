---
title: redis入门指南之Lua脚本
date: 2017-06-01 10:29:19
categories: redis系列
tags: [redis使用Lua脚本,Lua脚本,EVAL及EVALSHA,KEYS与ARGV]
description: 使用Redis的Lua编程脚本，实现复杂的复合操作，实现原子操作避免竞态条件。
---

###### 重要星级 ★★★★★

---


在进阶章节讲到实现访问频率限制功能，用来限制一个IP地址1分钟最多只能访问100次：  

    $isKeyExists=EXISTS rate.limiting:$IP
    if $isKeyExists is 1
        $times=INCR rate.limiting:$IP
        if $times>100
            print 访问频率超过限制，请稍后再试
            exit
    else
        MULTI
        INCR rate.limiting:$IP
        EXPIRE $keyName,60
        EXEC  
        
当时提到上面的代码会出现竞态条件，解决方法是用WATCH命令检测rate.limiting:$IP键的变动，但是这样做比较麻烦，而且还需要判断事务是否因为键被改动而没有执行。除此之外这段代码在不适用管道的情况下最多要向Redis请求5条命令，在网络传输上会浪费很多时间。  
我们这时最希望就是Redis直接提供一个“RATELIMITING”命令用来实现访问频率限制功能，这个命令只需要我们提供键名、时间限制和在时间限制内最多访问的次数三个参数就可以直接返回访问频率是否超限。就像下面这样：  

    if RATELIMITING rate.limiting:$IP,60 100
        print 访问频率超过限制，请稍后再试
    else 
        #没有超限，其他业务处理  


这种方式不仅代码简单、没有竞态条件（Redis的命令都是原子的），而且减少了通过网络发送和接收命令的传输开销。然而可惜的是Redis并没有提供这个命令，不过我们可以使用Redis脚本功能自己定义新的命令。  
### 1、脚本介绍  
Redis在2.6版推出了脚本功能，允许开发者用Lua语言编写脚本传到Redis中执行。在Lua脚本中可以调用大部分的Redis命令，也就是说可以写一段Lua脚本发送给Redis执行。使用脚本的好处：  
（1）减少网络开销。复合操作需要向Redis发送多次请求，如上例，而是用脚本功能完成同样的操作只需要发送一个请求即可，减少了网络往返时延。  
（2）原子操作。Redis会将整个脚本作为一个整体执行，中间不会被其他命令插入。换句话说在编写脚本的过程中无需担心会出现竞态条件，也就无需使用事务。事务可以完成的所有功能都可以用脚本来实现。  
（3）复用。客户端发送的脚本会永久存储在Redis中，这就意味着其他客户端（可以是其他语言开发的项目）可以复用这一脚本而不需要使用代码完成同样的逻辑。  
### 2、实例：访问频率限制  
因为无需考虑事务，使用Redis脚本实现访问频率限制非常简单。Lua代码如下：  

    local time=redis.call('incr',KEYS[1])
    if times==1 then --KEYS[1]键刚创建，所以为其设置生存时间
        redis.call('expire',KEYS[1],ARGV[1])
    end
    if times >tonumber(ARGV[2]) then
        return 0
    end
    return 1  

这段代码实现的功能与我们之前所做的类似，不过简洁了很多，即使不了解Lua语言也能猜出来大概意思。那么，该如何测试这个脚本呢？首先我们把这段代码存为ratelimiting.lua然后在命令行输入：  
redis-cli --eval /path/to/ratelimiting.lua rate.limiting:127.0.0.1 , 10 3，其中--eval参数是告诉redis-cli读取并运行后面的Lua脚本，/path/to/ratelimiting.lua是ratelimiting.lua文件的位置，后面跟着的是传给Lua脚本的参数。其中“，”前的rate.limiting:127.0.0.1是要操作的键，可以在脚本中使用KEYS[1]获取，“，”后面的10和3是参数，在脚本中能够使用ARGV[1]/ARGV[2]获得。结合脚本的内容可知这行命令的作用是将访问频率限制为每10秒最多3次，所以在终端中不断地运行此命令会发现当访问频率在10秒内小于或等于3次时返回1，否则返回0。  
**注意：上面命令中的“，”两边的空格不能省略，否则会出错。**  

## Lua 语法学习  
请参见一些网络学习地址或学习书籍，本人收集地址如下：
> [Lua程序设计](http://book.luaer.cn/)  
> [Lua在线手册](http://manual.luaer.cn/)  
> [Lua WIKI](http://lua-users.org/wiki/)  
> [GitHub Lua教程](https://github.com/wenquan0hf/lua/blob/master/TOC.md)  
> [Lua菜鸟教程](http://www.runoob.com/lua/lua-tutorial.html)  

## Redis与Lua  
编写Redis脚本的目的就是读写Redis的数据，本章主要介绍Redis与Lua交互的方法。  
### 1、在脚本中调用Redis命令  
在脚本中可以使用redis.call函数调用Redis命令。就像这样：  

    redis.call('set','foo','bar')
    local value=redis.call('get','foo') --value的值为bar

redis.call函数的返回值就是Redis命令的执行结果。Redis命令的返回值有5种类型，redis.call函数会将这5种类型的回复转换成对应的Lua的数据类型，具体的对应规则如下表（空结果比较特殊，其对应为Lua的false）。 
 
Redis返回值类型|Lua数据类型
---|---
整数回复|数字类型
字符串回复|字符串类型
多行字符串回复|表类型（数组形式）
状态回复|表类型（只有一个ok字段存储状态信息）
错误回复|表类型（只有一个err字段存储错误信息）  

Redis还提供了了redis.pcall函数，功能与redis.call相同，唯一区别是当命令执行出错时redis.pcall会记录错误并继续执行，而redis.call会直接返回错误，不会继续执行。  
### 2、从脚本中返回值  
在很多情况下都需要脚本返回值，比如前面的访问频率限制脚本会返回频率是否超限。在脚本中可以使用return语句将值返回给客户端，如果没有执行return语句则会默认返回nil。因为我们可以向调用其他Redis内置命令一样调用我们自己写的脚本，所以同样Redis会自动将脚本返回值的Lua数据类型转换成Redis的返回值类型。具体的转换规则见下表（其中Lua的false比较特殊，会被转换成空结果）。  
  
Lua数据类型|Redis返回值类型
---|---
数字类型|整数回复（Lua的数字类型会被自动转换成整数）
字符串类型|字符串回复
表类型（数组形式）|多行字符串回复
表类型（只有一个ok字段存储状态信息）|状态回复
表类型（只有一个err字段存储错误信息）|错误回复  

### 3、脚本相关命令  
#### 1.EVAL命令  
编写完脚本后最重要的就是在程序中执行脚本。Redis提供了EVAL命令可以使开发者像调用其他Redis内指命令一样调用脚本。EVAL命令的格式是：EVAL 脚本内容 key参数的数量 [key...] [arg ...]。可以通过key和arg这两类参数向脚本传递数据，他们的值可以在脚本中分别使用KEYS和ARGV两个类型的全局变量访问。比如希望用脚本功能实现一个SET命令,脚本内容是这样：  


    return redis.call('SET',KEYS[1],ARGV[1])  
    
现在打开redis-cli执行此脚本

    127.0.0.1:6379> eval "return redis.call('SET',KEYS[1],ARGV[1])" 1 foo bar
    OK
    127.0.0.1:6379> get foo
    "bar"  
    
其中要读写的键名应该作为key参数，其他的数据都作为arg参数。  
**注意：EVAL命令依据第二个参数将后面的所有参数分别存入脚本中的KEYS和ARGV两个表类型的全局变量中。当脚本不需要任务参数时也不能省略这个参数（设为0）**。  

#### 2.EVALSHA命令  
考虑到在脚本比较长的情况下，如果每次调用脚本都需要将这个脚本传给Redis会占用较多的带宽。为了解决这个问题，Redis提供了EVALSHA命令允许开发者通过脚本内容的SHA1摘要来执行脚本，改命令的用法和EVAL一样，只不过是将脚本内容替换成脚本内容的SHA1摘要。  
Redis在执行EVAL命令时会计算脚本的SHA1摘要并记录在脚本缓存中，执行EVALSHA命令时Redis会根据提供的摘要从脚本缓存中查找对应的脚本内容，如果找到了则执行脚本，否则会返回错误：“NOSCRIPT No matching script.Please use EVAL.”,在程序中使用EVALSHA命令的一般流程：  
（1）先计算脚本的SHA1摘要，并使用EVALSHA命令执行脚本。  
（2）获得返回值，如果返回“NOSCRIPT”错误则使用EVAL命令重新执行脚本。  
虽然这一流程略显麻烦，但值得庆幸的是很多编程语言的Redis客户端都会代替开发者完成这一流程。比如使用node\_redis客户端执行EVAL命令时，node\_redis会先尝试执行EVALSHA命令，如果失败才会执行EVAL命令。  

## 深入脚本  
### 1、KEYS与ARGV  
前面提到过向脚本传递参数分为KEYS和ARGV两类，前者表示要操作的键名，后者表示非键名参数。但事实上这一要求并不是强制的，比如 EVAL "return redis.call('get',KEYS[1])" 1 user:Bob可以获得user:Bob的键值，同样还可以使用EVAL "return redis.call('get','user:' .. ARGV[1])" 0 Bob完成同样的功能，此时我们虽然并未按照Redis的规则使用KEYS参数传递键名，但还是获得了正确的结果。  
虽然规则不是强制的，但不遵守规则依然有一定代价。Redis3.0版带有集群的功能，集群的作用是将数据库中的键分散到不同的节点上。这意味着在脚本执行前就需要知道脚本会操作哪些键以便于找到对应的节点，所以如果脚本中的键名没有使用KEYS参数传递则无法兼容集群。  
### 2、沙盒与随机数  
Redis脚本禁止使用Lua标准库中与文件或系统调用相关函数，在脚本中只允许对Redis的数据进行处理。并且Redis还通过禁用脚本的全局变量的方式保证每个脚本都要是相对隔离的，不会相互干扰。  
使用沙盒不仅是为了保证服务器的安全性，而且还确保了脚本的执行结果只和脚本本身和执行时传递的参数有关，不依赖外界条件（如 系统时间、系统中某个文件内容、其他脚本执行结果等。）这是因为在执行复制和AOF持久化操作时记录的是脚本的内容而不是脚本调用命令，所以必须保证在脚本内容和参数一样的前提下的执行结果是一样的。  
### 3、其他脚本相关命令  
除了EVAL和EVALSHA外，Redis还提供了了其他4个脚本相关的命令，一般都会被客户端封装起来，开发者很少能使用到。  
（1）、将脚本加入缓存：SCRIPT LOAD  
每次执行EVAL命令时Redis都会将脚本的SHA1摘要加入到脚本缓存中，以便下次客户端可以使用EVALSHA命令调用该脚本。如果只是希望将脚本加入脚本缓存而不执行则可以使用SCRIPT LOAD命令，返回值是脚本的SHA1摘要。如：  

    127.0.0.1:6379> script load "return 1"
    "e0e1f9fabfc9d4800c877a703b823ac0578ff8db"  

（2）、判断脚本是否已经被缓存：SCRIPT EXISTS  
SCRIPT EXISTS 命令可以同时查找1个或多个脚本的SHA1摘要是否被缓存，如：  

    127.0.0.1:6379> script exists e0e1f9fabfc9d4800c877a703b823ac0578ff8db
    1) (integer) 1  
    
（3）、清空脚本缓存：SCRIPT FLUSH  
Redis将脚本的SHA1摘要加入到脚本缓存后会永久保留，不会删除，但可以手动使用SCRIPT FLUSH命令清空脚本缓存：  

    127.0.0.1:6379> script flush
    OK  
    
（4）、强制终止当前脚本的执行：SCRIPT KILL  
如果想终止当前正在执行的脚本可以使用SCRIPT KILL命令。  

### 4、原子性和执行时间  
Redis的脚本执行时原子的，即脚本执行期间Redis不会执行其他命令。所有的命令都必须等待脚本执行完成后才能执行。为了防止某个脚本执行时间过长导致Redis无法提供服务（比如陷入死循环），Redis提供了**lua-time-limit**参数限制脚本的最长运行时间，默认为5秒钟。当脚本运行时间超过这一限制后，Redis将开始接受其他命令但不会执行（以确保脚本的原子性，因为此时脚本并没有被终止），而是会返回“BUSY”错误。限制我们可以打开2个redis-cli实例A和B来测试一下。首先A中执行一个死循环脚本：  

    127.0.0.1:6379> eval "while true do end" 0  --死循环  

然后在另一个B客户端执行一条命令:  

    127.0.0.1:6379> get foo
    (error) BUSY Redis is busy running a script. You can only call SCRIPT KILL or SHUTDOWN NOSAVE.  
    
这时实例B中命令并没有马上返回结果，因为、Redis已经被实例A发送的死循环脚本阻塞了，无法执行其他命令。且等待5秒钟之后实例B收到了“BUSY”错误，此时Redis虽然可以接受任何命令，但实际会执行的只有两个命令：SCRIPT KILL 和SHUTDOWN NOSAVE。在实例B中执行SCRIPT KILL命令可以终止当前脚本的运行，并且此时实例A中会返回错误:  

    --实例A
    127.0.0.1:6379> script kill
    OK  
    
    --实例B
    127.0.0.1:6379> eval "while true do end" 0
    (error) ERR Error running script (call to f_694a5fe1ddb97a4c6a1bf299d9537c7d3d0f84e7): @user_script:1: Script killed by user with SCRIPT KILL... 
    (343.29s)

需要注意的是如果当前执行的脚本对Redis的数据进行了修改（如调用SET、LPUSH或DEL等命令）则SCRIPT KILL 命令不会终止脚本的运行以防止脚本只执行了一部分。因为如果脚本只执行了一部分就被终止，会违背脚本的原子性要求，即脚本中的所有命令都要么执行，要么都不执行。比如在实例A中执行：  

    127.0.0.1:6379> eval "redis.call('SET','foo','bar') while true do end" 0
    --死循环卡住  
    
5秒钟后尝试在B中终止该脚本：  

    127.0.0.1:6379> script kill
    (error) UNKILLABLE Sorry the script already executed write commands against the dataset. You can either wait the script termination or kill the server in a hard way using the SHUTDOWN NOSAVE command.  
    
这是只能通过SHUTDOWN NOSAVE命令强制终止Redis。SHUTDOWN NOSAVE命令与SHUTDOWN命令的区别在于前者将不会进行持久化操作，这意味着所有发生在上一次快照后的数据库修改都会丢失。由于Redis脚本非常高效，所以在大部分情况下都不用担心脚本的性能。但同时由于脚本的强大功能，很多原本在程序中执行的逻辑都可以放到脚本中执行，这时就需要开发者根据具体的应用权衡到底哪些任务适合交给脚本。通常来说不应该在脚本中进行大量耗时的运算，因为毕竟Redis是单进程单线程执行脚本，而程序却能够多进程多线程运行。  
