# Kubernetes 日志收集（五）- 使用 Kafka 做中间层


# 前言

之前[基于 Fluent-bit Daemonset 的日志收集](/posts/kubernetes-logging-collection-4/)中，是由 fluent-bit 直接采集到 elasticsearch 中，当我们的日志量非常大的情况下，这样会造成 elasticsearch 压力过大，甚至会因此丢失日志，所以我们最好在采集器到保存的中间加上一个中间层，我们选择使用 kafka 作为中间层

# 配置 Output

fluent-bit 其余配置保持不变，需要替换 Output 的相关配置

```ini
[OUTPUT]
  name      kafka
  match     kube.*
  brokers   <your brokers addresses>
  topics    <your topic name>

  # sasl auth
  rdkafka.security.protocol   SASL_PLAINTEXT
  rdkafka.sasl.mechanisms     SCRAM-SHA-256
  rdkafka.sasl.username       <user>
  rdkafka.sasl.password       <password>
```

- name: 指定使用 kafka 输出插件
- match: 匹配的日志记录源 tag
- brokers: kafka 的 brokers 列表
- topics: 指定 Kafka 中的 topic 名称，可以配置多个
- sasl: sasl 的相关配置如果 kafka 没有使用可以删除
  - rdkafka.security.protocol: 指定 Kafka 的安全协议
  - rdkafka.sasl.mechanisms: 指定 SASL 认证机制
  - rdkafka.sasl.username: 指定 SASL 认证的用户名
  - rdkafka.sasl.password: 指定 SASL 认证的密码

## 动态的 Topic 配置

```ini
[OUTPUT]
  name      kafka
  match     kube.*
  brokers   <your brokers addresses>
  topics    <your topic name>
  topic_key <your topic name in record>

  dynamic_topic on

  # sasl auth
  rdkafka.security.protocol   SASL_PLAINTEXT
  rdkafka.sasl.mechanisms     SCRAM-SHA-256
  rdkafka.sasl.username       <user>
  rdkafka.sasl.password       <password>
```

- topic_key: 获取日志记录（record）中 topic_key 的值作为当前日志的 topic name，此时 topics 表示如果没有发现 record 中存在对应的 topic_key，将会使用的默认 topic 名称
- dynamic_topic: 启用动态 topic 的功能，当设置为 on 时，Kafka 插件会根据记录中的 topic_key 字段的值，动态决定将数据发送到哪个 topic；如果设置为 off，表示只有当 record 中 topic_key 的值配置在 topics 中有配置的情况下，才会发送到对应名字的 topic

更多配置参考: <https://docs.fluentbit.io/manual/pipeline/outputs/kafka>

这样配置完毕后，重启 fluent-bit 就会将日志传输到 kafka 中

我们将日志从 kafka 同步到 elasticsearch 可以有多种方式，如使用 fluent-bit/fluentd/logstash 等使用 kafka 作为数据源，elasticsearch 作为目标进行同步，或者使用 flink/编写自定义消费者 进行同步，或者使用 Kafka Connect 同步，这部分放到后面再说


---

> Author: [komorebi](https://blog.2nfree.com/)  
> URL: https://blog.2nfree.com/en/posts/kubernetes-logging-collection-5/  

