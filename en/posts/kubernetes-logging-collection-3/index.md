# Kubernetes 日志收集（三）- Logging Operator 中使用 Concat 多行合并相关问题


# 多行合并出现的问题

之前讲到容器的标准输出日志会将每一行打印的日志拆分开
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

我们使用需要 concat 在日志转发器中先合并相同的 log，配置如下所示

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
      separator: ''
      flush_interval: 5
```

但是
如果我们只配置 multiline_start_regexp: '/^\d{4}-\d{2}-\d{2}/'，在长时间没有新日志生成时，会出现如下错误：

```text
2024-05-13 03:10:16 +0000 [warn]: #0 send an error event to @ERROR: error_class=Fluent::Plugin::ConcatFilter::TimeoutError error="Timeout flush: kubernetes.var.log.containers.printorebi-test_print-logs-6fc021e614d408768bc84012f7ac34438bdb5d6d7ea8474a6307970e6af1ba66.log:default" location=nil tag="kubernetes.var.log.containers.print-logs-64fb98db85-x2hnd_komo1e614d408768bc84012f7ac34438bdb5d6d7ea8474a6307970e6af1ba66.log" time=2024-05-13 03:10:16.196873027 +0000
```

这时我们有两种解决方案

## 使用 multiline_end_regexp

我们可以将 concat 的配置变为如下配置：

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
```

multiline_end_regexp 用于匹配日志结尾的格式，使用正则 ```/\Z/``` 匹配字符串的结尾，这个方法并不能很好的适配所有的日志格式，还可以使用另一个方法

## 使用 flowLabel 和 timeout_label

在 fluentd 的 concat 插件中，timeout_label 配置可以让 concat 合并超时的日志通过其他 label 的相关 filter 进行处理
这个配置在 Logging Operator 中如下所示：

```yaml
apiVersion: logging.banzaicloud.io/v1beta1
kind: Flow
metadata:
  name: test-logging
  namespace: komorebi-test
spec:
  match:
  - select:
      labels:
        logging: golang
  filters:
  - concat:
      key: log
      multiline_start_regexp: '/^\d{4}-\d{2}-\d{2}/'
      separator: ''
      flush_interval: 1
      timeout_label: '@test-logging'
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
kind: Flow
metadata:
  name: test-logging-timeout
  namespace: komorebi-test
spec:
  flowLabel: '@test-logging'
  filters:
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
```

这里我们将超时的日志直接走 parser 进行匹配，然后通过 output 传输


---

> Author: [komorebi](https://blog.2nfree.com/)  
> URL: https://blog.2nfree.com/en/posts/kubernetes-logging-collection-3/  

