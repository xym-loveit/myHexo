---
title: redis入门指南-命令阶段I
date: 2017-05-23 15:48:38
categories: redis系列
tags: [redis命令,redis五种数据类型,字符串（string）,散列表（hash）,列表（list）,集合（set）,有序集合（zset）]
description: 了解了redis基础后，本章主要介绍redis五种数据类型及相应的命令。命令拾遗部分会对其他比较有用的命令进行补充介绍。
---
## 命令热身
### 1. 获得符合规则的键名列表  


```
KEYS pattern
```
pattern支持glob风格通配符格式，具体规则如下表：
符号|含义
---|---
？|匹配一个字符
*|匹配任意个（包括0个）字符
[]|匹配括号间任意字符，可以使用“-”符号表示一个范围，如a[b-d]可以匹配ab、ac、ad
\x|匹配字符x，用于转义符号。如要匹配？就需要使用 \\?

### 2. 判断键是否存在

```
EXISTS key
```
如果存在返回1，否则返回0。如：


```
127.0.0.1:6379> exists test
(integer) 1
127.0.0.1:6379> exists noexists
(integer) 0
127.0.0.1:6379>
```


### 3. 删除键


```
DEL key [key ...]
```
可以删除一个或多个键，返回数字是删除键的个数。如：


```
127.0.0.1:6379> del test
(integer) 1
127.0.0.1:6379> del test
(integer) 0
```
第二次执行del命令返回0，因为test已经被删除了，实际并没有删除任何键，所以返回值为0。

**技巧：DEL 命令的参数不支持通配符，但我们可以结合linux的管道和xargs命令自己实现删除所有符合规则的键。比如要删除所有以"user:\*"开头的键,就可以执行redis-cli KEYS "user:\*"|xargs redis-cli DEL。另外由于DEL命令支持多个键作为参数，所以还可以执行redis-cli DEL 'redis-cli KEYS "user:\*"' 来达到同样的效果，但是性能更好。**

### 4. 获得键值的数据类型

```
TYPE key
```
TYPE 命令用来获取键值的数据类型，返回值可能是string（字符串类型）、hash（散列类型）、list（列表类型）、set（集合类型）、zset（有序集合类型）。例如：

```
127.0.0.1:6379> set str str123
OK
127.0.0.1:6379> type str
string --字符串
127.0.0.1:6379> lpush list1 l1 l2 l3 l4
(integer) 4
127.0.0.1:6379> type list1
list --列表
127.0.0.1:6379> hmset people:1 name zhangsan age 22 work javacoder
OK
127.0.0.1:6379> type people:1
hash --散列
127.0.0.1:6379> zadd zset1 101 a 102 b
(integer) 2
127.0.0.1:6379> type zset1
zset --有序集合
127.0.0.1:6379> sadd set1 java oracle servet spring hibernate
(integer) 5
127.0.0.1:6379> type set1
set --集合
```

---

## 字符串类型
字符串类型是redis中最基本的数据类型，它能存储任何形式的字符串，包括二进制数据。你可以用其存储用户邮箱、json化的对象甚至是一张图片。一个字符串类型键允许存储的数据最大容量是512MB。
字符串类型是其他4中数据类型的基础，其他数据类型和字符串的差别从某种角度来说只是组织字符串的形式不同。

### 1、命令
* 取值赋值

```
SET key value
GET key
```
* 递增数字

```
INCR key
```
前面说过字符串可以存储任何形式的字符串，当存储的字符串是整数形式时，redis提供了一个使用的命令INCR，其作用是让当前键值递增，并返回递增后的值，用法：

```
127.0.0.1:6379> incr num
(integer) 1
127.0.0.1:6379> incr num
(integer) 2
```

包括INCR在内的所有redis命令都是原子操作，无论有多少客户端连接到redis执行命令都不可能出现竞态条件。


### 2、实践
提示：redis对于键的命名并没有强制的要求，但比较好的实践是用“对象类型:对象ID:对象属性”来命名一个键，如使用键user:1:friends来存储ID为1的用户的好友列表，对于多个单词则推荐使用“.”分隔，一方面是沿用以前的习惯（redis之前的版本键名不能包含空格等特殊符号），另一方面是在redis-cli中容易输入，无需使用双引号包裹。另外为了日后维护方便，键的命名一定要有意义，如u:f:1的可读性显然不如user:1:friends好（虽然采用较短的名称可以节省存储空间，但由于键值的长度往往大于键名的长度，所以这部分的节省大部分情况下并不如可读性来得重要）。

### 3、命令拾遗  
* 增加指定整数

```
INCRBY key increment
```
INCRBY命令与INCR命令基本一样，只不过前者可以通过increment 参数指定一次增加的数值，如：

