# Kubernetes 接入基于 Cephadm 部署的 Ceph 集群


# 前置条件

## 准备工作

1. 部署完毕的 Kubernetes 集群
2. 在 Ceph 集群节点上安装：docker，cephadm，并且更新 cephadm 到最新版本，以及配置 Ceph 节点之间的 ssh 互信

```bash
# 生成公钥并导入到注册到每台服务器
ssh-keygen
cat ~/.ssh/id_rsa.pub
vim ~/.ssh/authorized_keys

# 互相连接测试
ssh root@10.0.16.33
ssh root@10.0.16.34
ssh root@10.0.16.35

# 添加 /etc/hosts
10.0.16.33 ceph1
10.0.16.34 ceph2
10.0.16.35 ceph3

# docker 安装
apt-get update
apt-get install -y \
    apt-transport-https \
    ca-certificates \
    curl \
    gnupg-agent \
    software-properties-common
curl -fsSL https://mirrors.ustc.edu.cn/docker-ce/linux/ubuntu/gpg | sudo apt-key add -
add-apt-repository -y \
   "deb [arch=amd64] https://mirrors.ustc.edu.cn/docker-ce/linux/ubuntu/ \
  $(lsb_release -cs) \
  stable"
apt-get update
apt-get install -y docker-ce docker-ce-cli containerd.io

# cephadm 安装
apt-get install -y cephadm
cephadm add-repo --release reef
cephadm install
cephadm version
```

{{< admonition type="danger" open=true >}}
**最好不要修改 ceph 集群服务器的 ssh 默认端口，否则会导致 ceph 集群引导失败，如果修改了 ssh 的默认端口，则需要创建对应的 ssh 配置**
如：

```bash
mkdir -pv /etc/ceph/
cat > /etc/ceph/ceph.ssh_config <<EOF
Host *
  Port 2222
  StrictHostKeyChecking no
EOF
```

并在启动引导指定对应的 config

```bash
cephadm bootstrap --ssh-config /etc/ceph/ceph.ssh_config ......
```

{{< /admonition >}}

## 集群信息

| **主机名** | **IP**     | **系统版本**   | **备注**                  |
| ---------- | ---------- | -------------- | ------------------------- |
| ceph1      | 10.0.16.33 | ubuntu 22.04.3 | Ceph 集群管理节点+osd节点 |
| ceph2      | 10.0.16.34 | ubuntu 22.04.3 | Ceph 集群节点+osd节点     |
| ceph3      | 10.0.16.35 | ubuntu 22.04.3 | Ceph 集群节点+osd节点     |
| master1    | 10.0.16.36 | ubuntu 22.04.3 | K8S 控制平面              |
| node1      | 10.0.16.37 | ubuntu 22.04.3 | K8S 工作节点              |
| node2      | 10.0.16.38 | ubuntu 22.04.3 | K8S 工作节点              |
| node3      | 10.0.16.39 | ubuntu 22.04.3 | K8S 工作节点              |

# Ceph 集群部署

## 在节点 ceph1 初始化整个集群

```bash
cephadm bootstrap --log-to-file --mon-ip 10.0.16.33 --ssh-private-key /root/.ssh/id_rsa --ssh-public-key /root/.ssh/id_rsa.pub
```

{{< admonition type="info" open=true >}}

# 引导完毕后会出现默认配置，可以登录dashboard修改密码

Ceph Dashboard is now available at:

  URL: <https://ceph1:8443/>
  User: admin
  Password: 8ega86tepk

Enabling client.admin keyring and conf on hosts with "admin" label
Saving cluster configuration to /var/lib/ceph/80285dea-af83-11ee-83b8-23d931c2e040/config directory
Enabling autotune for osd_memory_target
You can access the Ceph CLI as following in case of multi-cluster or non-default config:

  sudo /usr/sbin/cephadm shell --fsid 80285dea-af83-11ee-83b8-23d931c2e040 -c /etc/ceph/ceph.conf -k /etc/ceph/ceph.client.admin.keyring

Or, if you are only running a single cluster on this host:

  sudo /usr/sbin/cephadm shell

Please consider enabling telemetry to help improve Ceph:

  ceph telemetry on

For more information see:

  <https://docs.ceph.com/en/latest/mgr/telemetry/>

Bootstrap complete.
{{< /admonition >}}

## 在节点一上安装 ceph 相关管理工具

```bash
cephadm install ceph-volume ceph-common ceph-osd ceph-mon
```

## 添加其他节点

```bash
ceph orch host add ceph2 10.0.16.34
ceph orch host add ceph3 10.0.16.35
ceph orch apply mon --placement="ceph1,ceph2,ceph3"

# 查看集群 host 列表
ceph orch host ls
```

## 添加磁盘

```bash
ceph orch daemon add osd ceph1:/dev/sdb
ceph orch daemon add osd ceph1:/dev/sdc
ceph orch daemon add osd ceph2:/dev/sdb
ceph orch daemon add osd ceph2:/dev/sdc
ceph orch daemon add osd ceph3:/dev/sdb
ceph orch daemon add osd ceph3:/dev/sdc
```

