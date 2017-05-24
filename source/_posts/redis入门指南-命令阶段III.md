---
title: redis入门指南-命令阶段III
date: 2017-05-24 09:23:42
categories: redis系列
tags: [redis命令,redis五种数据类型,有序集合（zset）]
description: 了解了redis基础后，本章主要介绍有序集合（zset）数据类型及相应的命令。命令拾遗部分会对其他比较有用的命令进行补充介绍
---
## 有序集合类型  
在集合类型的基础上有序集合类型为集合中的每个元素都关联了一个分数，这使得我们不仅可以完成插入、删除和判断元素是否存在等集合类型支持的操作，还能够获得分数最高（或最低）的前N个元素、获得指定分数范围内的元素等与分数有关的操作。虽然集合中每个元素都是不同的，但是他们的分数却可以相同。 

有序集合类型和列表类型的区别：  
（1）二者都是有序的。  
（2）二者都可以获得某一范围的元素。  


（3）列表类型是通过链表实现的，获取靠近两端的数据速度极快，而当元素增多后，访问中间数据的速度会较慢，所以它更加适合实现如“新鲜事”或“日志”这样的很少访问中间元素的应用。 
（4）有序集合类型是使用散列表和跳跃表（skip list）实现的，所以即使读取位于中间部分的数据速度也很快（时间复杂度O（logN））。  
（5）列表中不能简单地调整某个元素的位置，但是有序集合可以（通过更改这个元素的分数）。
（6）有序集合要比列表类型更耗费内存。

### 1、命令  

* 增加元素  

```
ZADD key score member [score member ...]
```

ZADD命令用来向有序集合中加入一个元素和该元素的分数，如果该元素已经存在则会用新的分数替换原有的分数。ZADD命令的返回值是新加入到集合的元素的个数（不包含之前已经存在的元素）。如：

```
127.0.0.1:6379> zadd scores 89 tom 67 peter 100 david
(integer) 3
127.0.0.1:6379> zadd scores 76 peter --如果元素已存在，用新分数替换原有分数
(integer) 0
127.0.0.1:6379> zadd test 17E+307 a --分数支持双精度
(integer) 1
127.0.0.1:6379> zadd test 1.5 a 
(integer) 0
127.0.0.1:6379> zadd test 1.5 b
(integer) 1
127.0.0.1:6379> zadd test +inf c --分数赋值为正无穷大
(integer) 1
127.0.0.1:6379> zadd test -inf d --分数设置为负无穷大
(integer) 1
```

**+inf表示正无穷大
-inf表示负无穷大**

* 获得元素的分数

```
ZSCORE key member

127.0.0.1:6379> ZRANGE scores 0 -1 withscores
1) "peter"
2) "76"
3) "tom"
4) "89"
5) "david"
6) "100"
127.0.0.1:6379> zscore scores tom
"89

```

* 获得排名在某个范围的元素列表  

```
ZRANGE key start stop [WITHSCORES]
ZREVRANGE key start stop [WITHSCORES]
```
ZRANGE 命令会按照元素的分数从小到大的顺序返回索引从start到stop之间的所有元素（包括两端元素）。ZRANGE命令与LRANGE命令十分相似，如索引都是从0开始，负数代表从后向前查找（-1表示最后一个元素）。如：

```
127.0.0.1:6379> zrange scores 1 2
1) "tom"
2) "david"
127.0.0.1:6379> zrange scores 0 -1
1) "peter"
2) "tom"
3) "david"
```
如果同时获得元素的分数的话可以在ZRANGE命令尾部加上WITHSCORES参数，这时返回的数据格式就从，“元素1，元素2，...，元素
```
n”变成了“元素1，分数1，元素2，分数2,...,元素n，分数n”。如：
127.0.0.1:6379> zrange scores 0 -1 withscores
1) "peter"
2) "76"
3) "tom"
4) "89"
5) "david"
6) "100"
```
如果两个元素分数相同，redis会按照字典顺序（即“0”<“9”<"A"<"Z"<"a"<"z" 这样的顺序）来进行排列。  
ZREVRANGE命令和ZRANGE的唯一不同在于ZREVRANGE命令是按元素的分数从大到小的顺序给出结果。

* 获得指定分数范围的元素  

```
ZRANGEBYSCORE key min max [WITHSCORES] [LIMIT offset count]
ZREVRANGEBYSCORE key max min [WITHSCORES] [LIMIT offset count]--注意参数
```
ZRANGEBYSCORE命令参数虽然多，但是很好理解。该命令按照元素的分数从小到大的顺序返回分数在min和max之间（包括min和max）的元素。

