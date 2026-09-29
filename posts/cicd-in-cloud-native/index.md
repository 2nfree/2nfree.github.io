# 云原生体系下的CI/CD介绍


# CI/CD介绍

## 什么是CI/CD

CI/CD 是持续集成/持续交付（Continuous Integration / Continuous Delivery）的缩写，是一种开发流程和技术实践。CI/CD 将软件开发过程中的自动化和快速迭代理念贯穿始终，并通过自动化工具、流程和原则来实现。

## DevOps 工作流程

典型的 DevOps 工作流程可以简化为 4 个阶段

1. 版本控制：这是存储和管理源代码的阶段，研发人员将代码通过版本控件（如 git，svn 等）保存到统一的代码管理仓库（github，gitlab 等），版本控件包含代码的不同版本。
2. 持续集成：这是将源代码编译与验证的阶段，要求每当开发人员提交了新代码之后，就对整个应用进行构建，并对其执行全面的自动化测试集合。根据构建和测试结果，我们可以确定新代码和原有代码是否正确的集成在一起。
3. 持续交付：持续交付是持续集成的延伸，修改后的代码可以通过自动构建工具来编译，并使其准备好交付给最终用户，它的目标在于让软件的构建、测试与发布变得更快以及更频繁。
4. 持续部署：持续部署是在持续交付的基础上，把部署到生产环境的过程自动化。

## CI/CD 相关工具介绍

- Gitlab：GitLab 是一个 Web-based 的 Git 代码仓库管理工具，它提供了 Git 托管、代码审查、问题跟踪、持续集成/持续交付等功能。GitLab 分为社区版和企业版两个版本，可以自行部署在自己的服务器上或使用 GitLab 官方的 Cloud 版本；
- Harbor：Harbor 是一个开源的容器镜像仓库，主要用于存储和分发 Docker 镜像，它支持“企业级”的功能，例如镜像签名、访问控制、LDAP 集成等。Harbor 提供了一个安全、可靠和可扩展的环境，帮助用户快速部署和管理 Docker 镜像；
- Jenkins：开源的 CI/CD 工具，可用于自动化软件构建、测试和部署，Jenkins 提供了各种插件和可扩展性，可以适应多种工作场景和技术栈；
- Argo：Argo 是一个基于 Kubernetes 的开源工作流引擎，用于编排容器化应用的工作流，支持多个开发语言，可以帮助开发者更方便地创建和管理容器化应用；
- CircleCI：CircleCI 是一个基于云计算的持续集成和持续交付 (CI/CD) 平台，它能够使开发者无需花费大量时间在设置和维护构建管道上，只需要专注于代码编写和测试即可；
- source-to-image(S2I)：S2I 是一个开源的工具，用于在源代码中构建可运行的镜像，它可以轻松地将源代码转换为镜像，将源代码和构建脚本打包在一起，并将其注入到基础镜像中，从而生成应用程序的可运行镜像。

# 云原生 Devops 工具链示意图

{{< image src="cloud-native-devops.webp" >}}

# 流水线思想

流水线思想是指将软件开发、测试和部署等环节中的任务自动化，并将它们串联在一起形成一个流程。这个流程会在提交代码后触发，然后经过一系列的阶段，最终将代码从开发人员的环境中送到指定的环境中。
这个流程通常包括代码编译、自动化测试、代码静态分析、构建镜像、部署等多个步骤。流水线思想的目的是提高项目质量、加速发布周期和降低运行风险

## 持续发布流水线

在云原生体系中持续交付的过程就是通过自动化工具将代码打包成可运行容器镜像的过程，我们使用 Jenkins 完成这些工作

### 为程序编写 Dockerfile，用于构建镜像

例如：

```Dockerfile
FROM golang:1.20-alpine as Builder
ARG EXECUTABLE_APP
COPY ./ /app/
WORKDIR /app/
RUN CGO_ENABLED=0 GOOS=linux go build -a -o ${EXECUTABLE_APP} ./main.go

FROM alpine:3
ARG EXECUTABLE_APP
ENV OPTIONS="" \
    EXECUTABLE_APP=${EXECUTABLE_APP}
WORKDIR /opt/app
COPY --from=0 /app/$EXECUTABLE_APP /opt/app/
RUN chmod +x /opt/app/${EXECUTABLE_APP}
EXPOSE 7000
ENTRYPOINT exec /opt/app/${EXECUTABLE_APP} ${OPTIONS}
```

{{< admonition type="info" open=true >}}
发生了什么？

- 在此 Dockerfile 中，将源代码编译与运行镜像构建合在了一起，为多步构建
  - 拉取 ```golang:1.20-alpine``` 镜像作为源代码的 Builder
  - 定义构建时参数 EXECUTABLE_APP：构建完后的二进制文件的名称
  - 复制当前目录下的代码到 /app 下
  - 设定工作目录为 /app
  - 执行构建命令为 ```CGO_ENABLED=0 GOOS=linux go build -a -o ${EXECUTABLE_APP} ./main.go```
  - 拉取 ```alpine:3``` 的镜像作为容器的运行环境
  - 设置环境变量 OPTIONS 以及 EXECUTABLE_APP，将构建时参数赋值给对应的环境变量（因为构建时参数无法使用在 ENTRYPOINT 或 CMD 中）
  - 设定工作目录为 /opt/app
  - 将第一步构建好的二进制文件复制到 /opt/app/ 下
  - 赋予构建完的二进制文件可执行权限
  - 开放端口 7000
  - 设置容器运行命令，使用 shell 模式，命令为：```/opt/app/${EXECUTABLE_APP} ${OPTIONS}```
{{< /admonition >}}

