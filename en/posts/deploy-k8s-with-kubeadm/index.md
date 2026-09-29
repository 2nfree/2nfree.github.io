# Deploy Kubernetes With Kubeadm


# Foreword

## Introduction

This document provides a step-by-step guide to deploying a Kubernetes cluster using Kubeadm (version 1.21.11). The deployment process includes setting up the control plane node and worker nodes, and verifying the cluster's functionality.

# Preparation

## Servers

| **HOST** | **IP** | **SYSTEM** |
| --- | --- | --- |
| master1 | 192.168.1.10 | centos7 |
| master2 | 192.168.1.11 | centos7 |
| node1 | 192.168.1.20 | centos7 |
| node2 | 192.168.1.21 | centos7 |

## Server Port

Open the relevant firewall ports, modify the security group settings for the cloud server, and update the firewall rules on the local server.
For test you can set all trust or disable the firewall.

| **UNIT** | **PORT** |
| --- | --- |
| API Server | 8080（http）<br> 6443（https） |
| Controller Manager | 10252 |
| Scheduler | 10251 |
| kubelet | 10250 <br> 10255（read only） |
| etcd | 2379（client）<br> 2380（etcd cluster） |
| DNS | 53（UDP TCP） |
| CNI Calico | 179 |

## Basic Tools

```bash
yum update -y
yum install net-tools curl wget epel-release vim gcc wget make libtool expat-devel pcre-devel openssl-devel libxml2-devel -y
yum install git lsof ncdu psmisc htop openssl lrzsz -y
yum install conntrack ipvsadm ipset jq sysstat iptables libseccomp -y
```

# Install Container Runtime

{{< admonition type="info" title="Container Runtime" open=true >}}
To run Kubernetes, container runtime support is required, so we need to install a container runtime on the server.
The most commonly used container runtime is Docker, but the officially recommended container runtime is Containerd.
{{< /admonition >}}

{{< admonition type="tip" open=true >}}
Note: In Kubernetes version 1.24, support for docker-shim was removed. If you are installing a Kubernetes version higher than 1.24, it is recommended to install Containerd.
{{< /admonition >}}

{{< tabs >}}
{{% tab title="Docker" %}}
```bash
yum install -y yum-utils \
  device-mapper-persistent-data \
  lvm2

yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo

# install
yum install -y docker-ce docker-ce-cli containerd.io

mkdir -pv /etc/docker

cat > /etc/docker/daemon.json <<"EOF"
{
  "log-driver":"json-file",
  "log-opts": {
    "max-size": "5m",
    "max-file":"3"
  },
  "exec-opts": ["native.cgroupdriver=systemd"]
}
EOF

# start
systemctl start docker
systemctl enable docker
systemctl status docker
```
{{% /tab %}}
{{% tab title="Containerd" %}}
```bash
# load mod
modprobe overlay
modprobe br_netfilter

cat <<EOF | sudo tee /etc/modules-load.d/containerd.conf
overlay
br_netfilter
EOF

# install the nerdctl toolkit
wget https://github.com/containerd/nerdctl/releases/download/v1.0.0/nerdctl-full-1.0.0-linux-amd64.tar.gz
tar Cxzvvf /usr/local nerdctl-full-1.0.0-linux-amd64.tar.gz
systemctl enable --now containerd
systemctl enable --now buildkit

sudo mkdir -p /etc/containerd
sudo containerd config default | sudo tee /etc/containerd/config.toml

# edit config
SystemdCgroup = true
....

[plugins."io.containerd.grpc.v1.cri".registry]
      config_path = "/etc/containerd/certs.d"

# create dir
mkdir /etc/containerd/certs.d/docker.io -pv

cat > /etc/containerd/certs.d/docker.io/hosts.toml << EOF
server = "https://docker.io"
[host."https://docker.mirrors.ustc.edu.cn"]
  capabilities = ["pull", "resolve"]
EOF

systemctl restart containerd
```
{{% /tab %}}
{{< /tabs >}}

