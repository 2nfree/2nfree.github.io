# Deploy Doris


# Foreword

Apache Doris is an MPP-based real-time data warehouse known for its high query speed. For queries on large datasets, it returns results in sub-seconds. It supports both high-concurrent point queries and high-throughput complex analysis. It can be used for report analysis, ad-hoc queries, unified data warehouse, and data lake query acceleration. Based on Apache Doris, users can build applications for user behavior analysis, A/B testing platform, log analysis, user profile analysis, and e-commerce order analysis.

## Server Info

| **Host** | **IP** | **OS** |
| --- | --- | --- |
| doris_fe | 192.168.1.10 | centos7 |
| doris_be1 | 192.168.1.20 | centos7 |
| doris_be2 | 192.168.1.21 | centos7 |
| doris_be3 | 192.168.1.22 | centos7 |

---

| **Instance** | **Port** | **Default Port** | **Communication Direction** | **Description** |
| --- | --- | --- | --- | --- |
| BE | be_port | 9060 | FE --> BE | thrift server port on BE, receiving requests from FE |
| BE | webserver_port | 8040 | BE <--> BE | http server port on BE |
| BE | heartbeat_service_port | 9050 | FE --> BE | heartbeat service port (thrift) on BE, receiving heartbeats from FE |
| BE | brpc_port | 8060 | FE <--> BE, BE <--> BE | brpc port on BE, used for communication between BEs |
| FE | http_port | 8030 | FE <--> FE, user <--> FE | http server port on FE |
| FE | rpc_port | 9020 | BE --> FE, FE <--> FE | thrift server port on FE, configuration of each FE should be consistent |
| FE | query_port | 9030 | user <--> FE | MySQL server port on FE |
| FE | edit_log_port | 9010 | FE <--> FE | port on FE for bdbje communication |

## Prepare

### Set Max Open Files (on all servers)

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

### Modify file descriptors (on all servers)

```bash
echo fs.file-max = 6553560  >> /etc/sysctl.d/doris.conf
sysctl -p /etc/sysctl.d/doris.conf

# uncomment and modify system DefaultLimitNOFILE
sed -i 's/^#DefaultLimitNOFILE=.*$/DefaultLimitNOFILE=204800/' /etc/systemd/system.conf
# uncomment and modify system DefaultLimitNPROC
sed -i 's/^#DefaultLimitNPROC=.*$/DefaultLimitNPROC=204800/' /etc/systemd/system.conf

# uncomment and modify user DefaultLimitNOFILE
sed -i 's/^#DefaultLimitNOFILE=.*$/DefaultLimitNOFILE=204800/' /etc/systemd/user.conf
# uncomment and modify user DefaultLimitNPROC
sed -i 's/^#DefaultLimitNPROC=.*$/DefaultLimitNPROC=204800/' /etc/systemd/user.conf
```

### Disable Wwap (on all servers)

```bash
swapoff -a
sed -ri 's/.*swap.*/#&/' /etc/fstab
```

### Install Java (on FE)

{{< tabs >}}
{{% tab title="openjdk" %}}
```bash
yum -y install java-1.8.0-openjdk java-1.8.0-openjdk-devel
```
{{% /tab %}}
{{% tab title="oracle jdk" %}}

You need to download the installation package from the oracle jdk official website and upload it to the server

```bash
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
{{% /tab %}}
{{< /tabs >}}

### Install Mysql Client (on FE)

```bash
yum remove mysql-libs -y

wget https://dev.mysql.com/get/Downloads/MySQL-5.7/mysql-community-common-5.7.39-1.el7.x86_64.rpm
wget https://dev.mysql.com/get/Downloads/MySQL-5.7/mysql-community-libs-5.7.39-1.el7.x86_64.rpm
wget https://dev.mysql.com/get/Downloads/MySQL-5.7/mysql-community-client-5.7.39-1.el7.x86_64.rpm

rpm -ivh mysql-community-common-5.7.39-1.el7.x86_64.rpm
rpm -ivh mysql-community-libs-5.7.39-1.el7.x86_64.rpm
rpm -ivh mysql-community-client-5.7.39-1.el7.x86_64.rpm
```

### Install ODBC (on all servers)

```bash
yum install unixODBC -y
wget https://dev.mysql.com/get/Downloads/Connector-ODBC/8.0/mysql-connector-odbc-8.0.30-1.el7.x86_64.rpm
rpm -ivh mysql-connector-odbc-8.0.30-1.el7.x86_64.rpm

# Check
ls /usr/lib64/libmyodbc8w.so
```

### Edit Hosts (on FE)

```bash
192.168.1.10 doris.example.com
```

# Deploy Doris

## Download

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

## Edit Start Scripts

Find the following part of the script (FE)

```bash
if [ ${RUN_DAEMON} -eq 1 ]; then
    nohup $LIMIT $JAVA $final_java_opt -XX:OnOutOfMemoryError="kill -9 %p" org.apache.doris.PaloFe ${HELPER} "$@" >> $LOG_DIR/fe.out 2>&1 < /dev/null &
else
    export DORIS_LOG_TO_STDERR=1
    $LIMIT $JAVA $final_java_opt -XX:OnOutOfMemoryError="kill -9 %p" org.apache.doris.PaloFe ${HELPER} "$@" < /dev/null