```
127.0.0.1:6379> ZRANGEBYSCORE scores 80 100 withscores
1) "tom"
2) "89"
3) "david"
4) "100"
```
如果希望分数范围不包含端点值，可以在分数加上“（”符号。如，希望返回80分到100分的数据，可以含80分，但不包含100分，则修改命令如下：

```
127.0.0.1:6379> ZRANGEBYSCORE scores 80 (100 withscores
1) "tom"
2) "89"
127.0.0.1:6379> ZRANGEBYSCORE scores 80 100 withscores
1) "tom"
2) "89"
3) "david"
4) "100"
```

min和max还支持无穷大，通ZADD命令一样，-inf和+inf分别表示负无穷大和正无穷大。比如你希望得到所有分数高于80分（不包含80分）的人的名单，但你却不知道最高分是多少，这时就可以使用+inf。如：

```
127.0.0.1:6379> ZRANGEbyscore scores (80 +inf withscores
1) "tom"
2) "89"
3) "david"
4) "100"
```
LIMIT offset count与SQL 中的用法基本相同，即在获得的元素列表的基础上向后偏移offset个元素，并且只获取前count个元素。如：

```
127.0.0.1:6379> zrange scores 0 -1 withscores
 1) "admin"
 2) "69"
 3) "peter"
 4) "76"
 5) "tom"
 6) "89"
 7) "david"
 8) "100"
 9) "xym"
10) "102"
```
获取分数高于60分的从第二个人开始的三个人：

```
127.0.0.1:6379> ZRANGEBYSCORE scores 60 +inf LIMIT 1 3 withscores
1) "peter"
2) "76"
3) "tom"
4) "89"
5) "david"
6) "100"
```

ZREVRANGEBYSCORE 命令不仅是按照元素的分数从大往小的顺序给出结果，而且它的min和max参数的顺序和ZRANGEBYSCORE命令拾相反的。如：  

获取分数低于100分的前三个人：

```
127.0.0.1:6379> zrevrangebyscore scores 100 0 LIMIT 0 3 WITHscores
1) "david"
2) "100"
3) "tom"
4) "89"
5) "peter"
6) "76"
```
* 增加某个元素的分数  

```
ZINCRBY key increment member
```
ZINCRBY 命令可以增加一个元素的分数，返回的是更改后的分数。

```
127.0.0.1:6379> zincrby scores 8 xym
"110"
127.0.0.1:6379> zrange scores 0 -1 withscores
 1) "admin"
 2) "69"
 3) "peter"
 4) "76"
 5) "tom"
 6) "89"
 7) "david"
 8) "100"
 9) "xym"
10) "110"
```

如果指定的元素不存在，redis在执行命令前会先去建立并将它的分数赋为0再去执行操作。

### 2、命令拾遗

* 获得集合中的元素的数量  

```
ZCARD key

127.0.0.1:6379> zcard scores
(integer) 5
127.0.0.1:6379> zrange scores 0 -1 withscores
 1) "admin"
 2) "69"
 3) "peter"
 4) "76"
 5) "tom"
 6) "89"
 7) "david"
 8) "100"
 9) "xym"
10) "110"
```
* 获得指定分数范围内的元素个数  

```
127.0.0.1:6379> zcount scores (80 +inf
(integer) 3
```
* 删除一个或多个元素  

```
ZREM key member [member ...]
```

ZREM 命令的返回值是成功删除的元素数量（不包含本来就不存在的元素）

```
127.0.0.1:6379> zrange scores 0 -1 withscores
 1) "admin"
 2) "69"
 3) "peter"
 4) "76"
 5) "tom"
 6) "89"
 7) "david"
 8) "100"
 9) "xym"
10) "110"
127.0.0.1:6379> zrem scores tom
(integer) 1
127.0.0.1:6379> zrange scores 0 -1 withscores
1) "admin"
2) "69"
3) "peter"
4) "76"
5) "david"
6) "100"
7) "xym"
8) "110"
```
* 按照排名范围删除元素  

```
ZREMRANGEBYRANK key start top
```
ZREMRANGEBYRANK命令按照元素的分数从小到大的顺序（即索引0表示最小的值）删除处在指定排名范围内的所有元素，并返回删除的元素数量。如：

