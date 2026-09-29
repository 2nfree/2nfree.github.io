# Apache Flink Kubernetes Operator


# 前言

Flink Kubernetes Operator 扩展了 Kubernetes API，能够管理和操作 Flink 部署，该 Operator 具有以下特点：

- 部署和监控 Flink Application 和 Session 的 Kubernetes Deployment
- 升级、暂停和删除 Kubernetes Deployment
- 完整的日志记录和指标集成
- 灵活的部署以及与 Kubernetes 工具的本机集成

{{< image src="overview.webp" >}}

# 部署 flink operator

## 安装 cert-manager

在安装 flink operator 之前，需要在 Kubernetes 集群中安装 cert-manager，如果集群之前已经安装过 cert-manager，则无需安装

{{< tabs >}}
{{% tab title="通过 manifests 安装" %}}
```bash
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.13.3/cert-manager.yaml
```
{{% /tab %}}

{{% tab title="通过 helm 安装" %}}
```bash
helm repo add jetstack https://charts.jetstack.io
helm repo update
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.13.3/cert-manager.crds.yaml
helm install cert-manager jetstack/cert-manager --namespace cert-manager --create-namespace --version v1.13.3
```
{{% /tab %}}
{{< /tabs >}}

使用命令查看

```bash
kubectl get all -n cert-manager

NAME                                          READY   STATUS      RESTARTS   AGE
pod/cert-manager-57688f5dc6-gqjj6             1/1     Running     0          10m
pod/cert-manager-cainjector-d7f8b5464-h9gt9   1/1     Running     0          10m
pod/cert-manager-startupapicheck-hcth4        0/1     Completed   0          10m
pod/cert-manager-webhook-58fd67545d-knq2q     1/1     Running     0          10m

NAME                           TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)    AGE
service/cert-manager           ClusterIP   10.110.70.247   <none>        9402/TCP   10m
service/cert-manager-webhook   ClusterIP   10.104.112.12   <none>        443/TCP    10m

NAME                                      READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/cert-manager              1/1     1            1           10m
deployment.apps/cert-manager-cainjector   1/1     1            1           10m
deployment.apps/cert-manager-webhook      1/1     1            1           10m

NAME                                                DESIRED   CURRENT   READY   AGE
replicaset.apps/cert-manager-57688f5dc6             1         1         1       10m
replicaset.apps/cert-manager-cainjector-d7f8b5464   1         1         1       10m
replicaset.apps/cert-manager-webhook-58fd67545d     1         1         1       10m

NAME                                     COMPLETIONS   DURATION   AGE
job.batch/cert-manager-startupapicheck   1/1           7m57s      10m
```

## 安装 flink-operator

flink operator 可以直接通过 helm 安装

```bash
# <OPERATOR-VERSION> 为版本号，可以在 https://flink.apache.org/downloads/ 查看选择对应的 operator 版本
helm repo add flink-operator-repo https://downloads.apache.org/flink/flink-kubernetes-operator-<OPERATOR-VERSION>/

# 这里安装 1.7.0 版本
helm repo add flink-operator-repo https://downloads.apache.org/flink/flink-kubernetes-operator-1.7.0/
helm install flink-kubernetes-operator flink-operator-repo/flink-kubernetes-operator --namespace flink --create-namespace

#查看
kubectl get all -n flink

NAME                                             READY   STATUS    RESTARTS   AGE
pod/flink-kubernetes-operator-7bdcc785dd-rkvkw   2/2     Running   0          3m57s

NAME                                     TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)   AGE
service/flink-operator-webhook-service   ClusterIP   10.100.207.157   <none>        443/TCP   3m57s

NAME                                        READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/flink-kubernetes-operator   1/1     1            1           3m57s

NAME                                                   DESIRED   CURRENT   READY   AGE
replicaset.apps/flink-kubernetes-operator-7bdcc785dd   1         1         1       3m57s
```

至此，flink operator 已经成功安装完毕

# 使用 flink operator 部署 flink job

在 flink operator 中，官方提供了 FlinkDeployment 和 FlinkSessionJobs 两种 CRD，可以部署基于 Application 模式或基于 Session 模式部署的 flink job
可以访问 <https://github.com/apache/flink-kubernetes-operator/tree/main/examples> 查看相关的示例

