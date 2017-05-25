---
title: redis入门指南之提高篇I
date: 2017-05-24 19:59:52
categories: redis系列
tags: [redis命令,redis事务,生存时间,缓存,访问频率限制]
description: 通过使用redis事务、生存时间、缓存等redis的高级特性，更好在生产环节适宜的场景中使用它处理问题。
---

###### 重要星级 ★★★★★

---

## Redis事务  

### 1、概述
Redis中的事务（transaction）是一组命令集合。事务同命令一样都是redis的最小执行单位，一个事务中的命令要么都执行，要么都不执行。事务的应用非常普遍，如银行转账过程中A给B汇款，首先系统从A的账户将钱划走，然后向B的账户增加相应的金额。这二个步骤必须属于同一个事务，要么全执行，要么全不执行。否则只执行第一步，钱就凭空消失了，这显然让人无法接受。  
事务的原理是先将属于一个事务的命令发送给Redis，然后再让Redis依次执行这个命令。如：

```
127.0.0.1:6379> multi
OK
127.0.0.1:6379> sadd "user:1:following" 2
QUEUED
127.0.0.1:6379> sadd "user:2:followers" 1
QUEUED
127.0.0.1:6379> exec
1) (integer) 1
2) (integer) 1
```
以上代码演示了事务的使用方式。首先使用multi命令告诉Redis：“下面我发给你的命令属于同一个事务，你先不要执行，而是把他们暂时存起来。”Redis回答：“OK。” 而后我们发送了2个sadd命令来实现关注和被关注的操作，可以看到Redis遵守了承若，没有执行这些命令，而是返回QUEUED表示这两条命令已经进入等待执行的事务队列中了。  
当把所有同一个事务中的执行的命令都发给Redis后，我们使用EXEC命令告诉Redis将等待执行的事务队列中所有命令（即刚才所有返回QUEUED的命令）按照发送顺序依次执行。EXEC命令的返回值就是这些命令的返回值组成的列表，返回值顺序和命令的顺序相同。  
Redis保证一个事务中的所有命令要么都执行，要么都不执行。如果在发送EXEC命令前客户端断线了，则Redis会清空事务队列，事务中的所有命令都不会执行。而一旦客户端发送了EXEC命令，所有的命令就都会被执行，即使此后客户端断线也没关系，因为Redis中已经记录了所有要执行的命令。  
除此之外，Redis的事务还能保证一个事务内的命令依次执行而不被其他命令插入。试想客户端A需要执行几条命令，同时客户端B发送了一条命令，如果不适用事务，则客户端B的命令可能会插入到客户端A的几条命令中执行。如果不希望发生这种情况，也可以使用事务。

### 2、错误处理  
如果事务中某个命令执行出错，Redis会怎么处理呢？  
#### （1）语法错误。语法错误指命令不存在或者命令参数的个数不对。如：

```
127.0.0.1:6379> multi
OK
127.0.0.1:6379> set key value
QUEUED
127.0.0.1:6379> set key
(error) ERR wrong number of arguments for 'set' command
127.0.0.1:6379> errorcommand key
(error) ERR unknown command 'errorcommand'
127.0.0.1:6379> exec
(error) EXECABORT Transaction discarded because of previous errors.
```
跟在MULTI命令后执行了3个命令：一个是正确命令，成功地加入了事务队列；其余二个命令都有语法错误。而只要有一个命令有语法错误，执行EXEC命令后Redis就会直接返回错误，连语法正确的命令也不会执行。  
#### （2）运行错误。运行错误指在命令执行时出现的错误，比如使用散列类型的命令操作集合类型的键，这种错误在实际执行之前Redis是无法发现的，所以在事务里这样的命令是会被Redis接受并执行的。如果事务里的一条命令出现了运行错误，事务里其他的命令依然会继续执行（包括出错命令之后的命令）。如：

```
127.0.0.1:6379> multi
OK
127.0.0.1:6379> set key 1
QUEUED
127.0.0.1:6379> sadd key 2
QUEUED
127.0.0.1:6379> set key 3
QUEUED
127.0.0.1:6379> exec
1) OK
2) (error) WRONGTYPE Operation against a key holding the wrong kind of value
3) OK
127.0.0.1:6379> get key
"3"
```
可见虽然sadd key 2出现了错误，但是set key 3 依然执行了。  
Redis的事务没有关系型数据库事务提供的回滚（rollback）功能。为此开发者必须在事务执行出错后自己收拾烂摊子（将事务恢复到执行前的状态等）  
不过由于Redis不支持回滚功能，也是的Redis在事务上可以保持简洁和快速。  

