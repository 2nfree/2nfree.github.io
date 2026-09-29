# Doris安装部署


# 前言

Apache Doris 是一个基于 MPP 架构的高性能、实时的分析型数据库，以极速易用的特点被人们所熟知，仅需亚秒级响应时间即可返回海量数据下的查询结果，不仅可以支持高并发的点查询场景，也能支持高吞吐的复杂分析场景。基于此，Apache Doris 能够较好的满足报表分析、即席查询、统一数仓构建、数据湖联邦查询加速等使用场景，用户可以在此之上构建用户行为分析、AB 实验平台、日志检索分析、用户画像分析、订单分析等应用。

## 基本信息

| **主机名** | **内网IP** | **操作系统** |
| --- | --- | --- |
| doris_fe | 192.168.1.10 | centos7 |
| doris_be1 | 192.168.1.20 | centos7 |
| doris_be2 | 192.168.1.21 | centos7 |
| doris_be3 | 192.168.1.22 | centos7 |

<br>

| **实例名称** | **端口名称** | **默认端口** | **沟通方向** | **描述** |
| --- | --- | --- | --- | --- |
| BE | be_port | 9060 | FE --> BE | BE 用于接收来自 FE 的请求 |
| BE | webserver_port | 8040 | BE <--> BE | BE |
| BE | heartbeat_service_port | 9050 | FE --> BE | BE上的心跳服务端口（thrift），用于接收FE的心跳 |
| BE | brpc_port | 8060 | FE <--> BE, BE <--> BE | BE用于BE之间的通信 |
| FE | http_port | 8030 | FE <--> FE, user <--> FE | FE 上的 HTTP 服务器端口 |
| FE | rpc_port | 9020 | BE --> FE, FE <--> FE | FE上的thrift server端口，每个fe的配置需要一致 |
| FE | query_port | 9030 | user <--> FE | FE |
| FE | edit_log_port | 9010 | FE <--> FE | FE |

## 准备工作

### 设置系统最大打开文件句柄数（all）

```bash
cat > /etc/security/limits.conf <<EOF
* soft noproc 204800
* hard noproc 204800
* soft nofile 204800
* hard nofile 204800
EOF

echo > /etc/security/limits.d/20-nproc.conf 

ulimit -n 204800
ulimit -u 204800
```

### 修改文件描述符（all）

```bash
echo fs.file-max = 6553560  >> /etc/sysctl.d/doris.conf
sysctl -p /etc/sysctl.d/doris.conf

vim /etc/systemd/system.conf
#修改如下内容
DefaultLimitNOFILE=204800
DefaultLimitNPROC=204800

vim /etc/systemd/user.conf
#修改如下内容
DefaultLimitNOFILE=204800
DefaultLimitNPROC=204800
```

### 关闭交换分区（all）

```bash
#关闭交换分区
swapoff -a
sed -ri 's/.*swap.*/#&/' /etc/fstab
```

### 安装java（doris_fe）

```bash
#openjdk
yum -y install java-1.8.0-openjdk java-1.8.0-openjdk-devel

#oracle jdk
#解压jdk后设置环境变量
tar -zxvf jdk-8u321-linux-x64.tar.gz
mv jdk1.8.0_321 /usr/local/java8

cat >> /etc/profile <<EOF
#java env
export JAVA_HOME=/usr/local/java8
export JRE_HOME=\$JAVA_HOME/jre
export CLASSPATH=.:\$JAVA_HOME/lib:\$JRE_HOME/lib
export PATH=\$JAVA_HOME/bin:\$PATH
EOF

source /etc/profile
```

### 安装mysql客户端（doris_fe）

```bash
yum remove mysql-libs -y

wget https://dev.mysql.com/get/Downloads/MySQL-5.7/mysql-community-common-5.7.39-1.el7.x86_64.rpm
wget https://dev.mysql.com/get/Downloads/MySQL-5.7/mysql-community-libs-5.7.39-1.el7.x86_64.rpm
wget https://dev.mysql.com/get/Downloads/MySQL-5.7/mysql-community-client-5.7.39-1.el7.x86_64.rpm

rpm -ivh mysql-community-common-5.7.39-1.el7.x86_64.rpm
rpm -ivh mysql-community-libs-5.7.39-1.el7.x86_64.rpm
rpm -ivh mysql-community-client-5.7.39-1.el7.x86_64.rpm
```