以下是两种不同模式部署的解析：

- 基于 Application 模式的部署

```yaml
apiVersion: flink.apache.org/v1beta1
kind: FlinkDeployment
metadata:
  namespace: flink
  name: test-flink
spec:
  image: flink:1.17
  flinkVersion: v1_17
  # 这部分用于配置当前 flink job 的相关配置
  flinkConfiguration:
    # 配置 flink 的 slot 数量，影响 flink 并行度，
    # 当 slot 数量与 job.parallelism 数量一致时，只会启动单个 taskmanager
    # 当 slot 数量为 job.parallelism 整数倍时，会启动对应数量的 taskmanager
    taskmanager.numberOfTaskSlots: "2"
    # 配置 flink job 的最大内存分配
    taskmanager.memory.process.size: "6 gb"
    taskmanager.memory.managed.fraction: "0.1"
    # 配置 savepoints 和 checkpoints 保存位置，注意这个地方配置的位置为持久化检查点 pvc 挂载的地方
    # 按照当前配置为挂载到 /flinkdata，也可以自行选择挂载位置，记得对应
    state.savepoints.dir: file:///flinkdata/savepoints
    state.checkpoints.dir: file:///flinkdata/checkpoints
    high-availability: org.apache.flink.kubernetes.highavailability.KubernetesHaServicesFactory
    high-availability.storageDir: file:///flinkdata/ha
    execution.checkpointing.interval: 5 s
    execution.checkpointing.timeout: 10 min
    execution.checkpointing.externalized-checkpoint-retention: RETAIN_ON_CANCELLATION
    execution.checkpointing.tolerable-failed-checkpoints: "2"
    restart-strategy: fixed-delay
    restart-strategy.fixed-delay.attempts: "3"
    restart-strategy.fixed-delay.delay: 10 s
    # 开启指标导出
    metrics.reporter.prom.class: org.apache.flink.metrics.prometheus.PrometheusReporter
  # 这部分是 flink 的日志配置，可以根据自己的需要修改
  logConfiguration:
    ....
  serviceAccount: flink
  # 该部分是 flink 主容器的模板配置，jobmanager 和 taskmanager 都会使用该模板，也就是说在此模板的相关配置会同步
  podTemplate:
    apiVersion: v1
    kind: Pod
    metadata:
      name: pod-template
    spec:
      # 设置安全上下文使用 uid 9999
      securityContext:
        runAsUser: 9999
        runAsGroup: 9999
        fsGroup: 9999
      containers:
      - name: flink-main-container
        volumeMounts:
          # flink 检查点的持久化 pvc 需要挂载到 /flinkdata 下
          - mountPath: /flinkdata
            name: test-flink-flinkdata
          # 需要让 /opt/flink/downloads 文件夹在 init容器、jobmanager 和 taskmanager 中共通，用于放置运行的 jar 包
          - mountPath: /opt/flink/downloads
            name: downloads
      volumes:
        # 这里是相关卷的配置，对应到前面使用的挂载
        - name: test-flink-flinkdata
          persistentVolumeClaim:
            claimName: test-flink-flinkdata
        - name: downloads
          emptyDir: {}
  jobManager:
    resource:
      memory: "2048m"
    podTemplate:
      apiVersion: v1
      kind: Pod
      metadata:
        name: task-manager-pod-template
      spec:
        # 在使用 FlinkDeployment 直接运行 job 时，无法直接从远程依赖仓库拉取 flink job 的 jar 包，需要使用 wget 在容器运行前下载对应的 flink jar 包
        initContainers:
          - name: wget
            image: netdata/wget
            volumeMounts:
              - mountPath: /opt/flink/downloads
                name: downloads
            command:
            - /bin/sh
            - -c
            - "wget -O /opt/flink/downloads/test.jar https://xxxx.com/test.jar"
  taskManager:
    resource:
      # 和 taskmanager.memory.process.size 一致
      memory: "6gb"
  job:
    # 这里的地址必须要和 wget -O 下载的位置一致
    jarURI: local:///opt/flink/downloads/test.jar
    # 配置 flink job 的并行度
    # 当 slot 数量与 job.parallelism 数量一致时，只会启动单个 taskmanager
    # 当 slot 数量为 job.parallelism 整数倍时，会启动对应数量的 taskmanager
    parallelism: 2
    upgradeMode: last-state
    state: running
```

