# Kubernetes 日志收集（四）- 基于 Fluent-Bit Daemonset 的日志收集


# 架构图

{{< image src="fluentbit-daemonset.webp" >}}

- 使用 DaemonSet 模式部署 Fluent Bit，使每个 Kubernetes 集群节点都运行一个 Fluent-Bit 容器
- 将 Kubernetes 容器运行时的日志文件夹挂载到 Fluent-Bit 容器中以供采集
- 通过 Kubernetes 提供的相关接口获取采集日志的相关元数据

# 配置方法

## 容器运行时日志解析

通过 Fluent-Bit 的多行日志分析进行日志解析与合并，相关配置如下

```ini
    [INPUT]
      name             tail
      tag              kube.*
      db               /tail-db/tail-containers-state.db
      db.locking       true
      mem_buf_limit    10mb
      multiline.parser multiline_docker
      path             /var/log/containers/*.log
      refresh_interval 5

    [FILTER]
      name             kubernetes
      match            kube.*
      buffer_size      0
      kube_token_ttl   600
      kube_url         https://kubernetes.default.svc:443
      use_kubelet      off
      merge_log        on
```

在该配置中通过 tail input 配置获取容器的日志，并通过 kubernetes filter 插件对日志添加 k8s 元数据

```ini
    [MULTILINE_PARSER]
      name          multiline_docker
      parser        docker
      key_content   log
      type          regex
      flush_timeout 2000
      rule          "start_state"  "/^\d{4}-\d{2}-\d{2} \d{2}:\d{2}:\d{2}(\.|\,)\d{3}(\s*)\w+/"  "cont_state"
      rule          "cont_state"  "/^(?!\d{4}-\d{2}-\d{2} \d{2}:\d{2}:\d{2}(\.|\,)\d{3}\s+\w+).*/"  "cont_state"

    [PARSER]
      Name   json
      Format json
      Time_Key time
      Time_Format %Y-%m-%dT%H:%M:%S.%LZ

    [PARSER]
      name        docker
      format      json
      decode_field_as  escaped_utf8  log  do_next
```

在该配置中定义了多行匹配规则 multiline_docker，匹配逻辑为：
先对原始容器日志进行 docker 的匹配规则解析（如果使用的是 containerd，需要将 docker json格式修改为对应的 cri 格式）
开始部分 -》 开始部分正则匹配（这里的正则匹配的是时间戳+日志等级的格式）-》合并部分
合并部分 -》 合并规则正则匹配（这里的正则匹配的是非时间戳+日志等级的格式）-》合并部分
如下格式类型

```text
2024-05-22 02:29:29.781 ERROR log_test/main.go:64 main.main 打印测试日志： 2024-05-22 02:29:29.781
main.main
        D:/GolandProjects/log_test/main.go:64
runtime.main
        C:/Program Files/Go/src/runtime/proc.go:271
2024-05-22 02:29:29.781 ERROR log_test/main.go:65 main.main
        select
        m.id,
        m.parent_id,
        m.menu_type,
        .....
```

其中每部分日志的开头都为 “2024-05-22 02:29:29.781 ERROR”，而其余部分都为其他多行日志，那么在日志匹配规则下，“2024-05-22 02:29:29.781 ERROR” 该格式会视为所有日志的开头部分

```text
main.main
        D:/GolandProjects/log_test/main.go:64
runtime.main
        C:/Program Files/Go/src/runtime/proc.go:271
```

此部分视为日志延续行内容直到匹配到下一个 “2024-05-22 02:29:29.781 ERROR” 部分对日志进行拆分

## 标签选择器

因为基于 DaemonSet 的模式采集了 K8S 集群所有的日志，所以需要使用标签选择器对不同的标签进行分类，保留需要的日志，舍弃不需要的日志，配置如下：

```ini
    [FILTER]
      name      lua
      match     kube.*
      script    scripts.lua
      call      label_filter

    [FILTER]
      name      rewrite_tag
      match     kube.*
      rule      $log_tag ^(.*)$ $log_tag true

    [FILTER]
      name      lua
      match     *
      script    scripts.lua
      call      remove_key_field
```

