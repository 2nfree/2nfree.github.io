# Spark Operator


# 前言

在 Kubernetes 中运行 Spark 有很多种方式

如直接部署 Deployment 运行 Spark 集群容器（这种方式相当于只是使用 Kubernetes 托管了运行 Spark 集群的容器，本身 Kubernetes 与 Spark 集群无任何交互）

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name:  spark-cluster-master
  namespace: spark
  labels:
    app:  spark-cluster-master
spec:
  selector:
    matchLabels:
      app: spark-cluster-master
  replicas: 1
  template:
    metadata:
      labels:
        app:  spark-cluster-master
    spec:
      securityContext:
        runAsUser: 0
        runAsGroup: 0
        fsGroup: 0
      containers:
      - name:  spark-cluster-master
        image:  apache/spark:v3.1.3
        workingDir: /opt/spark
        command: ["/bin/bash", "-c", "/opt/spark/sbin/start-master.sh && tail -f /opt/spark/logs/*.out"]
        livenessProbe:
          tcpSocket:
            port: 8080
          initialDelaySeconds: 30
          periodSeconds: 60
        resources:
          requests:
            cpu: 1000m
            memory: 2Gi
          limits:
            cpu: 1000m
            memory: 2Gi
        ports:
        - containerPort: 8080
          name: ui
        - containerPort: 7077
          name: rpc
      restartPolicy: Always
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name:  spark-cluster-worker
  namespace: spark
  labels:
    app:  spark-cluster-worker
spec:
  selector:
    matchLabels:
      app: spark-cluster-worker
  replicas: 1
  template:
    metadata:
      labels:
        app:  spark-cluster-worker
    spec:
      securityContext:
        runAsUser: 0
        runAsGroup: 0
        fsGroup: 0
      containers:
      - name:  spark-cluster-worker
        image:  apache/spark:v3.1.3
        workingDir: /opt/spark
        command: ["/bin/bash", "-c", "/opt/spark/sbin/start-worker.sh spark://spark-cluster-master.spark:7077 && tail -f /opt/spark/logs/*.out"]
        resources:
          requests:
            cpu: 2000m
            memory: 4Gi
          limits:
            cpu: 2000m
            memory: 4Gi
      restartPolicy: Always
---
apiVersion: v1
kind: Service
metadata:
  name: spark-cluster-master
  namespace: spark
spec:
  selector:
    app: spark-cluster-master
  type: NodePort
  ports:
  - name: rpc
    port: 7077
  - name: ui
    port: 8080

```

或者是用 spark-submit，官方文档地址：[spark-on-k8s](https://spark.apache.org/docs/latest/running-on-kubernetes.html)

但是对于以上方式来说，我还是更喜欢使用 [spark-operator](https://github.com/GoogleCloudPlatform/spark-on-k8s-operator)

# Spark Operator 介绍

{{< admonition type="info" open=true >}}
在 Spark Operator 中，Spark Job 属于一次性的执行任务，即执行完毕后会销毁容器，Spark-Operator 提供了两种关于 Spark Job 的 CRD：
SparkApplication 和 ScheduledSparkApplication
其中 SparkApplication 为一次性任务，即本次运行完毕后，将会不再运行
ScheduledSparkApplication 为定时任务，会根据对应配置的时间，定时启动 SparkApplication
{{< /admonition >}}

# 安装 Spark Operator

安装 Spark Operator 我们可以通过官方提供的 helm charts 进行安装

```bash
# 添加 helm repo
helm repo add spark-operator https://googlecloudplatform.github.io/spark-on-k8s-operator

# 安装
helm install spark-operator spark-operator/spark-operator \
  --namespace spark \
  --create-namespace \
  --set metrics.enable=true \
  --set webhook.enable=true
```

{{< admonition type="tip" open=true >}}
PS：
默认情况下 Operator 将会监视并处理每个命名空间中的 CRD，如果想要限制对应的命名空间，可以添加 --set sparkJobNamespace=namespace 设置
--set webhook.enable=true 是为了开启 mutating admission webhook，使 CRD 具有额外的功能，例如可以将 ConfigMap 挂载到 SparkApplication 和 ScheduledSparkApplication
{{< /admonition >}}

# Spark CRD 配置示例

## SparkApplication

```yaml
apiVersion: "sparkoperator.k8s.io/v1beta2"
kind: SparkApplication
metadata:
  name: spark-app
  namespace: spark