### 安装odbc依赖（all）

```bash
yum install unixODBC -y
wget https://dev.mysql.com/get/Downloads/Connector-ODBC/8.0/mysql-connector-odbc-8.0.30-1.el7.x86_64.rpm
rpm -ivh mysql-connector-odbc-8.0.30-1.el7.x86_64.rpm

#检查
ls /usr/lib64/libmyodbc8w.so
```

### 域名转发（doris_fe）

```bash
#修改/etc/hosts文件
192.168.1.10 doris.example.com
```

# 下载安装doris

## 下载doris

```bash
mkdir -pv /data/doris
mkdir -pv /data/logs/doris

#fe
wget https://dist.apache.org/repos/dist/release/doris/1.1/1.1.2-rc05/apache-doris-fe-1.1.2-bin.tar.gz
tar -zxvf apache-doris-fe-1.1.2-bin.tar.gz
cd apache-doris-fe-1.1.2-bin
cp -a fe /data/doris/fe

#be
wget https://dist.apache.org/repos/dist/release/doris/1.1/1.1.2-rc05/apache-doris-be-1.1.2-bin-x86_64.tar.gz
tar -zxvf apache-doris-be-1.1.2-bin-x86_64.tar.gz
cd apache-doris-be-1.1.2-bin-x86_64
cp -a be /data/doris/be
```

## 修改doris启动脚本

```bash
#fe
#将启动脚本中的
if [ ${RUN_DAEMON} -eq 1 ]; then
    nohup $LIMIT $JAVA $final_java_opt -XX:OnOutOfMemoryError="kill -9 %p" org.apache.doris.PaloFe ${HELPER} "$@" >> $LOG_DIR/fe.out 2>&1 < /dev/null &
else
    export DORIS_LOG_TO_STDERR=1
    $LIMIT $JAVA $final_java_opt -XX:OnOutOfMemoryError="kill -9 %p" org.apache.doris.PaloFe ${HELPER} "$@" < /dev/null
fi
#改为（去掉最后的 & 符号）
if [ ${RUN_DAEMON} -eq 1 ]; then
    nohup $LIMIT $JAVA $final_java_opt -XX:OnOutOfMemoryError="kill -9 %p" org.apache.doris.PaloFe ${HELPER} "$@" >> $LOG_DIR/fe.out 2>&1 < /dev/null
else
    export DORIS_LOG_TO_STDERR=1
    $LIMIT $JAVA $final_java_opt -XX:OnOutOfMemoryError="kill -9 %p" org.apache.doris.PaloFe ${HELPER} "$@" < /dev/null
fi

#be
#将启动脚本中的
if [ ${RUN_DAEMON} -eq 1 ]; then
    nohup $LIMIT ${DORIS_HOME}/lib/doris_be "$@" >> $LOG_DIR/be.out 2>&1 < /dev/null &
else
    export DORIS_LOG_TO_STDERR=1
    $LIMIT ${DORIS_HOME}/lib/doris_be "$@" 2>&1 < /dev/null
fi
#改为（去掉最后的 & 符号）
if [ ${RUN_DAEMON} -eq 1 ]; then
    nohup $LIMIT ${DORIS_HOME}/lib/doris_be "$@" >> $LOG_DIR/be.out 2>&1 < /dev/null
else
    export DORIS_LOG_TO_STDERR=1
    $LIMIT ${DORIS_HOME}/lib/doris_be "$@" 2>&1 < /dev/null
fi
```

## 安装supervisor

```bash
yum -y install supervisor

systemctl start supervisord
systemctl status supervisord
systemctl enable supervisord

#查看supervisor的文件描述符 不符合doris要求
cat /proc/$(ps -aux | grep supervisor | grep -v grep | awk '{print $2}')/limits
```