```lua
    function label_filter(tag, timestamp, record)
        if record["kubernetes"] == nil then
            return 0, timestamp, record
        end
    
        local namespace = record["kubernetes"]["namespace_name"]
        local labels = record["kubernetes"]["labels"]
        local app = record["kubernetes"]["labels"]["app"]
        local log_tag = nil
    
        if labels ~= nil then
            if labels["logging"] == "java" then
                log_tag = string.format("java.%s.%s", namespace, app)
            elseif labels["component"] == "jobmanager" then
                log_tag = string.format("flink.jobmanager.%s.%s", namespace, app)
            elseif labels["component"] == "taskmanager" then
                log_tag = string.format("flink.taskmanager.%s.%s", namespace, app)
            elseif labels["logging"] == "nginx" then
                log_tag = string.format("nginx.%s", namespace)
            end
        end
    
        if log_tag then
            record["log_tag"] = log_tag
            return 2, timestamp, record
        else
            return 0, timestamp, record
        end
    end
    
    function remove_key_field(tag, timestamp, record)
        fields_to_remove = {"stream", "kubernetes.pod_id", "kubernetes.docker_id", "kubernetes.container_hash",
                            "kubernetes.container_image", "kubernetes.labels", "kubernetes.annotations"}
        for _, field in ipairs(fields_to_remove) do
            local keys = {}
            for key in string.gmatch(field, "[^.]+") do
                table.insert(keys, key)
            end
            local current = record
            for i = 1, #keys - 1 do
                current = current[keys[i]]
                if not current then
                    break
                end
            end
            if current then
                current[keys[#keys]] = nil
            end
        end
        return 1, timestamp, record
    end
```

通过 lua 脚本获取不同的 label 内容，```label_filter```根据 label 内容分配不同的 日志 tag，```remove_key_field```清理部分不需要的 Kubernetes 元数据

## 根据修改后的 tag，进行日志的正则分析匹配

```ini
    [FILTER]
      name      parser
      match     java.*
      key_name  log
      parser    java_log
      reserve_data on
    
    [FILTER]
      name      parser
      match     flink.*
      key_name  log
      parser    flink
      reserve_data on

    [FILTER]
      name      parser
      match     nginx.*
      key_name  log
      parser    json
      reserve_data on

    [PARSER]
      name              java_log
      format            regex
      regex             /^(?<time>\d{4}-\d{2}-\d{2} \d{2}:\d{2}:\d{2}.\d{3})(\s*)(?<loglevel>\w*)(\s*)\[(?<trace_id>\S*)\]\[[\w\-]*;(?<thread>[^\[\]\;]*);(?<span_id>\S*)\](\s*)(?<caller>\S*)(?<message>[\S\s]*)/
      time_key          time
      time_format       %Y-%m-%d %H:%M:%S.%L
      time_keep         On
      time_offset       +0800

    [PARSER]
      name              flink
      format            regex
      regex             /^(?<time>\d{4}-\d{2}-\d{2} \d{2}:\d{2}:\d{2},\d{3})(\s+)(?<loglevel>\w+)(\s+)(?<classname>[\w\.]+)(?<message>[\S\s]*)/
      time_key          time
      time_format       %Y-%m-%d %H:%M:%S,%L
      time_keep         On
```

这里的正则 PARSER 需要根据自己的需要来配置

## 传输日志到 ES 或其他

```ini
    [OUTPUT]
      name              es
      match             flink.jobmanager.*
      host              【ES HOST】
      port              9200
      index             flink-jobmanager
      replace_dots      on
      retry_limit       2
      buffer_size       10mb
      tls               on
      tls.verify        off
      http_user         elastic
      http_passwd       【密码】
      suppress_type_name on
      trace_error       on

    [OUTPUT]
      name      http
      match     java.*
      host      log-alert.logging.svc.cluster.local
      port      7000
      uri       /api/v1/receiver
      format    json
```

# 完整配置

## DaemonSet

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: fluent-bit
  namespace: logging
