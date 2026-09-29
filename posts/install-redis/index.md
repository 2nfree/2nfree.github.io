# Redis 编译安装


# 初始准备

本次安装基于 Centos7

## 创建redis用户

```bash
groupadd redis
useradd -g redis redis -s /sbin/nologin -M
```

## 创建redis相关目录

```bash
#数据目录
mkdir -pv /data/redis
chown -R redis:redis /data/redis

#日志目录
mkdir -pv /data/logs/redis
chown -R redis:redis /data/logs/redis

#redis目录
mkdir -pv /usr/local/redis/{bin,etc}
```

# 编译安装

```bash
#安装依赖
yum install clang -y

cd /data/packages
#下载
wget http://download.redis.io/releases/redis-6.2.6.tar.gz

#解压
tar -zxvf redis-6.2.6.tar.gz

#编译安装
cd redis-6.2.6 && CC=clang make
#报错就 make MALLOC=libc
install -p -D -m 0755 src/redis-benchmark /usr/local/redis/bin/
install -p -D -m 0755 src/redis-check-aof /usr/local/redis/bin/
install -p -D -m 0755 src/redis-check-rdb /usr/local/redis/bin/
install -p -D -m 0755 src/redis-cli /usr/local/redis/bin/
install -p -D -m 0755 src/redis-sentinel /usr/local/redis/bin/
install -p -D -m 0755 src/redis-server /usr/local/redis/bin/
install -p -D -o redis redis.conf /usr/local/redis/etc/
ln -svf /usr/local/redis/bin/redis-cli /usr/bin/redis-cli
```

# 配置相关

## 修改配置文件

```bash
#修改配置文件
export REDIS_HOME=/usr/local/redis
sed -i s@'^logfile .*'@"logfile /data/logs/redis/redis.log"@ $REDIS_HOME/etc/redis.conf
sed -i s@'^dir .*'@"dir /data/redis"@ $REDIS_HOME/etc/redis.conf
sed -i s@'^pidfile .*'@"pidfile /data/redis/redis_6379.pid"@ $REDIS_HOME/etc/redis.conf
sed -i s@'^bind .*'@"bind 0.0.0.0"@ $REDIS_HOME/etc/redis.conf
```

## 配置 shutdown脚本 脚本

```bash
#创建redis-shutdown脚本
vim /usr/local/redis/bin/redis-shutdown
```

```bash
#!/bin/bash
test x"$REDIS_DEBUG" != x && set -x

REDIS_CLI=/usr/local/redis/bin/redis-cli

SERVICE_NAME="$1"
if [ -z "$SERVICE_NAME" ]; then
   SERVICE_NAME=redis
fi

CONFIG_FILE="/usr/local/redis/etc/$SERVICE_NAME.conf"

HOST=`awk '/^[[:blank:]]*bind/ { print $2 }' $CONFIG_FILE | tail -n1`
PORT=`awk '/^[[:blank:]]*port/ { print $2 }' $CONFIG_FILE | tail -n1`
PASS=`awk '/^[[:blank:]]*requirepass/ { print $2 }' $CONFIG_FILE | tail -n1`
SOCK=`awk '/^[[:blank:]]*unixsocket\s/ { print $2 }' $CONFIG_FILE | tail -n1`

HOST=${HOST:-127.0.0.1}
if [ "$SERVICE_NAME" = redis ]; then
    PORT=${PORT:-6379}
else
    PORT=${PORT:-26739}
fi

[ -z "$PASS"  ] || ADDITIONAL_PARAMS="-a $PASS"

if [ -e "$SOCK" ] ; then
        $REDIS_CLI -s $SOCK $ADDITIONAL_PARAMS shutdown
else
        $REDIS_CLI -h $HOST -p $PORT $ADDITIONAL_PARAMS shutdown
fi
```

```bash
chmod +x /usr/local/redis/bin/redis-shutdown
```

## 创建 Systemd 配置

```bash
cat >/usr/lib/systemd/system/redis.service<<EOF
[Unit]
Description=Redis persistent key-value database
After=network.target
After=network-online.target
Wants=network-online.target

[Service]
ExecStart=/usr/local/redis/bin/redis-server /usr/local/redis/etc/redis.conf --supervised systemd
ExecStop=/usr/local/redis/bin/redis-shutdown
Type=simple
User=redis
Group=redis
RuntimeDirectory=redis
RuntimeDirectoryMode=0755
LimitNOFILE=10240

[Install]
WantedBy=multi-user.target
EOF

systemctl daemon-reload
systemctl start redis
systemctl enable redis
systemctl status redis
```


---

> 作者: [komorebi](https://blog.2nfree.com/)  
> URL: https://blog.2nfree.com/posts/install-redis/  

