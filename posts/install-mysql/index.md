# Mysql 安装


# 准备工作

## 初始化环境

```bash
#创建用户和组
groupadd mysql &&  useradd -g mysql mysql -s /sbin/nologin -M

#初始化数据存放目录
mkdir -pv /data/mysql && chown -R mysql:mysql /data/mysql
```

# 安装

## 下载 Mysql

```bash
mkdir /data/packages
cd /data/packages
wget https://downloads.mysql.com/archives/get/p/23/file/mysql-5.7.33-linux-glibc2.12-x86_64.tar.gz

```

## 解压安装

```bash
tar -zxvf mysql-5.7.33-linux-glibc2.12-x86_64.tar.gz
mv mysql-5.7.33-linux-glibc2.12-x86_64  /usr/local/mysql
chown -R mysql:mysql  /usr/local/mysql
ln -svf /usr/local/mysql/bin/mysql /usr/bin/mysql
ln -svf /usr/local/mysql/bin/mysqldump /usr/bin/mysqldump
```

## 修改配置

```bash
cat > /etc/my.cnf <<EOF
[mysql]
port=3306
socket=/tmp/mysql.sock
[mysqld]
bind-address=0.0.0.0
port=3306
user=mysql
basedir=/usr/local/mysql
datadir=/data/mysql
socket=/tmp/mysql.sock
back_log=500
init_connect='SET NAMES utf8'
default-storage-engine=INNODB
character-set-server=utf8
collation-server=utf8_general_ci
innodb_file_format=Barracuda
innodb_io_capacity=200
innodb_io_capacity_max=4000
innodb_open_files=500
key_buffer_size=260000000
lock_wait_timeout=3600
log_bin_trust_function_creators=1
log_output=FILE
long_query_time=5
max_allowed_packet=1073741824
max_heap_table_size=2147483648
query_cache_limit=10000000
query_cache_min_res_unit=2048
query_cache_size=300000000
query_cache_type=1
read_buffer_size=4194304
read_rnd_buffer_size=16777216
skip_name_resolve=0
slave_net_timeout=10
slow_query_log=1
sort_buffer_size=16777216
table_definition_cache=912
table_open_cache=1024
tmp_table_size=2147483648
wait_timeout=3600
EOF
```

## 初始化

```bash
yum install -y libaio
yum install -y perl-Data-Dumper
cd /usr/local/mysql/bin/
./mysql_install_db --user mysql --basedir=/usr/local/mysql --datadir=/data/mysql
rm -rf /data/mysql/ && ./mysqld --initialize --user=mysql  --basedir=/usr/local/mysql --datadir=/data/mysql

#A temporary password is generated for root@localhost: Mn?f4R_YDfBt
#进入数据库之后需要修改初始密码才能使用
```

# 添加系统服务

```bash
#添加系统服务脚本
cp /usr/local/mysql/support-files/mysql.server /etc/init.d/mysqld
chmod +x /etc/init.d/mysqld

#启动MySQL服务
systemctl daemon-reload
systemctl start mysqld 
systemctl status mysqld 
systemctl enable mysqld
```


---

> 作者: [komorebi](https://blog.2nfree.com/)  
> URL: https://blog.2nfree.com/posts/install-mysql/  

