# 从零实现CI/CD（一）- Jenkins的安装部署的多种方式


# 前言

| [Jenkins](https://jenkins.io) 是一个开源的自动化服务器，主要用于自动化构建、测试和部署软件。它是基于Java开发的，可以在多种环境中运行，如 Linux，Windows 及其他类Unix操作系统。
| Jenkins 非常灵活，提供了丰富的插件系统，通过安装和配置这些插件，可以使 Jenkins 支持多种工具和平台，从而适应各种不同的 CI/CD（持续集成/持续部署）需求。
| Jenkins 能够帮助团队实现自动化构建、测试和部署，加速软件交付的速度。尽管 Jenkins 本身不是云原生产品，但通过与现代云技术的集成，它能够有效支持云原生应用的开发和运维工作。

# 裸机部署

## 基于 war 包的安装

### 安装 java 环境

运行 jenkins 需要安装 java 环境，可选择使用 oracle-jdk 或 openjdk，此处采用安装 oracle-jdk 的方式
oracle-jdk 需要在 [Oracle官网](https://www.oracle.com/java/technologies/downloads/#java8) 下载，下载后需要将对应的压缩包上传至服务器

```bash
#解压
tar -zxvf jdk-8u321-linux-x64.tar.gz
mv jdk1.8.0_321 /usr/local/java8

#配置环境变量
cat >> /etc/profile <<EOF
#java env
export JAVA_HOME=/usr/local/java8
export JRE_HOME=\$JAVA_HOME/jre
export CLASSPATH=.:\$JAVA_HOME/lib:\$JRE_HOME/lib
export PATH=\$JAVA_HOME/bin:\$PATH
EOF

source /etc/profile

#测试环境
java -version
javac
java
```

### 下载 Jenkins war 包

可在 [这个网站](https://get.jenkins.io/) 选择对应不同 war 版本

```bash
mkdir /data/jenkins
cd /data/jenkins
# 将下载完的 war 包上传至文件夹内
```

使用supervisor托管

```ini
[program:jenkins]
command=/usr/local/java8/bin/java -jar /data/jenkins/jenkins.war
directory=/data/jenkins
autostart=true
autorestart=true
user=root
log_stdout=true
log_stderr=true
logfile=/data/logs/jenkins/jenkins.log
logfile_maxbytes=1MB
logfile_backups=10 
stdout_logfile = /data/logs/jenkins/jenkins_stdout.log
stdout_logfile_maxbytes=10MB
stdout_logfile_backups=3
environment=JENKINS_HOME="/data/jenkins"
```

配置完成后，启动jenkins

```bash
supervisorctl reread
supervisorctl start jenkins
```

## 基于 Docker 安装 Jenkins

### 编写配置

创建文件夹保存持久化数据

```bash
mkdir -pv /data/jenkins/{data,home}
```

编写 docker compose 配置
{{< admonition type="info" open=true >}}
这里将 docker 的 sock 文件同步挂载是因为需要让 jenkins 的 docker 插件能够操作宿主机上的 docker
{{< /admonition >}}

```yaml
version: '3.9'
services:
    blueocean:
        image: 'jenkinsci/blueocean:1.25.5'
        volumes:
            - '/data/jenkins/home:/root'
            - '/var/run/docker.sock:/var/run/docker.sock'
            - '/data/jenkins/data:/var/jenkins_home'
        ports:
            - '50000:50000'
            - '8080:8080'
        container_name: jenkins
        user: root
        restart: always
```

### 启动 jenkins

{{< tabs >}}
{{% tab title="compose-v1" %}}
```bash
docker-compose up -d
```
{{% /tab %}}
{{% tab title="compose-v2" %}}
```bash
docker compose up -d
```
{{% /tab %}}
{{< /tabs >}}

## 编写 nginx 配置

```nginx
upstream jenkins {
  keepalive 32; #保持连接
  server 127.0.0.1:8080; #代理地址
}

#Jenkins websocket代理需要
map $http_upgrade $connection_upgrade {
  default upgrade;
  '' close;
}

server {
  listen 80;
  #注意修改域名
  server_name [域名];

  #传递来自Jenkins的Nginx认为无效的头文件
  ignore_invalid_headers off;
  return 301 https://$server_name$request_uri;
}

server {
  listen 443 ssl;
  #注意修改域名
  server_name [域名];

  #Jenkins的数据目录
  root            /data/jenkins/war/;

  #ssl配置
  add_header Strict-Transport-Security "max-age=31536000";
  ssl_certificate  [证书];
  ssl_certificate_key  [证书];
  ssl_ciphers "ECDHE-RSA-AES256-GCM-SHA384:ECDHE-RSA-AES128-GCM-SHA256:DHE-RSA-AES256-GCM-SHA384:DHE-RSA-AES128-GCM-SHA256:ECDHE-RSA-AES256-SHA384:ECDHE-RSA-AES128-SHA256:ECDHE-RSA-AES256-SHA:ECDHE-RSA-AES128-SHA:DHE-RSA-AES256-SHA256:DHE-RSA-AES128-SHA256:DHE-RSA-AES256-SHA:DHE-RSA-AES128-SHA:AES256-GCM-SHA384:AES128-GCM-SHA256:AES256-SHA256:AES128-SHA256:AES256-SHA:AES128-SHA:HIGH:!aNULL:!eNULL:!EXPORT:!DES:!MD5:!PSK:!RC4!3DES!ADH";
  ssl_protocols TLSv1.2;
  ssl_prefer_server_ciphers on;

  access_log      /data/logs/nginx/jenkins-access.log;
  error_log       /data/logs/nginx/jenkins-error.log;

  #传递来自Jenkins的Nginx认为无效的头文件
  ignore_invalid_headers off;

  location ~ "^/static/[0-9a-fA-F]{8}\/(.*)$" {
    #将所有静态文件重写为对根目录的请求
    rewrite "^/static/[0-9a-fA-F]{8}\/(.*)" /$1 last;
  }

  location / {
      sendfile off;
      proxy_pass         http://jenkins;
      proxy_redirect     default;
      proxy_http_version 1.1;

      #Jenkins websocket代理需要
      proxy_set_header   Connection        $connection_upgrade;
      proxy_set_header   Upgrade           $http_upgrade;

      proxy_set_header   Host              $host;
      proxy_set_header   X-Real-IP         $remote_addr;
      proxy_set_header   X-Forwarded-For   $proxy_add_x_forwarded_for;
      proxy_set_header   X-Forwarded-Proto $scheme;
      proxy_max_temp_file_size 0;

      #设置上传限制
      client_max_body_size       10m;
      client_body_buffer_size    128k;

      proxy_connect_timeout      90;
      proxy_send_timeout         90;
      proxy_read_timeout         90;
      proxy_buffering            off;
      proxy_request_buffering    off;
      proxy_set_header Connection "";
  }

}
```

# 基于 Kubernetes Deployment 的 Jenkins 部署

## deployment.yaml

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name:  jenkins-deployment
  namespace: cicd
  labels:
    app:  jenkins-deployment
spec:
  selector:
    matchLabels:
      app: jenkins-deployment
  replicas: 1
  template:
    metadata:
      labels:
        app:  jenkins-deployment
    spec:
      containers:
      - name:  jenkins-deployment
        image:  jenkinsci/blueocean:1.25.5
        resources:
          requests:
            cpu: 2
            memory: 4Gi
          limits:
            cpu: 2
            memory: 4Gi
        ports:
        - containerPort:  8080
          name: web
          protocol: TCP
        - containerPort: 50000
          name: agent
          protocol: TCP
        volumeMounts:
        - name: localtime
          mountPath: /etc/localtime
          #持久化jenkins数据
        - name: jenkins-storage
          mountPath: /var/jenkins_home
          subPath: jenkins_home
          #把home目录挂载出来，防止容器重载后密钥丢失
        - name: jenkins-storage
          mountPath: /root
          subPath: root
        - name: docker-sock
          mountPath: /var/run/docker.sock
      volumes:
        - name: localtime
          hostPath:
            path: /etc/localtime
        - name: jenkins-storage
          persistentVolumeClaim:
            claimName: jenkins-storage
        - name: docker-sock
          hostPath:
            path: /var/run/docker.sock
      restartPolicy: Always
      securityContext:
        runAsUser: 0
        runAsGroup: 0
        fsGroup: 0
```

## pvc.yaml

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: jenkins-storage
  namespace: cicd
  labels:
    app: jenkins-storage
spec:
  storageClassName: ceph-rbd
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 50Gi
```

## service.yaml

```yaml
apiVersion: v1
kind: Service
metadata:
  name: jenkins
  namespace: cicd
  labels:
    app: jenkins
spec:
  ports:
  - name: jenkins
    port: 8080
    targetPort: 8080
    protocol: TCP
  - name: jenkins-agent
    port: 50000
    targetPort: 50000
    protocol: TCP
  selector:
    app: jenkins-deployment
```

## ingress.yaml

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: jenkins.example.com
  namespace: cicd
spec:
  ingressClassName: nginx
  tls:
    - hosts:
        - jenkins.example.com
      secretName: [证书名称]
  rules:
    - host: jenkins.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: jenkins
                port:
                  number: 8080
```

{{< admonition type="danger" open=true >}}
PS：注意配置文件中的存储类相关配置和ingress域名配置和证书配置
{{< /admonition >}}

## 启动 Jenkins

```bash
kubectl apply -f ./
```

# 初始化 jenkins

## 修改插件国内源下载（可选）

如果当前 Jenkins 部署在国内服务器时，下载插件会非常慢，需要我们替换成国内源下载
{{< tabs >}}
{{% tab title="war包或docker" %}}
找到 Jenkins 相关数据文件夹下的 hudson.model.UpdateCenter.xml 文件，将其中默认的源，修改为其他源

```xml
<?xml version='1.1' encoding='UTF-8'?>
<sites>
  <site>
    <id>default</id>
    <url>https://updates.jenkins.io/update-center.json</url>
  </site>
</sites>
```

修改为

```xml
<?xml version='1.1' encoding='UTF-8'?>
<sites>
  <site>
    <id>default</id>
    <url>https://mirrors.tuna.tsinghua.edu.cn/jenkins/updates/update-center.json</url>
  </site>
</sites>
```
{{% /tab %}}
{{% tab title="kubernetes" %}}
在 kubernetes 中，直接使用命令进入容器使用 sed 命令修改

```bash
kubectl exec -it -n cicd pod 【jenkins pod名字】 -- sed -i 's#https://updates.jenkins.io/update-center.json#https://mirrors.tuna.tsinghua.edu.cn/jenkins/updates/update-center.json#' /var/jenkins_home/hudson.model.UpdateCenter.xml
```
{{% /tab %}}
{{< /tabs >}}
{{< admonition type="info" open=true >}}
这里使用的是清华源，也可以根据需要换成其他的源
{{< /admonition >}}

## 获取初始密码登录jenkins

{{< tabs >}}
{{% tab title="war包或docker" %}}
```bash
# 进入对应的数据文件夹，获取 /secrets/initialAdminPassword 文件的内容
cd /data/jenkins/data
cat ./secrets/initialAdminPassword
```
{{% /tab %}}
{{% tab title="kubernetes" %}}
```bash
kubectl exec -it -n cicd pod 【jenkins pod名字】 -- cat /var/jenkins_home/secrets/initialAdminPassword
```
{{% /tab %}}
{{< /tabs >}}

## 使用获取的密码登录，并初始化 Jenkins

{{< image src="jenkins-1.webp" >}}

## 选择安装推荐的插件

{{< image src="jenkins-2.webp" >}}

## 等待插件安装完毕设置管理员账号密码和邮箱

{{< image src="jenkins-3.webp" >}}

## 安装完毕

{{< image src="jenkins-4.webp" >}}


---

> 作者: [komorebi](https://blog.2nfree.com/)  
> URL: https://blog.2nfree.com/posts/cicd-in-kubernetes-based-jenkins-1/  

