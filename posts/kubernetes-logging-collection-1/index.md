# Kubernetes 日志收集（一）- 基于 SideCar 模式的日志收集


# 架构图

{{< image src="kubernetes-logging-stack.webp" >}}

# FluentBit 配置

我们规定所有的日志都输出到 /applogs 下

## fluent-bit.conf

```ini
[SERVICE]
  Flush             1
  Log_Level         info
  Daemon            off
  HTTP_Server       off
  Parsers_File      parsers.conf

#设置输入日志位置及tag
[INPUT]
  Name               tail
  tag                kube.*
  path               /applogs/*.log
  Exclude_Path       /applogs/gc.log
  Multiline          On
  Parser_Firstline   java_log
  DB                 /var/log/flb_logs.db
  Buffer_Max_Size    5MB
  Mem_Buf_Limit      5MB
  Buffer_Chunk_Size  100KB
  Refresh_Interval   5
  Ignore_Older       10s
  Rotate_Wait        5

[INPUT]
  Name               tail
  tag                gc.*
  path               /applogs/gc.log
  Parser             java_log
  DB                 /var/log/flb_logs.db
  Buffer_Max_Size    5MB
  Mem_Buf_Limit      5MB
  Refresh_Interval   5
  Ignore_Older       10s
  Rotate_Wait        5

#添加相关K8S环境字段，通过环境变量传入
[FILTER]
  Name               record_modifier
  Match              *
  Record             app ${APP}
  Record             namespace ${POD_NAMESPACE}
  Record             nodename ${NODE_NAME}

#es采集配置 
[OUTPUT]
  Name               es
  Match              kube.*
  Host               elastic.example.com
  Port               9200
  Index              k8s-java-${APP}
  Replace_Dots       On
  Retry_Limit        2
  Buffer_Size        1MB
  tls                On
  tls.verify         Off
  HTTP_User          elastic
  HTTP_Passwd        password
  Suppress_Type_Name On

[OUTPUT]
  Name               es
  Match              gc.*
  Host               elastic.example.com
  Port               9200
  Index              k8s-gc-${APP}
  Replace_Dots       On
  Retry_Limit        2
  Buffer_Size        1MB
  tls                On
  tls.verify         Off
  HTTP_User          elastic
  HTTP_Passwd        password
  Suppress_Type_Name On
```

## parsers.conf

```ini
[PARSER]
  Name               java_log
  Format             regex
  Regex              /^(?<time>\d+-\d+-\d+\s\S+)(\s+)(?<loglevel>\w+)(\s+)\[(?<thread>[^\]]+)\](\s+)\[(?<caller>[^\]]+)\](\s+)-(\s+)(?<message>.*)/
  Time_Key           time
  Time_Format        %Y-%m-%d %H:%M:%S.%L
  Time_Keep          On
```

# Kubernetes 配置

## ConfigMap

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: java-logging-config
data:
  fluent-bit.conf: |
    [SERVICE]
      Flush             1
      Log_Level         info
      Daemon            off
      HTTP_Server       off
      Parsers_File      parsers.conf
    
    [INPUT]
      Name               tail
      tag                kube.*
      path               /applogs/*.log
      Exclude_Path       /applogs/gc.log
      Multiline          On
      Parser_Firstline   java_log
      DB                 /var/log/flb_logs.db
      Buffer_Max_Size    5MB
      Mem_Buf_Limit      5MB
      Buffer_Chunk_Size  100KB
      Refresh_Interval   5
      Ignore_Older       10s
      Rotate_Wait        5
    
    [INPUT]
      Name               tail
      tag                gc.*
      path               /applogs/gc.log
      Parser             java_log
      DB                 /var/log/flb_logs.db
      Buffer_Max_Size    5MB
      Mem_Buf_Limit      5MB
      Refresh_Interval   5
      Ignore_Older       10s
      Rotate_Wait        5
    
    [FILTER]
      Name               record_modifier
      Match              *
      Record             app ${APP}
      Record             namespace ${POD_NAMESPACE}
      Record             nodename ${NODE_NAME}
    
    [OUTPUT]
      Name               es
      Match              kube.*
      Host               elastic.example.com
      Port               9200
      Index              k8s-java-${APP}
      Replace_Dots       On
      Retry_Limit        2
      Buffer_Size        1MB
      tls                On
      tls.verify         Off
      HTTP_User          elastic
      HTTP_Passwd        password
      Suppress_Type_Name On
    
    [OUTPUT]
      Name               es
      Match              gc.*
      Host               elastic.example.com
      Port               9200
      Index              k8s-gc-${APP}
      Replace_Dots       On
      Retry_Limit        2
      Buffer_Size        1MB
      tls                On
      tls.verify         Off
      HTTP_User          elastic
      HTTP_Passwd        password
      Suppress_Type_Name On

  parsers.conf: |
    [PARSER]
      Name               java_log
      Format             regex
      Regex              /^(?<time>\d+-\d+-\d+\s\S+)(\s+)(?<loglevel>\w+)(\s+)\[(?<thread>[^\]]+)\](\s+)\[(?<caller>[^\]]+)\](\s+)-(\s+)(?<message>.*)/
      Time_Key           time
      Time_Format        %Y-%m-%d %H:%M:%S.%L
      Time_Keep          On
```

## Deployment

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-java-application
  labels:
    app: my-java-application
spec:
  replicas: 1
  revisionHistoryLimit: 3
  selector:
    matchLabels:
      app: my-java-application
  template:
    metadata:
      labels:
        app: my-java-application
    spec:
      #原始应用容器
      containers:
        - name: my-java-application
          image: my-java-application:test
          imagePullPolicy: IfNotPresent
          ports:
            - containerPort: 8080
          #日志所在卷挂载
          volumeMounts:
          - mountPath: /applogs
            name: log-volume
        #fluentbit sidecar
        - name: fluentbit-logger
          image: fluent/fluent-bit:1.9
          imagePullPolicy: IfNotPresent
          #传入k8s相关元数据到环境变量
          env:
            - name: NODE_NAME
              valueFrom:
                fieldRef:
                  fieldPath: spec.nodeName
            - name: POD_NAMESPACE
              valueFrom:
                fieldRef:
                  fieldPath: metadata.namespace
            - name: APP
              valueFrom:
                fieldRef:
                  fieldPath: metadata.labels['app']
          #配置和日志卷挂载
          volumeMounts:
          - mountPath: /fluent-bit/etc
            name: config
          - mountPath: /applogs
            name: log-volume
      volumes:
        - name: config
          configMap:
            name: java-logging-config
        - name: log-volume
          emptyDir: {}
```


---

> 作者: [komorebi](https://blog.2nfree.com/)  
> URL: https://blog.2nfree.com/posts/kubernetes-logging-collection-1/  

