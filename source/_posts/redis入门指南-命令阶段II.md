---
title: redis入门指南-命令阶段II
date: 2017-05-23 21:41:20
categories: redis系列
tags: [redis命令,redis五种数据类型,列表（list）,集合（set）]
description: 了解了redis基础后，本章主要介绍redis列表（list）和集合（set）数据类型及相应的命令。命令拾遗部分会对其他比较有用的命令进行补充介绍。
---

## 列表类型  

列表类型（list）可以存储一个有序的字符串列表，常用的操作是向列表二端添加元素或者获取列表的某一个片段。  
列表类型内部是使用双向链表（double linked list）实现，所以向列表两端添加元素的时间复杂度为O（1），获取越接近两端的元素速度就越快。这意味着即使有几千万个元素的列表，获取头部或尾部的10条就也是极快的（和从只有20个元素的列表中获取头部或尾部的10条记录的速度是一样的）。
列表类型适合用来记录日志，可以保证新加入新日志不会受到已有日志数量的影响。列表还可以作为队列、栈结构来使用，与散列表类型最多能容纳字段数量相同，一个列表类型最多能容纳2的23次方-1个元素。

### 1、命令  

* 向列表两端增加元素

```
LPUSH key value [value ...]
RPUSH key value [value ...]
```
LPUSH 命令用来向列表左边添加元素，返回值表示增加元素后列表的长度。

```
127.0.0.1:6379> lpush numbers 1
(integer) 1
```
这时numbers 键中数据如下：

