# 好用的 Kubernetes 命令


# 资源

## 获取对应命名空间所有 Pod 的资源请求和限制情况

{{< admonition type="info" open=true >}}
如果需要获取整个集群所有的，只需要使用 kubectl get pods -A ...... 就可以了
{{< /admonition >}}

### CPU Requests

```bash
kubectl get pods -n ${namespace} -o=jsonpath='{.items[*].spec.containers[*].resources.requests.cpu}' | \
tr -s '[:space:]' '\n' | \
awk 'BEGIN{sum=0}{
    if(tolower($1) ~ /mi?/) 
        sum += gensub(/[^0-9.]/, "", "g", $1)/1000; 
    else 
        sum += $1
    } 
    END{print sum " CPU Cores"}'
```

### CPU Limits

```bash
kubectl get pods -n ${namespace} -o=jsonpath='{.items[*].spec.containers[*].resources.limits.cpu}' | \
tr -s '[:space:]' '\n' | \
awk 'BEGIN{sum=0}{
    if(tolower($1) ~ /mi?/) 
        sum += gensub(/[^0-9.]/, "", "g", $1)/1000; 
    else 
        sum += $1
    } 
    END{print sum " CPU Cores"}'
```

### Mem Requests (单位：Gi)

```bash
kubectl get pods -n ${namespace} -o=jsonpath='{.items[*].spec.containers[*].resources.requests.memory}' | \
tr -s '[:space:]' '\n' | \
awk 'BEGIN{sum=0}{
    if(tolower($1) ~ /ki?/) 
        sum += gensub(/[^0-9.]/, "", "g", $1)/1048576; 
    else if(tolower($1) ~ /mi?/) 
        sum += gensub(/[^0-9.]/, "", "g", $1)/1024; 
    else if(tolower($1) ~ /gi?/) 
        sum += gensub(/[^0-9.]/, "", "g", $1); 
    else if(tolower($1) ~ /ti?/) 
        sum += gensub(/[^0-9.]/, "", "g", $1)*1024; 
    else 
        sum += $1
    } 
    END{print sum " GiB"}'
```

### Mem Limits (单位：Gi)

```bash
kubectl get pods -n ${namespace} -o=jsonpath='{.items[*].spec.containers[*].resources.limits.memory}' | \
tr -s '[:space:]' '\n' | \
awk 'BEGIN{sum=0}{
    if(tolower($1) ~ /ki?/) 
        sum += gensub(/[^0-9.]/, "", "g", $1)/1048576; 
    else if(tolower($1) ~ /mi?/) 
        sum += gensub(/[^0-9.]/, "", "g", $1)/1024; 
    else if(tolower($1) ~ /gi?/) 
        sum += gensub(/[^0-9.]/, "", "g", $1); 
    else if(tolower($1) ~ /ti?/) 
        sum += gensub(/[^0-9.]/, "", "g", $1)*1024; 
    else 
        sum += $1
    } 
    END{print sum " GiB"}'
```

## 获取每个命名空间资源限额情况

