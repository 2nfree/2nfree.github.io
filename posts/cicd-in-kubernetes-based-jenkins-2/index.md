# 从零实现CI/CD（二）- Jenkins 配置流水线


# 简单的 CICD 流程

{{< image src="cicd.webp" >}}

# 准备工作

在配置流水线之前需要进行一些准备工作

## 下载插件

需要下载以下插件：

- Kubernetes
- Docker
- Docker Pipeline
- Gitlab

如图所示：
{{< image src="jenkins-plugin-kubernetes.webp" >}}
{{< image src="jenkins-plugin-docker.webp" >}}
{{< image src="jenkins-plugin-gitlab.webp" >}}

## 在 jenkins 容器中生成密钥对并注册到 gitlab 中

```bash
docker exec -it jenkins /bin/bash
ssh-keygen
```

{{< image src="sshkey.webp" >}}

## 创建全局凭据

Jenkins可以创建以下凭据

- 秘密文本：令牌，例如 API 令牌（例如 GitHub 个人访问令牌），
- 用户名和密码：可以作为单独的组件或格式中的冒号分隔字符串处理username:password
- 秘密文件：本质上是文件中的秘密内容
- 带有私钥的 SSH 用户名：SSH 公钥/私钥对
- 证书：PKCS#12 证书文件和可选密码
- Docker 主机证书身份验证凭据

选择系统管理->凭据->系统->全局凭据 (unrestricted)

{{< image src="unrestricted.webp" >}}

### 创建全局 Gitlab SSH 凭据

- 类型为：SSH Username with private key
- 范围选择全局
- ID为凭据的唯一标识，这里设置为```global-sshkey```
- 配置username
- 设置容器内生成的私钥
{{< image src="ssh-username-with-private-key.webp" >}}

### 创建全局镜像仓库凭据

- 类型为：Username with password
- 范围选择全局
- 用户名为仓库的用户名
- 密码为仓库的密码
- ID为凭据的唯一标识，这里设置为```harbor-secret```
{{< image src="username-with-password.webp" >}}

### 创建全局K8S集群凭据

K8S 集群凭据需要创建两个，一个用于 Jenkins Kubernetes Cloud 插件连接至 Kubernetes 集群启动 Jenkins Agent 容器，另一个用于 Jenkinsfile 中部署应用到 Kubernetes（如果不需要部署可以略过）

### 用于插件的 Secret

- 类型为：Secret file
- 范围选择全局
- 直接上传 ~/.kube/config 文件
- ID为凭据的唯一标识，这里设置为```k8s-config-file```
{{< image src="secret-file.webp" >}}

### 用于部署的 Secret

- 类型为：Secret text
- 范围选择全局
- Secret 为 ~/.kube/config 内容的 base64 加密（```base64 ~/.kube/config > ~/k8s-config.txt```）
- ID为凭据的唯一标识，这里设置为```k8s-config```
{{< image src="secret-text.webp" >}}

{{< admonition type="info" open=true >}}
PS : 如果当前 jenkins 服务器不能直接连接内网 K8S 配置，需要修改 ~/.kube/config 中的 server 地址为 api-service 的代理地址
{{< /admonition >}}

# 流水线的 Jenkinsfile 配置

## 基于 Docker 容器的构建

在每个 stage 中，使用不同的 docker agent 容器运行命令：

- image 为用于构建的镜像
- args 为 docker 参数
- steps 中编写需要执行的命令

```groovy
stages {
  stage('Node Build') {
    agent {
      docker {
        image 'node:14'
        args '-v $HOME/node_modules:/node_modules -p 3000:3000'
      }
    }
    steps {
    ....
    }
  }
}
```

## 基于 Kubernetes Cloud 的构建

等待更新

## 全量 Jenkinsfile 参考

```groovy
pipeline {
    # 这里设置默认使用 Jenkins 本身的 master 节点作为运行节点
    agent {
        label "master"
    }
    # 这里通过读取我们配置的凭据，将其读取到环境变量中
    environment {
        HARBOR_SECRET = credentials('harbor-secret')
        K8S_CONFIG = credentials('k8s-config')
    }
    # 这里可以配置 gitlab 触发器，用于配置推送代码后自动运行 jenkins 流水线
    triggers {
        gitlab(
            triggerOnPush: true,
            triggerOnMergeRequest: false,
            branchFilterType: 'All',
            secretToken:"【触发器的 token】"
        )
    }
    # 这里配置运行前可以修改的参数
    parameters {
        string(name: 'HARBOR_HOST', defaultValue: '', description: 'harbor 镜像仓库地址')
        string(name: 'APP_NAME', defaultValue: '', description: '应用名称')
        string(name: 'BRANCH_NAME', defaultValue: '', description: '分支名称')
    }
    stages {
        stage('Node Build') {
            # 这里使用 nodejs 的容器用于编译代码
            agent {
                docker {
                    image 'node:14'
                    args '-v $HOME/node_modules:/node_modules'
                }
            }
            steps {
                # 这里使用 git 插件以及之前配置的全局凭据拉取代码
                git url: "【仓库的地址，使用ssh模式】",
                    credentialsId: 'global-sshkey',
                    branch: "${params.BRANCH_NAME}",
                    changelog: true
                # 这里可以将提交 hash 读取到环境变量中
                script{
                    env.COMMIT_ID = sh(returnStdout: true,script: 'git rev-parse --short HEAD').trim()
                }
                sh 'echo npm_version:$(npm -v)'
                sh 'echo node_version:$(node -v)'
                sh 'npm config get registry'
                sh 'npm install'
                sh 'npm run build'
                # 这里将编译后的代码打包为一个名为 app 的包，可以在后续的 step 中使用
                stash includes: 'dist/**/*', name: 'app'
            }
        }
        stage('Docker Build'){
            steps {
                # 解压 app 包
                unstash 'app'
                # 登录容器仓库并且构建和推送容器，如果使用 username with password 的凭据作为环境变量，后缀添加 _USR 和 _PSW 是分别获取用户名和密码
                sh "docker login -u ${HARBOR_SECRET_USR} -p ${HARBOR_SECRET_PSW} ${params.HARBOR_HOST}"
                sh "docker build -t 【这里写对应的镜像 TAG】 ."
                sh "docker push 【这里写对应的镜像 TAG】"
                sh "docker rmi 【这里写对应的镜像 TAG】"
            }
        }
        stage('Deploy'){
            steps {
                ......
            }
        }
    }
}
```

# 配置流水线

1. 创建任务，选择类型为流水线
{{< image src="pipeline.webp" >}}

2. 将写好的 Jenkinsfile 放入下方
{{< image src="pipeline-jenkinsfile.webp" >}}

这里也可以选择 pipeline script from scm ，使用代码仓库中保存的 Jenkinsfile
{{< image src="pipeline-jenkinsfile-with-scm.webp" >}}

保存后，直接运行流水线即可


---

> 作者: [komorebi](https://blog.2nfree.com/)  
> URL: https://blog.2nfree.com/posts/cicd-in-kubernetes-based-jenkins-2/  

