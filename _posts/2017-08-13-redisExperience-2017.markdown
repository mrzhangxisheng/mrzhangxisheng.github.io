---
layout:     post
title:      "redis笔记"
subtitle:   " \"学无止境\""
date:       post-08-13 16:38:00
author:     "Zhangxs"
header-img: "img/2017-redisbg_2017.jpg"
catalog: true
tags:
    - 技术
---



## 概述
redis一个持久化缓存工具，在当今互联网应用中是非常火。所以打算自学，这篇博客是第一篇。基本命令的操作.....

<p id = "build"></p>
---

## 博文
redis 工具的下载及安装，安装在liunx环境下，可以电脑上安装一个虚拟机[安装教程](http://www.cnblogs.com/wyw-action/articles/6833265.html)。我这里有一套教程2013年的，入门还是可以的[视频教程](http://pan.baidu.com/s/1eR9LFUE) 密码：63fe 具体的安装命令操作这里面都有。

##### 1、redis启动
```
zhang869011602@ubuntu:~$ cd Downloads/redis
zhang869011602@ubuntu:~/Downloads/redis$ ls
bin  dump.rdb  redis.conf  redis.conf~
zhang869011602@ubuntu:~/Downloads/redis$ ./bin/redis-server ./redis.conf
4472:C 13 Aug 01:56:40.681 # oO0OoO0OoO0Oo Redis is starting oO0OoO0OoO0Oo
4472:C 13 Aug 01:56:40.681 # Redis version=4.0.1, bits=64, commit=00000000, modified=0, pid=4472, just started
4472:C 13 Aug 01:56:40.681 # Configuration loaded

```
我这里是配置了，服务进程投胎启动的，所以关闭终端是没有问题的。你需要配置.在redis.conf 文件中找到daemonize属性，默认是on，将其改为yes就可以了。
```
daemonize yes
```
##### 2、redis的database默认是16个。如果你需要修改，可以去redis.conf文件修改他的属性。

##### 3、这里是一些基本命令,是我学习时整理的笔记。
```
set key value  --设置键值对
get key        --根据key获取值


查询所有的key
keys *  可以模糊查询 例如：keys n*  keys n*n   
*匹配多个字符  ？匹配单个字符  []中间可以写正则


randomkey 在库中随机获取一个key

type key 判断该key的数据类型

exisit key  查询key是否存在

rename oldname newname 修改key的名称  这个命令是不会去判断newname是否存在的，如果存在会直接覆盖原有key的值
renamenx oldname newname  修改key的名称   这个会去判断新newname是否存在。如果存在则不会修改。返回0

redis 在redis.conf配置文件中默认是存在16个数据库的。顺序是从0-15

redis 打开默认在0数据库

select 1 这就打开了编号为1的数据库

move keyName 2  把当前库的key名字为keyName的迁移到数据库2  那么使用exisit keyName 当前库，该key就不存在了

ttl key  查看key的生命周期 秒为单位
expire key 整型数据  设置key的生命周期 秒为单位

pttl key  查看key的生命周期 毫秒为单位
pexpire key 整型数据  设置key的生命周期 毫秒为单位

persist key 当前key设置为永久key

flushdb 刷新redis数据库。所有的键值都清空

setrange key 偏移量 需要替换的字符

append key 所要添加的字符串

getrange key 开始  结束  截取key值

incr key   自增1

decr key   自减1

incrby key n(整形)   增加n

decrby key n(整形)   减少n

incrbyfloat key n(浮点型) 增加n

setbit key  偏移量  替换的值         数据换二进制根据偏移量替换相应的值
```
##### 4、这里是redis的一些集合链表命令操作（list,set,order set,hash）
```
link 链表结构

lpush key value 
作用: 把值插入到链接头部

rpop key
作用: 返回并删除链表尾元素

rpush,lpop: 不解释

lrange key start  stop
作用: 返回链表中[start ,stop]中的元素
规律: 左数从0开始,右数从-1开始


lrem key count value
作用: 从key链表中删除 value值
注: 删除count的绝对值个value后结束
Count>0 从表头删除
Count<0 从表尾删除

ltrim key start stop
作用: 剪切key对应的链接,切[start,stop]一段,并把该段重新赋给key

lindex key index
作用: 返回index索引上的值,
如  lindex key 2

llen key
作用:计算链接表的元素个数
redis 127.0.0.1:6379> llen task
(integer) 3
redis 127.0.0.1:6379> 

linsert  key after|before search value
作用: 在key链表中寻找’search’,并在search值之前|之后,.插入value
注: 一旦找到一个search后,命令就结束了,因此不会插入多个value


rpoplpush source dest
作用: 把source的尾部拿出,放在dest的头部,
并返回 该单元值

场景: task + bak 双链表完成安全队列
Task列表                             bak列表
		
		


业务逻辑:
1:Rpoplpush task bak
2:接收返回值,并做业务处理
3:如果成功,rpop bak 清除任务. 如不成功,下次从bak表里取任务

brpop ,blpop  key timeout
作用:等待弹出key的尾/头元素, 
Timeout为等待超时时间
如果timeout为0,则一直等待

场景: 长轮询Ajax,在线聊天时,能够用到

Setbit 的实际应用

场景: 1亿个用户, 每个用户 登陆/做任意操作  ,记为 今天活跃,否则记为不活跃

每周评出: 有奖活跃用户: 连续7天活动
每月评,等等...

思路: 

Userid   dt  active
1        2013-07-27  1
1       2013-0726   1

如果是放在表中, 1:表急剧增大,2:要用group ,sum运算,计算较慢


用: 位图法 bit-map
Log0721:  ‘011001...............0’

......
log0726 :   ‘011001...............0’
Log0727 :  ‘0110000.............1’


1: 记录用户登陆:
每天按日期生成一个位图, 用户登陆后,把user_id位上的bit值置为1

2: 把1周的位图  and 计算, 
位上为1的,即是连续登陆的用户


redis 127.0.0.1:6379> setbit mon 100000000 0
(integer) 0
redis 127.0.0.1:6379> setbit mon 3 1
(integer) 0
redis 127.0.0.1:6379> setbit mon 5 1
(integer) 0
redis 127.0.0.1:6379> setbit mon 7 1
(integer) 0
redis 127.0.0.1:6379> setbit thur 100000000 0
(integer) 0
redis 127.0.0.1:6379> setbit thur 3 1
(integer) 0
redis 127.0.0.1:6379> setbit thur 5 1
(integer) 0
redis 127.0.0.1:6379> setbit thur 8 1
(integer) 0
redis 127.0.0.1:6379> setbit wen 100000000 0
(integer) 0
redis 127.0.0.1:6379> setbit wen 3 1
(integer) 0
redis 127.0.0.1:6379> setbit wen 4 1
(integer) 0
redis 127.0.0.1:6379> setbit wen 6 1
(integer) 0
redis 127.0.0.1:6379> bitop and  res mon feb wen
(integer) 12500001


如上例,优点:
1: 节约空间, 1亿人每天的登陆情况,用1亿bit,约1200WByte,约10M 的字符就能表示
2: 计算方便


集合 set 相关命令

集合的性质: 唯一性,无序性,确定性

注: 在string和link的命令中,可以通过range 来访问string中的某几个字符或某几个元素
但,因为集合的无序性,无法通过下标或范围来访问部分元素.

因此想看元素,要么随机先一个,要么全选

sadd key  value1 value2
作用: 往集合key中增加元素

srem value1 value2
作用: 删除集合中集为 value1 value2的元素
返回值: 忽略不存在的元素后,真正删除掉的元素的个数

spop key
作用: 返回并删除集合中key中1个随机元素

随机--体现了无序性

srandmember key
作用: 返回集合key中,随机的1个元素.

sismember key  value
作用: 判断value是否在key集合中
是返回1,否返回0

smembers key
作用: 返回集中中所有的元素

scard key
作用: 返回集合中元素的个数

smove source dest value
作用:把source中的value删除,并添加到dest集合中



sinter  key1 key2 key3
作用: 求出key1 key2 key3 三个集合中的交集,并返回
redis 127.0.0.1:6379> sadd s1 0 2 4 6
(integer) 4
redis 127.0.0.1:6379> sadd s2 1 2 3 4
(integer) 4
redis 127.0.0.1:6379> sadd s3 4 8 9 12
(integer) 4
redis 127.0.0.1:6379> sinter s1 s2 s3
1) "4"
redis 127.0.0.1:6379> sinter s3 s1 s2
1)"4"

sinterstore dest key1 key2 key3
作用: 求出key1 key2 key3 三个集合中的交集,并赋给dest


suion key1 key2.. Keyn
作用: 求出key1 key2 keyn的并集,并返回

sdiff key1 key2 key3 
作用: 求出key1与key2 key3的差集
即key1-key2-key3 


order set 有序集合
zadd key score1 value1 score2 value2 ..
添加元素
redis 127.0.0.1:6379> zadd stu 18 lily 19 hmm 20 lilei 21 lilei
(integer) 3

zrem key value1 value2 ..
作用: 删除集合中的元素

zremrangebyscore key min max
作用: 按照socre来删除元素,删除score在[min,max]之间的
redis 127.0.0.1:6379> zremrangebyscore stu 4 10
(integer) 2
redis 127.0.0.1:6379> zrange stu 0 -1
1) "f"

zremrangebyrank key start end
作用: 按排名删除元素,删除名次在[start,end]之间的
redis 127.0.0.1:6379> zremrangebyrank stu 0 1
(integer) 2
redis 127.0.0.1:6379> zrange stu 0 -1
1) "c"
2) "e"
3) "f"
4) "g"

zrank key member
查询member的排名(升续 0名开始)

zrevrank key memeber
查询 member的排名(降续 0名开始)

ZRANGE key start stop [WITHSCORES]
把集合排序后,返回名次[start,stop]的元素
默认是升续排列 
Withscores 是把score也打印出来

zrevrange key start stop
作用:把集合降序排列,取名字[start,stop]之间的元素

zrangebyscore  key min max [withscores] limit offset N
作用: 集合(升续)排序后,取score在[min,max]内的元素,
并跳过 offset个, 取出N个
redis 127.0.0.1:6379> zadd stu 1 a 3 b 4 c 9 e 12 f 15 g
(integer) 6
redis 127.0.0.1:6379> zrangebyscore stu 3 12 limit 1 2 withscores
1) "c"
2) "4"
3) "e"
4) "9"


zcard key
返回元素个数

zcount key min max
返回[min,max] 区间内元素的数量


zinterstore destination numkeys key1 [key2 ...] 
[WEIGHTS weight [weight ...]] 
[AGGREGATE SUM|MIN|MAX]
求key1,key2的交集,key1,key2的权重分别是 weight1,weight2
聚合方法用: sum |min|max
聚合的结果,保存在dest集合内

注意: weights ,aggregate如何理解?
答: 如果有交集, 交集元素又有socre,score怎么处理?
 Aggregate sum->score相加   , min 求最小score, max 最大score

另: 可以通过weigth设置不同key的权重, 交集时,socre * weights

详见下例
redis 127.0.0.1:6379> zadd z1 2 a 3 b 4 c
(integer) 3
redis 127.0.0.1:6379> zadd z2 2.5 a 1 b 8 d
(integer) 3
redis 127.0.0.1:6379> zinterstore tmp 2 z1 z2
(integer) 2
redis 127.0.0.1:6379> zrange tmp 0 -1
1) "b"
2) "a"
redis 127.0.0.1:6379> zrange tmp 0 -1 withscores
1) "b"
2) "4"
3) "a"
4) "4.5"
redis 127.0.0.1:6379> zinterstore tmp 2 z1 z2 aggregate sum
(integer) 2
redis 127.0.0.1:6379> zrange tmp 0 -1 withscores
1) "b"
2) "4"
3) "a"
4) "4.5"
redis 127.0.0.1:6379> zinterstore tmp 2 z1 z2 aggregate min
(integer) 2
redis 127.0.0.1:6379> zrange tmp 0 -1 withscores
1) "b"
2) "1"
3) "a"
4) "2"
redis 127.0.0.1:6379> zinterstore tmp 2 z1 z2 weights 1 2
(integer) 2
redis 127.0.0.1:6379> zrange tmp 0 -1 withscores
1) "b"
2) "5"
3) "a"
4) "7"

Hash 哈希数据类型相关命令

hset key field value
作用: 把key中 filed域的值设为value
注:如果没有field域,直接添加,如果有,则覆盖原field域的值

hmset key field1 value1 [field2 value2 field3 value3 ......fieldn valuen]
作用: 设置field1->N 个域, 对应的值是value1->N
(对应PHP理解为  $key = array(file1=>value1, field2=>value2 ....fieldN=>valueN))


hget key field
作用: 返回key中field域的值

hmget key field1 field2 fieldN
作用: 返回key中field1 field2 fieldN域的值

hgetall key
作用:返回key中,所有域与其值

hdel key field
作用: 删除key中 field域

hlen key
作用: 返回key中元素的数量

hexists key field
作用: 判断key中有没有field域

hinrby key field value
作用: 是把key中的field域的值增长整型值value

hinrby float  key field value
作用: 是把key中的field域的值增长浮点值value

hkeys key
作用: 返回key中所有的field

kvals key
作用: 返回key中所有的value
```
---
第一篇就讲到这里,下一篇我会模拟微博的使用。



## 后记

学无止境。Tomorrow Will Be Better

—— Zhangxs 后记于 2017.08.13