```
127.0.0.1:6379> incrby num 20
(integer) 22
127.0.0.1:6379> incrby num 30
(integer) 52
```
* 减少指定的整数

```
DECR key
DECRBY key decrement
```
DECR命令与INCR命令用法相同，只不过时让键值递减，例如：

```
127.0.0.1:6379> decr num
(integer) 51
127.0.0.1:6379> decrby num 30
(integer) 21
127.0.0.1:6379>
```
DECR和INCR及DECRBY和INCRBY是相反的操作，可通过正负数字实现对方相同的功能。


* 指定增加浮点数

```
INCRBYFLOAT key increment
```
INCRBYFLOAT命令类似INCRBY 命令，差别是前者可以递增一个双精度浮点数，如：

```
127.0.0.1:6379> incrbyfloat fnum 2.764
"2.764"
127.0.0.1:6379> incrbyfloat fnum 2E+4
"20002.76399999999999935"
```
* 向尾部追加值

```
APPEND key value
```
APPEND作用是向键值的末尾追加value。如果键不存在则将改键的值设为value，即相当于SET key value。返回值是追加后的字符串长度。例如：

```
127.0.0.1:6379> append key " world"
(integer) 11
127.0.0.1:6379> get hello
(nil)
127.0.0.1:6379> get key
"hello world"
```
append命令的第二个参数加双引号，原因是该参数包含了空格，在redis-cli中输入需要双引号以示区分。

* 获取字符串长度

```
STRLEN key
```
STRLEN命令返回键值的长度，如果键不存在则返回0。例如：

```
127.0.0.1:6379> set key 你好
OK
127.0.0.1:6379> get key
"\xe4\xbd\xa0\xe5\xa5\xbd"
127.0.0.1:6379> strlen key
(integer) 6
```

* 同时获得/设置多个键值  

```
MGET key[key ...]
MSET key value [key value ...]
```


MGET/MSET与GET/SET相似，不过MGET/MSET可以同时获得/设置多个键的键值，例如:


```
127.0.0.1:6379> mset k1 v1 k2 v2 k3 v3 k4 v4
OK
127.0.0.1:6379> get k3
"v3"
127.0.0.1:6379> get k4
"v4"
127.0.0.1:6379> mget k1 k2 k3 k4
1) "v1"
2) "v2"
3) "v3"
4) "v4"
```

* 位操作


```
GETBIT key offset --获取字符串类型键指定位置的二进制位的值（0或1），索引从0开始
SETBIT key offset value --设置字符串类型键指定位置的二进制值，返回改位置旧值
BITCOUNT key [start] [end] --获得字符串类型键中值为1的二进制位个数
BITOP operation destkey key [key ...]--对多个字符串类型键进行位运算，并将结果存储在destkey参数指定的键中，bitop支持的操作命令（operation参数）有AND/OR/XOR/NOT。
```

一个字节有8个二进制位组成，Redis通过以上命令可以直接对二进制位进行操作。为演示首先准备将foo赋值为bar：


```
127.0.0.1:6379> set foo bar
OK
```
bar的三个字母对应的ASCII码分别为98/97/114转换为二进制如下：  

