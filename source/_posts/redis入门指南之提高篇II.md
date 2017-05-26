---
title: redis入门指南之提高篇II
date: 2017-05-26 10:58:16
categories: redis系列
tags: [redis命令,SORT排序,Redis排序,SORT参数讲解,排序性能,排序缓存]
description: Redis中Sort排序命令详解，关于BY、GET、 STORE参数详解；排序应注意的性能问题等。
---

###### 重要星级 ★★★★★

---

## Redis排序  
### 1、有序集合的集合操作  
集合类型提供了强大的集合操作命令，但是如果需要排序就要用到有序集合类型。Redis作者在设计Redis的命令时考虑到了不同类型的使用场景，对于不常用到的或者在不损失过多性能的前提下可以使用现有命令来实现的功能，Redis就不会单独提供命令来实现。这一原则使得Redis在拥有强大功能的同时保持着相对精简的命令。  
有序集合常见的使用场景是大数据排序，如游戏玩家排行榜，所有很少会需要获得键中的全部数据。同样Redis认为开发者在做完交集、并集运算后不需要直接获得全部结果，而是会希望将结果存入新的键中以便后续处理。这解释了为什么有序集合只有ZINTERSTORE和ZUNIONSTORE命令而没有ZINTER和ZUNION命令。  
当然实际使用中确实会有直接获取集合运算结果的情况，除了等待Redis加入相关命令，我们还可以使用multi，zinterstore，zrange，del和exec这5个命令自己实现ZINTER。  
### 2、SORT命令  
SORT命令可以对列表类型、集合类型和有序集合类型键进行排序，并且可以完成与关系型数据库中的链接查询相类似的任务。如：  

    //对结合类型进行操作
    SMEMBERS tag:java:posts
    1) "1"
    2) "2"
    3) "6"
    4) "10"
    5) "12"
    127.0.0.1:6379> sort tag:java:posts DESC
    1) "12"
    2) "10"
    3) "6"
    4) "2"
    5) "1"
    //对列表类型进行操作
    127.0.0.1:6379> lpush mylist 4 2 6 1 3 7
    (integer) 6
    127.0.0.1:6379> sort mylist
    1) "1"
    2) "2"
    3) "3"
    4) "4"
    5) "6"
    6) "7"
    //对有序集合类型排序会忽略元素的分数，只针对元素自身的值进行排序
    127.0.0.1:6379> zadd myset 50 2 40 3 20 1 60 5
    (integer) 4
    127.0.0.1:6379> sort myset
    1) "1"
    2) "2"
    3) "3"
    4) "5"
    
除了可以排列数字，SORT命令还可以通过ALPHA参数实现按照字典顺序排列非数字元素，如：  

    127.0.0.1:6379> lpush mylist a c e d B C A
    (integer) 7
    127.0.0.1:6379> 
    127.0.0.1:6379> sort mylist
    (error) ERR One or more scores can't be converted into double
    127.0.0.1:6379> sort mylist alpha
    1) "a"
    2) "A"
    3) "B"
    4) "c"
    5) "C"
    6) "d"
    7) "e"

SORT命令还支持LIMIT参数来返回指定范围的结果。用法和SQL语句一样，LIMIT offset count，表示跳过前offset个元素并获取之后的count个元素。SORT命令参数可以组合使用，如：  

    127.0.0.1:6379> sort tag:java:posts DESC
    1) "12"
    2) "10"
    3) "6"
    4) "2"
    5) "1"
    127.0.0.1:6379> sort tag:java:posts DESC limit 1 2
    1) "10"
    2) "6"
    127.0.0.1:6379> sort tag:java:posts DESC limit 1 3
    1) "10"
    2) "6"
    3) "2"

### 3、BY参数  
很多情况下列表（或集合、有序集合）中存储的元素值代表的是对象ID（如标签集合中存储的是文章对象的ID），单纯对这些ID自身排序有时意义并不大。更多的时候我们希望根据ID对应的对象的某个属性进行排序。我们可以通过有序集合键来存储文章ID列表，以修改时间作为有序集合的分数，所有文章的ID顺序和文章的发布时间（或更新时间）并不是完全一致。因此对ID本身排序就没什么意义了。使用散列表存储文章对象，其中time字段的值就是文章的发布时间。假设现在我们知道ID为“2”、”6“、”12“和”26“的四篇文章的time值分别为”1352619200“，”1352619600“，”1352620100“和”1352620000“（Unix时间）。如果要按照文章的发布时间递减排列结果应为”12“，”26“，”6“，”2“。位了获得这一的结果，需要使用SORT命令的另一个强大的参数BY。  
BY参数的语法为“BY 参考键”。其中参考键可以是字符串类型键或者是散列类型键的某个字段（表示为键名->字段名）。如果提供了BY参数，SORT命令将不再依据元素自身的值排序，而是对每个元素使用元素的值替换参考键中的第一个“*”并获取其值，然后依据该值对元素排序。如：  

    //添加测试数据
    127.0.0.1:6379> hmset post:2 time 1352619200 name java编程思想 price 86.6
    OK
    127.0.0.1:6379> hmset post:26 time 1352620000 name java核心技术 price 98.2
    OK
    127.0.0.1:6379> hmset post:12 time 1352620000 name Java并发编程的艺术 price 43.5
    OK
    127.0.0.1:6379> hmset post:6 time 1352619600  name java2 price 2.5
    OK
    127.0.0.1:6379> hmset post:5 time 1352622221 name java3 price 15
    OK
    127.0.0.1:6379> ZRANGE tag:java:posts 0 -1
    1) "2"
    2) "6"
    3) "12"
    4) "26"
    5) "5"
    //使用BY参数按post对象的time属性倒序，且26和12的time值相同时，我们发现26在前12在后
    127.0.0.1:6379> sort tag:java:posts BY post:\*->time DESC 
    1) "5"
    2) "26"
    3) "12"
    4) "6"
    5) "2"