### 3、WATCH命令  
我们知道在一个事务中只有当所有命令都依次执行完后才能得到每个结果的返回值，可是有些情况下需要先获得一条命令的返回值，然后再根据这个值执行下一条命令。例如，介绍INCR命令时曾经说过使用GET 和SET命令自己实现incr函数会出现竞态条件，伪代码如下：  

```
def incr($key)
    $value=GET $key
    if not $value
            $value=0
    $value=$value+1
    SET $key,$value
    return $value
```
因为事务中的每一个命令的执行结果都是最后一期返回的，所以无法将前一条命令的结果作为下一条命令的参数，即在执行SET命令时无法获得GET命令的返回值，也就无法做到增1的功能。所以无法使用事务来实现incr函数。  
为了解决这个问题，我们需要换种思路。即在GET获得键值后保证该键值不被其他客户端修改，直到函数执行完成后才允许其他客户端修改该键值，这样也可以防止竞态条件。要实现这一思路需要请出事务家族中的另一位成员：WATCH。WATCH命令可以监控一个或多个键，一旦其中有一个键被修改（或删除），之后的事务就不会执行。监控一直持续到EXEC命令（事务中的命令是在EXEC之后才执行的，所以在MULTI命令后可以修改WATCH监控的键值）。如：

```
127.0.0.1:6379> set key 1
OK
127.0.0.1:6379> watch key
OK
127.0.0.1:6379> set key 2
OK
127.0.0.1:6379> multi
OK
127.0.0.1:6379> set key 3
QUEUED
127.0.0.1:6379> exec
(nil)
127.0.0.1:6379> get key
"2"
```
上面例子在执行WATCH命令后、事务执行前修改了key的值（即set key 2），所以最后的事务中的命令SET key 3没有执行，EXEC命令返回空结果。学会了WATCH命令就可以通过事务自己实现incr函数了，伪代码如下：  

```
def incr($key)
    WATCH $key
    $value=GET $key
    if not $value
            $value=0
    $value=$value+1
    MULTI
    SET $key,$value
    result=EXEC
    return result[0]
```
**提示：由于WATCH命令的作用只是当被监控的键值被修改后阻止之后一个事务的执行，而不能保证其他客户端不修改这一键值，所以我们需要在EXEC执行失败后重新执行整个函数。**  
执行EXEC命令后会取消对所有键的监控，如果不想执行事务中的命令也可以使用UNWATCH命令来取消监控。比如我们要实现hsetxx函数，作用于HSETNX命令类似，只不过是仅当字段存在时才赋值。为了避免竞态条件我们使用事务来完成这一功能：

```
def hsetxx($key,$field,$value)
    WATCH $key
    $isFieldExists =HEXISTS $key,$field
    if $isFieldExists is 1
        MULTI
        HSET $key,$field,$value
        EXEC
    else
        UNWATCH
    return $isFieldExists
```
在代码中会判断要赋值的字段是否存在，如果字段不存在的话就不执行事务中的命令，但需要使用UNWATCH命令来保证下一个事务的执行不会受到影响。

## 生存时间  

### 1、命令  
EXPIRE命令的使用方法为EXPIRE key seconds，其中seconds参数表示键的生存时间，单位是秒。如想要让session:2345 键在15分钟后删除：  

```
127.0.0.1:6379> set session:2345 1234
OK
127.0.0.1:6379> expire session:2345 900
(integer) 1
```
EXPIRE命令返回1表示设置成功，返回0表示键不存在或设置失败。例如：  

```
127.0.0.1:6379> del session:2345
(integer) 1
127.0.0.1:6379> expire session:2345 900
(integer) 0
```
如果想知道一个键还有多久时间删除，可以使用TTL命令。返回值为键的剩余时间（单位是秒）。

```
127.0.0.1:6379> set foo bar
OK
127.0.0.1:6379> expire foo 20
(integer) 1
127.0.0.1:6379> ttl foo
(integer) 8
127.0.0.1:6379> ttl foo
(integer) 4
127.0.0.1:6379> ttl foo
(integer) 1
127.0.0.1:6379> ttl foo
(integer) -2
127.0.0.1:6379> get foo
(nil)
```
可见随着时间的不同，foo键的生存时间逐渐减少，20秒后foo键会被删除。当然不存在时TTL命令会返回-1，另外同样会返回-1的是没有为键设置生存时间（即永久存在的，这是建立一个键之后的默认情况）。

```
127.0.0.1:6379> set k1 v1
OK
127.0.0.1:6379> ttl k1
(integer) -1
127.0.0.1:6379> ttl k3
(integer) -1
```
如果想取消键的生存时间设置（即将键恢复成永久的），可以使用PERSIST命令。如果生存时间被成功清除则返回1；否则返回0（因为键不存在或键本来就是永久的）：  