## 修改supervisor服务文件

```bash
#在服务文件的[Service]下添加
vim /usr/lib/systemd/system/supervisord.service
LimitCORE=infinity
LimitNOFILE=204800
LimitNPROC=204800
```

```properties
[Unit]
Description=Process Monitoring and Control Daemon
After=rc-local.service nss-user-lookup.target

[Service]
LimitCORE=infinity
LimitNOFILE=204800
LimitNPROC=204800
Type=forking
ExecStart=/usr/bin/supervisord -c /etc/supervisord.conf

[Install]
WantedBy=multi-user.target
```

```bash
#重启supervisor并查看文件描述符
systemctl daemon-reload
systemctl restart supervisord
systemctl status supervisord
cat /proc/$(ps -aux | grep supervisor | grep -v grep | awk '{print $2}')/limits
```

## 修改配置文件

### fe:修改对应值

```properties
priority_networks = 192.168.1.0/24
```

### be:修改对应值

```properties
storage_root_path = ${DORIS_HOME}/storage

# 是否记录stream load
enable_stream_load_record=true
# 是否进行使用page cache进行index的缓存
disable_storage_page_cache=true
# 文件句柄缓存清理的间隔
cache_clean_interval=180
# 单节点上所有的导入线程占据的内存上限
load_process_max_memory_limit_bytes=5368709120
```

### supervisor配置文件

```bash
cat > /etc/supervisord.d/DorisFE.ini <<"EOF"
[program:DorisFE]
#进程名称
process_name=DorisFE
#工作目录
directory=/data/doris/fe
#启动命令
command=sh /data/doris/fe/bin/start_fe.sh
#自启动
autostart=true
#自动重启
autorestart=true
#运行用户
user=root
#进程数1
numprocs=1
#启动重试次数
startretries=3
#是否停止子进程（必须设置）
stopasgroup=true
#是否杀死子进程（必须设置）
killasgroup=true
#启动5秒后，如果还是运行状态才认为进程已经启动
startsecs=5
#日志
redirect_stderr = true
stdout_logfile_maxbytes = 20MB
stdout_logfile_backups = 10
stdout_logfile=/data/logs/doris/DorisFE.log
EOF
```

```bash
cat > /etc/supervisord.d/DorisBE.ini <<"EOF"
[program:DorisBE]
#进程名称
process_name=DorisBE
#工作目录
directory=/data/doris/be
#启动命令
command=sh /data/doris/be/bin/start_be.sh
#自启动
autostart=true
#自动重启
autorestart=true
#运行用户
user=root
#进程数1
numprocs=1
#启动重试次数
startretries=3
#是否停止子进程（必须设置）
stopasgroup=true
#是否杀死子进程（必须设置）
killasgroup=true
#启动5秒后，如果还是运行状态才认为进程已经启动
startsecs=5
#日志
redirect_stderr = true
stdout_logfile_maxbytes = 20MB
stdout_logfile_backups = 10
stdout_logfile=/data/logs/doris/DorisBE.log
EOF
```

## 启动doris

```bash
supervisorctl reread
supervisorctl update
supervisorctl start all
supervisorctl status
```

# 配置doris服务

```bash
mysql -hdorisdb.example.com -uroot -P9030
```

```sql
#配置be地址
ALTER SYSTEM ADD BACKEND "192.168.1.20:9050";
ALTER SYSTEM ADD BACKEND "192.168.1.21:9050";
ALTER SYSTEM ADD BACKEND "192.168.1.22:9050";

#修改账号密码
SET PASSWORD FOR '用户' = PASSWORD('密码');

# 设置stream load请求数量限制
ADMIN SET FRONTEND CONFIG ("max_running_txn_num_per_db" = "10000");
```

安装完毕，后续如果doris需要开放公网访问，需要吧DNS解析到doris fe的内网地址，如果解析到fe的公网地址则无法访问


---

> 作者: [komorebi](https://blog.2nfree.com/)  
> URL: https://blog.2nfree.com/posts/deploy-doris/  

