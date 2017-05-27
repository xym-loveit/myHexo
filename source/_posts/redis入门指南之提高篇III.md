---
title: redis入门指南之提高篇III
date: 2017-05-26 18:21:49
categories: redis系列
tags: [Redis任务队列,Redis优先级队列,Redis发布订阅,Redis管道概念,Redis键值存储详解]
description: Redis中的任务队列（消费者/生产者模式）、发布订阅的使用、从Redis节省空间到详解Redis存储数据结构（节约内存必读）。
---

###### 重要星级 ★★★★★

---
## 消息通知  
任务队列顾名思义，就是“传递任务的队列”。与任务队列进行交互的尸体有两类，一类是生产者（producer），一类是消费者（consumer）。生产者会将需要处理的任务放入队列中，而消费者则不断地从队列中读入任务信息并执行。  
使用任务队列有如下好处：  
（1）松耦合。生产者和消费者无需知道彼此的实现细节，只需要约定好任务的描述格式。这使得生产者和消费者可以由不同的团队使用不同的编程语言编写。  
（2）易于扩展消费者可以有多个，而且可以分布在不同的服务器中，借此可以轻易地降低单台服务器的负载。

![多个消费者消费生产者放入队列的任务](http://op7wplti1.bkt.clouddn.com/producerandConsumer.png)

### 1、Redis实现任务队列  
说到任务队列自然想到之前介绍的LPUSH和RPOP命令实现队列的概念。如果要实现队列，只需要让生产者将任务使用LPUSH命令加入某个键中，另一边让消费者不断地使用RPOP命令从该键中取出任务即可。消费者伪代码如下：  

    loop //无限循环读取任务队列中的内容
        $task=RPOP queue
        if $task
            //如果任务队列中有任务执行它
            execute($task)
        else
            //如果没有则等待1秒以免过于频繁请求数据
            wait 1 second

到此一个使用Redis实现的简单任务队列就写好了。不好还有一点不完美的地方:当任务队列中没有任务时消费者每秒都会调用一次RPOP命令查看是否有新任务。如果可以实现一旦有新任务队列就通知消费者就好了。其实借助BRPOP命令就可以实现这一的需求。  
BRPOP命令和RPOP命令相似，唯一的区别是当列表中没有元素时BRPOP命令会一直阻塞住连接，知道有新元素加入。如上代码可改为： 

    loop
        //如果任务队列中没有新任务，BRPOP命令会一直阻塞，不会执行execute()。
        $task=BRPOP queue,0
        //返回值是数组，数组第二个元素时我们需要的任务。
        execute($task[1])

BRPOP命令接受2个参数，第一个是键名，第二个是超时时间，单位是秒。当超过了此时间仍然没有获得新元素就会返回nil。上例中超时时间为“0”，表示不限制等待的时间，即如果没有新元素加入列表就会永远组塞下去。  
当获得一个元素后BPOP命令返回二个值，分别是键名和元素值。为了测试BPOP命令，我们可以打开2个redis-cli实例，在实例A中：  

    127.0.0.1:6379> brpop queue 0

键入回车后实例1会处于阻塞状态，这时在实例B中向queue中加入一个元素： 

    127.0.0.1:6379> lpush queue task
    (integer) 1

在LPUSH命令执行后实例A马上就返回了结果： 

    127.0.0.1:6379> brpop queue 0
    1) "queue"
    2) "task"
    (73.70s)
    
同时会发现queue中的元素已经被取走： 

    127.0.0.1:6379> llen queue
    (integer) 0

除了BRPOP命令外，Redis还提供了BLPOP，和BRPOP的区别在于从队列取元素时BLPOP会从左边取。  

### 2、优先级队列  

BRPOP命令可以同时接受多个键，其完整的命令格式为BRPOP key [key ...] timeout，如BRPOP queue:1 queue:2 0。意义是同时检测多个键，如果所有键都没有元素则阻塞，如果其中有一个键有元素则会从该键中弹出元素。例如，打开两个redis-cli实例，在实例A中:  

    127.0.0.1:6379> BLPOP queue:1 queue:2 queue:3 0
    
在实例B中：  

    127.0.0.1:6379> lpush queue:2 task
    (integer) 1
    
则实例A中返回： 

    1) "queue:2"
    2) "task"
    (15.54s)