- 基于 Session 模式的部署

```yaml
# 初始化一个 FlinkDeployment
apiVersion: flink.apache.org/v1beta1
kind: FlinkDeployment
metadata:
  name: test-flink
spec:
  image: flink:1.17
  flinkVersion: v1_17
  jobManager:
    resource:
      memory: "2048m"
  taskManager:
    resource:
      memory: "2048m"
  serviceAccount: flink
---
# 配置 FlinkSessionJob 提交到对应的 FlinkDeployment 运行
apiVersion: flink.apache.org/v1beta1
kind: FlinkSessionJob
metadata:
  name: test-job1
spec:
  deploymentName: test-flink
  job:
    jarURI: https://repo1.maven.org/maven2/org/apache/flink/flink-examples-streaming_2.12/1.16.1/flink-examples-streaming_2.12-1.16.1-TopSpeedWindowing.jar
    parallelism: 4
    upgradeMode: stateless
---
apiVersion: flink.apache.org/v1beta1
kind: FlinkSessionJob
metadata:
  name: test-job2
spec:
  deploymentName: test-flink
  job:
    jarURI: https://repo1.maven.org/maven2/org/apache/flink/flink-examples-streaming_2.12/1.16.1/flink-examples-streaming_2.12-1.16.1.jar
    parallelism: 2
    upgradeMode: stateless
    entryClass: org.apache.flink.streaming.examples.statemachine.StateMachineExample
```

# Application 模式和 Session 模式的区别

Application 模式和 Session 模式的主要区别为是否需要实现高可用（High Availability，HA）

在使用 Application 模式时，每个 flink job 都会独占一个独立的 flink 集群，这个集群只运行这一个 flink 作业，并且可以通过持久化配置的方式，实现 HA
而 Session 模式为启动一个共享的 flink 集群，这个集群可以运行多个提交的 flink job，这样的好处是，无需每次都启动新的 flink 集群用于运行作业，但是并不能依靠 Operator 的方式来实现 HA，但是可以通过持久化相关数据来侧面实现

关于 Flink 的 HA 可以参考：<https://nightlies.apache.org/flink/flink-docs-master/docs/deployment/ha/overview/>

# 访问 flink 集群

在 FlinkDeployment 启动后，会创建一个 xxx-rest 的 service 到 Kubernetes 集群内，其中 xxx 为 FlinkDeployment 的名称
可以通过配置 NodePort 或者 Ingress 的方式将该服务代理到集群外部以供访问

# 导出 flink 集群指标到 Prometheus

{{< admonition type="warning" open=true >}}
该步骤的前提条件为：集群中已经使用 prometheus-operator 安装了 prometheus 的相关组件
{{< /admonition >}}

当我们在 FlinkDeployment 中配置如下配置时

```yaml
...
spec:
  image: flink:1.17
  flinkVersion: v1_17
  flinkConfiguration:
    # 开启指标导出
    metrics.reporter.prom.class: org.apache.flink.metrics.prometheus.PrometheusReporter
...
```

会在 flink 集群中开启指标导出的接口，默认端口为 9249，api 为 metrics，我们需要配置一个 Service 和 ServiceMonitor 来收集这些指标到 Prometheus

- Service 配置：

```yaml
apiVersion: v1
kind: Service
metadata:
  name: test-flink-metrics-service
  namespace: flink
  labels:
    app: test-flink-metrics-service
spec:
  ports:
  - name: http-metrics
    port: 9249
    protocol: TCP
    targetPort: 9249
  selector:
    app: test-flink
```

- ServiceMonitor 配置：

```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: test-flink
  namespace: monitoring
spec:
  endpoints:
  - port: http-metrics
    interval: 10s
  selector:
    matchLabels:
      app: test-flink-metrics-service
  namespaceSelector:
    matchNames:
    - flink
```

dashboard 参考：

- <https://grafana.com/grafana/dashboards/14911-flink/>
- <https://grafana.com/grafana/dashboards/11049-flink-dashboard/>


---

> 作者: [komorebi](https://blog.2nfree.com/)  
> URL: https://blog.2nfree.com/posts/flink-operator/  

