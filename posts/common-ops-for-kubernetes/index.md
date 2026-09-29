# 常见 Kubernetes 维护场景


# Kubeadm更新证书

```bash
# 创建用于备份的相关文件夹，防止出现问题
mkdir -pv /data/kube-config-backup/{kubernetes,kube,yaml}

# 备份配置文件
cp -ra /etc/kubernetes/ /data/kube-config-backup/kubernetes/
cp -ra ~/.kube/ /data/kube-config-backup/kube/

# 重新生成证书
kubeadm certs renew all

# 检查证书时间
kubeadm certs check-expiration

# 重新生成配置
kubeadm init phase kubeconfig all

# 复制管理配置
rm -rf /root/.kube/*
cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
chown $(id -u):$(id -g) $HOME/.kube/config

# 重启 kube-system，将 manifests 中的配置移除并移入来重启
mv /etc/kubernetes/manifests/*.yaml /data/kube-config-backup/yaml/
mv /data/kube-config-backup/yaml/*.yaml /etc/kubernetes/manifests/

# 在所有节点重启 kubelet
systemctl restart kubelet

# 重启 prometheus
kubectl rollout restart -n monitoring deploy
kubectl rollout restart -n monitoring statefulsets
kubectl rollout restart -n monitoring daemonset

```

# kubeadm 添加新节点

## 简单方法

```bash
# 生成token并打印join语句
kubeadm token create --print-join-command

# 使用生成的 join 语句在对应节点执行添加

# 加入 node
kubeadm join [ControlPlaneAddress] --token [Token] \
--discovery-token-ca-cert-hash [TokenCAHash]

# 加入 master
kubeadm join [ControlPlaneAddress] --token [Token] \
        --discovery-token-ca-cert-hash [TokenCAHash] \
        --control-plane --certificate-key [Cert]
```

## 手动生成

```bash
# 生成 token (1)
kubeadm token create

# 获取 --discovery-token-ca-cert-hash 值(2)
openssl x509 -pubkey -in /etc/kubernetes/pki/ca.crt | openssl rsa -pubin -outform der 2>/dev/null | openssl dgst -sha256 -hex | sed 's/^.* //'

# 生成 upload-certs (3)
kubeadm init phase upload-certs --upload-certs

# 加入 node
kubeadm join [ControlPlaneAddress] --token [1的值] \
--discovery-token-ca-cert-hash [2的值]

# 加入 master
kubeadm join [ControlPlaneAddress] --token [1的值] \
        --discovery-token-ca-cert-hash [2的值] \
        --control-plane --certificate-key [3的值]
```

# Deployment的Rollout与Rollback

{{< admonition type="tip" open=true >}}
Deployment 的 rollout 当且仅当 Deployment 的 pod template（例如.spec.template）中的 label 更新或者镜像更改时被触发
其他更新，例如:扩容 Deployment 不会触发 rollout
{{< /admonition >}}

```bash
#查看 rollout 的状态
kubectl rollout status deployment [deployment名称]

#查看 rollout 的历史版本
kubectl rollout history deployment [deployment名称]

#查看单个版本的详细信息
kubectl rollout history deployment [deployment名称] --revision=2

#回退当前的 rollout 到之前的版本
kubectl rollout undo deployment [deployment名称]

#也可以使用 —revision 参数指定某个历史版本
kubectl rollout undo deployment [deployment名称] --to-revision=2
```

# Kubernetes 污点和容忍度

## 污点相关命令

```bash
#查看污点
kubectl describe nodes [node name] | grep Taints

#删除污点
kubectl taint node [node name] [Taint name]-

#添加污点
kubectl taint node [node name] [Taint name]:[Taint value]
```

## 容忍度

配置容忍度需要在工作负载上声明 Toleration
下面两个 Toleration 都被设置为可以容忍具有污点的 Node，使 Pod 能够调度到有污点的节点上

```yaml
tolerations:
- key: "key"
  operator: "Equal"
  value: "value"
  effect: "NoSchedule"
```

```yaml
tolerations:
- key: "key"
  operator: "Exists"
  effect: "NoSchedule"
```

Pod 的 Toleration 声明中的 key 和 effect 需要与 Taint 的设置保持一致，并且满足以下条件之一：

- operator 的值是 Exists（无需指定 value）
- operator 的值是 Equal 并且 value相等

```yaml
tolerations:
- key: "node-role.kubernetes.io/master"
  operator: "Equal"
  value: "node-role.kubernetes.io/master"
  effect: "NoSchedule"
```

```yaml
tolerations:
- key: "node-role.kubernetes.io/master"
  operator: "Exists"
  effect: "NoSchedule"
```

# Kubernetes 资源限额

## Request 和 Limit（在工作负载或 Pod 中）

