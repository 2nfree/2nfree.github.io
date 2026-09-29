# Rook-Ceph 开启卷快照功能


# 前言

CSI Snapshotter 为 Kubernetes 容器存储接口（CSI）提供了创建卷快照（VolumeSnapshot）的功能。
它定义了三种容器自定义资源定义 (Custom Resource Definition)，分别为 VolumeSnapshot，VolumeSnapshotContent 和 VolumeSnapshotClass。
卷快照的功能在 Kubernetes v1.12 中作为 Alpha 功能引入，并在 Kubernetes 1.17 中提升为 Beta 功能。在 Kubernetes 1.20 中，卷快照功能移至 GA。
如果 Rook 所在的 Kubernetes 集群版本过低，可能需要使用低版本的 CSI Snapshotter，具体参考官方原文：

{{< admonition type="info" open=true >}}

If only Alpha snapshots are available, enable snapshotter in rook-ceph-operator-config or helm chart values.yaml, change the external-snapshotter image to registry.k8s.io/sig-storage/csi-snapshotter:v1.2.2 and refer to the [alpha snapshots documentation](https://github.com/rook/rook/blob/release-1.3/Documentation/ceph-csi-drivers.md#rbd-snapshots)
VolumeSnapshot betav1 is deprecated in Kubernetes 1.20+ and removed in 1.24.0. If you still require betav1 snapshots, change the external-snapshotter image to registry.k8s.io/sig-storage/csi-snapshotter:v5.0.1 and refer to the [betav1 snapshots documentation](https://rook.github.io/docs/rook/v1.8/ceph-csi-snapshot.html#rbd-snapshots)

译：如果只有 Alpha 快照可用，请在 rook-ceph-operator-config 或 helm Chart values.yaml 中启用快照程序，将外部快照程序图像更改为 registry.k8s.io/sig-storage/csi-snapshotter:v1.2.2 并参考 [alpha 快照文档](https://github.com/rook/rook/blob/release-1.3/Documentation/ceph-csi-drivers.md#rbd-snapshots)
VolumeSnapshot betav1 在 Kubernetes 1.20+ 中已弃用，并在 1.24.0 中删除。如果您仍然需要 betav1 快照，请将外部快照映像更改为 registry.k8s.io/sig-storage/csi-snapshotter:v5.0.1 并参阅 [betav1 快照文档](https://rook.github.io/docs/rook/v1.8/ceph-csi-snapshot.html#rbd-snapshots)
{{< /admonition >}}

本文基于 CSI Snapshotter 7.0.1、Rook Ceph 1.10.2、Kubernetes 1.23.8，如果没有安装过 Rook-ceph 可以参考：[安装 Rook Ceph 作为 K8S 的存储服务](https://blog.2nfree.com/2023/02/17/deploy-rook-ceph-on-k8s/) 或 [Kubernetes 接入基于 Cephadm 部署的 Ceph 集群](https://blog.2nfree.com/2024/01/12/deploy-ceph-by-cephadm/)

# 安装 Snapshot-Controller

## 导入 crd 与 安装 snapshot-controller

Github：[Snapshotter](https://github.com/kubernetes-csi/external-snapshotter)

```bash
git clone -b release-7.0 https://github.com/kubernetes-csi/external-snapshotter.git

# 导入 crd
kubectl kustomize external-snapshotter/client/config/crd | kubectl create -f -

# 安装 snapshot-controller
kubectl -n kube-system kustomize external-snapshotter/client/config/snapshot-controller| kubectl create -f -
```

如果只需要使用基于 Rook-ceph 的卷快照的话，csi-snapshotter 不需要安装，Rook-ceph 的 csi 中已经包含了了这部分的功能

| 功能 | CSI-Snapshotter | 	Rook-Ceph |
| --- | --- | --- |
| 支持的卷类型 | 所有 CSI 卷 | CephFS 卷和 RBD 卷等 |
| 快照管理 | 使用 Kubernetes 内置的快照 API | 使用 Rook-Ceph 提供的原生工具 |
| 可扩展性 | 支持新的 CSI 卷驱动程序 | 支持新的 Ceph 功能 |


## 查看

```bash
kubectl get crd | grep snap

kubectl get pod -n kube-system -l app.kubernetes.io/name=snapshot-controller
```

# 配置卷快照类

就像 StorageClass 为管理员提供了一种在配置卷时描述其提供的存储“类”的方法一样，VolumeSnapshotClass 也提供了一种在配置卷快照时描述存储“类”的方法。

## ceph-rbd-snapclass

```yaml
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshotClass
metadata:
  name: ceph-rbd-snapclass
driver: rook-ceph.rbd.csi.ceph.com
parameters:
  clusterID: rook-ceph-external
  csi.storage.k8s.io/snapshotter-secret-name: rook-csi-rbd-provisioner
  csi.storage.k8s.io/snapshotter-secret-namespace: rook-ceph-external
deletionPolicy: Delete
```

## cephfs-snapclass

```yaml
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshotClass
metadata:
  name: cephfs-snapclass
driver: rook-ceph.cephfs.csi.ceph.com
parameters:
  clusterID: rook-ceph-external
  csi.storage.k8s.io/snapshotter-secret-name: rook-csi-cephfs-provisioner
  csi.storage.k8s.io/snapshotter-secret-namespace: rook-ceph-external
deletionPolicy: Delete
```

## 配置解析

每个 VolumeSnapshotClass 包含字段 driver 、 deletionPolicy 和 parameters ，当需要动态配置属于该类的 VolumeSnapshot 时使用这些字段。

- driver：卷快照类有一个驱动程序，用于确定使用什么 CSI 卷插件来配置 VolumeSnapshots。必须指定该字段；对于 Rook-ceph 来说，RBD 为：rook-ceph.rbd.csi.ceph.com、CephFS 为：rook-ceph.cephfs.csi.ceph.com
- deletionPolicy：卷快照类具有删除策略。它使您能够配置当要删除与 VolumeSnapshotContent 绑定的 VolumeSnapshot 对象时，该内容会发生什么情况。卷快照类的deletionPolicy可以是 Retain 或 Delete 。必须指定该字段。如果删除策略为 Delete ，则底层存储快照将与 VolumeSnapshotContent 对象一起删除。如果删除策略为 Retain ，则底层快照和 VolumeSnapshotContent 都会保留。
- parameters：这里的参数需要我们填写对应的 Ceph 集群的名称以及连接到集群的 secret 相关配置

## 查看配置的卷快照类

```bash
kubectl get volumesnapshotclasses.snapshot.storage.k8s.io
```

# 创建快照测试

格式：

```yaml
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshot
metadata:
  name: 【快照名称】
spec:
  volumeSnapshotClassName: 【快照存储类】
  source:
    persistentVolumeClaimName: 【需要备份的 PVC 名称，需要与快照存储类使用同一CSI】
```

```bash
kubectl apply -f - <<"EOF"
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshot
metadata:
  name: mypvc-snap
spec:
  volumeSnapshotClassName: ceph-rbd-snapclass
  source:
    persistentVolumeClaimName: mypvc
EOF
```

# 恢复快照测试

格式：

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: 【PVC 名称】
spec:
  storageClassName: 【存储类】
  dataSource:
    name: 【快照名称】
    kind: VolumeSnapshot
    apiGroup: snapshot.storage.k8s.io
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi # 这里的大小可以大于原始快照的大小
```

```bash
kubectl apply -f - <<"EOF"
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mypvc-restore
spec:
  storageClassName: ceph-rbd-snapclass
  dataSource:
    name: mypvc-snap
    kind: VolumeSnapshot
    apiGroup: snapshot.storage.k8s.io
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi # 这里的大小可以大于原始快照的大小
EOF
```


---

> Author: [komorebi](https://blog.2nfree.com/)  
> URL: https://blog.2nfree.com/en/posts/volume-snapshots-with-rook-ceph/  