fi
```

Remove the final `&`

```bash
if [ ${RUN_DAEMON} -eq 1 ]; then
    nohup $LIMIT $JAVA $final_java_opt -XX:OnOutOfMemoryError="kill -9 %p" org.apache.doris.PaloFe ${HELPER} "$@" >> $LOG_DIR/fe.out 2>&1 < /dev/null
else
    export DORIS_LOG_TO_STDERR=1
    $LIMIT $JAVA $final_java_opt -XX:OnOutOfMemoryError="kill -9 %p" org.apache.doris.PaloFe ${HELPER} "$@" < /dev/null
fi
```

Find the following part of the script (BE)

```bash
if [ ${RUN_DAEMON} -eq 1 ]; then
    nohup $LIMIT ${DORIS_HOME}/lib/doris_be "$@" >> $LOG_DIR/be.out 2>&1 < /dev/null &
else
    export DORIS_LOG_TO_STDERR=1
    $LIMIT ${DORIS_HOME}/lib/doris_be "$@" 2>&1 < /dev/null
fi
```

Remove the final `&`

```bash
if [ ${RUN_DAEMON} -eq 1 ]; then
    nohup $LIMIT ${DORIS_HOME}/lib/doris_be "$@" >> $LOG_DIR/be.out 2>&1 < /dev/null
else
    export DORIS_LOG_TO_STDERR=1
    $LIMIT ${DORIS_HOME}/lib/doris_be "$@" 2>&1 < /dev/null
fi
```

## Install Supervisor

```bash
yum -y install supervisor

systemctl start supervisord
systemctl status supervisord
systemctl enable supervisord

# Check if Supervisor's file descriptors do not meet Doris' requirements.
cat /proc/$(ps -aux | grep supervisor | grep -v grep | awk '{print $2}')/limits
```

## Modify Service File

```bash
vim /usr/lib/systemd/system/supervisord.service
```

Add the following configuration under the `[Service]` section.

```bash
LimitCORE=infinity
LimitNOFILE=204800
LimitNPROC=204800
```

Like:

```ini
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

Restart supervisor and check

```bash
systemctl daemon-reload
systemctl restart supervisord
systemctl status supervisord
cat /proc/$(ps -aux | grep supervisor | grep -v grep | awk '{print $2}')/limits
```

## Edit Doris Config

### FE

Configure the IP CIDR based on the network interface IP.

```properties
priority_networks = 192.168.1.0/24
```

### BE

```properties
storage_root_path = ${DORIS_HOME}/storage

# Whether to log stream load
enable_stream_load_record=true
# Whether to disable using page cache for index caching
disable_storage_page_cache=true
# Interval for file handle cache cleanup
cache_clean_interval=180
# Memory limit for all import threads on a single node
load_process_max_memory_limit_bytes=5368709120
# Stream load request limit
max_running_txn_num_per_db=10000
```

### Doris Supervisor Config

```bash
cat > /etc/supervisord.d/DorisFE.ini <<"EOF"
[program:DorisFE]
process_name=DorisFE
directory=/data/doris/fe
command=sh /data/doris/fe/bin/start_fe.sh
autostart=true
autorestart=true
user=root
numprocs=1
startretries=3
# Whether to stop subprocesses (must be set)
stopasgroup=true
# Whether to kill subprocesses (must be set)
killasgroup=true
startsecs=5
redirect_stderr = true
stdout_logfile_maxbytes = 20MB
stdout_logfile_backups = 10
stdout_logfile=/data/logs/doris/DorisFE.log
EOF
```

```bash
cat > /etc/supervisord.d/DorisBE.ini <<"EOF"
[program:DorisBE]
process_name=DorisBE
directory=/data/doris/be
command=sh /data/doris/be/bin/start_be.sh
autostart=true
autorestart=true
user=root
numprocs=1
startretries=3
# Whether to stop subprocesses (must be set)
stopasgroup=true
# Whether to kill subprocesses (must be set)
killasgroup=true
startsecs=5
redirect_stderr = true
stdout_logfile_maxbytes = 20MB
stdout_logfile_backups = 10
stdout_logfile=/data/logs/doris/DorisBE.log
EOF
```

## Start Doris Cluster

```bash
supervisorctl reread
supervisorctl update
supervisorctl start all
supervisorctl status
```

# Configure Doris Cluster

```bash
mysql -hdorisdb.example.com -uroot -P9030
```

```sql
-- ADD BE
ALTER SYSTEM ADD BACKEND "192.168.1.20:9050";
ALTER SYSTEM ADD BACKEND "192.168.1.21:9050";
ALTER SYSTEM ADD BACKEND "192.168.1.22:9050";

-- edit default password
SET PASSWORD FOR 'username' = PASSWORD('password');
```

After the installation is complete, if Doris needs to be accessed from the public internet, the DNS should be resolved to the internal IP address of the Doris **FE**. If the DNS resolves to the public IP address of the **FE**, it will not be accessible.


---

> Author: [komorebi](https://blog.2nfree.com/)  
> URL: https://blog.2nfree.com/en/posts/deploy-doris/  