spec:
  type: Scala
  mode: cluster
  image: "your.repo.com/spark/spark-test:v1" # 配置 spark 的镜像，镜像的构建需要 apache/spark 镜像 + jar 包进行构建
  imagePullPolicy: Always
  imagePullSecrets: # 这里配置镜像仓库对应的 Secret
    - your-repo-secret
  mainClass: com.fs.MySpark # spark jar 包的 main 执行类
  mainApplicationFile: "local:///opt/spark/examples/jars/my-spark.jar" # spark jar包所在位置
  sparkVersion: "3.1.3" # spark 对应的版本
  restartPolicy:
    type: Never
  driver: # driver 对应的配置
    cores: 1
    memory: "4096m"
    labels:
      version: 3.1.3
    serviceAccount: spark
    env:
    - name: TZ
      value: "Asia/Shanghai"
    volumeMounts:
    - name: spark-properties # 之前使用 helm 安装 operator 时配置了 --set webhook.enable=true 的话，就可以配置挂载 ConfigMap
      mountPath: /opt/spark/examples/jars/conf
  executor: # executor 对应的配置
    cores: 1
    instances: 1
    memory: "4g"
    memoryOverhead: "4g"
    env:
    - name: TZ
      value: "Asia/Shanghai"
    labels:
      version: 3.1.3
    volumeMounts:
    - name: spark-properties
      mountPath: /opt/spark/examples/jars/conf
  volumes:
    - name: spark-properties
      configMap:
        name: spark-properties
```

## ScheduledSparkApplication

```yaml
apiVersion: "sparkoperator.k8s.io/v1beta2"
kind: ScheduledSparkApplication
metadata:
  name: spark-scheduled-app
  namespace: spark
spec:
  schedule: "00 16 * * *" # 定时配置
  concurrencyPolicy: Allow # 允许并发
  successfulRunHistoryLimit: 1 # 成功的保存历史次数
  failedRunHistoryLimit: 3  # 失败的保存历史次数
  template: # 这里的模板配置与 SparkApplication 一致
    type: Scala
    mode: cluster
    image: "your.repo.com/spark/spark-test:v1"
    imagePullPolicy: Always
    imagePullSecrets:
      - your-repo-secret
    mainClass: com.fs.MySpark
    mainApplicationFile: "local:///opt/spark/examples/jars/my-spark.jar"
    sparkVersion: "3.1.1"
    restartPolicy:
      type: Never
    driver:
      ...
    executor:
      ...
```

# 构建 Spark Job 容器

## 打包 Jar

```bash
mvn clean package -Dfile.encoding=UTF-8 -DskipTests=true
```

## 构建容器

```Dockerfile
FROM apache/spark:v3.1.3
ARG JAR_FILE
ENV JAR_FILE=${JAR_FILE}
USER root
COPY ./target/$JAR_FILE /opt/spark/examples/jars
RUN chmod -R +r /opt/spark/examples/jars
USER 185
```

```bash
# 找到 target 文件夹下，带有 with-dependencies.jar 的 jar 包
docker build --build-arg JAR_FILE=`ls ./target/*with-dependencies.jar | cut -d '/' -f3"` -t your.repo.com/spark/myspark-app:v1 .
```

# 接入 Prometheus

## 配置 Service

```yaml
kind: Service
apiVersion: v1
metadata:
  labels:
    app.kubernetes.io/instance: spark-operator
    app.kubernetes.io/name: spark-operator
  name: spark-operator-metrics-service
  namespace: spark
spec:
  ports: 
    - name: http-metrics
      port: 10254
      protocol: TCP
      targetPort: 10254
  selector:
    app.kubernetes.io/instance: spark-operator
    app.kubernetes.io/name: spark-operator
```

## 配置 ServiceMonitor

```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: spark-operator-metrics
  namespace: monitoring
  labels:
    app.kubernetes.io/instance: spark-operator
    app.kubernetes.io/name: spark-operator
spec:
  endpoints:
  - port: http-metrics
    interval: 10s 
  selector:
    matchLabels:
      app.kubernetes.io/instance: spark-operator
      app.kubernetes.io/name: spark-operator
  namespaceSelector:
    matchNames:
    - spark
```


---

> 作者: [komorebi](https://blog.2nfree.com/)  
> URL: https://blog.2nfree.com/posts/spark-operator/  