如果多个键都有元素则按照从左到右的顺序取第一个的一个元素。我们现在queue:2和queue:3中各加入一个元素： 

    127.0.0.1:6379> lpush queue:2 task2
    (integer) 1
    127.0.0.1:6379> lpush queue:3 task3
    (integer) 1

然后执行BRPOP命令：  

    127.0.0.1:6379> BRPOP queue:1 queue:2 queue:3 0
    1) "queue:2"
    2) "task2"

借此特性可以实现区分优先级的任务队列。我们分别使用queue:confirm.email和queue：notification.email两个键存储发送确认邮件（注册网站需要发送确认邮箱正确性邮件）和发送通知邮件（一旦有新博客文章就发送邮件信息提醒）两种任务，然后将消费者的代码改为： 

    loop
        $task=BRPOP queue:confirm.email,queue:notification.email,0
        execute($task[1])

这时一旦发送确认邮件的任务被加入到queue:confirm.email队列中，无论queue:notification.email还有多少任务，消费者都会优先完成发送确认邮件的任务。  

### 3、发布/订阅模式  
除了实现任务队列外，Redis还提供了一组命令可以让开发者实现“发布/订阅”（publish/subscribe）模式。“发布/订阅”模式同样可以实现进程间的消息传递，其原理是这样的：  
“发布/订阅”模式中包含两种角色，分别是发布者和订阅者。订阅者可以订阅一个或若干个频道（channel），而发布者可以向指定的频道发送消息，所有订阅此频道的订阅者都会收到此消息。发布者发布消息的命令是publish，用法是publish channel message，如向channel1.1发送“hello”： 

    127.0.0.1:6379> publish channel1.1 hello
    (integer) 0

这样消息就发出去了。PUBLISH命令的返回值表示接收到这条消息的订阅者数量。因为此时没有客户端订阅channel1.1，所以返回0。发出去的消息不会被持久化，也就是说当有客户端订阅channel1.1后只能收到后续发布到该频道的消息，之前发送的就收不到了。订阅频道的命令时SUBSCRIBE，可以同时订阅多个频道，用法是SUBSCRIBE channel [channel ...]。现在新开一个redis-cli实例A，用它来订阅channel1.1： 

    127.0.0.1:6379> subscribe channel1.1
    Reading messages... (press Ctrl-C to quit)
    1) "subscribe"
    2) "channel1.1"
    3) (integer) 1

执行SUBSCRIBE命令后客户端会进入订阅状态，处于此状态下的客户端不能使用除SUBSCRIBE/UNSUBSCRIBE/PSUBSCRIBE/PUNSUBSCRIBE这4个属于“发布/订阅”模式的命令之外的其他命令，否则会报错。  
进入订阅状态后客户端可能收到三种类型的回复。每种类型的回复都包含3个值，第一个值是消息的类型，根据消息类型的不同，第二、第三个值的含义也不同。消息类型可能取值有：  
（1）Subscribe。表示订阅成功的反馈信息。第二个值是订阅成功的频道名称，第三个值是当前客户端订阅的频道数量。  
（2）message。这个类型的回复使我们最关心的，它表示接受到的消息。第二个值表示产生消息的频道名称，第三个是消息的内容。  
（3）unsubscribe。表示成功取消订阅某个频道。第二个值是对应的频道名称，第三个值是当前客户端订阅频道的数量，当此值为0时客户端会退出订阅状态，之后就可以执行其他非“发布/订阅”模式的命令了。  
上例中当实例A订阅了channel1.1进入订阅状态后收到了一条subscribe类型的回复，这时我们打开另一个redis-cli实例B，并向channel1.1发送一条消息：  

    127.0.0.1:6379> publish channel1.1 hello
    (integer) 1

返回值为1表示有一个客户端订阅了channel1.1，此时实例A收到了类型为message的回复：  
127.0.0.1:6379> subscribe channel1.1

    Reading messages... (press Ctrl-C to quit)
    1) "subscribe"
    2) "channel1.1"
    3) (integer) 1
    1) "message"
    2) "channel1.1"
    3) "hello"

使用UNSUBSCRIBE命令可以取消订阅指定的频道，用法为UNSUBSCRIBE [channel [channel...]]，如果不指定频道则会取消订阅所有频道。  