```
127.0.0.1:6379> set foo bar
OK
127.0.0.1:6379> persist foo --键本来就是永久的则返回0
(integer) 0
127.0.0.1:6379> expire foo 20
(integer) 1
127.0.0.1:6379> ttl foo
(integer) 15
127.0.0.1:6379> persist foo --键有生存时间则设置为永久，成功返回1
(integer) 1
127.0.0.1:6379> ttl foo
```
除了PERSIST命令之外，使用SET或GETSET命令为键赋值也会同时清除键的生存时间，例如：  

```
127.0.0.1:6379> expire foo 20
(integer) 1
127.0.0.1:6379> ttl foo
(integer) 11
127.0.0.1:6379> set foo bar
OK
127.0.0.1:6379> ttl foo
(integer) -1
```
使用EXPIRE命令会重新设置键的生存时间，如：  

```
127.0.0.1:6379> set foo bar
OK
127.0.0.1:6379> expire foo 20
(integer) 1
127.0.0.1:6379> ttl foo
(integer) 18
127.0.0.1:6379> expire foo 30
(integer) 1
127.0.0.1:6379> ttl foo
(integer) 27
```
其他只对键值进行操作的命令（如：INCR/LPUSH/HSET/ZREM）均不会影响键的生存时间。  
EXPIRE命令的seconds参数必须是整数，所以最小单位是1秒。如果想要更精确的控制键的生存时间应该使用PEXPIRE命令，PEXPIRE命令与EXPIRE的唯一区别是前者的时间单位是毫秒，即PEXPIRE key 1000与EXPIRE key 1等价。对应地可以用PTTL 命令以毫秒为单位返回键的剩余时间。  
**提示：如果使用WATCH命令监测一个拥有生存时间的键，该键时间到期自动删除并不会被WATCH命令认为该键被改变，也就不会影响事务的执行。**  
另外还有2个相对不太常用的命令：EXPIREAT和PEXPIREAT。  
EXPIREAT命令与EXPIRE命令的差别在于前者使用unix时间作为第二个参数表示键的生存时间的截止时间。PEXPIREAT命令与EXPIREAT命令的区别是前者的时间单位是毫秒。如：  

```
127.0.0.1:6379> set foo val
OK
127.0.0.1:6379> get foo
"val"
127.0.0.1:6379> EXPIREAT foo 1498473048362 --时间戳，截止到年月日时分秒
(integer) 1
127.0.0.1:6379> ttl foo
(integer) 1496977350307
127.0.0.1:6379> ttl foo
(integer) 1496977350301
127.0.0.1:6379> PEXPIREAT foo 1498473048362000 --时间戳，截止到年月日时分秒，毫秒为单位
(integer) 1
127.0.0.1:6379> pttl foo
(integer) 1496977350234675
```
### 2、访问频率限制策略  

* 实现访问频率限制1
如需要限制每个用户（以IP计）一段时间的最大访问量。如：  
每分钟用户最多能访问某系统100个界面，思路是对每个用户使用一个名为“rate.limiting:用户ID”的字符串类型键，每次用户访问则使用INCR 命令递增该键的键值，如果递增后的值是1（第一次访问页面），则同时还要设置该键的生存时间为1分钟。这样每次用户访问页面时都读取该键的键值，如果超过100就表明该用户的访问频次超过了限制，需要提示用户稍后访问。该键每分钟自动被删除，所以下一分钟用户的访问次数又会重新计算，也就达到了限制访问频率的目的。伪代码如下：  

```
$isKeyExists=EXISTE rate.limiting:$IP
if $isKeyExists is 1
    $times=INCR rate.limiting:$IP
    if $times>100
        print 访问频率超过了限制，请稍后再试
        exit
else
    --第一次访问设置生存时间
    INCR rate.limiting:$IP
    EXPIRE $keyName,60
```
上述代码存在一个不太明显的问题：假如程序执行完倒数第二行后突然因为某种原因退出了，没能够为该键设置生存时间，那么该键会永久存在，导致使用对应IP的用户在管理员手动删除该键前最多只能访问100次，这是一个很严重的问题。  
为了保证建立键和为键设置生存时间一起执行，可以使用上面学习的事务功能，修改后代码如下：  

	$isKeyExists=EXISTE rate.limiting:$IP
	if $isKeyExists is 1
		$times=INCR rate.limiting:$IP
		if $times>100
			print 访问频率超过了限制，请稍后再试
			exit
	else
		--第一次访问设置生存时间,并通过事务保证原子性
		MULTI
		INCR rate.limiting:$IP
		EXPIRE $keyName,60
		EXEC

