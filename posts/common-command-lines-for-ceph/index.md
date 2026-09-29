# Ceph 常用命令


# Cluster

```bash
# 显示集群的状态
ceph status 或 ceph -s

# 显示集群健康状况
ceph health

# 查看集群磁盘空间的使用情况
ceph df

# 显示集群中正在运行的 Ceph 版本
ceph version

# 列出所有认证密钥
ceph auth list

# 用于部署一个或多个集群服务实例，例如 MGR、MON、OSD、RGW 等
ceph orch apply <service_type> [options]

# 显示集群中当前运行的所有服务及其状态
ceph orch ps

# 列出集群中配置的所有服务
ceph orch ls

# 添加一个或多个 OSD 守护进程
ceph orch daemon add osd <host>:<path> [--data-devices <devices>]

# 移除一个 OSD 守护进程
ceph orch daemon rm osd <osd_id>

# 列出集群中每个主机上的所有设备及其状态
ceph orch device ls

#  向集群中添加特定类型的守护进程
ceph orch daemon add <daemon_type> <host> [options]

# 从集群中删除特定类型的守护进程
ceph orch daemon rm <daemon_type> <host> [options]

# 重启特定类型的守护进程
ceph orch daemon restart <daemon_type>

# 重新配置特定类型的守护进程
ceph orch daemon reconfig <daemon_type>

# 使用指定的容器镜像开始升级集群
ceph orch upgrade start --image <container_image>

# 停止正在进行的集群升级
ceph orch upgrade stop

# 获取当前集群升级的状态信息
ceph orch upgrade status

# 检查 Orchestrator 的状态和当前在用的后端
ceph orch status

# 列出集群中已知的主机
ceph orch host ls

# 将新主机添加到集群
ceph orch host add <hostname> <ip>

# 从集群中删除一个主机
ceph orch host rm <hostname>
```

# OSD

```bash
# 查看每个 OSD 的上线/下线状态
ceph osd status

# 查看集群 OSD 映射和状态
ceph osd tree

# 将指定的 OSD 设置为 out 状态，启动数据迁移
ceph osd out <osd.?>

# 将指定的 OSD 设置为 in 状态
ceph osd in <osd.?>

# 移除指定的 OSD
ceph osd rm <osd.?>

# 重新设置 OSD 的权重
ceph osd reweight <osd.?> <weight>

# 设置 OSD 的特定标志，例如 noout, noscrub 等
ceph osd set <flag>

# 列出所有存储池
ceph osd lspools

# 创建一个新的存储池
ceph osd pool create <pool_name> <pg_num> <pgp_num>

# 删除存储池
ceph osd pool delete <pool_name> <pool_name> --yes-i-really-really-mean-it

# 设置存储池的参数
ceph osd pool set <pool_name> <key> <value>
```

# MON

```bash
# 显示监控守护进程状态
ceph mon stat

# 添加新的监控守护进程
ceph mon add <name> <ip:port>

# 移除监控守护进程
ceph mon rm <name>
```

# MDS

```bash
# 显示元数据服务器状态
ceph mds stat

# 列出所有 Ceph 文件系统
ceph fs ls

# 创建新的 Ceph 文件系统
ceph fs new <fs_name> <metadata_pool> <data_pool>
```

# Crash

```bash
# 查看最近组件崩溃的历史
ceph crash ls-new

# 查看所有组件崩溃的历史
ceph crash ls

# 查看对应崩溃的详情
ceph crash info <id>

# 归档对应的崩溃
ceph crash archive <id>

# 归档所有
ceph crash archive-all
```


---

> 作者: [komorebi](https://blog.2nfree.com/)  
> URL: https://blog.2nfree.com/posts/common-command-lines-for-ceph/  