### 4、按照规则订阅  
除了可以使用SUBSCRIBE命令订阅指定名称的频道外，还可以使用PSUBSCRIBE命令订阅指定的规则。规则支持glob风格通配符格式，如：  

    127.0.0.1:6379> psubscribe channel.?*
    Reading messages... (press Ctrl-C to quit)
    1) "psubscribe"
    2) "channel.?*"
    3) (integer) 1

规则channel.?*可以匹配channel.1和channel.10但不会匹配channel.。这时在实例B中发送消息：  

    127.0.0.1:6379> publish channel.1 hello
    (integer) 2

返回结果是2因为2个实例客户端都订阅了channel.1频道。实例psubscribe channel.?\*收到的回复是：  

    1) "pmessage"
    2) "channel.?*"
    3) "channel.1"
    4) "hello"

第一个值表示这条消息是通过PSUBSCRIBE命令订阅频道而收到的======，第二个值表示订阅时用的通配符，第三个值表示实际收到的消息的频道，第四个值则是消息内容。 

提示：使用PSUBSCRIBE命令可以重复订阅一个频道，如某客户端执行了PSUBSCRIBE channel.? channel.?\*，这时向channel.2发布消息后该客户端会收到2条消息，而同时PUBLISH命令返回的值也是2而不是1。同样的，如果有另一个客户端执行了SUBSCRIBE channel.10和PSUBSCRIBE channel.?\*的话，向channel.10发送命令该客户端也会收到两条消息（但是是两种类型，message和pmessage），同时publish命令会返回2。  
PUNSUBSCRIBE命令可以推定指定的规则，用法是PUNSUBSCRIBE [pattern [pattern ...]]，如果没有参数则会退订所有规则。  

**注意：使用PUNSUBSCRIBE命令只能退订通过PSUBSCRIBE命令订阅的规则，不会影响直接通过SUBSCRIBE命令订阅的频道；同样UNSUBSCRIBE命令也不会影响通过PSUBSCRIBE命令订阅的规则。另外容易出错的是使用PUNSUBSCRIBE命令退订某个规则时不会将其中的通配符展开，而是进行严格的字符串匹配，所以PUNSUBSCRIBE \* 无法退订channel.\* 规则，而是必须使用PUNSUBSCRIBE channel.*才能退订。**  

## 管道  
客户端和Redis使用TCP协议连接。不论是客户端向Redis发送命令还是Redis向客户端返回命令的执行结果，都需要经过网络传输，这两个部分的总耗时称为往返时延。根据网络性能不同，往返时延也不同，大致来说到本地回环地址（loop back address）的往返时延在数量级上相当于Redis处理一条简单命令（如LPUSH list 1 2 3）的时间。如果执行较多的命令，每个命令的往返时延累加起来对性能还是有一定影响的。  
在执行多个命令时每条命令都需要等待上一条命令执行完（即收到Redis的返回结果）才能执行，即使命令不需要上一条命令的执行结果。如要获得post:1、post:2和post:3这3个键中的title字段，需要执行三条命令，如下图：  