```bash
#!/bin/bash
set -e
namespaces=($(kubectl get ns --no-headers | awk '{print $1}'))
for namespace in "${namespaces[@]}"; do
    request_cpu=$(
        kubectl get pods -n ${namespace} -o=jsonpath='{.items[*].spec.containers[*].resources.requests.cpu}' | \
        tr -s '[:space:]' '\n' | \
        awk 'BEGIN{sum=0}{
            if(tolower($1) ~ /mi?/) 
                sum += gensub(/[^0-9.]/, "", "g", $1)/1000; 
            else 
                sum += $1
            } 
            END{print sum " CPU Cores"}'
    )
    request_memory=$(
        kubectl get pods -n ${namespace} -o=jsonpath='{.items[*].spec.containers[*].resources.requests.memory}' | \
        tr -s '[:space:]' '\n' | \
        awk 'BEGIN{sum=0}{
            if(tolower($1) ~ /ki?/) 
                sum += gensub(/[^0-9.]/, "", "g", $1)/1048576; 
            else if(tolower($1) ~ /mi?/) 
                sum += gensub(/[^0-9.]/, "", "g", $1)/1024; 
            else if(tolower($1) ~ /gi?/) 
                sum += gensub(/[^0-9.]/, "", "g", $1); 
            else if(tolower($1) ~ /ti?/) 
                sum += gensub(/[^0-9.]/, "", "g", $1)*1024; 
            else 
                sum += $1
            } 
            END{print sum " GiB"}'
    )
    limit_cpu=$(
        kubectl get pods -n ${namespace} -o=jsonpath='{.items[*].spec.containers[*].resources.limits.cpu}' | \
        tr -s '[:space:]' '\n' | \
        awk 'BEGIN{sum=0}{
            if(tolower($1) ~ /mi?/) 
                sum += gensub(/[^0-9.]/, "", "g", $1)/1000; 
            else 
                sum += $1
            } 
            END{print sum " CPU Cores"}'
    )
    limit_memory=$(
        kubectl get pods -n ${namespace} -o=jsonpath='{.items[*].spec.containers[*].resources.limits.memory}' | \
        tr -s '[:space:]' '\n' | \
        awk 'BEGIN{sum=0}{
            if(tolower($1) ~ /ki?/) 
                sum += gensub(/[^0-9.]/, "", "g", $1)/1048576; 
            else if(tolower($1) ~ /mi?/) 
                sum += gensub(/[^0-9.]/, "", "g", $1)/1024; 
            else if(tolower($1) ~ /gi?/) 
                sum += gensub(/[^0-9.]/, "", "g", $1); 
            else if(tolower($1) ~ /ti?/) 
                sum += gensub(/[^0-9.]/, "", "g", $1)*1024; 
            else 
                sum += $1
            } 
            END{print sum " GiB"}'
    )
    echo "${namespace}"
    echo "request cpu：${request_cpu} request mem：${request_memory}"
    echo "limit cpu：${limit_cpu} limit mem：${limit_memory}"
    echo "------------------------------------------"
done
```

# 存储

## 获取所有 PVC 的总请求大小

```bash
kubectl get pvc --no-headers -A | awk '
{
    size = $5
    unit = substr(size, length(size))
    value = substr(size, 1, length(size) - 1)

    if (unit == "Ki") value /= 1024 * 1024
    else if (unit == "Mi") value /= 1024
    else if (unit == "Gi") value = value
    else if (unit == "Ti") value *= 1024
    else if (unit == "Pi") value *= 1024 * 1024
    else if (unit == "Ei") value *= 1024 * 1024 * 1024

    total += value
}
END {print "Total PVC Size (Gi):", total}'
```

## 获取对应命名空间 PVC 的总请求大小

```bash
kubectl get pvc --no-headers -n ${namespace} | awk '
{
    size = $4
    unit = substr(size, length(size))
    value = substr(size, 1, length(size) - 1)

    # 转换到 Gi
    if (unit == "Ki") value /= 1024 * 1024
    else if (unit == "Mi") value /= 1024
    else if (unit == "Gi") value = value
    else if (unit == "Ti") value *= 1024
    else if (unit == "Pi") value *= 1024 * 1024
    else if (unit == "Ei") value *= 1024 * 1024 * 1024

    total += value
}
END {print "Total PVC Size (Gi):", total}'
```

# 容器镜像

## 列出当前命名空间所有 POD 使用的容器镜像

```bash
kubectl get pods -n ${namespace} -o jsonpath='{range .items[*]}{"\n"}{.metadata.name}{":\t"}{range .spec.containers[*]}{.image}{", "}{end}{end}' | sort
```


---

> 作者: [komorebi](https://blog.2nfree.com/)  
> URL: https://blog.2nfree.com/posts/useful-kubernetes-commands/  

