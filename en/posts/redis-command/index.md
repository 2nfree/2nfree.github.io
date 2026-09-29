# Redis Commands and Data Types


# Basic Commands

```bash
# test the connection, return pong for successful connection
ping
# switch databases `select [index]`
select 1
# view database size
dbsize
# view all keys
keys *
# clear all data
flushall
# clear the current database
flushdb
# whether the key exists `exists [key]`
exists name
# move the key to another database `move [key] [db]`
move name 1
# setting the key expiration time `expire [key] [second]`
expire name 10
# view key expiration time `ttl [key]`
ttl name
# delete key `del [key]`
del name
# view the type of key `type [key]`
type name
# set key value `set [key] [value]`
set name zhangsan
# get key value
get name
# performance test
redis-benchmark -h localhost -p 6379 -c 100 -n 100000
```

# Data types

## String

String can be a numeric type in addition to a string type, and is generally used for

- Counters
- Counting multi-unit quantities
- Fans
- Object cache storage

```bash
# set multiple values `mset [key] [value] [key1] [value2] ....`
mset name komorebi name1 komorebi
# get multiple values `mget [key] [key1]`
mget name name1
# set a value with an expiration time `setex [key] [second] [value]`
setex name 30 hello
# set a value if it does not exist `setnx [key] [value]`
setnx name hello
# set multiple values, determine if the values exist, and if they do, they are all not created (either they succeed together or fail together)
# `msetnx [key] [value] [key1] [value2] ....`
msetnx name komorebi name1 komorebi
# append characters to a key of type String, if the appended key does not exist in the current database, it is equivalent to set key
# `append [key] [value]`
append name "hello"
# get string length `strlen [key]`
strlen name
# increase by 1 `incr [key]`
incr views
# increase in specified number `incrby [key] [num]`
incrby views 10
# decrease by 1 `decr [key]`
decr views
# decrease in specified number `decrby [key] [num]`
decrby views 10
# string range, set -1 to open interval `getrange [key] [begin] [end]`
getrange name 0 3
getrange name 0 -1
# replace the string starting at the specified position `setrange [key] [index] [value]`
setrange name 1 xx
# set objects
# `set [key]:[id]{[key1]:[value1],[key2]:[value2]}`
# `mset [key]:[id]:[key1] [value1] [key]:[id]:[key2] [value2]`
set user:1{name:zhangsan,age:3}
mset user:1:name zhangsan user:1:age 3
# get the previous value, set the next value,if previous value doesn't exist, just set `getset [key] [value]`
getset name komorebi 
```

## List

```bash
# add a value to the list, inserted at the head of the list, the last insertion of the subscript 0 `lpush [key] [value]`
lpush list one
lpush list two
lpush list three
# adds a value to the end of the list, with the subscript being the last digit `rpush [key] [value]`
rpush list right
# get all values in the list `lrange [key] [begin] [end]`
lrange list 0 -1
# get the specified value in a list
lrange list 0 1
# remove the first element of the list `lpop [key]`
lpop list
# 移remove the last element of the list `rpop [key]`
rpop list
# get a value in the list by subscript `lindex [key] [index]`
lindex list 0
# Get the length of list `llen [key]`
llen list
# remove a value from the list Numbers can be >0 =0 <0 or other numbers
lrem list 1 one
```

## Set

The values in set cannot be duplicated

```bash
# add value
sadd myset "hello"
# view all elements
smembers myset
# determine if the current value exists in set
sismember myset hello
# view the number of current set values
scard myset
# remove an element
srem myset hello
# random selection of elements (number can be specified)
srandmember myset 2
# randomly remove an element
spop myset
# moves elements from one set to another
smove myset1 myset2 hello
# find the difference of two sets
sdiff myset1 myset2
# find the intersection of two sets
sinter myset1 myset2
# find the union of two sets
sunion myset1 myset2
```

## Hash

```bash
# create a hash
hset myhash field1 hello
# get hash value
hget myhash field1
# setting multiple values,same name overwrite
hmset myhash field1 hello field2 world
# get multiple values
hmget myhash field1 field2
# get all fields and values
hgetall myhash
# del key
hdel myhash field1
# get hash value lens
hlen myhash
# determine if the current value exists in hash
hexists myhash field1
# get all fields
hkeys myhash
# get all values
hvals myhash
# self-increase, -1 is equivalent to self-decrease
hincrby myhash field1 1
# determine if the current value exists, create if it doesn't
hsetnx myhash field2 hello
```

## Zset

zset adds sorted values to set

```bash
# add (need to add a number for sorting)
zadd myset 1 one
zadd myset 2 two
# get all values
zrange myset 0 -1
# get all values asc
zrevrange myset 0 -1
# set the range of the query (-inf +inf means negative infinity to positive infinity, withscores means add parameters)
zrangebyscore myset -inf +inf
zrangebyscore myset 1 2 withscores
# view how many elements exist
zcard myset
# remove element
zrem myset hello
# determine how many elements are in the interval
zcount myset 1 3
```

## geospatial

Redis geospatial indexes let you store coordinates and search for them. This data structure is useful for finding nearby points within a given radius or bounding box
The principle of geo is actually Zset, you can use the zset command to operate geo

```bash
# adds a location to a given geospatial index (note that longitude comes before latitude with this command).
geoadd china:city 121.47 31.23 shanghai
# get latitude and longitude
geopos china:city shanghai
# get the distance between two places
geodist china:city shanghai beijing km
# find the elements in the key range centered on the given latitude and longitude
# withdist for straight line distance, withcoord for displaying latitude and longitude, count set how many to get
georadius china:city 116 30 1000 km withdist withcoord count 3
# finds elements within a range centered on the position of an existing element Other parameters are the same as georadius
georadiusbymember china:city beijing 1000 km
# converts two-dimensional latitude and longitude to a one-dimensional string, if the strings are closer together, then the closer the distance is
geohash china:city beijing
```

## hyperloglog

HyperLogLog is a probabilistic data structure that estimates the cardinality of a set. As a probabilistic data structure, HyperLogLog trades perfect accuracy for efficient space utilization.
A{1,3,5,7,8,7,8}
B{1,3,5,7,8,8}
Base (number of elements that are not repeated) = 5, acceptable error

```bash
# add elements
pfadd mykey a b c d e f g
# count the number of elements
pfcount mykey
# merge two collections into mykey3 (remove duplicate elements)
pfmerge mykey3 mykey mykey2
```

## bitmaps

generally used to store both 0 and 1 states

```bash
# the first bit indicates the subscript and the second bit stores 1 or 0.
setbit sign 0 0
# get the status of a bit
getbit sign 0
# states of 1
bitcount sign
```


---

> Author: [komorebi](https://blog.2nfree.com/)  
> URL: https://blog.2nfree.com/en/posts/redis-command/  

