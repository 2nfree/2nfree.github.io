# 基于kubeadm的k8s部署


# 前言

## Kubernetes 概述

Kubernetes 是一个可移植、可扩展的开源平台，用于管理容器化的工作负载和服务，可促进声明式配置和自动化。 Kubernetes 拥有一个庞大且快速增长的生态，其服务、支持和工具的使用范围相当广泛。

Kubernetes 这个名字源于希腊语，意为“舵手”或“飞行员”。k8s 这个缩写是因为 k 和 s 之间有八个字符的关系。 Google 在 2014 年开源了 Kubernetes 项目。 Kubernetes 建立在 Google 大规模运行生产工作负载十几年经验的基础上， 结合了社区中最优秀的想法和实践。

# 准备工作

## 服务器清单

| **主机名** | **内网IP** | **操作系统** |
| --- | --- | --- |
| master1 | 192.168.1.10 | centos7 |
| master2 | 192.168.1.11 | centos7 |
| node1 | 192.168.1.20 | centos7 |
| node2 | 192.168.1.21 | centos7 |

## 服务器端口

防火墙开放相关端口号，云主机修改安全组，本地服务器修改防火墙规则

测试可以直接开启互信、关闭防火墙

| **组件** | **默认端口号** |
| --- | --- |
| API Server | 8080（http）<br> 6443（https） |
| Controller Manager | 10252 |
| Scheduler | 10251 |
| kubelet | 10250 <br> 10255（read only） |
| etcd | 2379（client）<br> 2380（etcd cluster） |
| 集群DNS | 53（UDP TCP） |
| CNI Calico | 179 |

## 安装基础工具

```bash
#更新yum源
yum update -y
#安装相关依赖和工具
yum install net-tools curl wget epel-release vim gcc wget make libtool expat-devel pcre-devel openssl-devel libxml2-devel -y
yum install git lsof ncdu psmisc htop openssl lrzsz -y
yum install conntrack ipvsadm ipset jq sysstat iptables libseccomp -y
```

# 安装容器运行时

{{< admonition type="info" title="容器运行时" open=true >}}
Kubernetes 的运行需要容器运行时的支持，所以我们需要在服务器上安装容器运行时

一般用的较多的容器运行时是Docker，但是官方推荐的容器运行时是Containerd
{{< /admonition >}}

{{< admonition type="tip" open=true >}}
PS：K8S在1.24版本，取消了对docker-shim的支持，如果安装K8S的版本高于1.24版本，建议安装Containerd
{{< /admonition >}}

{{< tabs >}}
{{% tab title="Docker" %}}
```bash
#安装yum工具
yum install -y yum-utils \
  device-mapper-persistent-data \
  lvm2

#官方仓库（国外服务器使用）
yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
#阿里云仓库（国内服务器使用）
yum-config-manager --add-repo http://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo

#安装docker
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

#启动docker
systemctl start docker
systemctl enable docker
systemctl status docker
```
{{% /tab %}}
{{% tab title="Containerd" %}}
```bash
#加载模块
modprobe overlay
modprobe br_netfilter

cat <<EOF | sudo tee /etc/modules-load.d/containerd.conf
overlay
br_netfilter
EOF

#安装nerdctl工具包
wget https://github.com/containerd/nerdctl/releases/download/v1.0.0/nerdctl-full-1.0.0-linux-amd64.tar.gz
tar Cxzvvf /usr/local nerdctl-full-1.0.0-linux-amd64.tar.gz
systemctl enable --now containerd
systemctl enable --now buildkit

sudo mkdir -p /etc/containerd
sudo containerd config default | sudo tee /etc/containerd/config.toml

#修改如下配置
SystemdCgroup = true
....

[plugins."io.containerd.grpc.v1.cri".registry]
      config_path = "/etc/containerd/certs.d"  # 镜像地址配置文件

#创建相应目录
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

# 服务器系统配置

## 关闭selinux

```bash
#永久关闭
sed -i 's/enforcing/disable/' /etc/selinux/config
#临时关闭
setenforce 0
```

## 关闭swap交换分区

```bash
#（部分云服务默认关闭，可以用free命令查看或者查看/etc/fstab）
#临时关闭
swapoff -a
#永久关闭
sed -ri 's/.*swap.*/#&/' /etc/fstab
```

## 将桥接的IPV4流量传递到iptables

```bash
#加载网桥过滤(安装containerd时已经配置)
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

