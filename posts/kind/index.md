# Kind


前提条件 : 需要安装 kind 和 kubectl 以及 docker 环境

# 安装 Kind

```bash
# 安装 Kind
curl -Lo ./kind https://kind.sigs.k8s.io/dl/v0.17.0/kind-linux-amd64
chmod +x ./kind
mv ./kind /usr/local/bin/kind

apt update && apt install -y apt-transport-https
curl https://mirrors.aliyun.com/kubernetes/apt/doc/apt-key.gpg | apt-key add - 
cat <<EOF >/etc/apt/sources.list.d/kubernetes.list
deb https://mirrors.aliyun.com/kubernetes/apt/ kubernetes-xenial main
EOF
apt update

#查看版本
apt policy kubectl
#版本取决于kind创建的集群版本，可以先创建后安装
apt install kubectl=1.25.3-00
```

# 使用配置文件启动 Kind 集群

```yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
name: kind
nodes:
- role: control-plane
  extraPortMappings:
  - containerPort: 30443
    hostPort: 443
  - containerPort: 30080
    hostPort: 80
- role: worker
- role: worker
- role: worker
```

```bash
kind create cluster --config=kind.yaml
```

{{< admonition type="info" open=true >}}
这段配置为给 ingress-nginx 预留的端口转发配置

```yaml
  extraPortMappings:
  - containerPort: 30443
    hostPort: 443
  - containerPort: 30080
    hostPort: 80
```

{{< /admonition >}}


---

> 作者: [komorebi](https://blog.2nfree.com/)  
> URL: https://blog.2nfree.com/posts/kind/  