### 构建容器并推送

```bash
docker login -u 【镜像仓库用户名】 -p 【镜像仓库密码】 【镜像仓库地址】
#容器名称命名：镜像仓库地址/项目空间/名称:容器TAG : harbor.fs.com/sre/java-test:v1.0
docker build --build-arg EXECUTABLE_APP=【构建后的可执行文件名称】 -t 【容器名称】 .
docker push 【容器名称】
```

### 将以上流程自动化的Jenkins流水线写法

```groovy
stage('Prepare'){
  steps {
    container('jnlp'){
      git url: "【代码仓库地址】",
          credentialsId: '【用于拉取代码的公钥】',
          branch: "【对应的分支】",
          changelog: true
      stash includes: '**/*', name: 'app'
    }
  }
}
stage('Docker Build'){
  steps {
    container('docker-cli'){
      unstash 'app'
      sh "docker login -u 【镜像仓库用户名】 -p 【镜像仓库密码】 【镜像仓库地址】"
      sh "docker build --build-arg EXECUTABLE_APP=【构建后的可执行文件名称】 -t 【容器名称】 ."
      sh "docker push 【容器名称】"
      sh "docker rmi 【容器名称】"
    }
  }
}
```

{{< admonition type="info" open=true >}}
发生了什么？

- 在 Prepare 阶段，使用了 jenkins 的 git 插件拉取代码
- 使用 stash 插件压缩了当前目录（源代码目录）
- 在 Docker Build 阶段，解压了源代码目录并执行了构建
{{< /admonition >}}

## 持续部署流水线

在云原生体系中持续部署的过程就是将已经打包好的可运行容器镜像，部署到我们的对应环境中，我们也可以使用 Jenkins 完成

### 在 k8s 中提交 yaml 配置，生成对应的资源

```yaml
kind: Service
apiVersion: v1
metadata:
  labels:
    app: 【应用名称】
  name: 【应用名称】
spec:
  ports:
    - name: http
      port: 【应用端口】
      protocol: TCP
      targetPort: 【应用端口】
  selector:
    app: 【应用名称】
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: 【应用名称】
  labels:
    app: 【应用名称】
spec:
  replicas: 1
  revisionHistoryLimit: 3
  selector:
    matchLabels:
      app: 【应用名称】
  template:
    metadata:
      labels:
        app: 【应用名称】
    spec:
      containers:
        - name: 【应用名称】
          image: 【镜像名称】
          imagePullPolicy: IfNotPresent
          ports:
            - containerPort: 【应用端口】
      imagePullSecrets:
        - name: 【镜像拉取secret】
```

```bash
kubectl apply -f 配置.yaml
```

### 将以上流程自动化的Jenkins流水线写法

```groovy
stage('Deploy'){
  steps {
    container('kubectl'){
      sh "mkdir -p ~/.kube"
      sh "echo 【K8S配置详情】 | base64 -d > ~/.kube/config"
      script {
        writeFile file: 'deploy.yaml', text: """
kind: Service
apiVersion: v1
metadata:
  labels:
    app: 【应用名称】
  name: 【应用名称】
spec:
  ports:
    - name: http
      port: 【应用端口】
      protocol: TCP
      targetPort: 【应用端口】
  selector:
    app: 【应用名称】
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: 【应用名称】
  labels:
    app: 【应用名称】
spec:
  replicas: 1
  revisionHistoryLimit: 3
  selector:
    matchLabels:
      app: 【应用名称】
  template:
    metadata:
      labels:
        app: 【应用名称】
    spec:
      containers:
        - name: 【应用名称】
          image: 【镜像名称】
          imagePullPolicy: IfNotPresent
          ports:
            - containerPort: 【应用端口】
      imagePullSecrets:
        - name: 【镜像拉取secret】
"""
      }
      sh "kubectl apply -f deploy.yaml"
    }
  }
}
```

{{< admonition type="info" open=true >}}
发生了什么？

- 在 Deploy 阶段，将 k8s config 保存到当前用户的 ~/.kube/config 下，用于连接 k8s 集群
- 使用 writeFile 插件，生成了对应资源的 yaml 配置
- 将 yaml 配置应用到 k8s 集群
{{< /admonition >}}

## 流水线解决了什么问题

1. 避免环境差异
解决本地开发环境与线上环境不一致出现的问题，通过流水线来解耦，保持代码编译环境和打包环境一致，研发人员只需关注被打包好的容器镜像即可。
2. 自动化
使用自动化的流水线来完成编译，发布和部署的过程，比起人工来说可以更加快速、准确地完成任务，防止人工因为粗心产生的错误。
3. 阶段性任务
支持不同的构建或发布频率，可以以每周，每天，每小时为细粒度运行流水线，且构建与发布可以解耦，实现随时构建随时发布；
支持不同交付环境，随时选择对应的交付环境，在不同的环境上，执行不同的测试和验证。

基于 kubernetes 的 CI/CD 电子书：<https://thenewstack.io/ebooks/kubernetes/ci-cd-with-kubernetes/>


---

> 作者: [komorebi](https://blog.2nfree.com/)  
> URL: https://blog.2nfree.com/posts/cicd-in-cloud-native/  

