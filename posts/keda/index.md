# KEDA - 基于 Kubernetes 的事件驱动自动缩放器


# 简介

KEDA（全称：Kubernetes-based Event Driven Autoscaler）是一个基于 Kubernetes 的事件驱动自动缩放器。 借助 KEDA，可以根据需要处理的事件数量来驱动 Kubernetes 中任何容器的扩展。

KEDA 是一个单一用途的轻量级组件，可以添加到任何 Kubernetes 集群中。 KEDA 与 Horizontal Pod Autoscaler 等标准 Kubernetes 组件一起工作，可以扩展功能而无需覆盖或重复。

借助 KEDA，您可以明确映射要使用事件驱动规模的应用程序，而其他应用程序继续运行。

优点：

- 为 Kubernetes 集群中的每个工作负载提供丰富的扩展能力
- 智能扩展您的事件驱动应用程序
- 适用于各种云平台、数据库、消息传递系统、遥测系统、CI/CD 等的 50 多个内置触发器
- 支持各种工作负载类型
- 支持跨各种云提供商和产品的触发器

# 原理图

{{< image src="keda.webp" >}}

# 部署 Keda

## 使用 helm 部署

```bash
helm repo add kedacore https://kedacore.github.io/charts
helm repo update
helm install keda kedacore/keda --namespace keda --create-namespace
```

如果需要使用不同版本的 crd，可以访问 [Keda Release](https://github.com/kedacore/keda/releases) 使用其中的 `keda-2.xx.x-crds.yaml` 文件

## 使用 Manifest 文件

访问 [Keda Release](https://github.com/kedacore/keda/releases) 获取 Manifest 文件
其中 `keda-2.xx.x.yaml` 是包含所有功能的 Keda（包括Admission Webhooks）
`keda-2.xx.x-core.yaml` 是安装所需的最少 KEDA 组件（不包含Admission Webhooks）

关于 [Admission Webhooks](https://keda.sh/docs/concepts/admission-webhooks/)

{{< admonition type="danger" open=true >}}
注意事项：
如果在 apply manifest 文件过程中出现了报错，请参考 [ISSUE](https://github.com/kedacore/keda/issues/4740) 使用 `kubectl apply --server-side -f ...`
{{< /admonition >}}

# CRD资源

安装 KEDA 时，它会创建四个自定义资源，这些自定义资源使您能够将事件源（以及对该事件源的身份验证）映射到 Deployment、StatefulSet、自定义资源或 Job 以进行扩展。

- scaledobjects.keda.sh：表示事件源（例如 Rabbit MQ）与 Kubernetes Deployment、StatefulSet 或定义 /scale 子资源的任何自定义资源之间所需的映射。
- scaledjobs.keda.sh：表示事件源和 Kubernetes Job 之间的映射。
- triggerauthentications.keda.sh：包含用于监视事件源的身份验证配置或 Secret。
- clustertriggerauthentications.keda.sh：包含用于监视事件源的身份验证配置或 Secret。

其中调度事件源如下所示：

```yaml
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: {scaled-object-name}
spec:
  # 对目标资源的引用，如Deployment 、 StatefulSet 或自定义资源
  scaleTargetRef:
    # 可选值，目标资源的 api 版本，默认为 apps/v1
    apiVersion: apps/v1
    # 可选值，目标资源的类型，默认为 Deployment
    kind: Deployment
    # 必填值，目标资源的名称，必须和 ScaledObject 在同一个命名空间
    name: nginx
    # 可选值，目标资源监听容器名称，默认为 .spec.template.spec.containers[0]
    envSourceContainerName: nginx
  # 可选值，轮询间隔，默认为 30 秒
  pollingInterval:  30
  # 可选值，最后一个触发器报告活动后将资源缩放回 0 之前等待的时间，默认为 5 分钟
  cooldownPeriod:   300
  # 可选值，空闲副本计数，默认为忽略改选项，该属性唯一支持的值为 0
  # 如果设置了此属性，KEDA 会将资源缩减至此副本数
  # 如果目标触发器上有一些活动，KEDA 会立即将目标资源扩展至minReplicaCount，然后由 HPA 处理扩展
  idleReplicaCount: 0
  # 可选值，最小副本数，默认值为 0
  minReplicaCount:  1
  # 可选值，最大副本数，默认值为 100
  maxReplicaCount:  100
  # 定义了在缩放器处于错误状态时要回退至多少副本数
  fallback:
    # 如果设置了 fallback 则 failureThreshold 和 replicas 必填
    # 如下示例为：当实例连续 3 次不可用时，KEDA 将更改 HPA 指标，以便部署将扩展到 2 个副本
    failureThreshold: 3
    replicas: 2
  # 高级配置
  advanced:
    # 可选值，指定在删除了该 ScaledObject 时目标资源是否应缩减至原始副本计数
    # 默认为 false，即保持在 ScaledObject 删除时的相同数量
    restoreToOriginalReplicaCount: true/false
    # KEDA 将创建的 HPA 资源
    horizontalPodAutoscalerConfig:
      # # 可选值，默认为 keda-hpa-{scaled-object-name}
      name: keda-hpa
      # 可选值，配置 HPA 的 behavior 字段
      behavior:
        scaleDown:
          stabilizationWindowSeconds: 300
          policies:
          - type: Percent
            value: 100
            periodSeconds: 15
  # 触发器，数组，配置触发扩容的触发器
  triggers:
  # 触发器的类型，当前以 CPU 触发器举例
  - type: cpu
    #要使用的指标类型，选项为 Utilization 或 AverageValue
    metricType: Utilization
    metadata:
      #触发缩放操作的值
      type: Utilization
      value: "60"
      # 根据其 CPU 进行扩展的特定容器的名称，而不是整个 Pod。如果未指定则默认为空
      containerName: ""
  ......
```

# 配置示例

如下的配置规定了一个名为 nginx 的 Deployment 的弹性伸缩，其中最小分片为 4，最大分片为 6，触发弹性伸缩的条件为 CPU 使用率达到 50% 以上持续 30 秒以上
关于 CPU 及内存的相关缩放，需要提供节点监控数据才可以使用，如部署 prometheus 的 metrics-server

```yaml
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: nginx-scale
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: nginx
    envSourceContainerName: nginx
  pollingInterval:  30
  cooldownPeriod:   300
  minReplicaCount:  4
  maxReplicaCount:  6
  fallback:
    failureThreshold: 3
    replicas: 4
  triggers:
  - type: cpu
    metricType: Utilization
    metadata:
      value: "50"
      containerName: "nginx"
```

# 缩放器类别

参考官方文档：[Scalers](https://keda.sh/docs/scalers/)


---

> 作者: [komorebi](https://blog.2nfree.com/)  
> URL: https://blog.2nfree.com/posts/keda/  

