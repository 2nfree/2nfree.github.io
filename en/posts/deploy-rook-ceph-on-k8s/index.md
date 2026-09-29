# Deploy Rook Ceph on Kubernetes


# Foreword

## Persistent Storage

Persistent storage for containers is a crucial means of preserving the storage state of containers. Storage plugins mount a remote data volume in the container, based on a network or other mechanisms, ensuring that files created inside the container are actually stored on a remote storage server or distributed across multiple nodes. This way, there is no binding relationship with the current host machine.

As a result, no matter which host you start a new container on, you can request to mount the specified persistent storage volume, thereby accessing the data stored in the volume.

Due to Kubernetes' loosely coupled design, most storage solutions, such as Ceph, GlusterFS, NFS, and others, can provide persistent storage capabilities for Kubernetes.

## Ceph

Ceph is a distributed storage system that provides file, block and object storage and is deployed in large scale production clusters.

Official website: [ceph.io](https://ceph.io/)

## Rook

Rook is an open source cloud-native storage orchestrator, providing the platform, framework, and support for Ceph storage to natively integrate with cloud-native environments.

Rook automates deployment and management of Ceph to provide self-managing, self-scaling, and self-healing storage services. The Rook operator does this by building on Kubernetes resources to deploy, configure, provision, scale, upgrade, and monitor Ceph.

Official website: [rook.io](https://rook.io/)

GitHub: [rook/rook](https://github.com/rook/rook)

# Prepare

{{< admonition type="info" open=true >}}
PS：Before installing Rook, ensure that the Kubernetes cluster has at least three available nodes to meet Ceph's high availability requirements.
{{< /admonition >}}

## Disk

Rook uses Ceph to create distributed storage services. During the initial installation, Rook automatically detects all the disks on the nodes and creates OSD services to manage them.

Rook monitors and discovers available devices based on the following criteria: The device has no partitions and does not have a formatted file system.

Rook will not use devices that do not meet the above criteria. Additionally, you can modify the configuration file to specify which nodes or devices will be used.

The following steps are for installing Rook on Kubernetes (CentOS):

```bash
# Check disk status
lsblk -f

# Initialize Disk
yum install gdisk -y
sgdisk --zap-all /dev/vdb
dd if=/dev/zero of="/dev/vdb" bs=1M count=100 oflag=direct,dsync
blkdiscard /dev/vdb
partprobe /dev/vdb

# Install lvm2
yum install lvm2 -y

# Enable rbd module
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

# Install

## Deploy Rook Operator

```bash
cd /data/packages

git clone --single-branch --branch release-1.9 https://github.com/rook/rook.git
```

```bash
cd /data/packages/rook/deploy/examples

# apply common manifests
kubectl create -f crds.yaml
kubectl create -f common.yaml
```

- This file is a Kubernetes manifests file, and you can directly use Kubernetes commands to deploy all the resources defined in the file to the cluster.
- There is only one point to note: if you want to install Rook and the corresponding Ceph containers into a specific project, it is recommended to first create the project and the namespace.
- The common file resources will automatically create a namespace called `rook-ceph`, and all subsequent resources and containers will be installed within this namespace.

```bash
# Modify the `operator.yaml` configuration to enable automatic disk discovery.
# By default, automatic discovery is disabled. Once enabled, Rook will automatically create OSDs when new raw disk devices are connected.
# However, note that unmounting a disk will not remove the OSD.
ROOK_ENABLE_DISCOVERY_DAEMON: true

# Install operator
kubectl create -f operator.yaml
```

- The operator is the core of the entire Rook system, and all subsequent functions such as cluster creation, automatic orchestration, and expansion are implemented based on the operator.
- After the installation is complete, you need to wait for all operators to be running normally before proceeding with the installation of the Ceph distributed cluster.

Check the installation status by running the following command, and wait until all the pods are in the Running state before proceeding to the next step

```bash
kubectl -n rook-ceph get pod
```

This will display the status of all pods in the rook-ceph namespace. Ensure all pods are in the Running state before continuing with the Ceph distributed cluster installation.

## Deploy rook-ceph

With a single YAML orchestration file, you can manage the deployment of the entire Ceph component, including disk configuration, cluster setup, and other related operations.

By applying this YAML file with the kubectl command, you can automatically deploy and configure the entire Ceph system within your Kubernetes environment. Here's a simplified command for applying the file:

```bash
kubectl apply -f cluster.yaml
```

The OSD containers must exist and be running properly. If all the OSD pods are running successfully, the cluster installation is considered successful.

To verify the status of the OSD pods, use the following command:

```bash
kubectl -n rook-ceph get pod
```

Check that all OSD pods (e.g., osd-0, osd-1, osd-2, or any others based on your configuration) are in the Running state. If all these OSD pods are running without any issues, such as CrashLoopBackOff or Error, the Ceph cluster installation can be deemed successful.

## Check

```bash
# Apply toolbox pod
kubectl apply -f toolbox.yaml

# Enter the toolbox pod
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- bash

# Check
ceph status
ceph osd status
ceph df 
rados df
```

## StorageClass

After the cluster setup is complete, the next step is to create storage. Currently, Ceph supports three types of storage: block storage, file system storage, and object storage.

- **Block Storage**: This is the storage solution officially integrated with Kubernetes and is considered the most stable option. However, block storage currently does not support multi-host read-write access (it is limited to `RWO` - ReadWriteOnce).
  
- **File System Storage**: It supports multi-host access and offers good performance, making it suitable for scenarios requiring concurrent access across multiple nodes.
  
- **Object Storage**: Due to its poor I/O performance, it is generally not recommended for pvc use cases.

Once the storage system is created, you must add a **StorageClass** for the system. Only then can the Ceph storage be used directly by the Kubernetes cluster through the Kubernetes storage class.

Given the various options and detailed configuration required, it is recommended to refer to the official Rook documentation for guidance: [Storage Configuration](https://rook.github.io/docs/rook/latest-release/Storage-Configuration/Block-Storage-RBD/block-storage/).

PS：It is recommended to use the **XFS** file system instead of **EXT4**, as EXT4 creates a `lost+found` directory after formatting. Some containers, such as MySQL, require the mounted data disk to be completely empty. If using EXT4 is necessary, you can choose to mount it to a parent directory to avoid conflicts caused by the `lost+found` folder.

Using **XFS** ensures a cleaner mount point without the need for extra adjustments, making it more suitable for containers that expect an empty volume.

### CephFS

#### Create a File System Pool

To create a file system pool in Ceph, especially for testing purposes, you can use a default configuration. For production environments, it is recommended to adjust the number of replicas (shards) to 2 or 3 for better data redundancy.

```yaml
apiVersion: ceph.rook.io/v1
kind: CephFilesystem
metadata:
  name: cephfs
  namespace: rook-ceph
spec:
  metadataPool:
    replicated:
      # replicas 1 for test
      size: 1
      requireSafeReplicaSize: false
  dataPools:
    - name: replicated
      failureDomain: osd
      replicated:
        # replicas 1 for test
        size: 1
        requireSafeReplicaSize: false
  preserveFilesystemOnDelete: false
  metadataServer:
    activeCount: 1
    activeStandby: true
```

```bash
kubectl apply -f ceph-filesystem.yaml

# Check pod status
kubectl -n rook-ceph get pod -l app=rook-ceph-mds

# Enter the toolbox pod
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- bash
ceph status
```

#### Create StorageClass

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
# apply
kubectl apply -f storageclass-cephfs.yaml

# check
kubectl get sc
```

#### Set Default StorageClass

```bash
kubectl patch sc rook-cephfs -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'
```

#### Create Test PVC

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
kubectl apply -f pvc-cephfs-test.yaml
kubectl get pvc -A
kubectl delete -f pvc-cephfs-test.yaml
```

To expose the Ceph dashboard via an Ingress, you can apply one of the following methods based on your cluster configuration and requirements. Below are three typical options for exposing the dashboard

```text
dashboard-external-https.yaml
dashboard-ingress-https.yaml
dashboard-loadbalancer.yaml
```

```bash
# Get the default password of the dashboard
kubectl -n rook-ceph get secret rook-ceph-dashboard-password -o jsonpath="{['data']['password']}"|base64 --decode && echo
```

## Join the ceph cluster when adding a new node

```bash
# tag new node
kubectl label node [node-name] role=rook-osd-node

# restart operator
kubectl delete pod -n rook-ceph $(kubectl get pod -n rook-ceph | grep operator | awk '{print$1}')
```


---

> Author: [komorebi](https://blog.2nfree.com/)  
> URL: https://blog.2nfree.com/en/posts/deploy-rook-ceph-on-k8s/  