**上例中SORT命令会去读取post:2、post:6、post:12、post:26、post:5几个散列键中的time字段的值并以此决定tag:java:posts键中各个文章ID的顺序。 ** 

除了散列类型之外，参考键还可以是字符串类型，如：  

    127.0.0.1:6379> lpush sortList 2 1 3
    (integer) 3
    127.0.0.1:6379> set score:1 50
    OK
    127.0.0.1:6379> set score:2 100
    OK
    127.0.0.1:6379> set score:3 -10
    OK
    127.0.0.1:6379> sort sortList by score:\* DESC
    1) "2"
    2) "1"
    3) "3"

当参考键名不包含“*”时（即常量键名，与元素值无关），SORT命令将不会执行排序操作，因为Redis认为这种情况时没有意义的（因为所有要比较的值都一样）。例如：  

    127.0.0.1:6379> sort sortList by abc
    1) "3"
    2) "1"
    3) "2"
    
例中abc是常量名（甚至abc键可以不存在），此时SORT的结果与LRANGE的结果相同，没有执行排序操作。在不需要排序但需要借助SORT命令获得与元素相关联的数据时，常量键是很有用的。如果几个元素的参考键值相同，则SORT命令会再去比较元素本身的值来决定元素的顺序。如：  

    127.0.0.1:6379> lpush sortList 4
    (integer) 4
    127.0.0.1:6379> set score:4 50
    OK
    127.0.0.1:6379> sort sortList BY score:\* DESC
    1) "2"
    2) "4" //4的参考值和1一样也是50，但比较元素本身4比1大，所以4在前
    3) "1"
    4) "3"

当某个元素的参考键不存在时，会默认参考键的值为0：  

    127.0.0.1:6379> LPUSH sortList 5
    (integer) 5
    127.0.0.1:6379> sort sortList BY score:\* DESC
    1) "2"
    2) "4"
    3) "1"
    4) "5"//默认参考键的值为0，比3的参考键的值（-10）要大，所以排列在3前
    5) "3"

补充：参考键虽然支持散列类型，但是“*”只能在“->”符号前面（即键名部分）才有用，在“->”后（即字段名部分）会被当成字段名本身而不会作为占位符被元素值替换，即常量键名。但是实际运行时会发现一个有趣的结果：  

    127.0.0.1:6379> sort sortList BY somekey->somefield:\*
    1) "1"
    2) "2"
    3) "3"
    4) "4"
    5) "5"

上面提到了当参数键名是常量键名时SORT命令将不会执行排序操作，然而上面例子中确实进行了排序，而且只是对元素本身进行排序。这是因为Redis判断参考键名是不是常量键名的方式是判断参考键名中是否包含“\*”，而somekey->somefield:\*中包含“\*”所以不是常量键名。所以在排序的时候Redis对每个元素都会读取键somekey中的somefield:\*字段（"\*"不会被替换），无论能否获得其值，每个元素的参考键值是相同的，所以Redis会按照元素本身的大小排列。  

### 4、GET参数  
GET参数不影响排序，它的作用是使SORT命令的返回结果不再是元素自身的值，而是GET参数中指定的键值。GET参数的规则和BY参数一样，GET参数也支持字符串类型和散列类型的键，并使用“\*”作为占位符。要实现在排序后直接返回ID对应的文章标题，代码如下：  

    //按post的time属性倒序，返回post的name属性值
    127.0.0.1:6379> sort tag:java:posts BY post:\*->time DESC GET post:\*->name
    1) "java3"
    2) "java\xe6\xa0\xb8\xe5\xbf\x83\xe6\x8a\x80\xe6\x9c\xaf"
    3) "Java\xe5\xb9\xb6\xe5\x8f\x91\xe7\xbc\x96\xe7\xa8\x8b\xe7\x9a\x84\xe8\x89\xba\xe6\x9c\xaf"
    4) "java2"
    5) "java\xe7\xbc\x96\xe7\xa8\x8b\xe6\x80\x9d\xe6\x83\xb3"