```yaml
spec:
  containers:
    resources:
      requests:
        cpu: 2  # 有时会采用写法为 2000m，其中1000m约等于1核
        memory: 2Gi # 也可使用Mi代替，如 2048Mi
      limits:
        cpu: 2
        memory: 2Gi
```

K8S 通过 Request 和 Limit 两个抽象概念来支持资源的升级与配额的管理
对于以下两种资源 K8S 都有权来管理
Request 主要是应用发布时对容器的使用量进行灵活配置，即 K8S 会根据该值决定调度的节点
Limit 主要是对应用容器进行资源限制，即应用容器的最大资源配额

## QoS(Quality of Service)

当集群资源不足时，K8S 会根据 Pod 标记的 QoS 类别做剔除决策，腾出空闲资源

- 当 Request = Limit 时，QoS 为保证型（Guaranteed），只有当使用量超限时，才会被 Kill
- 当 Request < Limit 时（包含只设置了 Request，未设置 Limit 的工作负载或 Pod），QoS 为突发型（Burstable），当节点资源不足时，这类 Pod 可能会被 Kill 掉
- 当既不设置 Request，又不设置 Limit 时，QoS 为最大努力型（BestEffort），只要当节点资源不够调度时，这类 Pod 首先就会被kill掉

## ResourceQuota/LimitRange（对于命名空间限制）

ResourceQuota/LimitRange 用于控制特定命名空间中的资源使⽤量，最终实现集群的公平使⽤和成本的控制

- 限制运⾏状态的Pod的计算资源⽤量
- 限制持久存储卷的数量以控制对存储的访问
- 限制负载均衡器的数量以控制成本
- 防⽌滥⽤⽹络端⼝
- 提供默认的计算资源Requests以便于系统做出更优化的调度

示例：

### 命名空间总资源限制

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: object-counts
  namespace: demo
spec:
  hard:
    persistentvolumeclaims: "2" # 持久存储卷数量
    services.loadbalancers: "2" # 负载均衡器数量
    services.nodeports: "0" # NodePort数量
    pods: "4" # Pod数量
    requests.cpu: "1" # 限制请求CPU数量
    requests.memory: 1Gi # 限制请求内存大小
    limits.cpu: "2" # 限制最小CPU数量限制
    limits.memory: 2Gi # 限制最小内存限制
```

### 命名空间内默认资源配置

```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: limits
  namespace: demo
spec:
  limits:
  - default:
      cpu: 200m
      memory: 512Mi
    defaultRequest:
      cpu: 100m
      memory: 256Mi
    type: Container
```

### 计算资源配额

| 资源名称 | 描述 |
| --- | --- |
| `limits.cpu` | 所有非终止状态的 Pod，其 CPU 限额总量不能超过该值 |
| `limits.memory` | 所有非终止状态的 Pod，其内存限额总量不能超过该值 |
| `requests.cpu` | 所有非终止状态的 Pod，其 CPU 需求总量不能超过该值 |
| `requests.memory` | 所有非终止状态的 Pod，其内存需求总量不能超过该值 |
| `hugepages-<size>` | 对于所有非终止状态的 Pod，针对指定尺寸的巨页请求总数不能超过此值 |
| `cpu` | 与 `requests.cpu` 相同 |
| `memory` | 与 `requests.memory` 相同 |

### 存储资源配额

| 资源名称 | 描述 |
| --- | --- |
| `requests.storage` | 所有 PVC，存储资源的需求总量不能超过该值 |
| `persistentvolumeclaims` | 在该命名空间中所允许的 PVC 总量 |
| `<storage-class-name>.storageclass.storage.k8s.io/requests.storage` | 在所有与 `<storage-class-name>` 相关的 PVC 中，存储请求的总和不能超过该值 |
| `<storage-class-name>.storageclass.storage.k8s.io/persistentvolumeclaims` | 在与 `<storage-class-name>` 相关的所有 PVC 中，命名空间中可以存在的 PVC 总数 |

其他请参考官方文档：[资源配额](https://kubernetes.io/zh-cn/docs/concepts/policy/resource-quotas/)

# 代理外部服务

在 Kubernetes 中，使用 Service 和 Endpoints 可以代理外部的服务，让 K8S 集群内部的工作负载通过 Service 连接外部的服务

```yaml
apiVersion: v1
kind: Service
metadata:
  name: external-api
  namespace: external
  labels:
    app: external-api
spec:
  type: ClusterIP
  ports:
  - name: http
    port: 8080
    protocol: TCP
---
kind: Endpoints
apiVersion: v1
metadata:
  name: external-api
  namespace: external
  labels:
    app: external-api
subsets:
  - addresses:
      - ip: 外部的IP
    ports:
      - name: http
        port: 8080
```


---

> 作者: [komorebi](https://blog.2nfree.com/)  
> URL: https://blog.2nfree.com/posts/common-ops-for-kubernetes/  

