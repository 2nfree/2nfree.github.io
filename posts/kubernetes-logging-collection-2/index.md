# Kubernetes 日志收集（二）- 基于 Logging Operator 的日志收集


# Logging Operator 介绍

Logging operator 基于 CRD 规定和管理日志收集架构，我们通过相关规定的资源可以在 K8S 中轻松的部署日志采集器、日志转发器与相关的日志路由规则

## 自定义资源

每次修改完相关 CRD 时，对应的资源需要一段时间之后才会收到影响，需要等待一段时间

### Logging
>
> `Logging` 定义用于收集和传输日志消息的集群的日志记录基础架构

- `Logging` 包含 `Fluent Bit` 日志收集器（基于 `DemonSet` 部署）以及 `Fluentd` 和 `Syslog-ng` 日志转发器（基于 `StatefulSet` 部署）的配置，在新版本中，可以使用 `FluentbitAgent` 代替 `Fluent Bit` 的配置，将其与日志转发器隔离开
- 日志收集器（`Fluent Bit`）作为 `Daemonset` 部署在节点上，主要用于收集节点上的日志并传入日志转发器
- 日志转发器可以接收、过滤和转换传入的日志，并将它们传输到一个或多个目标输出，Logging Operator 支持 `Fluentd` 和 `Syslog-ng` 作为日志转发器，`Syslog-ng` 支持多线程处理可提供更高的性能，`Fluentd` 支持丰富的输入输出源以及各种插件，可以根据各种需要选择不同的日志转发器
- 在创建 `Logging` 时，会建立 `controlNamespace`，即 Logging Operator 的管理命名空间，`Fluentd`|`Syslog-ng` 和 `Fluent Bit` 部署在此命名空间中，默认情况下，仅在此命名空间中评估诸如 `ClusterOutput` 和 `ClusterFlow` 之类的全局资源（即使它们在任何其他命名空间，除非 `allowClusterResourcesFromAllNamespaces` 设置为 true）

### Flow
>
> `Flow` 将选定的日志消息路由到指定的输出，它包含了 `Flow` 与 `ClusterFlow`

- `Flow` 是一个 `namespaced` 资源，因此仅收集来自相同命名空间的日志。可以指定 `match` 语句根据 Kubernetes labels、容器和主机名来选择或排除日志（匹配语句按照定义和处理的顺序进行评估，直到第一个匹配的 select 或 exclude 规则应用为止）
- `ClusterFlow` 定义了一个没有命名空间限制的 `Flow`。它也只在 `controlNamespace` 中有效。 `ClusterFlow` 从所有命名空间中选择日志
- `Flow` 和 `ClusterFlow` 是针对 `Fluentd Forwarder` 的 CRD 资源，如果我们要使用 `Syslog-ng` 作为 `Forwarder`，需要将对应的名称改为：`SyslogNGFlow` 和 `SyslogNGClusterFlow`

### OutPut
>
> `OutPut` 是日志转发器将日志消息发送到的目的地，如常用的 Elasticsearch、Loki 或 Kafka 等

- `Output` 也是 `namespaced` 资源，定义了 `Flow` 可以发送日志消息的输出。意味着只有同一命名空间内的 `Flow` 可以访问它。可以在这些定义中使用 `secrets`，但它们也必须位于同一命名空间中。输出是日志转发的最后阶段
- `ClusterOutput` 定义没有命名空间限制的输出
- 同 `Flow`，如果要使用`Syslog-ng`，要将对应的名称改为：`SyslogNGFlow` 和 `SyslogNGClusterOutput`

