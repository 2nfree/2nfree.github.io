# Redis 命令与数据类型


# 基本命令

```bash
#测试连接，返回pong连接成功
ping
#切换数据库 select [index]
select 1
#查看数据库大小
dbsize
#查看所有的key
keys *
#清空所有
flushall
#清空当前数据库
flushdb
#判断是否存在key exists [key]
exists name
#移动key到另一个数据库 move [key] [db]
move name 1
#设置key过期时间 expire [key] [second]
expire name 10
#查看key剩余时间 ttl [key]
ttl name
#移除key del [key]
del name
#查看key的类型 type [key]
type name
#设置基本值 set [key] [value]
set name zhangsan
#获取key
get name
#测试并发：100个并发连接 100000请求
redis-benchmark -h localhost -p 6379 -c 100 -n 100000
```

# 数据类型

## String 类型

String 除了字符串类型还能是数字类型，一般用于

- 计数器
- 统计多单位数量
- 粉丝数
- 对象缓存存储

```bash
#设置多个值 mset [key] [value] [key1] [value2] ....
mset name komorebi name1 komorebi
#获取多个值 mget [key] [key1]
mget name name1
#设置一个值，指定时间过期 setex [key] [second] [value]
setex name 30 hello
#设置一个值，如果该值已经存在，则不设置 setnx [key] [value]
setnx name hello
#设置多个值，同时判断值是否存在，存在则全部不创建（要么一起成功，要么一起失败）msetnx [key] [value] [key1] [value2] ....
msetnx name komorebi name1 komorebi
#向String类型的key中追加字符 append [key] [value]
#如果追加的key在当前数据库不存在，则相当于set key
append name "hello"
#获取字符串长度 strlen [key]
strlen name
#自增1 incr [key]
incr views
#指定数字增加 incrby [key] [num]
incrby views 10
#自减1 decr [key]
decr views
#指定数字减少 decrby [key] [num]
decrby views 10
#字符串范围 getrange [key] [begin] [end]
#后面为-1 为开区间
getrange name 0 3
getrange name 0 -1
#替换指定位置开始的字符串 setrange [key] [index] [value]
setrange name 1 xx
#设置对象 set [key]:[id]{[key1]:[value1],[key2]:[value2]}
#mset [key]:[id]:[key1] [value1] [key]:[id]:[key2] [value2]
set user:1{name:zhangsan,age:3}
mset user:1:name zhangsan user:1:age 3
#得到上一个值，设置下一个值 不存在就直接set getset [key] [value]
getset name komorebi 
```

## List 类型（列表）

```bash
#向list中添加值,插入的时候为列表的头部，最后一个插入的下标为0 lpush [key] [value]
lpush list one
lpush list two
lpush list three
#向list中的列表尾部添加值，下标为最后一位 rpush [key] [value]
rpush list right
#获取list中的所有值 lrange [key] [begin] [end]
lrange list 0 -1
#获取list中的指定值
lrange list 0 1
#移除list中的第一个元素 lpop [key]
lpop list
#移除list中的最后一个元素 rpop [key]
rpop list
#通过下标获取list中的某一个值 lindex [key] [index]
lindex list 0
#获取list中的长度 llen [key]
llen list
#移除list中的一个值 数字可以为>0 =0 <0 或者为其他数字
lrem list 1 one
```

## Set 类型（集合）

```bash
#set中的值不能重复
#添加
sadd myset "hello"
#查看所有元素
smembers myset
#判断当前值是不是存在set中
sismember myset hello
#查看当前set值的个数
scard myset
#移除某一个元素
srem myset hello
#随机抽取元素（可以指定个数）
srandmember myset 2
#随机移除一个元素
spop myset
#将元素从一个set移动到另一个set
smove myset1 myset2 hello
#求两个set的差集
sdiff myset1 myset2
#求两个set的交集
sinter myset1 myset2
#求两个set的并集
sunion myset1 myset2
```

## Hash 类型（map）

本质上和String类型没有什么区别

```bash
#创建一个hash
hset myhash field1 hello
#查询hash中的值
hget myhash field1
#设置多个值 会覆盖
hmset myhash field1 hello field2 world
#获取多个值
hmget myhash field1 field2
#获取hash里所有的键值对
hgetall myhash
#删除某个字段
hdel myhash field1
#获取hash的长度
hlen myhash
#判断hash中的某个field是否存在
hexists myhash field1
#只获得所有的field
hkeys myhash
#只获得所有的值
hvals myhash
#给hash中的指定field值自增 如果设置-1相当于自减
hincrby myhash field1 1
#判断是否存在，不存在就创建
hsetnx myhash field2 hello
```

## Zset 类型（有序集合）

set的基础上增加了值

```bash
#添加（需要添加一个数来进行排序）
zadd myset 1 one
zadd myset 2 two
#获取所有的值
zrange myset 0 -1
#从高到低获取所有的值
zrevrange myset 0 -1
#设定查询的范围（-inf +inf表示负无穷到正无穷，withscores表示加上参数）
zrangebyscore myset -inf +inf
zrangebyscore myset 1 2 withscores
#查看存在多少个元素
zcard myset
#移除对应的元素
zrem myset hello
#判断区间内共有多少元素
zcount myset 1 3
```

## geospatial（地理位置）

geo的底层实现原理其实就是Zset
可以使用zset命令操作geo

```bash
#一般直接通过java程序一次性导入
#参数 key 经度 纬度 名称
#geoadd 添加指定的经纬度 可以同时添加多个
geoadd china:city 121.47 31.23 shanghai
#geopos获取经纬度
geopos china:city shanghai
#geodist获取两地之间的距离 单位m：米，km：千米，mi：英里，ft：英尺
geodist china:city shanghai beijing km
#georadius以给定的经纬度为中心，找key范围内的元素
#后面跟withdist为直线距离 withcoord为显示经纬度 count设置获取多少个
georadius china:city 116 30 1000 km withdist withcoord count 3
#以已有元素位置为中心，找范围内的元素 其他参数与georadius一样
georadiusbymember china:city beijing 1000 km
#将二维的经纬度转换为一维的字符串，如果字符串越接近，那么距离越近
geohash china:city beijing
```

## hyperloglog（基数统计算法）

什么是基数
A{1,3,5,7,8,7,8}
B{1,3,5,7,8,8}
基数（不重复的元素的个数）=5，可以接受误差

```bash
#添加元素
pfadd mykey a b c d e f g
#统计元素的个数
pfcount mykey
#合并两个集合为mykey3（去掉重复元素）
pfmerge mykey3 mykey mykey2
```

## bitmaps

位存储
一般用于存储0和1两种状态

```bash
#设置位存储 第一位表示下标 第二位存1或者0
setbit sign 0 0
#查询某一位的状态
getbit sign 0
#统计为1的状态
bitcount sign
```


---

> 作者: [komorebi](https://blog.2nfree.com/)  
> URL: https://blog.2nfree.com/posts/redis-command/  

