# Redis 高级功能


# 事务

redis事务本质：一组命令的集合，一个事务的所有命令都会被序列化，在事务执行的过程中，会按照顺序执行
一次性、顺序性、排他性
redis的事务没有隔离级别的概念
所有的命令在事务中，并没有直接被执行，只有发起执行命令才会被执行
redis单条命令是保存原子性的，但是事务不保证原子性

## redis的事务执行

1. 开启事务（multi）
2. 命令入队
3. 执行事务（exec）

放弃事务的命令为：discard

## 事务异常

1. 编译型异常（代码有问题，命令有错）：事务中所有的命令都不会被执行
2. 运行时异常：如果事务队列存在语法性，那么执行命令的时候，其他命令可以正常执行，错误命令抛出异常

## 锁

### 悲观锁

| 悲观，认为什么时候都会出问题，无论做什么都会加锁

### 乐观锁

| 乐观，认为什么时候都不会出现问题，所以不会加锁。更新数据的时候去判断，在此期间是否有人修改这个数据
| 获取version
| 更新的时候比较version

监视对象使用 watch [key]  解锁使用 unwatch

```bash
#shell1
127.0.0.1:6379> set money 200
OK
127.0.0.1:6379> set out 0
OK
127.0.0.1:6379> watch money #监视money
OK
127.0.0.1:6379> MULTI
OK
127.0.0.1:6379> decrby money 10
QUEUED
127.0.0.1:6379> incrby out 10
QUEUED
127.0.0.1:6379> exec #执行失败
(nil)
#shell2
127.0.0.1:6379> get money
"200"
127.0.0.1:6379> set money 1000 #另外一个线程修改了元素
OK
```

# redis 配置

## redis 中对大小写不敏感

{{< image src="redis-config1.webp" >}}

## 可以包含其他配置文件

{{< image src="redis-config2.webp" >}}

## 网络相关

```bash
#绑定ip
bind 127.0.0.1
#保护模式
protected-mode yes
#端口设置
port 6379
```

## 通用配置

```bash
#以守护进程的方式运行，默认为no
daemonize yes
#如果以后台方式运行，需要指定一个pid文件
/var/run/redis_6379.pid

#日志
# Specify the server verbosity level.
# This can be one of:
# debug (a lot of information, useful for development/testing)
# verbose (many rarely useful info, but not a mess like the debug level)
# notice (moderately verbose, what you want in production probably) 生产环境
# warning (only very important / critical messages are logged)
loglevel notice
#日志的输出位置及文件名
logfile 
#数据库数量 默认16个
database 16
#是否总是显示logo
always-show-logo yes
```

## 快照

```bash
# 如果900s内 至少有 1 key进行了修改，就进行持久化操作
save 900 1
# 如果300s内 至少有 10 key进行了修改，就进行持久化操作
save 300 10 
# 如果60s内 至少有 10000 key进行了修改，就进行持久化操作
save 60 10000
# 之后的持久化会自定义测试

# 持久化出现错误是否继续工作
stop-writes-on-bgsave-error yes
# 是否压缩rdb文件
rdbcompression yes
# 保存rdb文件时进行错误检查
rdbchecksum yes
# rdb文件保存的目录
dir ./
```

## 安全

```bash
# 设置密码
config set requirepass 123456
# 登录
auth 123456
# 设置能连接上redis的客户端的最大数量
maxclients 10000
# redis设置最大的内存
maxmemory <byte>
# 内存达到上限的处理策略
# noeviction: 不删除策略, 达到最大内存限制时, 如果需要更多内存, 直接返回错误信息。（默认值）
# allkeys-lru: 所有key通用; 优先删除最近最少使用(less recently used ,LRU) 的 key。
# volatile-lru: 只限于设置了 expire 的部分; 优先删除最近最少使用(less recently used ,LRU) 的 key。
# allkeys-random: 所有key通用; 随机删除一部分 key。
# volatile-random: 只限于设置了 expire 的部分; 随机删除一部分 key。
# volatile-ttl: 只限于设置了 expire 的部分; 优先删除剩余时间(time to live,TTL) 短的key。
maxmemory-policy noeviction
```

## AOF设置

```bash
# 默认不开启aof模式，默认使用rdb方式持久化，在大部分情况下rdb基本够用
appendonly no
# 持久化文件名字
appendfilename "appendonly.aof"
#每次修改都会 sync。消耗性能
appendfsync always
#每秒执行一次 sync，可能会丢失这一秒的数据。
appendfsync everysec
#不执行 sync，这个时候操作系统自己同步数据，速度最快
appendfsync no
```


---

> 作者: [komorebi](https://blog.2nfree.com/)  
> URL: https://blog.2nfree.com/posts/redis-config/  