## 官方文档及架构图
>
> [https://kube-logging.dev/docs/](https://kube-logging.dev/docs/)

# Logging Operator 安装

需要提前安装 helm

```bash
helm upgrade --install --wait \
     --create-namespace --namespace logging \
     --set testReceiver.enabled=true \
     logging-operator oci://ghcr.io/kube-logging/helm-charts/logging-operator
```

以上命令除了安装 Logging Operator 外，还会安装一个测试用的 deployment `logging-operator-test-receiver`，它侦听 HTTP 端口，接收 JSON 消息，并将它们写入标准输出 (stdout)，我们在配置日志转发器时，可以接入这个服务，一边检查我们的日志格式是否有问题。

```bash
# 查看服务运行情况
kubectl get deploy -n logging
```

# 配置日志收集器和日志转发器

{{< admonition type="danger" open=true >}}
在本文中，我们使用 `Fluentd` 作为日志转发器
{{< /admonition >}}

## Fluentd（CRD：Logging）

这里配置了一个三分片的 `Fluentd` 并且配置了 pod 亲和性使其不调度在一个节点，使用 pvc 对 buffer 数据进行了持久化

```yaml
apiVersion: logging.banzaicloud.io/v1beta1
kind: Logging
metadata:
  name: logging-collector
spec:
  controlNamespace: logging
  fluentd:
    scaling:
      replicas: 3
    bufferStorageVolume:
      pvc:
        spec:
          storageClassName: 【这里修改成集群的存储类】
          accessModes: [ "ReadWriteOnce" ]
          resources:
            requests:
              storage: 5Gi
    affinity:
      podAntiAffinity:
        requiredDuringSchedulingIgnoredDuringExecution:
          - labelSelector:
              matchExpressions:
              - key: app.kubernetes.io/name
                operator: In
                values:
                  - fluentd
            topologyKey: kubernetes.io/hostname
```

## Fluent-bit（CRD：FluentbitAgent）

这里额外配置了 `/data/docker/containers` 是因为修改过 docker 的默认 `data-root` 文件夹，如果默认的 docker 数据保存在 `/var/lib/docker` 则不需要添加
配置 `tolerations` 使 `daemonset` 可以调度到 master 上让我们后续可以收集 `kube-system` 相关日志

```yaml
apiVersion: logging.banzaicloud.io/v1beta1
kind: FluentbitAgent
metadata:
  name: logging-collector-agent
spec:
  extraVolumeMounts:
  - source: /data/docker/containers/
    destination: /data/docker/containers/
    readOnly: true
  tolerations:
  - key: "node-role.kubernetes.io/master"
    operator: "Exists"
    effect: "NoSchedule"
```

> 当我们 apply `Logging` 和 `FluentbitAgent` 时，会启动名为 `logging-collector-fluentd` 的 `Fluentd`（statefulset）和名为 `logging-collector-agent-fluentbit` 的 `Fluent-Bit`（daemonset），此时 `Fluent-Bit` 就会采集节点上的日志到 `Fluentd` 中

# 检查

## 文件挂载检查

查看 `daemonset` 的详细信息查看我们配置的额外挂载是否生效

```bash
kubectl describe daemonsets.apps -n logging logging-collector-agent-fluentbit
```

```text
Volumes:
  varlibcontainers:
  Type:          HostPath (bare host directory volume)
  Path:          /var/lib/docker/containers
  HostPathType:
  varlogs:
  Type:          HostPath (bare host directory volume)
  Path:          /var/log
  HostPathType:
  extravolumemount0:
  Type:          HostPath (bare host directory volume)
Path:          /data/docker/containers/
HostPathType:
  config:
  Type:        Secret (a volume populated by a Secret)
  SecretName:  logging-collector-agent-fluentbit
  Optional:    false
  positiondb:
  Type:       EmptyDir (a temporary directory that shares a pod's lifetime)
  Medium:
  SizeLimit:  <unset>
  buffers:
  Type:       EmptyDir (a temporary directory that shares a pod's lifetime)
  Medium:
  SizeLimit:  <unset>
```

## 配置检查

```bash
kubectl get secret logging-collector-agent-fluentbit -n logging -o jsonpath='{.data.fluent-bit\.conf}'|base64 --decode
```

```ini
[SERVICE]
    Flush        1
    Grace        5
    Daemon       Off
    Log_Level    info
    Parsers_File /fluent-bit/etc/parsers.conf
    Coro_Stack_Size    24576
    storage.path  /buffers
[INPUT]
    Name         tail
    DB  /tail-db/tail-containers-state.db
    DB.locking  true
    Mem_Buf_Limit  5MB
    Parser  docker
    Path  /var/log/containers/*.log
    Refresh_Interval  5
    Skip_Long_Lines  On
    Tag  kubernetes.*
[FILTER]
    Name        kubernetes
    Buffer_Size  0
    K8S-Logging.Exclude  On
    Kube_CA_File  /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
    Kube_Tag_Prefix  kubernetes.var.log.containers
    Kube_Token_File  /var/run/secrets/kubernetes.io/serviceaccount/token
    Kube_Token_TTL  600
    Kube_URL  https://kubernetes.default.svc:443
    Match  kubernetes.*
    Merge_Log  On
    Use_Kubelet  Off
[OUTPUT]
    Name tcp
    Match *
    Host logging-collector-syslog-ng.logging.svc.cluster.local.
    Port 601
    Format json_lines
    json_date_key ts
    json_date_format iso8601
```

## 日志等级（可选）

修改日志级别可以查看采集是否有问题，如下所示将 `logLevel` 配置为 `trace`

```yaml
apiVersion: logging.banzaicloud.io/v1beta1
kind: FluentbitAgent
metadata:
  name: logging-collector-agent
spec:
  logLevel: trace
......
```

`Fluentd` 同上

```yaml
apiVersion: logging.banzaicloud.io/v1beta1
kind: Logging
metadata:
  name: logging-collector
spec:
  controlNamespace: logging
  fluentd:
    logLevel: trace
  ......
```

# Flow 和 OutPut

## 部署打印测试日志的容器

通过 golang 编写一个一直打印多行错误日志的 Deployment 用于测试，日志格式如下：

```text
2024-03-20 06:31:44.934 ERROR log_test/main.go:19 main.main 日志测试 2024-03-20 06:31:44.934
main.main
        log_test/main.go:19
runtime.main
        runtime/proc.go:271
```

代码如下：

```golang
package main

import (
 "os"
 "time"

 "go.uber.org/zap"
 "go.uber.org/zap/zapcore"
)

func init() {
 InitZapLogger()
}

var Logger *zap.Logger

func InitZapLogger() {
 Logger = zap.New(
  zapcore.NewTee(
   zapcore.NewCore(
    encoderConfig(),
    zapcore.AddSync(os.Stdout),
    zapcore.DebugLevel,
   ),
  ),
  zap.Development(),
  zap.AddCaller(),
  zap.AddStacktrace(zap.ErrorLevel),
 )
}

func encoderConfig() zapcore.Encoder {
 zapEncode := zapcore.EncoderConfig{
  MessageKey:          "Message",
  LevelKey:            "Level",
  TimeKey:             "Timestamp",
  NameKey:             "Name",
  CallerKey:           "Caller",
  FunctionKey:         "Function",
  StacktraceKey:       "Stacktrace",
  SkipLineEnding:      false,
  LineEnding:          zapcore.DefaultLineEnding,
  EncodeLevel:         zapcore.CapitalLevelEncoder,
  EncodeTime:          encodeTime,
  EncodeDuration:      zapcore.SecondsDurationEncoder,
  EncodeCaller:        zapcore.ShortCallerEncoder,
  EncodeName:          zapcore.FullNameEncoder,
  NewReflectedEncoder: nil,
  ConsoleSeparator:    " ",
 }
 return zapcore.NewConsoleEncoder(zapEncode)
}

func encodeTime(t time.Time, enc zapcore.PrimitiveArrayEncoder) {
 enc.AppendString(t.Format("2006-01-02 15:04:05.000"))
}

func main() {
 ticker := time.NewTicker(5 * time.Second)
 defer func() {
  ticker.Stop()
 }()
 for range ticker.C {
  Logger.Sugar().Errorf("日志测试 %s", time.Now().Format("2006-01-02 15:04:05.000"))
 }
}
```

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: print-logs
  labels:
    app: print-logs
    logging: golang
spec:
  selector:
    matchLabels:
      app: print-logs
  replicas: 1
  template:
    metadata:
      labels:
        app: print-logs
        logging: golang
    spec:
      containers:
      - name: print-logs
        image: print-test-log
      restartPolicy: Always
```

## 配置 Flow 和 OutPut （使用logging-operator-test-receiver测试）

```yaml
apiVersion: logging.banzaicloud.io/v1beta1
kind: Flow
metadata:
  name: test-logging
spec:
  match:
  - select:
      labels:
        logging: golang
  localOutputRefs:
    - test-receiver
---
apiVersion: logging.banzaicloud.io/v1beta1
kind: Output
metadata:
  name: test-receiver
spec:
  http:
    endpoint: http://logging-operator-test-receiver:8080
    content_type: application/json
    buffer:
      type: memory
      tags: time
      timekey: 1s
      timekey_wait: 0s
```

> 创建完毕后，会在 `default` 命名空间创建 `Flow：log-generator` 和 `OutPut：test-receiver`，并将 `kubernetes label` 为 `logging=golang` 的日志传输到`logging-operator-test-receiver`打印

# 日志的格式化

## 多行日志的初步合并

在使用 docker 作为 kubernetes 的容器运行时时，容器日志会将每一行打印的日志拆分开
实际日志为：

```text
2024-03-20 06:31:44.934 ERROR log_test/main.go:19 main.main 日志测试 2024-03-20 06:31:44.934
main.main
        log_test/main.go:19
runtime.main
        runtime/proc.go:271
```

会被拆分成：

```text
{"log":"2024-03-20 06:31:44.934 ERROR log_test/main.go:19 main.main 日志测试 2024-03-20 06:31:44.934\n","stream":"stdout","time":"2024-03-20T06:29:59.935462228Z"}
{"log":"main.main\n","stream":"stdout","time":"2024-03-20T06:29:59.935581674Z"}
{"log":"\u0009log_test/main.go:19\n","stream":"stdout","time":"2024-03-20T06:29:59.935610097Z"}
{"log":"runtime.main\n","stream":"stdout","time":"2024-03-20T06:29:59.935633418Z"}
{"log":"\u0009runtime/proc.go:271\n","stream":"stdout","time":"2024-03-20T06:29:59.93565774Z"}
```

需要在日志转发器中先合并相同的 log，再将对应的 log 进行格式化，这里仍然使用 `logging-operator-test-receiver` 测试

```yaml
apiVersion: logging.banzaicloud.io/v1beta1
kind: Flow
metadata:
  name: test-logging
spec:
  match:
  - select:
      labels:
        logging: golang
  filters:
  - concat:
      key: log
      multiline_start_regexp: '/^\d{4}-\d{2}-\d{2}/'
      multiline_end_regexp: '/\Z/'
      separator: ''
      flush_interval: 5
  localOutputRefs:
    - test-receiver
---
apiVersion: logging.banzaicloud.io/v1beta1
kind: Output
metadata:
  name: test-receiver
spec:
  http:
    endpoint: http://logging-operator-test-receiver:8080
    content_type: application/json
    buffer:
      type: memory
      tags: time
      timekey: 1s
      timekey_wait: 0s
```

运行之后发现，日志已经被合并到了 `key` 为 `log` 的部分

```text
[0] http.0: [[1710923000.932884258, {}], {"log"=>"2024-03-20 08:23:14.937 ERROR log_test/main.go:19 main.main 日志测试 2024-03-20 08:23:14.937
main.main
        D:/GolandProjects/log_test/main.go:19
runtime.main
        C:/Program Files/Go/src/runtime/proc.go:271
", "stream"=>"stdout", "time"=>"2024-03-20T08:23:14.937900447Z", "kubernetes"=>{"pod_name"=>"print-logs-64fb98db85-c4zdz", "namespace_name"=>"default", "pod_id"=>"d72c4042-cb45-4fe1-a684-eb4e56861049", "labels"=>{"app"=>"print-logs", "logging"=>"golang", "pod-template-hash"=>"64fb98db85"}, "annotations"=>{"cni.projectcalico.org/containerID"=>"24c125288decf36a19b5d333569258a45b2f41b2dfbbef407854145234ec7323", "cni.projectcalico.org/podIP"=>"10.244.166.182/32", "cni.projectcalico.org/podIPs"=>"10.244.166.182/32"}, "host"=>"node1", "container_name"=>"print-logs", "docker_id"=>"ffb6313adb1347f828348f128852ba110f608124f3a8aca68b400f238bfde71a","container_hash"=>"print-test-log@sha256:ad1a3e5bb60d81a6b13e8085c618244055c35807e7a05caabf50f77adc7a11e0", "container_image"=>"print-test-log"}}]
```

{{< admonition type="danger" open=true >}}
如果使用的容器运行时为 containerd，则需要对应的日志为：

```text
2024-04-10T02:15:17.527436711Z stdout F 2024-04-10 02:15:17.527 ERROR log_test/main.go:19 main.main 日志测试 2024-04-10 02:15:17.527
2024-04-10T02:15:17.527496993Z stdout F main.main
2024-04-10T02:15:17.527501224Z stdout F         D:/GolandProjects/log_test/main.go:19
2024-04-10T02:15:17.527503316Z stdout F runtime.main
2024-04-10T02:15:17.527505153Z stdout F         C:/Program Files/Go/src/runtime/proc.go:271
```

这时，fluentbit 会将后面的部分收集到 key 为 message 中，需要修改 Flow 的合并配置合并的 key 为 message

```yaml
  filters:
  - concat:
      key: message
```

{{< /admonition >}}

## 日志的字段格式化

```text
2024-03-20 06:31:44.934 ERROR log_test/main.go:19 main.main 日志测试 2024-03-20 06:31:44.934
main.main
        log_test/main.go:19
runtime.main
        runtime/proc.go:271
```

使用上述日志格式的作为参考编写正则表达式，将我们的数据使用正则命令将其拆分为如下部分

- time：2024-03-20 08:23:14.937
- loglevel：ERROR
- line：log_test/main.go:19
- func：main.main
- log部分保持不变

正则表达式如下：(正则测试网站：[https://regex101.com/](https://regex101.com/))

```text
/^(?<time>\d{4}-\d{2}-\d{2} \d{2}:\d{2}:\d{2}.\d{3})(\s+)(?<loglevel>\w+)(\s+)(?<line>[\w\.\:\/\-]+)(\s+)(?<func>[\w\.\:\/\-]+)/
```

配置日志字段格式化

```yaml
apiVersion: logging.banzaicloud.io/v1beta1
kind: Flow
metadata:
  name: test-logging
spec:
  match:
  - select:
      labels:
        logging: golang
  filters:
  - concat:
      key: log
      multiline_start_regexp: '/^\d{4}-\d{2}-\d{2}/'
      multiline_end_regexp: '/\Z/'
      separator: ''
      flush_interval: 5
  - parser:
      remove_key_name_field: false
      reserve_data: true
      parse:
        type: multi_format
        patterns:
        - format: regexp
          expression: /^(?<time>\d{4}-\d{2}-\d{2} \d{2}:\d{2}:\d{2}.\d{3})(\s+)(?<loglevel>\w+)(\s+)(?<line>[\w\.\:\/\-]+)(\s+)(?<func>[\w\.\:\/\-]+)/
  localOutputRefs:
    - test-receiver
---
apiVersion: logging.banzaicloud.io/v1beta1
kind: Output
metadata:
  name: test-receiver
spec:
  http:
    endpoint: http://logging-operator-test-receiver:8080
    content_type: application/json
    buffer:
      type: memory
      tags: time
      timekey: 1s
      timekey_wait: 0s
```

运行效果：

  [0] http.0: [[1710928830.558128299, {}], {"log"=>"2024-03-20 10:00:24.938 ERROR log_test/main.go:19 main.main 日志测试 2024-03-20 10:00:24.938
  main.main
    D:/GolandProjects/log_test/main.go:19
  runtime.main
    C:/Program Files/Go/src/runtime/proc.go:271
  ", "stream"=>"stdout", "time"=>"2024-03-20T10:00:24.939195916Z", "kubernetes"=>{"pod_name"=>"print-logs-64fb98db85-c4zdz", "namespace_name"=>"default", "pod_id"=>"d72c4042-cb45-4fe1-a684-eb4e56861049", "labels"=>{"app"=>"print-logs", "logging"=>"golang", "pod-template-hash"=>"64fb98db85"}, "annotations"=>{"cni.projectcalico.org/containerID"=>"24c125288decf36a19b5d333569258a45b2f41b2dfbbef407854145234ec7323", "cni.projectcalico.org/podIP"=>"10.244.166.182/32", "cni.projectcalico.org/podIPs"=>"10.244.166.182/32"}, "host"=>"node1", "container_name"=>"print-logs", "docker_id"=>"ffb6313adb1347f828348f128852ba110f608124f3a8aca68b400f238bfde71a", "container_hash"=>"print-test-log@sha256:ad1a3e5bb60d81a6b13e8085c618244055c35807e7a05caabf50f77adc7a11e0", "container_image"=>"print-test-log"}, "loglevel"=>"ERROR", "line"=>"log_test/main.go:19", "func"=>"main.main"}]

## 删除不需要的字段

看上述日志发现其中有很多数据我们都不需要，可以使用 `record_transformer` 删除不需要的字段

```yaml
apiVersion: logging.banzaicloud.io/v1beta1
kind: Flow
metadata:
  name: test-logging
spec:
  match:
  - select:
      labels:
        logging: golang
  filters:
  - concat:
      key: log
      multiline_start_regexp: '/^\d{4}-\d{2}-\d{2}/'
      multiline_end_regexp: '/\Z/'
      separator: ''
      flush_interval: 5
  - parser:
      remove_key_name_field: false
      reserve_data: true
      parse:
        type: multi_format
        patterns:
        - format: regexp
          expression: /^(?<time>\d{4}-\d{2}-\d{2} \d{2}:\d{2}:\d{2}.\d{3})(\s+)(?<loglevel>\w+)(\s+)(?<line>[\w\.\:\/\-]+)(\s+)(?<func>[\w\.\:\/\-]+)/
  - record_transformer:
      remove_keys: '$.kubernetes.pod_id,$.kubernetes.annotations,$.kubernetes.labels,$.kubernetes.docker_id,$.kubernetes.container_hash,$.kubernetes.container_image'
  localOutputRefs:
    - test-receiver
---
apiVersion: logging.banzaicloud.io/v1beta1
kind: Output
metadata:
  name: test-receiver
spec:
  http:
    endpoint: http://logging-operator-test-receiver:8080
    content_type: application/json
    buffer:
      type: memory
      tags: time
      timekey: 1s
      timekey_wait: 0s
```

# 日志输出到 ElasticSearch

## ElasticSearch OutPut 配置

在 default 命名空间创建 elastic 密码的 `secret`

```bash
kubectl create secret generic elastic-password --from-literal=password='密码'
```

```yaml
apiVersion: logging.banzaicloud.io/v1beta1
kind: Flow
metadata:
  name: test-logging
spec:
  match:
  - select:
      labels:
        logging: golang
  filters:
  - concat:
      key: log
      multiline_start_regexp: '/^\d{4}-\d{2}-\d{2}/'
      multiline_end_regexp: '/\Z/'
      separator: ''
      flush_interval: 5
  - parser:
      remove_key_name_field: false
      reserve_data: true
      parse:
        type: multi_format
        patterns:
        - format: regexp
          expression: /^(?<time>\d{4}-\d{2}-\d{2} \d{2}:\d{2}:\d{2}.\d{3})(\s+)(?<loglevel>\w+)(\s+)(?<line>[\w\.\:\/\-]+)(\s+)(?<func>[\w\.\:\/\-]+)/
  - record_transformer:
      remove_keys: '$.kubernetes.pod_id,$.kubernetes.annotations,$.kubernetes.labels,$.kubernetes.docker_id,$.kubernetes.container_hash,$.kubernetes.container_image'
  localOutputRefs:
    - elastic-receiver
---
apiVersion: logging.banzaicloud.io/v1beta1
kind: Output
metadata:
  name: elastic-receiver
spec:
  elasticsearch:
    host: 10.0.16.2
    port: 9200
    logstash_format: true
    logstash_prefix: my-test
    scheme: http
    user: elastic
    password:
      valueFrom:
        secretKeyRef:
          name: elastic-password
          key: password
    buffer:
      timekey: 1m
      timekey_wait: 30s
      timekey_use_utc: true
```

## 使用动态的索引名称

如果需要配置动态索引名称，需要在 buffer 中添加对应的 key 值，如我们需要在索引名称中添加命名空间与容器名称，配置如下：

```yaml
apiVersion: logging.banzaicloud.io/v1beta1
kind: Flow
metadata:
  name: test-logging
spec:
  match:
  - select:
      labels:
        logging: golang
  filters:
  - concat:
      key: log
      multiline_start_regexp: '/^\d{4}-\d{2}-\d{2}/'
      multiline_end_regexp: '/\Z/'
      separator: ''
      flush_interval: 5
  - parser:
      remove_key_name_field: false
      reserve_data: true
      parse:
        type: multi_format
        patterns:
        - format: regexp
          expression: /^(?<time>\d{4}-\d{2}-\d{2} \d{2}:\d{2}:\d{2}.\d{3})(\s+)(?<loglevel>\w+)(\s+)(?<line>[\w\.\:\/\-]+)(\s+)(?<func>[\w\.\:\/\-]+)/
  - record_transformer:
      remove_keys: '$.kubernetes.pod_id,$.kubernetes.annotations,$.kubernetes.labels,$.kubernetes.docker_id,$.kubernetes.container_hash,$.kubernetes.container_image'
  localOutputRefs:
    - elastic-receiver
---
apiVersion: logging.banzaicloud.io/v1beta1
kind: Output
metadata:
  name: elastic-receiver
spec:
  elasticsearch:
    host: 10.0.16.2
    port: 9200
    logstash_format: true
    logstash_prefix: my-test-${$.kubernetes.namespace_name}-${$.kubernetes.container_name}
    scheme: http
    user: elastic
    password:
      valueFrom:
        secretKeyRef:
          name: elastic-password
          key: password
    buffer:
      tags: tag,time,$.kubernetes.namespace_name,$.kubernetes.container_name
      timekey: 1m
      timekey_wait: 30s
      timekey_use_utc: true
```

## ElasticSearch 使用数据流模式

```yaml
apiVersion: logging.banzaicloud.io/v1beta1
kind: Output
metadata:
  name: elastic-receiver
spec:
  elasticsearch:
    host: 10.0.16.2
    port: 9200
    logstash_format: false
    index_name: my-test-${$.kubernetes.namespace_name}-${$.kubernetes.container_name}
    include_timestamp: true
    data_stream_enable: true
    data_stream_name: my-test-${$.kubernetes.namespace_name}-${$.kubernetes.container_name}
    data_stream_ilm_name: my-test
    data_stream_template_name: my-test
    scheme: https
    ssl_verify: false
    ssl_version: TLSv1_2
    user: elastic
    log_es_400_reason: true
    default_elasticsearch_version: "8.10.4"
    password:
      valueFrom:
        secretKeyRef:
          name: elastic-password
          key: password
    buffer:
      tags: tag,time,$.kubernetes.namespace_name,$.kubernetes.container_name
      timekey: 1m
      timekey_wait: 1m
      timekey_use_utc: true
```


---

> 作者: [komorebi](https://blog.2nfree.com/)  
> URL: https://blog.2nfree.com/posts/kubernetes-logging-collection-2/  