* 实现访问频率限制2  
事实上，上述代码仍然有个问题：如果一个用户在一分钟的第一秒访问了一次，在同一分钟的最后一秒访问了99次，又在下一分钟的第一秒访问了100次，这样的访问是可以通过现在的访问频率限制的，但实际上该用户在2秒钟内访问了199次，这与每个用户每分钟只能访问100次的限制差距较大。尽管这种情况比较极端，但在一些场合中还是需要粒度更小的控制方案。如果要精确控制每分钟最多访问100次，需要记录下用户每次访问的时间。因此对每个用户，我们使用一个列表类型的键来记录他最近100次访问时间。一旦键中的元素超过100个，就判断时间最早的元素距现在的时间是否小于1分钟。如果是则表示最近一分钟的访问次数超过了100频次限制；如果不是就将限制的时间加入到列表中，同时把最早的元素删除掉。伪代码如下:  

	$listLength=LLEN rate.limiting:$IP
	if $listLength <100
		访问次数不超过限制就加入
		LPUSH rate.limiting:$IP,now()
	else
		超过限制就根据时差分情况
		$time=LINDEX rate.limiting:$IP,-1
		如果最早的一次访问距离现在不超过1分钟，说明访问密集已经超出频次限制 
		if now()-$time<60
			print 访问频率超过限制，请稍后再试
		else
		说明最早的访问到距离现在已超过一分钟，没有超出限制
		LPUSH rate.limiting:$IP,now()
		踢掉最右端的元素（即最早访问时间）
		LTRIM rate.limiting:$IP,0,99
		
由于需要记录每次访问的时间，所以当要限制“A时间最多访问B次”时，如果“B”的数值较大，比方法会占用较多的存储空间，实际使用时还需要开发者自己权衡。除此之外该方法也会出现竞态条件。  

### 3、实现缓存（缓存占用内存解决方案）  
为了提高网站的负载能力，常常需要将一些访问频率较高但是对CPU或IO资源消耗较大的操作结果缓存起来，并希望让这些缓存过段时间自动过期。比如教务网站要对全校所有学生的各个科目成绩汇总排名，并在首页上显示前10名学生姓名，由于计算过程较耗资源，所以可以将结果使用Redis的字符串缓存起来。由于学生成绩总在不断变化，需要每隔二小时就重新计算一次排名，这可以通过给键设置生存时间的方式实现。每次用户访问首页时程序先去查询缓存是否存在，如果存在则直接使用缓存的值；否则重新计算排名并将结果赋值给该键并同时设置该键的生存时间为2小时。伪代码如下： 

	$rank=GET cache:rank
	if not $rank
		$rank=计算排名...
		MULTI
		SET cache:rank,$rank
		EXPIRE cache:rank,7200
		EXEC 

然后在一些场合中这种方法并不能满足需求。当服务器内存有限时，如果大量使用缓存键且生存时间设置的过长就会导致Redis占满内存；另一方面如果为了防止Redis占用内存过大而将缓存键的生存时间设得太短，就可能导致缓存命中率过低并且大量内存白白闲置。实际开发中会发现很难为缓存键设置合理的生存时间，为此可以限制Redis能够使用的最大内存，并让Redis按照一定的规则淘汰不需要的缓存键，这种方式只将Redis用作缓存系统时非常有用。  
具体的设置方法为：修改配置文件中的maxmemory参数，限制Redis最大可用内存大小（单位是字节），当超出这个限制时Redis会根据maxmemory-policy参数指定的策略来删除不需要的键，直到Redis内存小于指定内存。  
maxmemory-policy支持规则如下，其中LRU（Least Recently Used）算法即“最近最少使用”，其认为最近最少使用的键在未来一段时间内也不会被用到，即当需要时这些键是可以被删除的。  

规则|说明
---|---
volatile-lru|使用LRU算法删除一个键（只针对设置了生存时间的键）
allkeys-lru|使用LRU算法删除一个键
volatile-random|随机删除一个键（只针对设置了生存时间的键）
allkeys-random|随机删除一个键
volatile-ttl|删除生存时间最近的一个键
noevication|不删除键，值返回错误

如当maxmemory-policy设置为allkeys-lru时，一旦Redis占用的内存超过了限制，Redis会不断删除数据中最近最少使用的键，直到占用的内存小于限制值。

**LRU(Least Recently Used)最近最少使用：事实上Redis并不会准确地将整个数据库中最久未被使用的键删除，而是每次从数据库中随机取3个键并删除这3个键中最久未被使用的键。删除生存时间最接近的键的实现方法也是这样。“3”这个数字可以通过Redis的配置文件中的maxmemory-samples参数设置。**