```
127.0.0.1:6379> zrange scores 0 -1 withscores
1) "admin"
2) "69"
3) "peter"
4) "76"
5) "david"
6) "100"
7) "xym"
8) "110"
127.0.0.1:6379> ZREMRANGEBYRANK scores 0 1
(integer) 2
127.0.0.1:6379> zrange scores 0 -1 withscores
1) "david"
2) "100"
3) "xym"
4) "110"
```

* 按照分数范围删除元素  

```
ZREMRANGEBYSCORE key min max
```
ZREMRANGEBYSCORE命令会删除指定分数范围内所有元素，参数min和max的特性和ZRANGEBYSCORE命令中的一样。返回值是删除的元素数量。如:

```
127.0.0.1:6379> zrange scores 0 -1 withscores
1) "tom"
2) "78"
3) "admin"
4) "98"
5) "david"
6) "100"
7) "xym"
8) "110"
127.0.0.1:6379> zremrangebyscore scores (98 101
(integer) 1
127.0.0.1:6379> zrange scores 0 -1 withscores
1) "tom"
2) "78"
3) "admin"
4) "98"
5) "xym"
6) "110"
```
* 获得元素排名  

```
ZRANK key member
ZREVRANK key member
```
ZRANK命令会按照元素分数从小到大的顺序获得指定的元素的排名（从0开始，即分数最小的元素排名为0），ZREVRANK命名相反（分数最大的元素排名为0）。如：

```
127.0.0.1:6379> zrange scores 0 -1 withscores
1) "tom"
2) "78"
3) "admin"
4) "98"
5) "xym"
6) "110"
127.0.0.1:6379> zrank scores xym
(integer) 2
127.0.0.1:6379> zrevrank scores xym
(integer) 0
```
* 计算有序集合的交集  

```
ZINTERSTORE destination numkeys key [key ...] [WEIGHTS weight [weight ...]] [AGGREGATE SUM|MIN|MAX]
```
ZINTERSTORE 命令用来计算多个有序集合的交集并将结果存储在destination键中（同样以有序集合的类型存储），返回值为destination键中元素的个数。destination键中元素的分数是由AGGREGATE参数决定的。  
（1）当AGGREGATE时SUM（默认值）时，destination键中的元素的分数是每个参与计算的集合中该元素分数的和。如： 

```
127.0.0.1:6379> zadd store1 1 a 2 b
(integer) 2
127.0.0.1:6379> zadd store2 10 a 20 b
(integer) 2
127.0.0.1:6379> Zinterstore destStore 2 store1 store2
(integer) 2
127.0.0.1:6379> zrange destStore 0 -1 withscores
1) "a"
2) "11"
3) "b"
4) "22"
```
（2）当AGGREGATE时MIN时，destination键中的元素的分数是每个参与计算的集合中该元素分数的最小值。如：

```
127.0.0.1:6379> zinterstore destStore 2 store1 store2 AGGREGATE min
(integer) 2
127.0.0.1:6379> zrange destStore 0 -1 withscores
1) "a"
2) "1"
3) "b"
4) "2"
```

（3）当AGGREGATE时MAX时，destination键中的元素的分数是每个参与计算的集合中该元素分数的最大值。如：

```
127.0.0.1:6379> zinterstore destStore 2 store1 store2 AGGREGATE max
(integer) 2
127.0.0.1:6379> zrange destStore 0 -1 withscores
1) "a"
2) "10"
3) "b"
4) "20"
```
ZINTERSTORE 命令还能够通过WEIGHTS参数设置每个集合的权重，每个集合在参与计算时元素的分数会被乘上该集合的权重。如：

```
127.0.0.1:6379> zinterstore destStore 2 store1 store2 WEIGHTS 1 0.1
(integer) 2
127.0.0.1:6379> ZRANGE destStore 0 -1 withscores
1) "a"
2) "2"
3) "b"
4) "4"

127.0.0.1:6379> zrange store1 0 -1 withscores
1) "a"
2) "1"
3) "b"
4) "2"

127.0.0.1:6379> zrange store2 0 -1 withscores
1) "a"
2) "10"
3) "b"
4) "20"
```

**计算方式为：a分数1\*1+10\*0.1=2，b分数2\*1+20\*0.1=4，注意如果加WEIGHTS参数，则后面对应每一个有序集合的权重，是多值**  
其他有序集合命令请查阅[Redis命令中文在线参考](http://doc.redisfans.com/)