spec:
  revisionHistoryLimit: 3
  selector:
    matchLabels:
      app: fluent-bit
  template:
    metadata:
      labels:
        app: fluent-bit
    spec:
      serviceAccountName: fluent-bit
      tolerations:
      - key: "node-role.kubernetes.io/master"
        operator: "Exists"
        effect: "NoSchedule"
      containers:
      - name: fluentbit-logger
        image: fluent/fluent-bit:3.0.3
        imagePullPolicy: IfNotPresent
        volumeMounts:
          - mountPath: /fluent-bit/etc
            name: config
          - mountPath: /var/log
            name: pod-volume
            readOnly: true
          - mountPath: /data/docker/containers
            name: docker-volume
            readOnly: true
          - mountPath: /tail-db
            name: positiondb
          - mountPath: /buffers
            name: buffers
          - mountPath: /var/run/secrets/kubernetes.io/serviceaccount
            name: k8s-certs
            readOnly: true
      volumes:
      - name: pod-volume
        hostPath:
          path: /var/log
      - name: docker-volume
        hostPath:
          path: /data/docker/containers # 这里配置对应的容器运行时数据文件夹
      - name: config
        configMap:
          name: fluent-bit-config
      - name: buffers
        emptyDir: {}
      - name: positiondb
        emptyDir: {}
      - name: k8s-certs
        projected:
          sources:
          - serviceAccountToken:
              path: token
              expirationSeconds: 7200
          - configMap:
              name: kube-root-ca.crt
              items:
              - key: ca.crt
                path: ca.crt