![lpush1之后数据机构模型](http://op7wplti1.bkt.clouddn.com/lpush1.png)

LPUAH 命令还支持同时增加多个元素，如：

```
127.0.0.1:6379> lpush numbers 2 3
(integer) 3
```
LPUSH会先向列表左边加入“2”，然后再加入“3”，所以此时members 键中数据如下：

![lpush2之后数据机构模型](http://op7wplti1.bkt.clouddn.com/lpush2.png)

向列表右边增加元素使用RPUSH命令，用法和LPUSH命令一样：

```
127.0.0.1:6379> lpush numbers 2 3
(integer) 5
```
此时numbers键中的数据如下图：

![rpush之后数据机构模型](http://op7wplti1.bkt.clouddn.com/rpush.png)

* 从列表两端弹出元素

```
LPOP key 
RPOP key
```
有进有出，lpop命令可以从列表左边弹出一个元素，lpop命令执行二步操作：第一步是将列表左边的元素从列表中移除，第二部是返回被移除的元素值。如：从numbers列表左边弹出一个元素（也就是“3”）

```
127.0.0.1:6379> lpop numbers
"3"
```
此时numbers键中的数据如下图：

![lpop之后数据机构模型](http://op7wplti1.bkt.clouddn.com/lpop.png)

同样，rpop命令可以从右边弹出一个元素：

```
127.0.0.1:6379> rpop numbers
"-1"
```
此时numbers键中数据如下：

![rpop之后数据机构模型](http://op7wplti1.bkt.clouddn.com/rpop.png)

**结合上面的4个命令可以使用列表类型来模拟栈和队列的操作：如果想把列表当做栈，则搭配LPUSH和LPOP或RPUSH和RPOP，如果想当成队列，则搭配使用LPUSH和RPOP或RPUSH和LPOP。**

* 读取列表中元素个数

```
LLEN key
```

当键不存在时LLEN返回0：

```
127.0.0.1:6379> llen numbers
(integer) 3
```

* 获取列表片段

```
LRANGE key start stop
```
LARNGE 命令是列表类型最常用的命令之一，它能够获得列表中的某一片段。LRANGE命令将返回索引从start到stop之间的所有元素（包含两端的元素）。与大多数人的直觉相同，redis的列表起始索引为0：

```
127.0.0.1:6379> lrange numbers 0 2
1) "2"
2) "1"
3) "0"
```
LRANGE 命令也支持负索引，表示从右边开始计算序数，如“-1”表示最右边第一个元素，“-2”表示最右边第二个元素，以此类推：

```
127.0.0.1:6379> lrange numbers -2 -1
1) "1"
2) "0"
```
显然，LRANGE numbers 0 -1 可以获取列表中所有元素。另外一些特殊情况如下：  
（1）如果start的索引位置比stop的索引位置靠后，则会返回空列表
（2）如果stop大于实际的索引范围，则会返回到列表最后边的元素：

```
127.0.0.1:6379> lrange number 2 1
(empty list or set)

127.0.0.1:6379> lrange numbers 1 100
1) "1"
2) "0"
127.0.0.1:6379>
```

* 删除列表中指定的值

```
LREM key count value
```
LREM 命令会删除列表中前count个值为value的元素，返回值是实际删除的元素个数。
根据count值得不同，LREM命令执行的方式也会略有差异：  

    1. 当count>0 时LREM命令会从列表左边开始删除前count个值为value的元素；  
    2. 当count<0 时LREM命令会从列表右边开始删除前|count|个值为value的元素；  
    3. 当count=0 时LREM命令会删除所有值为value的元素。
    
```
127.0.0.1:6379> rpush numbers 2
(integer) 4
127.0.0.1:6379> lrange numbers 0 -1
1) "2"
2) "1"
3) "0"
4) "2"

#从右边删除第一个值为"2"的元素

127.0.0.1:6379> lrem numbers -1 2
(integer) 1
127.0.0.1:6379> lrange numbers 0 -1
1) "2"
2) "1"
3) "0"

```

### 2、命令拾遗

* 获得/设置指定索引的元素值  

```
LINDEX key index --用来返回指定索引的元素，索引从0开始,如果index为负数则表示从右边开始计算索引，最右边元素索引为-1

LSET key index value --将索引为index的元素赋值为value
```
e.g:

```
127.0.0.1:6379> lrange numbers 0 -1
1) "2"
2) "1"
3) "0"

127.0.0.1:6379> lindex numbers -2
"1"

127.0.0.1:6379> lset numbers -2 20
OK

127.0.0.1:6379> lrange numbers 0 -1
1) "2"
2) "20"
3) "0"

```

* 只保留列表指定片段
LTRIM key start end
LTRIM命令可以删除指定索引范围之外的所有元素，其指定列表范围的方法和LRANGE命令相同，如：

```
127.0.0.1:6379> lrange numbers 0 -1
1) "2"
2) "20"
3) "0"
127.0.0.1:6379> ltrim numbers 1 2
OK
127.0.0.1:6379> lrange numbers 0 -1
1) "20"
2) "0"
```
LTRIM命令常和LPUSH命令一起使用来限制列表中元素的数量，比如记录日志时我们希望只保留最近100条日志，则每次加入新元素时调用一次LTRIM命令即可：

```
LPUSH logs $newLog
LTRIM logs 0 99
```

* 向列表中插入元素  

```
LINSERT key BEFORE|AFTER pivot value
```
LINSERT命令会首先在列表中从左到右查找值为pivot的元素，然后根据第二个参数是BEFORE还是AFTER来决定将value插入到该元素的前面还是后面。LINSERT命令的返回值为插入后列表元素的个数。如：


```
127.0.0.1:6379> lrange numbers 0 -1
1) "20"
2) "0"
127.0.0.1:6379> lrange numbers 0 -1
1) "20"
2) "0"
127.0.0.1:6379> linsert numbers Before 20 110
(integer) 3
127.0.0.1:6379> lrange numbers 0 -1
1) "110"
2) "20"
3) "0"
127.0.0.1:6379> linsert numbers AFTER 20 21 
(integer) 4
127.0.0.1:6379> lrange numbers 0 -1
1) "110"
2) "20"
3) "21"
4) "0"
```

* 将元素从一个列表转到另一个列表  

```
RPOPLPUSH source destination
```
RPOPLPUSH是个很有意思的命令，从名字就可以看出它的功能：先执行RPOP命令再执行LPUSH命令。RPOPLPUSH命令会先从source列表类型键的右边弹出一个元素，然后将其将入到destination列表类型键的左边，并返回这个元素的值，整个过程是原子的。


当把列表类型作为队列使用时，RPOPLPUSH命令可以很直观地在多个队列中传递数据。当source和destination相同时，RPOPLPUSH命令会不断的将队尾的元素移到队首借助这个特性我们可以实现一个网站监控系统：使用一个队列存储需要监控的网址，然后监控程序不断地使用RPOPLPUSH命令循环取出一个网址来测试可用性。这里使用RPOPLPUSH命令的好处在于程序执行过程中仍然可以不断地向网址列表中加入新网址，而且整个系统容易扩展，允许多个客户端同事处理队列。

## 集合类型

集合类型和列表类型有相似之处，但很容易将他们区分开来，见下表：

集合类型和列表类型对比

特性|集合类型|列表类型
---|---|---
存储内容|至多2的23次方-1|至多2的23次方-1
有序性|否|是
唯一性|是|否

集合类型的常用操作是向集合中加入或删除元素，判断某个元素是否存在等，由于集合类型在redis内部是使用值为空的散列表（hash table）实现，所以这些操作的时间复杂度都是O（1）。最方便的是多个集合类型键之间还可以进行并集、交集和差集运算。

### 1、命令

* 增加/删除元素

```
SADD key member [member ...]
SREM key member [member ...]
```
SADD命令用来向集合中增加一个或多个元素，如果键不存在则会自动创建。因为在yige
 集合中不能有相同的元素，所以如果要加入的元素已经存在于集合中就会忽略这个元素。
 本命令的返回值是成功加入的元素数量（怒略的元素不计算在内）。如：

```
127.0.0.1:6379> sadd letters a
(integer) 1
127.0.0.1:6379> sadd letters a b c
(integer) 2
```
第二条命令返回值为2是因为元素“a” 已经存在，所以实际加入了2个元素。  
SREM 用来从集合中删除一个或多个元素，返回值为成功删除个数，如：

```
127.0.0.1:6379> srem letters c d
(integer) 1
```
由于“d”在集合中不存在，所以只删除了一个元素，返回值为1。

* 获得集合中所有元素

```
SMEMBERS key
```


SMEMBERS命令返回集合中所有元素

```
127.0.0.1:6379> smembers letters
1) "b"
2) "a"
```
* 判断元素是否在集合中
SISMEMBER key member
判断一个元素是否在集合是一个时间复杂度为O（1）的操作，无论集合中有多少元素，SISMEMBER命令始终可以极快返回结果。当值存在时SISMEMBER返回为1，当值不存在或键不存在时返回0，如：

```
127.0.0.1:6379> sismember letters a
(integer) 1
127.0.0.1:6379> sismember letters d
(integer) 0
```
* 集合间运算

```
SDIFF key [key ...]
SINTER key [key ...]
SUNION key [key ...]
```

（1）SDIFF命令用来对多个集合执行差集运算。集合A与集合B差集表示为A-B，代表所有属于A且不属于B的元素构成的集合即A-B={x∈A且x∉B }。如图

![sdiff集合差集](http://op7wplti1.bkt.clouddn.com/sdiff.png)

```
127.0.0.1:6379> sadd setA 1 2 3
(integer) 3
127.0.0.1:6379> sadd setB 2 3 4
(integer) 3
127.0.0.1:6379> Sdiff setA setB
1) "1"
127.0.0.1:6379> Sdiff setB setA
1) "4"
```
SDIFF命令支持同时传入多个键，如：

```
127.0.0.1:6379> sadd setC 2 3
(integer) 2
127.0.0.1:6379> Sdiff setB setA setC
1) "4"
```
计算顺序是先计算setA-setB，再将计算结果与setC做差集。  
（2）SINTER命令用来对多个集合执行交集运算。集合A与集合B交集表示为A∩B，代表所有属于A且属于B的元素构成的集合，即A∩B={x|x∈A且x∈B}。如图
 
 
![sinter集合交集](http://op7wplti1.bkt.clouddn.com/sinter.png) 

```
127.0.0.1:6379> sinter setA setB
1) "2"
2) "3"
127.0.0.1:6379> sinter setA setB setC
1) "2"
2) "3"
```
（3）SUNION命令用来对多个集合执行并集运算。集合A与集合B的并集表示为A∪B，代表所有
    属于A或属于B的元素构成的集合，即A∪B={x|x∈A或x∈B }。如    
    
    
![sunion 集合并集](http://op7wplti1.bkt.clouddn.com/sunion.png)

```
127.0.0.1:6379> sunion setA setB
1) "1"
2) "2"
3) "3"
4) "4"
127.0.0.1:6379> sunion setA setB setC
1) "1"
2) "2"
3) "3"
4) "4"
```

### 2、命令拾遗

* 获得集合中元素个数

```
SCARD key

127.0.0.1:6379> smembers letters
1) "b"
2) "a"
127.0.0.1:6379> scard letters
(integer) 2
```

* 进行集合运算并将结果存储

```
SDIFFSTORE destination key [key ...]
SINTERSTORE destination key [key ...]
SUNIONSTORE destination key [key ...]
```
SDIFFSTORE命令和SDIFF命令功能一样，唯一的区别就是前者不会直接返回运算结果，而是将结果存储在destination键中。  
SINTERSTORE命令常用于需要进行多步集合运算的场景中，如需要先计算差集再将结果和其他键计算交集。

* 随机获得集合中的元素  

```
SRANDMEMBER key [count]
```
SRANDMEMBER命令用来随机从集合中获取一个元素，如：  

```
127.0.0.1:6379> srandmember letters
"a"
127.0.0.1:6379> srandmember letters
"b"
```
还可以通过传递count参数来一次随机获取多个元素，根据count的正负不同，具体表现也不同。  
（1）当count为正数时，SRANDMEMBER会随机从集合里获得count个不重复的元素，如果count值大于集合中的元素个数，则SRANDMEMBER会返回集合中全部元素。  

（2）当count为负数时，SRANDMEMBER会随机从集合里获得|count|个元素，这些元素有可能相同。

```
127.0.0.1:6379> srandmember letters 2
1) "c"
2) "d"
127.0.0.1:6379> srandmember letters 2
1) "a"
2) "c"
127.0.0.1:6379> srandmember letters -2
1) "d"
2) "c"
127.0.0.1:6379> srandmember letters -2
1) "c"
2) "c"
127.0.0.1:6379> srandmember letters -10
 1) "b"
 2) "c"
 3) "a"
 4) "b"
 5) "c"
 6) "c"
 7) "b"
 8) "b"
 9) "d"
10) "a"
```
b出现的 几率比较大（原因参见redis入门指南第65页）

* 从集合中弹出一个元素      

```
SPOP key
```

spop会随机选择一个元素从集合中弹出


```
127.0.0.1:6379> smembers letters
1) "d"
2) "c"
3) "b"
4) "a"
127.0.0.1:6379> spop letters
"c"
127.0.0.1:6379> smembers letters
1) "d"
2) "b"
3) "a"
```