#生效配置
sysctl -p /etc/sysctl.d/kubernetes.conf
```

## 设置ipvs代理模式

在kubernetes中service有两种代理模型，一种是基于iptables的，一种是基于ipvs的
两者比较的话，ipvs的性能明显要高一些，但是如果要使用它，需要手动载入ipvs模块

```bash
yum install ipset ipvsadmin -y
```

开启模块
{{< tabs >}}
{{% tab title="低内核版本" %}}
```bash
cat > /etc/sysconfig/modules/ipvs.modules <<EOF
#!/bin/bash
modprobe -- ip_vs
modprobe -- ip_vs_rr
modprobe -- ip_vs_wrr
modprobe -- ip_vs_sh
modprobe -- nf_conntrack_ipv4
EOF

#添加可执行权限并执行
chmod +x /etc/sysconfig/modules/ipvs.modules
sh /etc/sysconfig/modules/ipvs.modules
```
{{% /tab %}}
{{% tab title="高内核版本" %}}
```bash
cat > /etc/sysconfig/modules/ipvs.modules <<EOF
#!/bin/bash
modprobe -- ip_vs
modprobe -- ip_vs_rr
modprobe -- ip_vs_wrr
modprobe -- ip_vs_sh
modprobe -- nf_conntrack
EOF

#添加可执行权限并执行
chmod +x /etc/sysconfig/modules/ipvs.modules
sh /etc/sysconfig/modules/ipvs.modules
```
{{% /tab %}}
{{< /tabs >}}

## 设置时间同步（一般云服务厂商已经设置，本地服务器需配置一下）

```bash
yum -y install chrony

#开启服务
systemctl start chronyd
systemctl enable chronyd
```

# 安装kubernetes组件

## 配置kubernetes组件的yum源

{{< tabs >}}
{{% tab title="官方源" %}}
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

#安装指定版本
#查看版本
yum search kubeadm --showduplicates

yum install -y kubectl-1.21.11-0.x86_64 kubeadm-1.21.11-0.x86_64 kubelet-1.21.11-0.x86_64 --disableexcludes=kubernetes
```
{{% /tab %}}
{{% tab title="镜像源" %}}
```bash
cat > /etc/yum.repos.d/kubernetes.repo <<EOF
[kubernetes]
name=Kubernetes
baseurl=https://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64/
enabled=1
gpgcheck=1
repo_gpgcheck=0
gpgkey=https://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg https://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg
EOF

yum install -y kubectl kubeadm kubelet --nogpgcheck

#安装指定版本
#查看版本
yum search kubeadm --showduplicates

yum install -y kubectl-1.21.11-0.x86_64 kubeadm-1.21.11-0.x86_64 kubelet-1.21.11-0.x86_64 --nogpgcheck
```
{{% /tab %}}
{{< /tabs >}}

## 启动kubelet

```bash
systemctl start kubelet
systemctl enable kubelet
#kubelet显示报错是正常的，因为当前集群节点没有启动
systemctl status kubelet
```

# 集群镜像准备

## 查看镜像版本

```bash
#查看当前镜像列表，此处加上 --kubernetes-version=v1.21.11 可以指定版本，--v=5可以检查网卡状态
kubeadm config images list
kubeadm config images list --kubernetes-version=v1.21.11
kubeadm config images list --kubernetes-version=v1.21.11 --v=5
```

## 下载镜像

{{< tabs >}}
{{% tab title="官方下载" %}}
```bash
kubeadm config images pull --kubernetes-version=v1.21.11

#检查镜像
docker images
```
{{% /tab %}}
{{% tab title="镜像源下载" %}}
镜像源可以使用脚本下载(注意版本对应)