```

## ConfigMap

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: fluent-bit-config
  namespace: logging
data:
  fluent-bit.conf: |
    [SERVICE]
      flush            1
      grace            5
      daemon           off
      log_level        info
      parsers_file     /fluent-bit/etc/parsers.conf
      coro_stack_size  32768
      storage.path     /buffers
      http_server      off

    @INCLUDE tail-kubernetes-log.conf
    @INCLUDE logging-label-filter.conf
    @INCLUDE parser-java-log.conf
    @INCLUDE parser-flink-log.conf
    @INCLUDE parser-nginx-log.conf
    @INCLUDE es-output.conf
    @INCLUDE log-alert.conf

  tail-kubernetes-log.conf: |
    [INPUT]
      name             tail
      tag              kube.*
      db               /tail-db/tail-containers-state.db
      db.locking       true
      mem_buf_limit    10mb
      multiline.parser multiline_docker
      path             /var/log/containers/*.log
      refresh_interval 5

    [FILTER]
      name             kubernetes
      match            kube.*
      buffer_size      0
      kube_token_ttl   600
      kube_url         https://kubernetes.default.svc:443
      use_kubelet      off
      merge_log        on

    [FILTER]
      name      lua
      match     kube.*
      script    scripts.lua
      call      get_pod_ip

    [FILTER]
      name      lua
      match     kube.*
      script    scripts.lua
      call      get_app_name

  logging-label-filter.conf: |
    [FILTER]
      name      lua
      match     kube.*
      script    scripts.lua
      call      label_filter

    [FILTER]
      name      rewrite_tag
      match     kube.*
      rule      $log_tag ^(.*)$ $log_tag true

    [FILTER]
      name      lua
      match     *
      script    scripts.lua
      call      remove_key_field

  parser-java-log.conf: |
    [FILTER]
      name      parser
      match     java.*
      key_name  log
      parser    java_log
      reserve_data on

  parser-flink-log.conf: |
    [FILTER]
      name      parser
      match     flink.*
      key_name  log
      parser    flink
      reserve_data on

  parser-nginx-log.conf: |
    [FILTER]
      name      parser
      match     nginx.*
      key_name  log
      parser    json
      reserve_data on

  es-output.conf: |
    [OUTPUT]
      name              es
      match             flink.jobmanager.*
      host              【ES HOST】
      port              9200
      index             flink-jobmanager
      replace_dots      on
      retry_limit       2
      buffer_size       10mb
      tls               on
      tls.verify        off
      http_user         elastic
      http_passwd       【密码】
      suppress_type_name on
      trace_error       on

  log-alert.conf: |
    [OUTPUT]
      name      http
      match     java.*
      host      log-alert.logging.svc.cluster.local
      port      7000
      uri       /api/v1/receiver
      format    json

  parsers.conf: |
    [MULTILINE_PARSER]
      name          multiline_docker
      parser        docker
      key_content   log
      type          regex
      flush_timeout 2000
      rule          "start_state"  "/^\d{4}-\d{2}-\d{2} \d{2}:\d{2}:\d{2}(\.|\,)\d{3}(\s*)\w+/"  "cont_state"
      rule          "cont_state"  "/^(?!\d{4}-\d{2}-\d{2} \d{2}:\d{2}:\d{2}(\.|\,)\d{3}\s+\w+).*/"  "cont_state"

    [PARSER]
      Name        json
      Format      json
      Time_Key    time
      Time_Format %Y-%m-%dT%H:%M:%S.%LZ

    [PARSER]
      name        docker
      format      json
      decode_field_as  escaped_utf8  log  do_next

    [PARSER]
      name              java_log
      format            regex
      regex             /^(?<time>\d{4}-\d{2}-\d{2} \d{2}:\d{2}:\d{2}.\d{3})(\s*)(?<loglevel>\w*)(\s*)\[(?<trace_id>\S*)\]\[[\w\-]*;(?<thread>[^\[\]\;]*);(?<span_id>\S*)\](\s*)(?<caller>\S*)(?<message>[\S\s]*)/
      time_key          time
      time_format       %Y-%m-%d %H:%M:%S.%L
      time_keep         On
      time_offset       +0800

    [PARSER]
      name              flink
      format            regex
      regex             /^(?<time>\d{4}-\d{2}-\d{2} \d{2}:\d{2}:\d{2},\d{3})(\s+)(?<loglevel>\w+)(\s+)(?<classname>[\w\.]+)(?<message>[\S\s]*)/
      time_key          time
      time_format       %Y-%m-%d %H:%M:%S,%L
      time_keep         On

  scripts.lua: |
    function label_filter(tag, timestamp, record)
        if record["kubernetes"] == nil then
            return 0, timestamp, record
        end
    
        local namespace = record["kubernetes"]["namespace_name"]
        local labels = record["kubernetes"]["labels"]
        local app = record["kubernetes"]["labels"]["app"]
        local log_tag = nil
    
        if labels ~= nil then
            if labels["logging"] == "java" then
                log_tag = string.format("java.%s.%s", namespace, app)
            elseif labels["component"] == "jobmanager" then
                log_tag = string.format("flink.jobmanager.%s.%s", namespace, app)
            elseif labels["component"] == "taskmanager" then
                log_tag = string.format("flink.taskmanager.%s.%s", namespace, app)
            elseif labels["logging"] == "nginx" then
                log_tag = string.format("nginx.%s", namespace)
            end
        end
    
        if log_tag then
            record["log_tag"] = log_tag
            return 2, timestamp, record
        else
            return 0, timestamp, record
        end
    end
    
    function get_pod_ip(tag, timestamp, record)
        if record["kubernetes"]["annotations"] and record["kubernetes"]["annotations"]["cni.projectcalico.org/podIP"] then
            record["podIP"] = record["kubernetes"]["annotations"]["cni.projectcalico.org/podIP"]
            return 2, timestamp, record
        else
            return 0, timestamp, record
        end
    end
    
    function get_app_name(tag, timestamp, record)
        if record["kubernetes"]["labels"] and record["kubernetes"]["labels"]["app"] then
            record["app"] = record["kubernetes"]["labels"]["app"]
            return 2, timestamp, record
        else
            return 0, timestamp, record
        end
    end
    
    function remove_key_field(tag, timestamp, record)
        fields_to_remove = {"stream", "kubernetes.pod_id", "kubernetes.docker_id", "kubernetes.container_hash",
                            "kubernetes.container_image", "kubernetes.labels", "kubernetes.annotations"}
        for _, field in ipairs(fields_to_remove) do
            local keys = {}
            for key in string.gmatch(field, "[^.]+") do
                table.insert(keys, key)
            end
            local current = record
            for i = 1, #keys - 1 do
                current = current[keys[i]]
                if not current then
                    break
                end
            end
            if current then
                current[keys[#keys]] = nil
            end
        end
        return 1, timestamp, record
    end
```

## ServiceAccount

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: fluent-bit
  namespace: logging
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: fluent-bit-cluster-role
rules:
- apiGroups: [""]
  resources: ["pods", "namespaces"]
  verbs: ["get", "list", "watch"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: fluent-bit-cluster-role-binding
subjects:
- kind: ServiceAccount
  name: fluent-bit
  namespace: logging
roleRef:
  kind: ClusterRole
  name: fluent-bit-cluster-role
  apiGroup: rbac.authorization.k8s.io
```


---

> 作者: [komorebi](https://blog.2nfree.com/)  
> URL: https://blog.2nfree.com/posts/kubernetes-logging-collection-4/  