![bar的二进制](http://op7wplti1.bkt.clouddn.com/barbin.png)

GETBIT命令可以获得一个字符串类型键值指定位置的二进制的值（0/1）索引从0开始：


```
127.0.0.1:6379> getbit foo 0
(integer) 0
127.0.0.1:6379> getbit foo 6
(integer) 1
```
如果需要获取的二进制的索引超出了键值的二进制的实际长度则默认位值是0：

```
127.0.0.1:6379> getbit foo 100
(integer) 0
```
SETBIT命令可以设置字符串类型键值指定位置的二进制位的值，返回值是该位置的旧值。如我们要将foo键值设置为aar，可以通过操作将foo键的二进制位的索引第6位设为0，第7位设为1：

```
127.0.0.1:6379> setbit foo 6 0
(integer) 1
127.0.0.1:6379> setbit foo 7 1
(integer) 0
127.0.0.1:6379> get foo
"aar"
```
BITCOUNT 命令可以获得字符串类型键中值是1的二进制位个数，例如：

```
127.0.0.1:6379> bitcount foo
(integer) 10
```
可以通过参数来限制统计字符的范围，如我们只希望统计前2个字节（即“aa”）

```
127.0.0.1:6379> bitcount foo 0 1
(integer) 6
```

通过BITOP命令我们可以对bar和aar进行or运算：

```
127.0.0.1:6379> set foo1 bar
OK
127.0.0.1:6379> set foo2 aar
OK
127.0.0.1:6379> bitop or res foo1 foo2
(integer) 3
127.0.0.1:6379> get res
"car"
```

运算过程如下图：  

![bit的or操作运算过程](http://op7wplti1.bkt.clouddn.com/orcalc.png)


## 散列类型
redis是采用字典结构以键值对的形式存储数据，而散列类型（hash）的健值也是一种字典结构，其存储了字段（field）和字段值的映射，但字段值只能是字符串，不支持其他类型数据，换句话说，散列类型不能嵌套其他的数据类型。一个散列类型键可以包含至多2的32次方-1个字段。

*提示：除了散列类型，redis的其他数据类型同样不支持数据类型嵌套。比如集合类型的每个元素都只能是字符串，不能是另一个集合或散列表等。*

散列类型适合存储对象：使用对象类别和ID构成键名，使用字段表示对象属性，而字段值则存储属性值。例如要存储ID为2的汽车对象，可以分别使用名为color、name和price的3个字段来存储该辆汽车的颜色、名称和价格。存储结构如图：

![散列表存储汽车对象模型](http://op7wplti1.bkt.clouddn.com/hashcar.png)

相比关系型数据库（数据是以二维表的形式存储，要求所有记录都拥有同样的属性，无法单独为某条记录增减属性）redis散列类型不存在字段冗余等问题。redis并不要求每个键都依据固定的结构存储，我们完全可以自由的为任何键增减字段而不影响其他键。


### 1、命令

* 赋值与取值

```
HSET key field value --
HGET key field --根据field获取对应键值
HMSET key field value [field value ...]
HMGET key field [field ...]
HGETALL key
```
HSET命令用来给字段赋值，而HGET命令用来获得字段值。用法如下：


```
127.0.0.1:6379> HSET car price 500
(integer) 1
127.0.0.1:6379> HSET car name BMW
(integer) 1
127.0.0.1:6379> HGET car name
"BMW"
```
HSET命令方便之处在于不区分插入和更新操作，这意味着修改数据时不用事先判断字段是否存在再来决定执行的是插入操作（insert）还是更新（update）操作。当执行的是插入操作时（即之前的字段不存在）HSET命令返回1，当执行的是更新操作时（即之前字段存在）HSET命令返回0。更进一步，当键本身不存在时，HSET命令还会自动创建它。

当需要同时设置多个字段的值时，可以使用HMSET命令。例如，下面语句：

```
HSET key field1 value1
HSET key field2 value2
```

可以用HMSET命令改写成

```
HMSET key field1 value1 field2 value2
```
相应地，HMGET命令可以同时获得多个字段的值：

```
127.0.0.1:6379> HMGET car name price
1) "BMW"
2) "500"
```
如果想获取键中所有字段和字段值却不知道键中有哪些字段时（如汽车对象的例子，每个对象拥有的属性都未必相同）应该使用HGETALL命令。如：

```
127.0.0.1:6379> HGETALL car
1) "price"
2) "500"
3) "name"
4) "BMW"
```

* 判断字段是否存在

```
HEXISTS key field
```
HEXISTS 命令用来判断一个字段是否存在。如果存在则返回1，否则返回0（如果键不存在也返回0）。

```
127.0.0.1:6379> Hexists car model
(integer) 0
127.0.0.1:6379> HSET car model c200
(integer) 1
127.0.0.1:6379> Hexists car model
(integer) 1
```
* 当字段不存在时赋值

```
HSETNX key field value
```
HSETNX命令和HSET命令类似，区别在于如果字段已经存在，HSETNX命令将不执行任何操作。
HSETNX 命令是原子操作，不用担心竞态条件。

* 增加数字

```
HINCRBY key field increment
127.0.0.1:6379> Hincrby person score 60
(integer) 60
```

* 删除字段

```
HDEL key field [field ...]
```
HDEL命令可以删除一个或多个字段，返回值是被删除的字段个数：

```
127.0.0.1:6379> HDEL car price
(integer) 1
127.0.0.1:6379> Hgetall car
1) "name"
2) "BMW"
3) "model"
4) "c200"
```

### 2、命令拾遗

* 只获取字段名或字段值

```
HKEYS key
HVALS key
```
有时仅仅需要获取键中所有字段的名字而不需要字段值，那么可以使用HKEYS命令，如：

```
127.0.0.1:6379> Hkeys car
1) "name"
2) "model"
```
HVALS命令和HKEYS命令相对应，HVALS命令用来获得键中的所有字段值，如：

```
127.0.0.1:6379> HVALS car
1) "BMW"
2) "c200"
```
* 获得字段数量

```
HLEN key

127.0.0.1:6379> Hlen car
(integer) 2
```