```bash
#!/bin/bash
images=(
kube-apiserver:v1.21.11
kube-controller-manager:v1.21.11
kube-scheduler:v1.21.11
kube-proxy:v1.21.11
pause:3.4.1
etcd:3.4.13-0

)
for imageName in ${images[@]} ; do
docker pull registry.cn-hangzhou.aliyuncs.com/google_containers/${imageName}
docker tag registry.cn-hangzhou.aliyuncs.com/google_containers/${imageName} k8s.gcr.io/${imageName}
docker rmi registry.cn-hangzhou.aliyuncs.com/google_containers/${imageName}
done

docker pull registry.cn-hangzhou.aliyuncs.com/google_containers/coredns:v1.8.0
docker tag registry.cn-hangzhou.aliyuncs.com/google_containers/coredns:v1.8.0 k8s.gcr.io/coredns/coredns:v1.8.0
docker rmi registry.cn-hangzhou.aliyuncs.com/google_containers/coredns:v1.8.0
```
{{% /tab %}}
{{< /tabs >}}

# 启动K8S控制平面（Control plane）

## 在master1中执行开启控制平面

```bash
#docker作为容器运行时（高版本中docker-shim被废弃，建议使用containerd）
kubeadm init \
  --control-plane-endpoint 192.168.1.10:6443 \
  --kubernetes-version=v1.21.11  \
  --pod-network-cidr=10.244.0.0/16 \
  --service-cidr=10.96.0.0/12 --upload-certs

#containerd
kubeadm init \
  --kubernetes-version v1.21.11  \
  --control-plane-endpoint 192.168.1.10:6443 \
  --image-repository registry.aliyuncs.com/google_containers \
  --pod-network-cidr 10.244.0.0/16 \
  --cri-socket /run/containerd/containerd.sock \
  --service-cidr=10.96.0.0/12 --upload-certs
```

保留以下数据

```bash
#暂时保留以下数据
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

## 在master2中执行加入控制平面

```bash
kubeadm join 192.168.1.10:6443 --token grjzmr.ttkq7t4l3w9r2fb7 \
        --discovery-token-ca-cert-hash sha256:d9f4a8c6d14584b998a3ca86fd7e727f7d9d779f80e834dcf19e9116f22f3fad \
        --control-plane --certificate-key 2346e1d55aa63ab30c3d7dca7529e9f91097383322e36336483a42ffa76e4c27
```

## 在所有master执行配置

```bash
mkdir -p $HOME/.kube
cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
chown $(id -u):$(id -g) $HOME/.kube/config
```

# 将node加入集群

```bash
kubeadm join 192.168.1.10:6443 --token grjzmr.ttkq7t4l3w9r2fb7 \
        --discovery-token-ca-cert-hash sha256:d9f4a8c6d14584b998a3ca86fd7e727f7d9d779f80e834dcf19e9116f22f3fad
```

# 安装CNI网络插件（在master1上进行就好）

```bash
#查看节点信息，发现当前节点not ready，需要安装CNI网络插件，这里选择Calico，也可以使用其他网络插件

#下载配置
wget https://docs.projectcalico.org/manifests/calico.yaml

#安装
kubectl apply -f calico.yaml

#国内无法连接calico，下载下面网址的镜像，导入到docker中，或者导入到镜像仓库中
https://github.com/projectcalico/calico

#下载对应版本的，查看版本
cat calico.yaml | grep image

#重新查看节点信息
kubectl get nodes
```

# 使用IPVS代理

```bash
#查看当前ipvs规则发现没有使用ipvs
ipvsadm -L -n

#修改kubeproxy配置
kubectl edit configmap -n kube-system kube-proxy
#修改部分为mode: "ipvs"

#删除重启kubeproxy
kubectl get pod -n kube-system -o wide | grep kube-proxy

#查看ipvs规则
ipvsadm -L -n
```

至此K8S集群已经搭建完毕，云主机可以通过配置NLB实现对外访问ApiService，本地服务器可以配置Keepalived+HAProxy对ApiService进行代理


---

> 作者: [komorebi](https://blog.2nfree.com/)  
> URL: https://blog.2nfree.com/posts/deploy-k8s-with-kubeadm/  