# Server System Config

## Disable Selinux

```bash
# permanent
sed -i 's/enforcing/disable/' /etc/selinux/config
# temporary
setenforce 0
```

## Disable SWAP

```bash
#（Some cloud services are turned off by default, which can be viewed with the free command or by looking at /etc/fstab）
# permanent
swapoff -a
# temporary
sed -ri 's/.*swap.*/#&/' /etc/fstab
```

## Pass bridged IPV4 traffic to iptables

```bash
# load mod
modprobe br_netfilter

cat > /etc/sysctl.d/kubernetes.conf <<EOF
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
net.ipv4.ip_forward = 1
vm.swappiness=0
vm.overcommit_memory = 1
vm.panic_on_oom = 0
fs.inotify.max_user_watches = 89100
EOF

sysctl -p /etc/sysctl.d/kubernetes.conf
```

## IPVS Config

In Kubernetes, there are two service proxy models: one based on iptables and another based on IPVS.
When comparing the two, the performance of IPVS is significantly higher. However, to use IPVS, you need to manually load the IPVS modules.

```bash
yum install ipset ipvsadmin -y
```

load mod
{{< tabs >}}
{{% tab title="low kernel version" %}}
```bash
cat > /etc/sysconfig/modules/ipvs.modules <<EOF
#!/bin/bash
modprobe -- ip_vs
modprobe -- ip_vs_rr
modprobe -- ip_vs_wrr
modprobe -- ip_vs_sh
modprobe -- nf_conntrack_ipv4
EOF

# Add executable permissions and execute
chmod +x /etc/sysconfig/modules/ipvs.modules
sh /etc/sysconfig/modules/ipvs.modules
```
{{% /tab %}}
{{% tab title="high kernel version" %}}
```bash
cat > /etc/sysconfig/modules/ipvs.modules <<EOF
#!/bin/bash
modprobe -- ip_vs
modprobe -- ip_vs_rr
modprobe -- ip_vs_wrr
modprobe -- ip_vs_sh
modprobe -- nf_conntrack
EOF

# Add executable permissions and execute
chmod +x /etc/sysconfig/modules/ipvs.modules
sh /etc/sysconfig/modules/ipvs.modules
```
{{% /tab %}}
{{< /tabs >}}

## Set up time synchronization (generally cloud service vendors have already set it up, local servers need to be configured)

```bash
yum -y install chrony

# start
systemctl start chronyd
systemctl enable chronyd
```

# Install Kubernetes

## Yum Repo

```bash
cat > /etc/yum.repos.d/kubernetes.repo <<EOF
[kubernetes]
name=Kubernetes
baseurl=https://packages.cloud.google.com/yum/repos/kubernetes-el7-x86_64
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://packages.cloud.google.com/yum/doc/yum-key.gpg https://packages.cloud.google.com/yum/doc/rpm-package-key.gpg
exclude=kube*
EOF

yum install -y kubectl kubeadm kubelet --disableexcludes=kubernetes

yum search kubeadm --showduplicates

yum install -y kubectl-1.21.11-0.x86_64 kubeadm-1.21.11-0.x86_64 kubelet-1.21.11-0.x86_64 --disableexcludes=kubernetes
```

## Start Kubelet

```bash
systemctl start kubelet
systemctl enable kubelet
# It is normal for kubelet to report errors because the current cluster node is not up.
systemctl status kubelet
```

# Images

## Get Image List

```bash
kubeadm config images list
kubeadm config images list --kubernetes-version=v1.21.11
kubeadm config images list --kubernetes-version=v1.21.11 --v=5
```

## Download

```bash
kubeadm config images pull --kubernetes-version=v1.21.11

# check
docker images && nerdctl --namespace k8s.io images
```

# Control Plane

## Init