## 创建 CephFS

```bash
# 创建文件系统
ceph fs volume create cephfs

# 在每个节点都配置 mds
ceph orch apply mds cephfs --placement="ceph1,ceph2,ceph3"
```

## 创建 RBD 存储池

```bash
# 如果是直接给 Kubernetes 用只需要创建存储池即可
ceph osd pool create rbd.data 64 64 replicated
rbd pool init rbd.data
```

# 性能测试

## 测试命令

每次测试时间为三分钟

```bash
# 测试写入性能且不清理测试数据，后续用于测试读取性能
rados bench -p rbd.data 180 write --no-cleanup

# 顺序读
rados bench -p rbd.data 180 seq

#随机读
rados bench -p rbd.data 180 rand
```

## 写入性能

  Total time run:         180.162     # 总体运行时间 (Total time run): 180.162秒，表示从测试开始到结束的总时间。
  Total writes made:      9728        # 总写入次数 (Total writes made): 9728次，表示在整个测试期间执行的总写入操作次数。
  Write size:             4194304     # 写入大小 (Write size): 4194304字节，每个写入操作的数据块大小，等于4MB。
  Object size:            4194304     # 对象大小 (Object size): 4194304字节，每个存储对象的大小，等于4MB。
  Bandwidth (MB/sec):     215.984     # 平均带宽 (Average Bandwidth): 215.984 MB/秒，表示平均每秒写入数据的量。
  Stddev Bandwidth:       34.9424     # 带宽标准差 (Stddev Bandwidth): 34.9424 MB/秒，表示带宽测量值的波动程度。这个数字较大，表明带宽数据在测试期间波动了不少。
  Max bandwidth (MB/sec): 400         # 最大带宽 (Max Bandwidth): 400 MB/秒，测试期间的最高带宽记录。
  Min bandwidth (MB/sec): 140         # 最小带宽 (Min Bandwidth): 140 MB/秒，测试期间的最低带宽记录。
  Average IOPS:           53          # 平均IOPS (Average IOPS): 53，基于4MB写入操作的对象，平均每秒完成53个IOPS。
  Stddev IOPS:            8.73561     # IOPS标准差 (Stddev IOPS): 8.73561，IOPS数值的波动程度。
  Max IOPS:               100         # 最大IOPS (Max IOPS): 100，测试期间的最高IOPS记录。
  Min IOPS:               35          # 最小IOPS (Min IOPS): 35，测试期间的最低IOPS记录。
  Average Latency(s):     0.296262    # 平均延迟 (Average Latency): 0.296262秒，一次写入操作的平均完成时间。
  Stddev Latency(s):      0.211683    # 延迟标准差 (Stddev Latency): 0.211683秒，延迟测量值的波动程度。较低的标准差通常意味着延迟比较一致。
  Max latency(s):         0.83413     # 最大延迟 (Max Latency): 0.83413秒，测试期间的最高延迟。
  Min latency(s):         0.0194977   # 最小延迟 (Min Latency): 0.0194977秒，测试期间的最低延迟。

## 顺序读取性能

这里因为读取速度较快，60秒就完成了读取

  Total time run:       60.2536
  Total reads made:     9728
  Read size:            4194304
  Object size:          4194304
  Bandwidth (MB/sec):   645.804
  Average IOPS:         161
  Stddev IOPS:          31.5648
  Max IOPS:             331
  Min IOPS:             112
  Average Latency(s):   0.0982576
  Max latency(s):       1.32703
  Min latency(s):       0.00750698

## 随机读取性能

  Total time run:       180.251
  Total reads made:     29233
  Read size:            4194304
  Object size:          4194304
  Bandwidth (MB/sec):   648.717
  Average IOPS:         162
  Stddev IOPS:          25.9462
  Max IOPS:             331
  Min IOPS:             104
  Average Latency(s):   0.0980925
  Max latency(s):       1.49632
  Min latency(s):       0.00163233

## 清理数据

```bash
rados -p rbd.data cleanup
```

# 使用 Rook 接入外部 Ceph

{{< admonition type="warning" open=true >}}
**K8S 接入外部 ceph 集群的最低要求为，外部 ceph 建立了 CephFS 以及 RBD 的相关存储池，需要存储池创建完毕并且集群健康状态处于 Healthy 的时候接入**
{{< /admonition >}}

## 导出当前 Ceph 的相关数据

{{< admonition type="tip" open=true >}}
Rook 官方提供了接入外部 Ceph 集群的相关脚本：<https://github.com/rook/rook/blob/master/deploy/examples/>
其中 ``create-external-cluster-resources.py`` 为导出当前 Ceph 集群相关配置数据的脚本
执行完脚本会出现导出数据，需要留存
{{< /admonition >}}

```bash
# 在 ceph1 节点运行
python3 create-external-cluster-resources.py --rbd-data-pool-name rbd.data --cephfs-filesystem-name cephfs --namespace rook-ceph-external --format bash
```

