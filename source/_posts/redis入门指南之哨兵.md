---
title: redis入门指南之哨兵
date: 2017-06-02 16:25:25
categories: redis系列
tags: [redis哨兵监控,哨兵（sentinel）的实现原理,哨兵的配置]
description: 通过使用redis主从模式提升redis负载能力，减小单点故障的可能，阅读本章使用哨兵（sentinel）可以配置出更强劲的Redis集群，重中之重。
---

###### 重要星级 ★★★★★

---

## 哨兵  

Redis中的复制的原理和使用方式，在一个典型的一主多从的Redis系统中，从数据库在整个系统中起到了数据冗余备份和读写分离的作用。当主数据库遇到异常中断服务后，开发者可以通过手动的方式选择一个从数据库来升格为主数据库，以使得系统能够继续提供服务。然后整个过程相对麻烦且需要人工介入，难以实现自动化。为此，Redis2.8中提供了哨兵工具来实现自动化的系统监控和故障恢复功能。  
### 1、什么是哨兵  
顾名思义，哨兵的作用就是监控Redis系统的运行状况。它的功能包含以下两个。  
（1）监控主数据库和从数据库是否正常运行。  
（2）主数据库出现故障时自动将从数据库转换为主数据库。  
哨兵是一个独立的进程，使用哨兵的一个典型架构如下图:  

