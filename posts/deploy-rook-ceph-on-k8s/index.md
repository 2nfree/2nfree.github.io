# 安装Rook Ceph作为K8S的存储服务


# 前言

## 容器的持久化存储

容器的持久化存储是保存容器存储状态的重要手段，存储插件会在容器里挂载一个基于网络或者其他机制的远程数据卷，使得在容器里创建的文件，实际上是保存在远程存储服务器上，或者以分布式的方式保存在多个节点上，而与当前宿主机没有任何绑定关系。
这样，无论你在其他哪个宿主机上启动新的容器，都可以请求挂载指定的持久化存储卷，从而访问到数据卷里保存的内容。
由于 Kubernetes 本身的松耦合设计，绝大多数存储项目，比如 Ceph、GlusterFS、NFS 等，都可以为 Kubernetes 提供持久化存储能力。

## Ceph

Ceph是一种高度可扩展的分布式存储解决方案，提供对象、文件和块存储。

官方网站：[ceph.io](https://ceph.io/)

## Rook

Rook 是一个开源的cloud-native storage编排, 提供平台和框架；为各种存储解决方案提供平台、框架和支持，以便与云原生环境本地集成。

官方网站：[rook.io](https://rook.io/)

GitHub仓库：[rook/rook](https://github.com/rook/rook)

# 部署环境准备

PS：在集群中至少有三个节点可用，满足ceph高可用要求

## rook使用存储方式

rook默认使用所有节点的所有资源，rook operator自动在所有节点上启动OSD设备，Rook会用如下标准监控并发现可用设备：

- 设备没有分区
- 设备没有格式化的文件系统
- Rook不会使用不满足以上标准的设备。另外也可以通过修改配置文件，指定哪些节点或者设备会被使用

```bash
#服务器添加新磁盘

#查看磁盘情况
lsblk -f

#初始化磁盘
yum install gdisk -y
sgdisk --zap-all /dev/vdb
dd if=/dev/zero of="/dev/vdb" bs=1M count=100 oflag=direct,dsync
blkdiscard /dev/vdb
partprobe /dev/vdb

#确认安装lvm2
yum install lvm2 -y
#启用rbd模块
modprobe rbd
cat > /etc/rc.sysinit << EOF
#!/bin/bash
for file in /etc/sysconfig/modules/*.modules
do
  [ -x \$file ] && \$file
done
EOF
cat > /etc/sysconfig/modules/rbd.modules << EOF
modprobe rbd
EOF
chmod 755 /etc/sysconfig/modules/rbd.modules
lsmod |grep rbd
```

# 开始安装

## 部署Rook Operator

```bash
cd /data/packages

#国外
git clone --single-branch --branch release-1.9 https://github.com/rook/rook.git

#国内无法访问建议手动下载上传到服务器上
```

```bash
cd /data/packages/rook/deploy/examples

#安装常规资源
kubectl create -f crds.yaml
kubectl create -f common.yaml
```

- 这个文件是K8S的编排文件直接便可以使用K8S命令将文件之中编排的资源全部安装到集群之中
- 需要注意的只有一点，如果你想讲rook和对应的ceph容器全部安装到一个特定的项目之中去，那么建议优先创建项目和命名空间
- common文件资源会自动创建一个叫做rook-ceph的命名空间，后续所有的资源与容器都会安装到这里面去。

```bash
#修改operator.yaml中的配置，开启磁盘自动发现
#默认不开启自动发现，开启自动发现后，当你接入新的裸磁盘设备时会自动创建osd，但是将磁盘取消挂载不会删除osd
ROOK_ENABLE_DISCOVERY_DAEMON: true

#安装操作器
kubectl create -f operator.yaml
```

- operator是整个ROOK的核心，后续的集群创建、自动编排、拓展等等功能全部是在operator的基础之上实现的
- 安装完成之后需要等待所有的操作器正常运行之后才能继续ceph分布式集群的安装

```bash
#查看安装状态，等待所有的pod都是running状态之后继续下一步
kubectl -n rook-ceph get pod
```

# 部署rook-ceph集群

```bash
#通过一个yaml编排文件能够对整个Ceph组件部署、硬盘配置、集群配置等一系列操作
kubectl apply -f cluster.yaml
#其中osd-0、osd-1、osd-2容器必须是存在且正常的，如果上述pod均正常运行成功，则视为集群安装成功。
kubectl -n rook-ceph get pod
```

国内无法下载镜像需要自己pull

```text
quay.io/ceph/ceph:v16.2.9
quay.io/cephcsi/cephcsi:v3.6.2
quay.io/csiaddons/k8s-sidecar:v0.2.1
quay.io/csiaddons/volumereplication-operator:v0.3.0
registry.k8s.io/sig-storage/csi-attacher:v3.4.0
registry.k8s.io/sig-storage/csi-node-driver-registrar:v2.5.1
registry.k8s.io/sig-storage/csi-provisioner:v3.1.0
registry.k8s.io/sig-storage/csi-resizer:v1.4.0
registry.k8s.io/sig-storage/csi-snapshotter:v6.0.1
registry.k8s.io/sig-storage/nfsplugin:v4.0.0
rook/ceph:v1.9.5


#国内无法下载k8s.gcr.io的镜像
#可以使用国外服务器下载，导入到镜像仓库，记住tag和版本要一致
#或者使用镜像仓库地址，记住tag和版本要一致
#quay.io == quay.mirrors.ustc.edu.cn
#k8s.gcr.io == registry.aliyuncs.com/google_containers
```

# 测试集群

```bash
#安装工具
kubectl apply -f toolbox.yaml

#进入管理pod
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- bash

#测试
ceph status
ceph osd status
ceph df 
rados df
```

# 存储类

集群搭建完毕之后便是存储的创建，目前Ceph支持块存储、文件系统存储、对象存储三种方案，K8S官方对接的存储方案是块存储，他也是比较稳定的方案，但是块存储目前不支持多主机读写（只能RWO）；

文件系统存储是支持多主机存储的性能也不错；对象存储系统IO性能太差不考虑，所以可以根据要求自行决定。

存储系统创建完成之后对这个系统添加一个存储类之后整个集群才能通过K8S的存储类直接使用Ceph存储。
因为此部分分类太多，建议参考[Storage Configuration](https://rook.github.io/docs/rook/latest/Storage-Configuration/Block-Storage-RBD/block-storage/)

PS：涉及到格式化类型建议使用XFS而不是EXT4，因为EXT4格式化后会生成一个lost+found文件夹，某些容器要求挂载的数据盘必须是空的，例如Mysql，如果必要使用，可选择挂载到上一级目录

# CephFS存储类（共享文件系统）

## 创建文件系统池（测试用，生产环境建议修改分片数为2或3）

```yaml
apiVersion: ceph.rook.io/v1
kind: CephFilesystem
metadata:
  name: cephfs
  namespace: rook-ceph
spec:
  metadataPool:
    replicated:
      #设置分片为 1
      size: 1
      requireSafeReplicaSize: false
  dataPools:
    - name: replicated
      failureDomain: osd
      replicated:
        #设置分片为 1
        size: 1
        requireSafeReplicaSize: false
  preserveFilesystemOnDelete: false
  metadataServer:
    activeCount: 1
    activeStandby: true
```

```bash
kubectl apply -f ceph-filesystem.yaml

#查看文件系统pod创建情况
kubectl -n rook-ceph get pod -l app=rook-ceph-mds

#进入管理pod
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- bash
ceph status
```

## 创建文件系统存储类

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: rook-cephfs
provisioner: rook-ceph.cephfs.csi.ceph.com
parameters:
  clusterID: rook-ceph
  fsName: cephfs
  pool: cephfs-replicated
  csi.storage.k8s.io/provisioner-secret-name: rook-csi-cephfs-provisioner
  csi.storage.k8s.io/provisioner-secret-namespace: rook-ceph
  csi.storage.k8s.io/controller-expand-secret-name: rook-csi-cephfs-provisioner
  csi.storage.k8s.io/controller-expand-secret-namespace: rook-ceph
  csi.storage.k8s.io/node-stage-secret-name: rook-csi-cephfs-node
  csi.storage.k8s.io/node-stage-secret-namespace: rook-ceph
reclaimPolicy: Delete
allowVolumeExpansion: true
```

```bash
#创建
kubectl apply -f storageclass-cephfs.yaml

#查看存储类
kubectl get sc
```

## 设置默认存储

```bash
kubectl patch sc rook-cephfs -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'
```

## 创建测试pvc

```bash
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: test-cephfs-pvc
  namespace: kube-system
spec:
  accessModes:
  - ReadWriteMany
  resources:
    requests:
      storage: 1Gi
  storageClassName: rook-cephfs
```

```bash
#创建测试用的pvc
kubectl apply -f pvc-cephfs-test.yaml


kubectl get pvc -A
kubectl delete -f pvc-cephfs-test.yaml
```

# 开启dashboard（三选一，根据集群情况选择）

```text
dashboard-external-https.yaml
dashboard-ingress-https.yaml
dashboard-loadbalancer.yaml
```

```bash
#获取dashboard默认密码
kubectl -n rook-ceph get secret rook-ceph-dashboard-password -o jsonpath="{['data']['password']}"|base64 --decode && echo
```

# 加入新node时加入ceph集群

```bash
#节点打标签
kubectl label node [node名字] role=rook-osd-node

#删除operator
kubectl delete pod -n rook-ceph $(kubectl get pod -n rook-ceph | grep operator | awk '{print$1}')
```


---

> 作者: [komorebi](https://blog.2nfree.com/)  
> URL: https://blog.2nfree.com/posts/deploy-rook-ceph-on-k8s/  