```bash
export NAMESPACE=rook-ceph-external
export ROOK_EXTERNAL_FSID=6b6d577c-b035-11ee-8b7f-e51965640d97
export ROOK_EXTERNAL_USERNAME=client.healthchecker
export ROOK_EXTERNAL_CEPH_MON_DATA=ceph1=10.0.16.33:6789
export ROOK_EXTERNAL_USER_SECRET=AQAUnp9loTImDxAAYZiXyoTfPBO8v96/K+hj4Q==
export ROOK_EXTERNAL_DASHBOARD_LINK=https://10.0.16.33:8443/
export CSI_RBD_NODE_SECRET=AQAUnp9lkPeFFhAAviD9KRUZPQK0mjJ3Cr0bpA==
export CSI_RBD_NODE_SECRET_NAME=csi-rbd-node
export CSI_RBD_PROVISIONER_SECRET=AQAUnp9lZqecGxAA2aGz/HlFX4pW1suQjJXmGA==
export CSI_RBD_PROVISIONER_SECRET_NAME=csi-rbd-provisioner
export CEPHFS_POOL_NAME=cephfs.cephfs.data
export CEPHFS_METADATA_POOL_NAME=cephfs.cephfs.meta
export CEPHFS_FS_NAME=cephfs
export CSI_CEPHFS_NODE_SECRET=AQAUnp9lbdkFIBAAq1d4PMi0GDJP4G7uQrcHnA==
export CSI_CEPHFS_PROVISIONER_SECRET=AQAUnp9le2PSIxAAcgvsxgZ+GPJXYkv+u8B4vQ==
export CSI_CEPHFS_NODE_SECRET_NAME=csi-cephfs-node
export CSI_CEPHFS_PROVISIONER_SECRET_NAME=csi-cephfs-provisioner
export MONITORING_ENDPOINT=10.0.16.33
export MONITORING_ENDPOINT_PORT=9283
export RBD_POOL_NAME=rbd.data
export RGW_POOL_PREFIX=default
```

## 执行外部集群数据导入脚本

{{< admonition type="tip" open=true >}}
<https://github.com/rook/rook/blob/master/deploy/examples/>
其中 ``import-external-cluster.sh`` 为导入当前 Ceph 集群相关配置数据到 Kubernetes 的脚本，注意使用与当前 Kubernetes 集群部署的相同版本的 Rook，需要切换 tag 下载对应版本的脚本，如果没有部署过 Rook，可以直接使用最新的
{{< /admonition >}}

当前 Kubernetes 集群使用的 Rook 版本为 1.10.2，故使用版本为：<https://github.com/rook/rook/blob/release-1.10/deploy/examples/import-external-cluster.sh>

下载完脚本后，需要按照图示将之前导出的数据写入脚本

{{< image src="script.webp" >}}

```bash
git clone --single-branch --branch release-1.10 https://github.com/rook/rook.git
cd rook
git checkout v1.10.2
cd deploy/examples/

# 如果之前没有安装过 Rook 则 apply 下面的配置文件先安装 Rook，如果已安装过 Rook，则不需要执行
kubectl apply -f common.yaml
kubectl apply -f crds.yaml
kubectl apply -f operator.yaml
```

### 导入数据

```bash
kubectl apply -f common-external.yaml

# 在这一步修改 import-external-cluster.sh，添加 ceph 集群导出数据
bash import-external-cluster.sh

# 等待所有服务启动
kubectl apply -f cluster-external.yaml

# 查看创建的存储类
kubectl get sc

# 查看集群状态
kubectl -n rook-ceph-external  get CephCluster

# 查看 csi 容器状态
kubectl get pod -n rook-ceph
```

{{< image src="cluster.png" >}}

### 创建 PVC 测试

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: rbd-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
  storageClassName: ceph-rbd
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: cephfs-pvc
spec:
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 1Gi
  storageClassName: cephfs
```

# 其他

## Prometheus

Cephadm 集群会自动启动一套 Prometheus 监控集群，相关配置如下（这里的端口不一定一致）：

| **服务**     | **地址**                 | **配置文件**                                                 |
| ------------ | ------------------------ | ------------------------------------------------------------ |
| Alertmanager | <http://10.0.16.33:9093/>  | /var/lib/ceph/【集群ID】/alertmanager.【所在host】/etc/alertmanager |
| Prometheus   | <http://10.0.16.33:9095/>  | /var/lib/ceph/【集群ID】/prometheus.【所在host】/etc/prometheus |
| Grafana      | <https://10.0.16.33:3000/> | /var/lib/ceph/【集群ID】/grafana.【所在host】/etc/grafana    |

### 如果修改了对应配置，直接重启服务

```plain
systemctl restart ceph-【集群ID】@alertmanager.【所在host】.service
systemctl restart ceph-【集群ID】@prometheus.【所在host】.service
systemctl restart ceph-【集群ID】@grafana.【所在host】.service
```


---

> 作者: [komorebi](https://blog.2nfree.com/)  
> URL: https://blog.2nfree.com/posts/deploy-ceph-by-cephadm/  