![哨兵监控](http://op7wplti1.bkt.clouddn.com/sentinelMonitor.png)  

在一个一主多从的Redis系统中，可以使用多个哨兵进行监控任务以保证系统足够稳健，如下图所示。注意，此时不仅哨兵会同时监控主数据库和从数据库，哨兵之间也会相互监控。  

![一主多从Redis集群多哨兵监控](http://op7wplti1.bkt.clouddn.com/multiSentinel.png)  

### 2、实例讲解  
在理解哨兵的原理前，我们首先实际使用一下哨兵，来了解哨兵是如何工作的。为了简单起见，我们将建立三个Redis的集群（一主两从）。我们使用Redis命令行客户端来获取复制状态，以保证复制配置正确。  
首先是主数据库：  

    --236服务器
    127.0.0.1:6379> info replication
    # Replication
    role:master
    connected_slaves:2
    slave0:ip=192.168.100.237,port=6379,state=online,offset=151945,lag=1
    slave1:ip=192.168.100.238,port=6379,state=online,offset=152104,lag=0
    master_repl_offset:152104
    repl_backlog_active:1
    repl_backlog_size:1048576
    repl_backlog_first_byte_offset:2
    repl_backlog_histlen:152103  

可见其连接了两个从数据库，配置正确。然后用相同的方法查看两个从数据库的配置：  

    --237服务器
    127.0.0.1:6379> info replication
    # Replication
    role:slave
    master_host:192.168.100.236
    master_port:6379
    master_link_status:up
    master_last_io_seconds_ago:0
    master_sync_in_progress:0
    slave_repl_offset:159045
    slave_priority:100
    slave_read_only:1
    connected_slaves:0
    master_repl_offset:0
    repl_backlog_active:0
    repl_backlog_size:1048576
    repl_backlog_first_byte_offset:0
    repl_backlog_histlen:0  
    --238服务器
    127.0.0.1:6379> info replication
    # Replication
    role:slave
    master_host:192.168.100.236
    master_port:6379
    master_link_status:up
    master_last_io_seconds_ago:1
    master_sync_in_progress:0
    slave_repl_offset:160972
    slave_priority:100
    slave_read_only:1
    connected_slaves:0
    master_repl_offset:0
    repl_backlog_active:0
    repl_backlog_size:1048576
    repl_backlog_first_byte_offset:0
    repl_backlog_histlen:0  

当出现的信息如上时，即证明一主二从的配置已经成功了。接下来配置哨兵。建立一个配置文件，如sentinel.conf，内容为：  

    sentinel monitor mymaster 192.168.100.236 6379 1  
    
其中mymaster表示要监控的主数据库的名字，可以自己定义一个。这个名字必须仅有大小写字母、数字和“.-_”这三种字符组成。后面2个参数表示主数据库的地址和端口号，这里我们要监控的是主数据库236。最后一个1表示最低通过票数。接下来启动sentinel进程，并将上述配置的路径传递给哨兵：  

    [root@KFCS1 src]# ./redis-sentinel ../sentinel.conf  

需要注意的是，配置哨兵监控一个系统时，只需要配置其监控主数据库即可，哨兵会自动发现所有复制该主数据库的从数据库。启动哨兵后，哨兵输出内容如下：  

    Sentinel ID is 5f9becd72d6c4e8d7e5c0a06836b1b79a8ad0550
    14246:X 02 Jun 17:39:46.909 # +monitor master mymaster 192.168.100.236 6379 quorum 1
    14246:X 02 Jun 17:39:46.911 * +slave slave 192.168.100.238:6379 192.168.100.238 6379 @ mymaster 192.168.100.236 6379
    14246:X 02 Jun 17:39:46.938 * +slave slave 192.168.100.237:6379 192.168.100.237 6379 @ mymaster 192.168.100.236 6379

其中+slave表示新发现了从数据库，可见哨兵成功地发现了两个从数据库。现在哨兵已经在监控这3个Redis实例了，这时我们将主数据库关闭（杀死进程或使用SHUTDOWN命令），等待指定时间后（可以配置，默认为30秒），哨兵会输出如下内容：  

    14246:X 02 Jun 17:40:34.699 # +sdown master mymaster 192.168.100.236 6379
    14246:X 02 Jun 17:40:34.700 # +odown master mymaster 192.168.100.236 6379 #quorum 1/1  
    
其中+sdown表示哨兵主观认为主数据库停止服务了，而+odown则表示哨兵客观认为主数据库停止服务了，关于主观和客观的区别后文会详细介绍。此时哨兵开始执行故障恢复，即挑选一个从数据库，将其升格为主数据库，输出如下：  

    14246:X 02 Jun 17:40:34.700 # +try-failover master mymaster 192.168.100.236 6379
    14246:X 02 Jun 17:40:34.714 # +vote-for-leader 5f9becd72d6c4e8d7e5c0a06836b1b79a8ad0550 2
    14246:X 02 Jun 17:40:34.714 # +elected-leader master mymaster 192.168.100.236 6379
    14246:X 02 Jun 17:40:34.714 # +failover-state-select-slave master mymaster 192.168.100.236 6379
    14246:X 02 Jun 17:40:34.815 # +selected-slave slave 192.168.100.238:6379 192.168.100.238 6379 @ mymaster 192.168.100.236 6379
    14246:X 02 Jun 17:40:34.815 * +failover-state-send-slaveof-noone slave 192.168.100.238:6379 192.168.100.238 6379 @ mymaster 192.168.100.236 6379
    14246:X 02 Jun 17:40:34.891 * +failover-state-wait-promotion slave 192.168.100.238:6379 192.168.100.238 6379 @ mymaster 192.168.100.236 6379
    14246:X 02 Jun 17:40:35.072 # +promoted-slave slave 192.168.100.238:6379 192.168.100.238 6379 @ mymaster 192.168.100.236 6379
    14246:X 02 Jun 17:40:35.072 # +failover-state-reconf-slaves master mymaster 192.168.100.236 6379
    14246:X 02 Jun 17:40:35.134 * +slave-reconf-sent slave 192.168.100.237:6379 192.168.100.237 6379 @ mymaster 192.168.100.236 6379
    14246:X 02 Jun 17:40:36.104 * +slave-reconf-inprog slave 192.168.100.237:6379 192.168.100.237 6379 @ mymaster 192.168.100.236 6379
    14246:X 02 Jun 17:40:36.104 * +slave-reconf-done slave 192.168.100.237:6379 192.168.100.237 6379 @ mymaster 192.168.100.236 6379
    14246:X 02 Jun 17:40:36.186 # +failover-end master mymaster 192.168.100.236 6379
    14246:X 02 Jun 17:40:36.186 # +switch-master mymaster 192.168.100.236 6379 192.168.100.238 6379
    14246:X 02 Jun 17:40:36.186 * +slave slave 192.168.100.237:6379 192.168.100.237 6379 @ mymaster 192.168.100.238 6379
    14246:X 02 Jun 17:40:36.187 * +slave slave 192.168.100.236:6379 192.168.100.236 6379 @ mymaster 192.168.100.238 6379  
    
+try-failover表示哨兵开始进行故障恢复，+failover-end表示哨兵完成故障恢复，期间涉及的内容比较复杂，包括领头哨兵的选举、备选从数据库的选择等，放到后面介绍，此处只需要关注最后3条输出。+switch-master表示主数据库从236转换到了238，即238服务器升格为主数据库，同时两个+slave则列出了新的主数据库的2个从数据库。其中236就是之前停止服务的主数据库，可见哨兵并没有彻底清除停止服务的实例信息，这是因为停止服务的实例有可能会在之后的某个时间恢复服务，这时哨兵会让其重新加入进来，所以当实例停止服务后，哨兵会更新该实例的信息，使得当其重新加入后可以按照当前信息继续对外提供服务。此例中236主数据库实例停止服务了，而238服务器的从数据库已经升格为主数据库，当236实例恢复服务后，会转变为238实例的从数据库来运行，所以哨兵将236服务器实例的信息修改成了238实例的从数据库。  
故障恢复完成后，可以使用Redis命令行客户端重新检查237/238两台服务器上的实例信息：  

    --237服务器
    127.0.0.1:6379> info replication
    # Replication
    role:slave
    master_host:192.168.100.238
    master_port:6379
    master_link_status:up
    master_last_io_seconds_ago:0
    master_sync_in_progress:0
    slave_repl_offset:308379
    slave_priority:100
    slave_read_only:1
    connected_slaves:0
    master_repl_offset:0
    repl_backlog_active:0
    repl_backlog_size:1048576
    repl_backlog_first_byte_offset:0
    repl_backlog_histlen:0
    --238服务器
    127.0.0.1:6379> info replication
    # Replication
    role:master
    connected_slaves:1
    slave0:ip=192.168.100.237,port=6379,state=online,offset=311494,lag=1
    master_repl_offset:311494
    repl_backlog_active:1
    repl_backlog_size:1048576
    repl_backlog_first_byte_offset:2
    repl_backlog_histlen:311493  
    
可以看到238服务器上实例已经确实升格为主数据库了，同时237服务器上的实例是其从数据库。整个故障恢复过程就此完成。那么我们重新启动236服务器上的Redis实例，监控到的日志输出如下:  

    14246:X 02 Jun 17:42:27.637 # -sdown slave 192.168.100.236:6379 192.168.100.236 6379 @ mymaster 192.168.100.238 6379
    14246:X 02 Jun 17:42:37.643 * +convert-to-slave slave 192.168.100.236:6379 192.168.100.236 6379 @ mymaster 192.168.100.238 6379

-sdown表示实例236已经恢复服务了（与+sdown相反）同时+convert-to-slave表示将236服务器的实例设置为238服务器实例的从数据库。这时使用Redis命令行客户端查看236实例的复制信息为：  

    127.0.0.1:6379> info replication
    # Replication
    role:slave
    master_host:192.168.100.238
    master_port:6379
    master_link_status:up
    master_last_io_seconds_ago:0
    master_sync_in_progress:0
    slave_repl_offset:333678
    slave_priority:100
    slave_read_only:1
    connected_slaves:0
    master_repl_offset:0
    repl_backlog_active:0
    repl_backlog_size:1048576
    repl_backlog_first_byte_offset:0
    repl_backlog_histlen:0  

同时238端口的复制信息为:  

    127.0.0.1:6379> info replication
    # Replication
    role:master
    connected_slaves:2
    slave0:ip=192.168.100.237,port=6379,state=online,offset=311494,lag=1
    slave1:ip=192.168.100.236,port=6379,state=online,offset=311494,lag=0
    master_repl_offset:311494
    repl_backlog_active:1
    repl_backlog_size:1048576
    repl_backlog_first_byte_offset:2
    repl_backlog_histlen:311493

正如预期一样，238实例的从数据库变为了2个，236成功恢复服务。

### 3、实现原理  
一个哨兵进程启动时会读取配置文件的内容，通过如下的配置找出需要监控的主数据库：  

    sentinel monitor master-name ip redis-port quorum

其中master-name是一个由大小写字母、数字和“.-_”组成的主数据库的名字，因为考虑到故障恢复后当前监控的系统的主数据库的地址和端口号会产生变化，所以哨兵提供了命令可以通过主数据库的名字获取当前系统的主数据库的地址和端口号。  
ip 表示当前系统中主数据库的地址，而redis-port则表示端口号。  
quorum用来表示执行故障恢复操作前至少需要几个哨兵节点同意。一个哨兵节点可以同时监控多个Redis主从系统，只需要提供多个sentinel monitor 配置即可，例如：  

    sentinel monitor mymaster 127.0.0.1 6379 2
    sentinel monitor othermaster 192.168.100.238 6379 1

同时多个哨兵节点也可以同时监控同一个Redis主从系统，从而形成网状结构。具体实践时如何协调哨兵与主从系统的数量关系将在后文介绍。  
配置文件中还可以定义其他监控相关的参数，每个配置选项都包含主数据的名字使得监控不同主数据库时可以使用不同的配置参数。如：  

    sentinel down-after-milliseconds mymaster 60000
    sentinel down-after-milliseconds othermaster 10000
    
上面的两行配置分别配置了mymaster和othermaster的sentinel down-after-milliseconds选项分别为60000和10000。  
哨兵启动后，会与要监控的主数据库建立两条连接，这两条连接的建立方式与普通的Redis客户端无异。其中一条连接用来订阅该主数据库的\_\_sentinel\_\_:hello频道以获取其他同样监控该数据库的哨兵节点的信息，另外哨兵也需要定期向主数据库发送INFO等命令来获取主数据库本身的信息，因为之前介绍过当客户端的连接进入订阅模式时就不能再执行其他命令了，所以这时哨兵会使用另外一条连接来发送这些命令。和主数据库的连接建立完成后，哨兵会定时执行下面3个操作。  
（1）每10秒哨兵会向主数据库和从数据库发送INFO命令。  
（2）每2秒哨兵会向主数据库和从数据库的\_\_sentinel\_\_:hello频道发送自己的信息。  
（3）每1秒哨兵会向主数据库、从数据库和其他哨兵节点发送PING命令。  
这3个操作贯穿哨兵进程的整个生命周期中，非常重要，可以说了解了这3个操作的意义就能够了解哨兵工作原理的一半内容了。  
首先，发送INFO命令使得哨兵可以获得当前数据库的相关信息（包括运行ID、复制信息等）从而实现新节点的自动发现。前面说配置哨兵监控Redis主从系统时只需要指定主数据库的信息即可，因为哨兵正是借助INFO命令来获取所有复制该主数据库的从数据库信息的。启动后，哨兵向主数据库发送INFO命令，通过解析返回结果来得知从数据库列表，而后对每个从数据库同样建立2个连接，2个连接的作用和前面介绍的与主数据库建立的2个连接完全一致。在此之后哨兵会每10秒定时向已知的所有主从数据库发送INFO命令来获取信息更新并进行相应操作，比如对新增的从数据库建立连接并加入监控列表，对主从数据库的角色变化（由故障恢复操作引起）进行信息更新等。  
接下来哨兵向主从数据库的\_\_sentinel\_\_:hello频道发送信息来与同样监控该数据库的哨兵分享自己的信息。发送的消息内容为：  
> <哨兵的地址>,<哨兵的端口>,<哨兵的运行ID>,<哨兵的配置版本>,<主数据的名字>,<主数据库的地址>,<主数据库的端口>,<主数据库的配置版本>  

可以看到消息包括的哨兵的基本信息，以及其监控的数据库的信息。前文介绍过，哨兵会订阅每个其监控的数据库的\_\_sentinel\_\_:hello频道，所以当其他哨兵收到消息后，会判断发送消息的哨兵是不是新发现的哨兵。如果是则将其加入已发现的哨兵列表中并创建一个到其的连接（与数据库不同，哨兵与哨兵之间只会创建一条连接用来发送PING命令，而不需要创建另外一条连接来订阅频道，因为哨兵只需要订阅数据库的频道即可实现自动发现其他哨兵）。同时哨兵会判断信息中主数据的配置版本，如果该版本比当前记录的主数据库的版本高，则更新主数据库的数据。配置版本的作用将在后面介绍。  
实现了自动发现从数据库和其他哨兵节点后，哨兵要做的就是定时监控这些数据库和节点有没有停止服务。这是通过每隔一定时间向这些节点发送PING命令实现的。时间间隔与down-after-milliseconds选项有关，当down-after-milliseconds的值小于1秒时，哨兵会每隔down-after-milliseconds指定的时间发送一次PING命令，当down-after-milliseconds的值大于1秒时，哨兵会每隔1秒发送一次PING命令。如：  

    --每隔1秒发送一次PING命令
    sentinel down-after-milliseconds mymaster 60000
    --每隔600毫秒发送一次PING命令
    sentinel down-after-milliseconds mymaster 600

当超过down-after-milliseconds选项指定时间后，如果被PING的数据库或节点仍然未进行回复，则哨兵认为其主观下线(subjectively down)。主观下线表示从当前的哨兵进程看来，该节点已经下线。如果该节点是主数据库，则哨兵会进一步判断是否需要对其进行故障恢复：哨兵发送SENTINEL is-master-down-by-addr命令询问其他哨兵节点以了解他们是否也认为该主数据库主观下线，如果达到指定数量时，哨兵会认为其客观下线（objectively down），并选举领头的哨兵节点对主从系统发起故障恢复。这个指定数量即为前文介绍的quorum参数。如下配置：  

    sentinel monitor mymaster 127.0.0.1 6379 2

该配置表示只有当至少2个sentinel节点（包括当前节点）认为该主数据库主观下线时，当前哨兵节点才会认为该主数据库客观下线。进行接下来的选举领头哨兵步骤。  
虽然当前哨兵节点发现了主数据库客观下线，需要故障恢复，但是故障恢复需要有领头的哨兵来完成，这样可以保证同一时间只有一个哨兵节点来执行故障恢复。选举领头哨兵的过程使用后了Raft算法，具体如下：  
（1）发现主数据库客观下线的哨兵节点（下面称作A）向每个哨兵节点发送命令，要求对方选自己成为领头哨兵。  
（2）如果目标哨兵节点没有选过其他人，则会同意将A设置成领头哨兵。  
（3）如果A发现有超过半数且超过quorum参数值的哨兵节点同意选自己成为领头哨兵，则A成功成为领头哨兵。  
（4）当有多个哨兵节点同时参选领头哨兵，则会出现没有任何节点当选的可能。此时每个参选节点将等待一个随机时间重新发起参选请求，进行下一轮选举，直到选举成功。  
具体过程可以参考Raft算法的过程<http://www.cnblogs.com/mindwind/p/5231986.html>。因为要成为领头哨兵必须有超过半数的哨兵节点支持，所以每次选举最多只会选出一个领头哨兵。  
选出领头哨兵后，领头哨兵将会开始对主数据库进行故障恢复。故障恢复的过程相对简单，具体如下：  
首先领头哨兵将从停止服务的主数据库的从数据库中挑选一个充当新的主数据库。挑选的依据如下：  
（1）所有在线的从数据库中，选择优先级最高的从数据库。优先级可以通过slave-priority选项来设置。  
（2）如果有多个最高优先级的从数据库，则复制的命令偏移量越大（即复制越完整）越优先。  
（3）如果以上条件都一样，则选择运行ID较小的从数据库。  
选出一个从数据库后，领头哨兵将向从数据库发送SLAVEOF NO ONE 命令使其升格为主数据库。而后领头哨兵向其他从数据库发送SLAVEOF命令来使其成为新主数据库的从数据库。最后一步则是更新内部的记录，将已经停止服务的旧的主数据库更新成为新的主数据库的从数据库，使得当其恢复服务时自动以从数据库的身份继续服务。  

### 4、哨兵的部署  
哨兵以独立进程的方式对一个主从系统进行监控，监控的效果好坏与否取决于哨兵的视角是否有代表性。如果一个主从系统中配置的哨兵较少，哨兵对整个系统的判断的可靠性就会降低。极端情况下，当只有一个哨兵时，哨兵本身就可能会发生单点故障。整体来讲，相对稳妥的哨兵部署方案是使得哨兵的视角尽可能地与每个节点的视角一致，即：  
（1）为每个节点（无论是主数据库还是从数据库）部署一个哨兵。  
（2）使每个哨兵与其对应的节点网络环境相同或相近。  
这样的部署方案可以保证哨兵的视角拥有较高的代表性和可靠性。举例：当网络分区后，如果哨兵认为某个分区是主要分区，即意味着从每个节点观察，该分区均为主分区。同时设置quorum的值为N/2+1（其中N为哨兵节点数量），这样使得只有当大部分哨兵节点同意后才会进行故障恢复。  
当系统中的节点较多时，考虑到每个哨兵都会和系统中的所有节点建立连接，为每个节点分配一个哨兵会产生较多连接，尤其是当进行客户端分片时使用多个哨兵节点监控多个主数据库会因为Redis不支持连接复用而产生大量冗余连接，同时如果Redis节点负载较高，会在一定程度上影响其对哨兵的回复以及与其节点的通信。所以配置哨兵时还需要根据实际的生产环境情况进行选择。
