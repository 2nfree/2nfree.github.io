# Useful Kubernetes Commands


# Resources

## Get resource requests and limits for all pods in namespace

{{< admonition type="info" open=true >}}
If you need to get all the resource requests and limits of cluster, just use ``kubectl get pods -A ...``
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

### Mem Requests (Units: Gi)

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

### Mem Limits (Units: Gi)

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

## Get the resources for each namespace

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

# Storage

## Get the total request size for all PVCs

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

## Get the total request size for all PVCs in namespace

```bash
kubectl get pvc --no-headers -n ${namespace} | awk '
{
    size = $4
    unit = substr(size, length(size))
    value = substr(size, 1, length(size) - 1)

    # convert to Gi
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

# Container Images

## List all container images used by PODs in namespace

```bash
kubectl get pods -n ${namespace} -o jsonpath='{range .items[*]}{"\n"}{.metadata.name}{":\t"}{range .spec.containers[*]}{.image}{", "}{end}{end}' | sort
```


---

> Author: [komorebi](https://blog.2nfree.com/)  
> URL: https://blog.2nfree.com/en/posts/useful-kubernetes-commands/  