![不使用管道时多条命令执行示意图](http://op7wplti1.bkt.clouddn.com/executeCommand.png)  

Redis的底层通信协议对管道（pipelining）提供了支持。通过管道可以一次性发动多条命令并在执行完后一次性将结果返回，当一组命令中每条命令都不依赖于之前命令的执行结果时就可以将这组命令一起通过管道发出。管道通过减少客户端与Redis的通信次数来实现降低往返时延累计值的目的，如下图：  

![管道执行多条命令示意图](http://op7wplti1.bkt.clouddn.com/pipeExecute.png)  

## 节省空间  
相比于硬盘而言，内存在今天仍然显得比较昂贵。而Redis是一个基于内存的数据库，所有的数据都存储在内存中，所以如何优化存储，减少内存空间占用对成本控制来说是一个非常重要的话题。  
### 1、精简键名和键值  
精简键名和键值是最直观的减少内存占用的方式，如将键名very.important.person:20改成VIP:20。当然精简键名一定要把握尺度，不能单纯为了节约空间而使用不易理解的键名（比如将VIP:20修改为V:20，这样既不易维护，还容易造成命名冲突）。又比如一个存储用户性别的字符串类型键的取值是male和female，我们可以将其修改成m和f来为每条记录节约几个字节空间（更好的办法是使用0和1来表示性别）。  
### 2、内部编码优化  
有时候仅凭精简键名和键值所减少的空间并不足以满足需求，这时就需要根据Redis内部的编码规则来节省更多的空间。Redis为每种数据类型都提供了两种内部编码方式，以散列类型为例，散列类型是通过散列表实现，这样就可以实现O（1）时间复杂度的查找、赋值操作，然而当键中元素很少的时候，O（1）的操作并不会比O（n）有明显的性能提高，所以这种情况下Redis会采用一种更为紧凑但性能稍差（获取元素的时间复杂度为O（n））的内部编码方式。内部编码方式的选择对于开发者来说是透明的，Redis会根据实际情况自动调整。当键中元素变多时Redis会自动将该键的内部编码方式转换成散列表。如果想查看一个键的内部编码方式可以使用OBJECT ENCODING 命令，如：  

    127.0.0.1:6379> type score:2
    string
    127.0.0.1:6379> get score:2
    "100"
    127.0.0.1:6379> object encoding score:2
    "int"
    127.0.0.1:6379> type post:5
    hash
    127.0.0.1:6379> object encoding post:5
    "ziplist"

Redis的每个键值都是使用一个redisObject结构体保存的，redisObject的定义如下：  

    typedef struct redisObject{
        unsigned type:4;
        unsigned notused:2;
        unsigned encoding:4;
        unsigned lru:22; /* lru time(relative to server.lruclock) */
        int refcount;
        void *ptr;
    } robj;
    
其中type字段表示的是键值的数据类型，取值可以是如下内容：  

    #define REDIS_STRING 0
    #define REDIS_LIST 1
    #define REDIS_SET 2
    #define REDIS_ZSET 3
    #define REDIS_HASH 4

encoding字段表示的就是Redis键值内部编码方式，取值如下：  

    #define REDIS_ENCODING_RAW 0 /* Raw representation */
    #define REDIS_ENCODING_INT 1 /* Encoded as integer */
    #define REDIS_ENCODING_HT 2  /* Encoded as hash table */
    #define REDIS_ENCODING_ZIPMAP 3 /* Encoded as zipmap */
    #define REDIS_ENCODING_LINKEDLIST 4 /* Encoded as regular linked list */
    #define REDIS_ENCODING_ZIPLIST 5 /* Encoded as ziplist */
    #define REDIS_ENCODING_INTSET 6 /* Encoded as intset */
    #define REDIS_ENCODING_SKIPLIST 7 /* Encoded as skiplist */
    
各个数据类型可能采用的内部编码方式及其相应的OBJECT ENCODING 命令执行结果如下：  

![Redis每种数据类型都采用二种编码方式中的一种存储](http://op7wplti1.bkt.clouddn.com/redisEncoding.png)

#### 1、字符串类型  
Redis使用一个sdshdr类型的变量来存储字符串，而redisObject的ptr字段指向的是该变量的地址。sdshdr的定义如下： 

    struct sdshdr{
        int len;
        int free;
        char buf[];
    }

其中len字段表示的是字符串的长度，free字段表示buf中的剩余空间，而buf字段存储的才是字符串的内容。所以当执行SET key foobar时，存储键值需要占用的空间是sizeof(redisObject)+sizeof(sdshdr)+strlen("foobar")=30字节，如下图：  

![字符串键值的存储结构](http://op7wplti1.bkt.clouddn.com/strstructure.png)  

而当键值内容可以用一个64位有符号整数表示时，Redis会将键值转换成long类型来存储。如SET key 123456，实际占用空间是sizeof(redisObject)=16字节，比存储“foobar”节省了一般的存储空间，如下图：  

![字符串键值的存储结构2](http://op7wplti1.bkt.clouddn.com/strstructure2.png)   

redisObject中的refcount字段存储的是该键值被引用数量，即一个键值可以被多个键引用。Redis启动后会预先建立10000个分别存储从0到9999这些数字的redisObject类型变量作为共享对象，如果要设置的字符串在这10000个数字内（如SET key1 123）则可以直接引用共享对象而不用再建立一个redisObject了，也就是说存储键值占用的空间是0字节，如下图：  

![Redis直接引用共享对象](http://op7wplti1.bkt.clouddn.com/shareObject.png)  

由此可见，使用字符串类型键存储对象ID这种小数字是非常节省存储空间的，Redis只需存储键名和一个对共享对象的引用即可。  
**提示：当通过配置文件参数maxmemory设置了Redis可用的最大空间大小时，Redis不会使用共享对象，因为对于每个键值都需要使用一个redisObject来记录其LRU信息。**  

#### 2、散列类型  
散列类型的内部编码方式可能是REDIS_ENCODING_HT或REDIS_ENCODING_ZIPLIST。在配置文件中可以定义使用REDIS_ENCODING_ZIPLIST方式编码散列表类型的时机：  

    hash-max-ziplist-entries 512
    hash-max-ziplist-value 64
    
当散列类型键的字段个数少于hash-max-ziplist-entries参数值且每个字段名和字段值的长度都小于hash-max-ziplist-value 参数值（单位为字节）时，Redis就会使用REDIS_ENCODING_ZIPLIST来存储该键，否则就会使用REDIS_ENCODING_HT。转换过程是透明的，每当键值变更后Redis都会自动判断是否满足条件来完成转换。  

REDIS_ENCODING_HT编码即散列表，可以实现O（1）时间复杂度的赋值取值等操作，其字段和字段值都是使用redisObject存储的，所以前面讲到的字符串类型键值的优化方法同样适用于散列类型键的字段和字段值。  

提示 Redis的键值对存储也是通过散列表实现的，与REDIS_ENCODING_HT编码方式类似，但键名并非使用redisObject存储，所以键名“123456”并不会比“abcdef”占用更少空间。之所以不对键名进行优化是因为绝大多数情况下键名都不会是纯数字。  

    补充知识 Redis支持多数据库，每个数据库中的数据都是通过结构体redisDb存储。redisDb的定义如下：  
    typedef struct redisDb{
        dict *dict; /* The keyspace for this DB */
        dict *expires; /* Timeout of keys with a timeout set */
        dict *blocking_keys; /* Keys with clients waiting for data (BLPOP) */
        dict *ready_keys; /* Blocked keys that received a PUSH */
        dict *watched_keys; /* WATCHED keys for MULTI/EXEC CAS */
        int id;
    } redisDb;
    dict类型就是散列表结构，expires存储的是数据的过期时间。当Redis启动时会根据配置文件中databases参数指定的数量创建若干个redisDb类型变量存储不同数据库中的数据。

REDIS\_ENCODING\_ZIPLIST编码类型是一种紧凑的编码格式，它牺牲了部分读取性能以换取极高的空间利用率，适合在元素较少的时使用。该编码类型同样还在列表类型和有序集合类型中使用。REDIS\_ENCODING\_ZIPLIST编码结构下图所示:  

![REDIS_ENCODING_ZIPLIST编码的内存结构](http://op7wplti1.bkt.clouddn.com/REDIS_ENCODING_ZIPLIST_bmjg.png)  
其中zlbytes是uint32\_t类型，表示整个结构占用的空间。zltail也是uint32\_t类型，表示到最后一个元素的偏移，记录zltail是的程序可以直接定位到尾部元素而无需遍历整个结构，执行从尾部弹出（对列表类型而言）等操作时速度更快。zllen是uint16_t类型，存储的是元素的数量。zlend是一个单字节标识，标记结构的末尾，值永远是255。  
在REDIS\_ENCODING\_ZIPLIST中每个元素由4个部分组成。第一个部分用来存储前一个元素的大小以实现倒序查找，当前一个元素的大小小于245字节时第一个部分占用1个字节，否则会占用5个字节。

第二、三个部分分别是元素的编码类型和元素大小，当元素的大小小于或等于63个字节时，元素的编码类型是ZIP\_STR\_06B（即0<<6），同时第三个部分用6个二进制位来记录元素的长度，所以第二、三个部分总占用空间是1字节。当元素的大小大于63且小于或等于16383字节时，第二、三个部分总占用空间是2字节。当元素的大小大于16383字节时，第二、三个部分总占用空间是5字节。  
第四个部分是元素的实际内容，如果元素可以转换成数字的话Redis会使用相应的数字类型来存储以节省空间，并用第二、第三个部分来表示数字的类型（int16_t/int32_t等）。使用REDIS_ENCODING_ZIPLIST编码存储散列类型时元素的排列方式是：元素1存储字段1，元素2存储字段值1，以此类推，如下图：  

![使用REDIS_ENCODING_ZIPLIST编码存储散列类型内存结构](http://op7wplti1.bkt.clouddn.com/REDIS_ENCODING_ZIPLIST_2.png)    

例如，当执行命令HSET hkey foo bar 命令后，hkey键值的内存结构如下图：  

![使用REDIS_ENCODING_ZIPLIST编码存储散列类型内存结构2](http://op7wplti1.bkt.clouddn.com/REDIS_ENCODING_ZIPLIST_3.png)   

下次需要执行HSET hkey foo anothervalue 时Redis需要从头开始找到值为foo的元素（查找时每次都会跳过一个元素以保证只查找字段名），找到后删除其下一个元素，并将新值anothervalue 插入。删除和插入都需要移动后面的内存数据，而且查找操作也需要遍历才能完成，可想而知当散列键中的数据多时性能将很低，所以不宜将**hash-max-ziplist-entries**和**hash-max-ziplist-value** 两个参数设置得很大。  

#### 3、列表类型  
列表类型的内部编码方式可能是REDIS_ENCODING_LINKEDLIST或REDIS_ENCODING_ZIPLIST。同样在配置文件中可以使用REDIS_ENCODING_ZIPLIST方式编码的时机:  

    list-max-ziplist-entries 512
    list-max-ziplist-value 64
    
具体转换方式和散列类型一样，REDIS_ENCODING_LINKEDLIST编码方式即双向链表，链表中的每个元素是用redisObject存储的，所以此种编码方式下的元素值的优化方法与字符串的键值相同。  
而是用REDIS_ENCODING_ZIPLIST编码方式时具体的表现和散列类型一样，由于REDIS_ENCODING_ZIPLIST编码方式同样支持倒序访问，所以采用此种编码方式时获取两端的数据依然较快。  

#### 4、集合类型  
集合类型的内部编码方式可能是REDIS_ENCODING_HT或REDIS_ENCODING_INTSET。当集中的所有元素都是整数且元素的个数小于配置文件中的set-max-intset-entries参数指定值（默认512）时Redis会使用REDIS_ENCODING_INTSET编码存储该集合，否则会使用REDIS_ENCODING_HT来存储。REDIS_ENCODING_INTSET编码存储结构体intset的定义如下： 

    typedef struct intset{
        uint32_t encoding;
        uint32_t length;
        int8_t contents[];
    } intset;

其中contents存储的就是集合中的元素值，根据encoding的不同，每个元素占用的字节大小不同。默认encoding是INTSET_ENC_INT16（即2个字节），当新增加的整数元素无法使用2个字节表示时，Redis会将该集合的encoding升级为INTSET_ENC_INT32（即2个字节）并调整之前所有元素的位置和长度，同样集合的encoding还可升级为INTSET_ENC_INT64（即8个字节）。  
REDIS_ENCODING_INTSET编码以有序的方式存储元素（所以使用SMEMBERS命令获得的结果是有序的），使得可以使用二分算法查找元素。然后无论是添加还是删除元素，Redis都需要调整后面元素的内存位置，所以当集合中元素太多时性能较差。当新增加的元素不是整数或集合中的元素数量超过了set-max-intset-entries参数指定值时，Redis会自动将该集合的存储结构转换成REDIS_ENCODING_HT。  

#### 5、有序集合类型  
有序集合类型的内部编码方式可能是REDIS_ENCODING_SKIPLIST或REDIS_ENCODING_ZIPLIST。同样在配置文件中可以使用REDIS_ENCODING_ZIPLIST方式编码的时机:  

    zset-max-ziplist-entries 128
    zset-max-ziplist-value 64

具体规则和散列表一样。当编码方式是REDIS_ENCODING_SKIPLIST时，Redis使用散列表和跳跃列表（skip list）两种数据结构来存储有序集合类型键值，其中散列表用来存储元素值与元素分数的映射关系以实现O（1）时间复杂度的ZSCORE等命令。跳跃列表用来存储元素的分数及其到元素值的映射以实现排序的功能。Redis对跳跃表的实现进行了几点修改，其中包括允许跳跃列表中的元素（即分数）相同，还有为跳跃链表每个节点增加了指向前一个元素的指针以实现倒序查找。采用此种编码方式时，元素值是使用redisObject存储的，所以可以使用字符串类型键值的优化方式优化元素值，而元素的分数是使用double类型存储的。  
使用REDIS_ENCODING_ZIPLIST编码时有序集合存储的方式按照“元素1的值，元素1的分数，元素2的值，元素2的分数”的顺序排列，并且分数是有序的。