在一个SORT命令中可以有多个GET参数（而BY参数只有一个），所以还可以如下使用：  

    127.0.0.1:6379> sort tag:java:posts BY post:\*->time DESC GET post:\*->name GET post:\*->time
     1) "java3"
     2) "1352622221"
     3) "java\xe6\xa0\xb8\xe5\xbf\x83\xe6\x8a\x80\xe6\x9c\xaf"
     4) "1352620000"
     5) "Java\xe5\xb9\xb6\xe5\x8f\x91\xe7\xbc\x96\xe7\xa8\x8b\xe7\x9a\x84\xe8\x89\xba\xe6\x9c\xaf"
     6) "1352620000"
     7) "java2"
     8) "1352619600"
     9) "java\xe7\xbc\x96\xe7\xa8\x8b\xe6\x80\x9d\xe6\x83\xb3"
    10) "1352619200"
  
可见有N个GET参数，每个元素返回的结果就有N行。如果还需要返回文章的ID，可以使用GET #,也就是说GET# 会返回元素本身 

    127.0.0.1:6379> sort tag:java:posts BY post:\*->time DESC GET post:\*->name GET post:\*->time GET #
     1) "java3"
     2) "1352622221"
     3) "5"
     4) "java\xe6\xa0\xb8\xe5\xbf\x83\xe6\x8a\x80\xe6\x9c\xaf"
     5) "1352620000"
     6) "26"
     7) "Java\xe5\xb9\xb6\xe5\x8f\x91\xe7\xbc\x96\xe7\xa8\x8b\xe7\x9a\x84\xe8\x89\xba\xe6\x9c\xaf"
     8) "1352620000"
     9) "12"
    10) "java2"
    11) "1352619600"
    12) "6"
    13) "java\xe7\xbc\x96\xe7\xa8\x8b\xe6\x80\x9d\xe6\x83\xb3"
    14) "1352619200"
    15) "2"

### 5、STORE参数  
默认情况下SORT会直接返回排序结果，如果希望保存排序结果，可以使用STORE参数。如果希望把记过保存到sort.result键中：  

    127.0.0.1:6379> sort tag:java:posts BY post:\*->time DESC GET post:\*->name GET post:\*->time GET # STORE sort.result
    (integer) 15
    127.0.0.1:6379> lrange sort.result 0 -1
     1) "java3"
     2) "1352622221"
     3) "5"
     4) "java\xe6\xa0\xb8\xe5\xbf\x83\xe6\x8a\x80\xe6\x9c\xaf"
     5) "1352620000"
     6) "26"
     7) "Java\xe5\xb9\xb6\xe5\x8f\x91\xe7\xbc\x96\xe7\xa8\x8b\xe7\x9a\x84\xe8\x89\xba\xe6\x9c\xaf"
     8) "1352620000"
     9) "12"
    10) "java2"
    11) "1352619600"
    12) "6"
    13) "java\xe7\xbc\x96\xe7\xa8\x8b\xe6\x80\x9d\xe6\x83\xb3"
    14) "1352619200"
    15) "2"

保存后的键的类型为列表类型，如果键已经存在则会覆盖它。加上STORE参数后SORT命令的返回值为结果的个数。STORE参数常用来结合EXPIRE命令缓存排序结果，如下面的伪代码：  

    //判断是否存在之前排序结果的缓存
    $isCacheExists=Exists cache.sort
    if $isCacheExists is 1
        //如果存在直接返回
        return LRANGE cache.sort,0 -1
    else 
        //如果不存在，则使用SORT命令排序并将结果存入cache.sort键中作为缓存
        $sortResult=SORT some.list STORE cache.sort
        //设置生成时间为10分钟
        EXPIRE cache.sort,600
        //返回排序结果
        return $sortResult

### 6、性能优化  
SORT是Redis中最强大最复杂的命令之一，如果使用不好很容易成为性能瓶颈。SORT命令的时间复杂度是O（n+mlogm），其中n表示要排序的列表（集合或有序集合）中的元素个数，m表示要返回的元素个数。当n较大的时候SORT命令的性能相对较低，并且Redis在排序前会建立一个长度为n的容器来存储待排序的元素，虽然是一个临时的过程，但如果同时进行较多的大数据量排序操作则会严重影响性能。  
所以开发中使用SORT命令时需要注意以下几点：  
（1）尽可能减少待排序键中元素数量（使n尽可能小）。
（2）使用LIMIT参数只获取需要的数据（使m尽可能小）。
（3）如果要排序的数据量较大，尽可能使用STORE参数将结果缓存。