```bash
# docker (docker-shim is deprecated in higher versions, containerd is recommended)
kubeadm init \
  --control-plane-endpoint 192.168.1.10:6443 \
  --kubernetes-version=v1.21.11  \
  --pod-network-cidr=10.244.0.0/16 \
  --service-cidr=10.96.0.0/12 --upload-certs

# containerd
kubeadm init \
  --kubernetes-version v1.21.11  \
  --control-plane-endpoint 192.168.1.10:6443 \
  --image-repository registry.aliyuncs.com/google_containers \
  --pod-network-cidr 10.244.0.0/16 \
  --cri-socket /run/containerd/containerd.sock \
  --service-cidr=10.96.0.0/12 --upload-certs
```

```bash
# retain following data, it will be used later
Your Kubernetes control-plane has initialized successfully!

To start using your cluster, you need to run the following as a regular user:

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

Alternatively, if you are the root user, you can run:

  export KUBECONFIG=/etc/kubernetes/admin.conf

You should now deploy a pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
  https://kubernetes.io/docs/concepts/cluster-administration/addons/

You can now join any number of the control-plane node running the following command on each as root:

 kubeadm join 192.168.1.10:6443 --token grjzmr.ttkq7t4l3w9r2fb7\        --discovery-token-ca-cert-hash sha256:d9f4a8c6d14584b998a3ca86fd7e727f7d9d779f80e834dcf19e9116f22f3fad \
        --control-plane --certificate-key 2346e1d55aa63ab30c3d7dca7529e9f91097383322e36336483a42ffa76e4c27

Please note that the certificate-key gives access to cluster sensitive data, keep it secret!
As a safeguard, uploaded-certs will be deleted in two hours; If necessary, you can use
"kubeadm init phase upload-certs --upload-certs" to reload certs afterward.

Then you can join any number of worker nodes by running the following on each as root:

kubeadm join 192.168.1.10:6443 --token grjzmr.ttkq7t4l3w9r2fb7 \
        --discovery-token-ca-cert-hash sha256:d9f4a8c6d14584b998a3ca86fd7e727f7d9d779f80e834dcf19e9116f22f3fad
```

## Execute Join Control Plane in master2

```bash
kubeadm join 192.168.1.10:6443 --token grjzmr.ttkq7t4l3w9r2fb7 \
        --discovery-token-ca-cert-hash sha256:d9f4a8c6d14584b998a3ca86fd7e727f7d9d779f80e834dcf19e9116f22f3fad \
        --control-plane --certificate-key 2346e1d55aa63ab30c3d7dca7529e9f91097383322e36336483a42ffa76e4c27
```

## Perform configuration on all masters

```bash
mkdir -p $HOME/.kube
cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
chown $(id -u):$(id -g) $HOME/.kube/config
```

# Add the node to the cluster

```bash
kubeadm join 192.168.1.10:6443 --token grjzmr.ttkq7t4l3w9r2fb7 \
        --discovery-token-ca-cert-hash sha256:d9f4a8c6d14584b998a3ca86fd7e727f7d9d779f80e834dcf19e9116f22f3fad
```

# Install the CNI network plugin (do it on master)

```bash
# Check the node information, found that the current node is not ready, you need to install CNI, here choose Calico, you can also use others

# download manifests
wget https://docs.projectcalico.org/manifests/calico.yaml

# apply
kubectl apply -f calico.yaml

# check
kubectl get nodes
```

# Use IPVS

```bash
# view current IPVS rules
ipvsadm -L -n

# edit kube-proxy config
kubectl edit configmap -n kube-system kube-proxy
# Edit: `mode: "ipvs"`

# restart kube-proxy，get pod and delete pod
kubectl get pod -n kube-system -o wide | grep kube-proxy

# check
ipvsadm -L -n
```

At this point the K8S cluster has been built, the cloud host can be configured through the NLB to achieve external access to the ApiService, the local server can be configured with Keepalived + HAProxy on the ApiService proxy!


---

> Author: [komorebi](https://blog.2nfree.com/)  
> URL: https://blog.2nfree.com/en/posts/deploy-k8s-with-kubeadm/  

