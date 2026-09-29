# Mysql Installation


# Prepare

## Initialization

```bash
# Creating Users and Groups
groupadd mysql &&  useradd -g mysql mysql -s /sbin/nologin -M

# Initialize the data storage directory
mkdir -pv /data/mysql && chown -R mysql:mysql /data/mysql
```

# Install

## Download Mysql

```bash
mkdir /data/packages
cd /data/packages
wget https://downloads.mysql.com/archives/get/p/23/file/mysql-5.7.33-linux-glibc2.12-x86_64.tar.gz

```

## Unzip

```bash
tar -zxvf mysql-5.7.33-linux-glibc2.12-x86_64.tar.gz
mv mysql-5.7.33-linux-glibc2.12-x86_64  /usr/local/mysql
chown -R mysql:mysql  /usr/local/mysql
ln -svf /usr/local/mysql/bin/mysql /usr/bin/mysql
ln -svf /usr/local/mysql/bin/mysqldump /usr/bin/mysqldump
```

## Config

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

## Initialization DB

```bash
yum install -y libaio
yum install -y perl-Data-Dumper
cd /usr/local/mysql/bin/
./mysql_install_db --user mysql --basedir=/usr/local/mysql --datadir=/data/mysql
rm -rf /data/mysql/ && ./mysqld --initialize --user=mysql  --basedir=/usr/local/mysql --datadir=/data/mysql

# A temporary password is generated for root@localhost: Mn?f4R_YDfBt (There will be randomized passwords here)
# After entering the database you need to change the initial password to use it
```

# Adding system service

```bash
# create service scripts
cp /usr/local/mysql/support-files/mysql.server /etc/init.d/mysqld
chmod +x /etc/init.d/mysqld

systemctl daemon-reload
systemctl start mysqld 
systemctl status mysqld 
systemctl enable mysqld
```


---

> Author: [komorebi](https://blog.2nfree.com/)  
> URL: https://blog.2nfree.com/en/posts/install-mysql/  

