[Argo Workflows](https://argoproj.github.io/argo-workflows/) 是一个[云原生](https://en.wikipedia.org/wiki/Cloud-native_computing)的通用的工作流引擎。本教程主要介绍如何用其完成持续集成（Continous Integration, CI）任务。

## 基本概念
对任何工具的基本概念有一致的认识和理解，是我们学习以及与他人交流的基础。

以下是本文涉及到的概念：

* WorkflowTemplate，工作流模板
* Workflow，工作流

为方便读者理解，下面就几个同类工具做对比：

| Argo Workflow | Jenkins |
|---|---|
| WorkflowTemplate | Pipeline |
| Workflow | Build |

## 安装
首先，你需要有一套 [Kubernetes](https://github.com/kubernetes/kubernetes/) 环境。下面的工具可以帮助你快速按照好一套 Kubernetes 环境：

> 推荐使用 [hd](https://github.com/LinuxSuRen/http-downloader) 安装下面的工具
>
> 安装 `hd` 的命令为：`curl https://linuxsuren.github.io/tools/install.sh|bash`

| 工具 | 工具安装 |使用 |
|---|---|---|
| [k3d](https://k3d.io/) | `hd i k3d` | `k3d cluster create` |
| [kubekey](https://github.com/kubesphere/kubekey) | `hd i kk` | `kk create cluster` |
| [minikube](https://github.com/kubernetes/minikube) | `hd i minikube` | `minikube start` |

当 Kubernetes 环境就绪后，就可以通过下面的命令会在命名空间（`argo`）下安装最新版本的 `Argo Workflow`：

```shell
kubectl create namespace argo
kubectl apply -n argo -f https://github.com/argoproj/argo-workflows/releases/latest/download/install.yaml
```

如果你的环境访问 GitHub 时有网络问题，可以使用下面的命令来安装：

```shell
docker run -it --rm -v $HOME/.kube/:/root/.kube --network host ghcr.io/linuxsuren/argo-workflows-guide:master
```

推荐使用的工具：

|||
|---|---|
| [k9s](https://k9scli.io/) | K9s is a terminal based UI to interact with your Kubernetes clusters. |

## 设置访问方式
我们可以用下面的方式或者其他方式来设置 Argo Workflows 的访问端口：

```shell
kubectl -n argo port-forward deploy/argo-server --address 0.0.0.0 2746:2746
```

> 需要注意的是，这里默认的配置下，服务器设置了自签名的证书提供 HTTPS 服务，因此，确保你使用 `https://` 协议进行访问。

例如，地址为：`https://10.121.218.242:2746/`

Argo Workflows UI 提供了多种认证登录方式，对于学习、体验等场景，我们可以通过下面的命令直接设置绕过登录：

```shell
kubectl patch deployment \
  argo-server \
  --namespace argo \
  --type='json' \
  -p='[{"op": "replace", "path": "/spec/template/spec/containers/0/args", "value": [
  "server",
  "--auth-mode=server"
]}]'
```

## 简单示例

下面是一个非常简单的示例：

```shell
cat <<EOF | kubectl apply -n default -f -
apiVersion: argoproj.io/v1alpha1
kind: WorkflowTemplate
metadata:
  name: hello-world
spec:
  entrypoint: hd # 执行入口，类似于 Go、Java 语言的的 main 函数
  templates:
  - container:
      args:
      - search
      - kubectl
      command:
      - hd
      image: ghcr.io/linuxsuren/hd:v0.0.70 # 任务镜像
      name: main
    name: hd
EOF
```

执行成功后，就可以在下面的地址访问到刚刚创建的工作流模板：

`https://10.121.218.242:2746/workflow-templates/default`

选择其中一个模板，点击 `SUBMIT` 按钮，并设置对应的参数后即可触发工作流的执行。

在 Workflows 的详情页面中，我们做如下的操作：

* RESUBMIT，使用相同的模板以及参数触发一次新的执行

### 小结
通过前面的步骤，我们可以观察到 Argo Workflow 有如下特点：

* 需要具备基本的容器知识
* 需要熟悉 Kubernetes 的基本资源，例如：PodTemplate、ConfigMap、Secret、Volume 等
* 实现特定任务，基本上是需要寻找官方提供的镜像，或自行构建镜像

## 完整示例

```shell
cat <<EOF | kubectl apply -n default -f -
apiVersion: argoproj.io/v1alpha1
kind: WorkflowTemplate
metadata:
  name: gogit
spec:
  entrypoint: main                  # 执行入口
  arguments:
    # 工作流全局参数
    parameters:
      - name: repo
        value: https://github.com/linuxsuren/gogit
      - name: branch
        value: master

  # Volume 模板申明，用于工作流中多个 Pod 之间共享存储
  # 例如：克隆代码、构建代码的 Pod 之间共享目录
  # 动态创建 Volume，与当前工作流的生命流程保持一致
  volumeClaimTemplates:
    - metadata:
        name: work
      spec:
        accessModes: ["ReadWriteOnce"]
        resources:
          requests:
            storage: 64Mi

  templates:
  - name: main
    dag:
      tasks:
        - name: clone
          template: clone   # 引用下面的模板，并传入参数
          arguments:
            parameters:
              - name: repo
                value: "{{workflow.parameters.repo}}"
              - name: branch
                value: "{{workflow.parameters.branch}}"
        - name: build
          template: build
          depends: checkout # 通过 depends 设置执行任务之间的顺序关系
  
  - name: clone
    inputs:
      parameters:
        - name: repo
        - name: branch
    container:
      volumeMounts:
        - mountPath: /work              # 共享该目录
          name: work
      image: alpine/git:v2.26.2
      workingDir: /work                 # 代码会克隆到这个目录中
      args:
        - clone
        - --depth
        - "1"
        - --branch
        - "{{inputs.parameters.branch}}"
        - --single-branch
        - "{{inputs.parameters.repo}}"
        - .
  - name: build
    container:
      image: golang:1.19
      volumeMounts:
        - name: work
          mountPath: /work
      workingDir: /work
      env:
        - name: GOPROXY     # 根据需要来设置相应的环境变量，例如这里的 Golang 代理
          value: https://goproxy.io,direct
      command:
        - make
      args:
        - build             # 执行 Makefile 中的 build 指令来构建 Golang 代码
EOF
```

### 小结

通过上面的例子，我们可以看到 Argo Workflows 的如下特点：

* 一个工作流中的多个任务之间默认是并行执行的，如果希望顺序执行则可以通过 `depends` 设置
* 工作流中可以申明多个任务模板，只有被 `entrypoint` 引用到的模板才会被执行
* 每执行一个任务都会对应启动一个 Pod
* 一个工作流之间的多个任务需要共享目录的话，需要挂载 Volume

## Webhook
所有主流 Git 仓库都是支持 webhook 的，借助 webhook 可以当代码发生变化后实时地触发工作流的执行。

Argo Workflows 利用 `WorkflowEventBinding` 将收到的 webhook 请求与 WorkflowTemplate 做关联。请参考下面的例子：

```shell
cat <<EOF kubectl apply -n default -f -
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: submit-workflow-template
rules:
  - apiGroups:
      - argoproj.io
    resources:
      - workfloweventbindings
    verbs:
      - list
  - apiGroups:
      - argoproj.io
    resources:
      - workflowtemplates
    verbs:
      - get
  - apiGroups:
      - argoproj.io
    resources:
      - workflows
    verbs:
      - create
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: github.com
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: github.com
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: submit-workflow-template
subjects:
  - kind: ServiceAccount
    name: github.com
    namespace: argo
---
apiVersion: v1
stringData:
  github.com: |			# 这里对应 ServiceAcccount 名称
    type: github		# 固定的几个类型
    secret: "my-uuid"
kind: Secret
metadata:
  name: argo-workflows-webhook-clients
type: Opaque
---
apiVersion: argoproj.io/v1alpha1
kind: WorkflowEventBinding
metadata:
  name: pull-request-binding
spec:
  event:
    # 通过 webhook 的 payload 对请求进行过滤，并联动触发对应的工作流模板
    selector: payload.project.name == "gogit" && payload.object_attributes.state == "opened"
  submit:
    workflowTemplateRef:
      name: gogit       # 关联工作流模板
    arguments:
      parameters:
      - name: branch
        valueFrom:
          # 从 webhook 的 payload 中提取值作为参数
          event: payload.object_attributes.source_branch
EOF
```

然后，在代码仓库中添加如下的 webhook 地址（其中，`default` 是 `WorkflowEventBinding` 所在的命名空间）：

```
https://argo-workflow-ip:port/api/v1/events/default/
```

上面的 Secret 名称 `argo-workflows-webhook-clients` 是固定的，所在命名空间也就是 webhook 地址中的 `default`。支持的 Git Provider 名称也是固定的几个：

* `bitbucket`
* `bitbucketserver`
* `github`
* `gitlab`

### 小结

从上面的例子中，我们可以看到：

* Argo Workflows 以申明式的资源将 webhook 与工作流模板做关联，非常地灵活
* webhook 绑定并不局限在 Git 代码仓库上，还可以与其他类型的 webhook 做关联
* 通过从 webhook 的 payload 中提取值，可以非常方便地为工作流模板参数传递值

## 关联代码仓库

对于不少的团队而言，会出于各种考虑而选择私有部署 Git 服务，例如：Gitlab、Gitee 等。而将工作流的执行结果与代码仓库的 Pull Request 相关联几乎是一个**标配**。以下是关联后的几点好处：

* 工作流执行失败后阻止 Pull Request 的合并
* 在 Pull Request 页面中可以直接看到工作流执行状态

下面会基于 https://github.com/LinuxSuRen/gogit/ 给出一个关联方案：

```shell
cat <<EOF | kubectl apply -n default -f -
apiVersion: v1
data:
  token: eW91ci10b2tlbg==
kind: Secret
metadata:
  name: git-secret
type: Opaque
---
apiVersion: argoproj.io/v1alpha1
kind: WorkflowTemplate
metadata:
  name: hook
spec:
  entrypoint: main
  arguments:
    parameters:
      - name: pr
        value: 1
      - name: argoServer
        value: https://localhost:8080

  hooks:
    exit:                   # 只有 exit 这个 hook 名称是固定的
      template: hook
    all:                    # 这里可以是任意字符串，重点在于 expression 这里的表达式
      template: hook
      expression: "true"    # 可以通过表达式 expression 对事件进行过滤

  templates:
  - container:
      args:
      - search
      - kubectl
      command:
      - hd
      image: ghcr.io/linuxsuren/hd:v0.0.70 # 任务镜像
      name: main
    name: hd
  - name: hook
    volumes:
      - name: git-secret
        secret:
          defaultMode: 0400
          secretName: git-secret    # 包含 token 字段的 Secret
    container:
      image: ghcr.io/linuxsuren/gogit:master@sha256:4855f4ffbc1644eb7246f94cc9ee12c793ed4c26ba18e1d4d9afa57b72f1e846
      args:
        - --provider=github         # 支持 GitHub、Gitlab 等，私有部署的话需要参数 --server 指定地址
        - --owner=LinuxSuRen        # 根据需要修改 owner、repo、username
        - --repo=gogit
        - --username=LinuxSuRen
        - --token=file:///root/.ssh/token
        - --pr={{workflow.parameters.pr}}
        - --target={{workflow.parameters.argoServer}}/workflows/{{workflow.namespace}}/{{workflow.name}}
        - --status={{workflow.status}}
      volumeMounts:
        - mountPath: /root/.ssh/
          name: git-secret
EOF
```

触发上面的工作流后，就会在指定的仓库 Pull Request 上出现构建状态。

### 参考链接

* [官方文档](https://argoproj.github.io/argo-workflows/lifecyclehook/)

### 小结

从这个示例中，我们可以看到：

* hook 机制依然是非常的灵活，但 expression 表达式可能会是一个具有挑战的部分
* hook 机制有点像是 Golang 的 `__init` 函数，作为特殊的入口，可以调用其他的模板

## 日志持久化
Argo Workflows 默认不会持久化工作流日志，而是从每个任务对应的 Pod 中获取日志。而对于 Kubernetes 来说，Pod 是一个没有保障的最小执行单元，可能会由于人为或者某种策略被删除。当 Pod 被删除后，日志就无法查看了。因此，对于生产环境而言，必须要持久化日志。

Argo Workflows 执行多种存储协议，以下是兼容 S3 的 MinIO 存储：

首先，下载、安装以及配置 minio。本文仅作学习、演示使用，生产环境中，请按照官方文档进行安装、配置。

```shell
hd i minio
minio server /tmp/minio --console-address ":9001"
```

然后，访问 minio 管理界面 `http://localhost:9001`，创建名为 `argo-workflow` 的 `bucket`。创建 `Access Key`，并写入下面的 `Secret` 中。

安装如下配置修改 `ConfigMap`：

```shell
kubectl create secret generic minio-workflow \
  --from-literal=accessKey=supersecret \
  --from-literal=secretKey=topsecret
cat <<EOF | kubectl apply -n default -f -
apiVersion: v1
kind: ConfigMap
metadata:
  name: workflow-controller-configmap
  namespace: argo
data:
  artifactRepository: |
    archiveLogs: true          # 全局设置使得所有工作流日志做持久化
    s3:
      bucket: argo-workflow    # 在 minio 中创建的 bucket
      endpoint: minio.minio.svc
      insecure: true                  # 当 minio 没有启用 TLS
      accessKeySecret:
        name: minio-workflow
        key: accessKey
      secretKeySecret:
        name: minio-workflow
        key: secretKey
EOF
```

完成上面的配置后，再次执行任意工作流，并将执行完成的 Pod 删除后，我们依然可以在 UI 上查看任务日志。并且，可以在 minio 中看到了新增的文件。每个 Pod 的日志在 minio 中分别以一个文件的形式存储。

我们可以通过 minio 的命令行客户端 `mc` 看到类似如下的文件：

```shell
mc alias set myminio http://localhost:9001 minioadmin minioadmin
# mc ls myminio/argo-workflow -r
[2022-12-09 10:53:31 CST]    20B STANDARD hello-world-5mjgp/hello-world-5mjgp-clone-3848310779/main.log
[2022-12-09 10:55:39 CST]  16KiB STANDARD hello-world-5mjgp/hello-world-5mjgp-image-2614052838/main.log
[2022-12-09 10:54:35 CST] 5.9KiB STANDARD hello-world-5mjgp/hello-world-5mjgp-scan-4101005739/main.log
[2022-12-09 10:53:51 CST]   435B STANDARD hello-world-5mjgp/hello-world-5mjgp-test-1532501286/main.log
```

### 参考链接

* [支持的外部存储类型](https://argoproj.github.io/argo-workflows/configure-artifact-repository/)
* [官方文档](https://argoproj.github.io/argo-workflows/configure-archive-logs/)

### 小结

通过上面的例子，我们可以看到：

* Argo Workflows 能以非侵入式的配置，使得工作流日志输出到对象存储等外部存储中
* Argo Workflows 的任务有输入、输出（input、output）的概念，日志的持久化是将日志作为输出写入到预先配置好的外部存储
* 日志的持久化，可以分别在全局 ConfigMap、Workflow Spec、WorkflowTemplate 中配置

## 引用已有模板中的任务

Argo Workflows 允许以三种方式引用已有的任务：

* 当前工作流
* 当前命名空间中的工作流模板
* 全局（Cluster 级别）的工作流模板

这相当于 Java、Golang 等编程语言中的引用方式，分别可以引用：当前源文件、当前包、其他包下的函数。

```yaml
apiVersion: argoproj.io/v1alpha1
kind: WorkflowTemplate
metadata:
  name: hello-world
spec:
  templates:
  - name: hd
    dag:
      tasks:
      - name: hd
        templateRef:            # 表示引用其他模板中的任务
          name: hook            # 模板名称
          template: hook        # 模板中的任务名称
          clusterScope: true    # 为 true 是从全局（Cluster）中查找模板，为 false 时从当前命名空间中查找
        arguments:
          parameters:           # 给所引用的任务传递参数
          - name: pr
            value: 1
```

### 小结

我们可以将公用的模板作为模板库，供工作流调用，这样就可以使得工作流变得简单。

## 任务模板类型

|||
|---|---|
| 容器 | 指定单个容器 |
| 脚本 ||
| [容器集合](https://argoproj.github.io/argo-workflows/container-set-template/) | 支持多个容器 |
| [Directed-Acyclic Graph (DAG)](https://argoproj.github.io/argo-workflows/walk-through/dag/) | 支持指定任务之间的依赖关系，默认会尽可能地并发执行。 |
| [HTTP](https://argoproj.github.io/argo-workflows/http-template/) | 支持发送 HTTP 请求 |
| 资源 | 直接操作 Kubernetes 资源 |

## SSO
TODO

## 插件机制
Argo Workflows 内置了[几种类型的任务模板](#任务模板类型)，这些任务类型或是方便解决特定问题，或是可以解决通用问题。此外，我们还可以通过[执行器（Executor）插件](https://argoproj.github.io/argo-workflows/plugins/)扩展 Argo Workflows 的功能。

执行器插件，会作为工作流 Pod 中 sidecar 的形式存在，通过 HTTP 提供服务。Argo Workflows 规定了 URI，以及 Request 和 Response。据此，我们可以看出来插件的几个特点：

* 插件可以用任何编程语言实现
* 执行插件任务时无需启动新的 Pod，减少了对 Pod 的消耗

该插件功能默认是未启用的，我们可以在控制器（Controller）中添加环境变量的方式启用插件功能。请参考如下配置：

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: workflow-controller
spec:
  template:
    spec:
      containers:
        - name: workflow-controller
          env:
            - name: ARGO_EXECUTOR_PLUGINS
              value: "true"
```

安装插件时，只需要添加一个 ConfigMap 即可。例如：

```yaml
apiVersion: v1
data:
  sidecar.automountServiceAccountToken: "false"
  sidecar.container: |
    args:
    - --provider
    - gitlab
    image: ghcr.io/linuxsuren/workflow-executor-gogit:master
    command:
    - workflow-executor-gogit
    name: gogit-executor-plugin
    ports:
    - containerPort: 3001
    resources:
      limits:
        cpu: 500m
        memory: 128Mi
      requests:
        cpu: 250m
        memory: 64Mi
    securityContext:
      allowPrivilegeEscalation: true
      runAsNonRoot: true
      runAsUser: 65534
kind: ConfigMap
metadata:
  labels:
    workflows.argoproj.io/configmap-type: ExecutorPlugin
  name: gogit-executor-plugin
  namespace: argo
```

我们可以把上面的 ConfigMap 添加到 Argo Workflows 控制器所在的命名空间中，也可以添加到执行工作流所在的命名空间中。另外，当存在多个同名的插件时，会以工作流所在命名空间的插件为主。

插件安装成功的话，你可以在控制器中查看到类似如下的日志输出：

```
level=info msg="Executor plugin added" name=gogit-executor-plugin
```

插件的使用方法如下：

```yaml
apiVersion: argoproj.io/v1alpha1
kind: WorkflowTemplate
metadata:
  name: plugin
  namespace: default
spec:
  entrypoint: main
  hooks:
    exit:
      template: status
    all:
      template: status
      expression: "true"
  templates:
  - container:
      args:
        - search
        - kubectl
      command:
        - hd
      image: ghcr.io/linuxsuren/hd:v0.0.70
    name: main
  - name: status
    plugin:
      gogit-executor-plugin:                    # 下面支持任何格式给插件传递参数
        owner: linuxsuren
        repo: test
        pr: "3"
```

这里有[更多社区维护的插件](https://argoproj.github.io/argo-workflows/plugin-directory/)，有通过 Python、Golang、Rust 等语言实现的。

如果你想了解如何开发一个插件，可以继续往后阅读。下面介绍插件机制对 HTTP 的请求、响应的规定：

* Request payload 中可以解析到与当前工作流的信息，包括：名称、命名空间、插件参数
  * 我们可以参考 [ExecuteTemplateArgs](https://github.com/argoproj/argo-workflows/blob/774bf47ee678ef31d27669f7d309dee1dd84340c/pkg/plugins/executor/template_executor_plugin.go#L19) 来解析请求
* Response 需要告知任务执行的状态
  * 我们可以参考 [ExecuteTemplateReply](https://github.com/argoproj/argo-workflows/blob/774bf47ee678ef31d27669f7d309dee1dd84340c/pkg/plugins/executor/template_executor_plugin.go#L32) 作为 HTTP 响应的数据

## 归档
Argo Workflow 支持将工作流执行记录（Workflow）的信息存储到 PostgreSQL 或 MySQL 中，以达到更长久地保存执行记录但又不会影响到
Kubernetes 集群的性能。

这里，给出一个归档（ [Archive](https://argoproj.github.io/argo-workflows/workflow-archive/) ）数据到 PostgreSQL 的配置方法：

首先，安装 [PostgreSQL](https://www.postgresql.org/) 。这里采用 Helm Chart 的方式来安装：
```shell
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update
cat > values.yaml <<EOF
auth:
  enablePostgresUser: true
  postgresPassword: "StrongPassword"
  username: "root"
  password: "root"
  database: "app_db"
EOF
helm install postgresql-dev -f values.yaml bitnami/postgresql
```

Argo Workflows 会以 Secret 的方式读取数据库的用户名、密码，下面是创建 Secret 的命令：
```shell
kubectl create secret generic --from-literal=username=root --from-literal=password=root argo-postgres-config -n argo
```

然后，参考下面的 ConfigMap 启用工作流的归档功能：
```yaml
apiVersion: v1
data:
  persistence: |
    archive: true
    postgresql:
      host: postgresql-dev.argocd.svc
      port: 5432
      database: app_db
      tableName: argo_workflows
      userNameSecret:
        name: argo-postgres-config
        key: username
      passwordSecret:
        name: argo-postgres-config
        key: password
kind: ConfigMap
metadata:
  name: workflow-controller-configmap
  namespace: argo
```

上面的配置步骤都完成，执行工作流后，我们可以在 UI 界面左侧菜单上看到归档的执行记录。也可以通过数据库命令行客户端连接数据库，查看数据的表记录信息：
```shell
export POSTGRES_PASSWORD=root
kubectl run postgresql-dev-client --rm --tty -i --restart='Never' --namespace default --image docker.io/bitnami/postgresql:14.1.0-debian-10-r80 --env="PGPASSWORD=$POSTGRES_PASSWORD" --command -- psql --host postgresql-dev.argocd.svc -U root -d app_db -p 5432
```

下面是一些 PostgreSQL 命令行客户端的参考：
```shell
\dt                                                   # 查看当前数据库中的表
select name,phase from argo_archived_workflows;       # 查看已归档的工作流执行记录
```
你会看到类似如下的输出：
```sql
app_db=> \dt
                    List of relations
 Schema |              Name              | Type  | Owner
--------+--------------------------------+-------+-------
 public | argo_archived_workflows        | table | root
 public | argo_archived_workflows_labels | table | root
 public | argo_workflows                 | table | root
 public | schema_history                 | table | root
(4 rows)

app_db=> select name,phase from argo_archived_workflows;
     name     |   phase
--------------+-----------
 plugin-pl6rx | Succeeded
 plugin-8gs7c | Succeeded
```

## GC
Argo Workflows 有个工作流执行记录（Workflow）的清理机制，也就是 Garbage Collect(GC)。GC 机制可以避免有太多的执行记录，
防止 Kubernetes 的后端存储 Etcd 过载。

我们可以在 ConfigMap 中配置期望保留的工作执行记录数量，这里支持为不同状态的执行记录设定不同的保留数量。配置方法如下：

```yaml
apiVersion: v1
data:
  retentionPolicy: |
    completed: 3
    failed: 3
    errored: 3
kind: ConfigMap
metadata:
  name: workflow-controller-configmap
  namespace: argo
```

需要注意的是，这里的清理机制会将多余的 Workflow 资源从 Kubernetes 中删除。如果希望能更多历史记录的话，建议启用并配置好归档功能。

## Golang SDK
Argo Workflows 官方[维护了 Golang、Java、Python 语言](https://argoproj.github.io/argo-workflows/client-libraries/)的 SDK。下面以 Golang 为例，讲解 SDK 的使用方法。

在运行下面的示例前，有两点需要注意的：

* Argo Workflows Server 地址
* Token

你可以选择直接使用 `argo-server` 的 Service 地址，将端口 `2746` 转发到本地，或将 Service 修改为 `NodePort`，或者其他方法暴露端口。也可以执行下面的命令，再启动一个 Argo 服务：

```shell
argo server
```

第二个，就是用户认证的问题了。如果你对 Kubernetes 认证系统非常熟悉的话，可以跳过这一段，直接找一个 Token。为了让你对 Argo 的用户认证更加了解，我们为下面的测试代码创建一个新的 ServiceAccount。

我们需要分别创建：

* Role，规定可以对哪些资源有哪些操作权限

```shell
kubectl create role demo --verb=get,list,update,create --resource=workflows.argoproj.io --resource=workflowtemplates.argoproj.io -n default
```

* ServiceAccount，代表一个用户

```shell
kubectl create serviceaccount demo -n default
```

* RoleBinding，将用户和角色（Role）进行绑定

```shell
kubectl create rolebinding demo --role=demo --serviceaccount=default:demo -n default
```

* Secret，关联一个 ServiceAccount，并自动生成 Token

```shell
kubectl apply -n default -f - <<EOF
apiVersion: v1
kind: Secret
metadata:
  name: demo.service-account-token
  annotations:
    kubernetes.io/service-account.name: demo
type: kubernetes.io/service-account-token
EOF
```

> 上面的例子中，我们使用的是 `Role` 和 `RoleBinding` ，这样的角色只能允许访问所在命名空间（namespace）的资源。上面创建的用户，只能够访问 `default` 这命名空间下的 `Workflow` 和 `WorkflowTemplate` 。
> 如果想要创建一个全局的角色以及绑定，可以使用 `ClusterRole` 和 `ClusterRoleBinding` 。

上面的用户创建完成后，我们就可以通过下面的命令拿到指定权限的 `Token` 了：

```shell
kubectl get secret -n default demo.service-account-token -ojsonpath={.data.token}|base64 -d
```

接下来，创建一个 Golang 工程，并将下面的示例代码拷贝到源文件 `main.go` 中。

```shell
mkdir demo
cd demo
go mod init github.com/linuxsuren/demo
go get github.com/argoproj/argo-workflows/v3@v3.4.4
go mod tidy
```

示例代码：

```golang
package main

import (
	"fmt"
	"github.com/argoproj/argo-workflows/v3/pkg/apiclient"
	"github.com/argoproj/argo-workflows/v3/pkg/apiclient/workflow"
	"github.com/argoproj/argo-workflows/v3/pkg/apiclient/workflowtemplate"
	"github.com/argoproj/argo-workflows/v3/pkg/apis/workflow/v1alpha1"
	metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
)

/**
** Before run this demo, please create a parameterless WorkflowTemplate in namespace default.
** In this demo, we will print all the WorkflowTemplates in namespace default.
** Then run a Workflow base on the first WorkflowTemplate.
 */

func main() {
	opt := apiclient.Opts{
		ArgoServerOpts: apiclient.ArgoServerOpts{
			URL:                "localhost:31808", // argo-server address
			Path:               "/",
			Secure:             true,
			InsecureSkipVerify: true,
		},
		AuthSupplier: func() string {
			return "Bearer your-token"
		},
	}
	ctx, client, err := apiclient.NewClientFromOpts(opt) // the context will carry on auth
	if err != nil {
		panic(err)
	}

	wftClient, err := client.NewWorkflowTemplateServiceClient()
	if err != nil {
		fmt.Println("failed to get the WorkflowTemplates client", err)
		return
	}
	defaultNamespace := "default"

	fmt.Println("get the WorkflowTemplate list from", defaultNamespace)
	wftList, err := wftClient.ListWorkflowTemplates(ctx, &workflowtemplate.WorkflowTemplateListRequest{
		Namespace: defaultNamespace,
	})
	if err != nil {
		fmt.Println("failed to list WorkflowTemplates", err)
		return
	}
	for _, wft := range wftList.Items {
		fmt.Println(wft.Namespace, wft.Name)
	}

	if wftList.Items.Len() > 0 {
		wft := wftList.Items[0]

		wfClient := client.NewWorkflowServiceClient()
		_, err := wfClient.CreateWorkflow(ctx, &workflow.WorkflowCreateRequest{
			Namespace: defaultNamespace,
			Workflow: &v1alpha1.Workflow{
				ObjectMeta: metav1.ObjectMeta{
					GenerateName: wft.Name,
				},
				Spec: v1alpha1.WorkflowSpec{
					WorkflowTemplateRef: &v1alpha1.WorkflowTemplateRef{
						Name: wft.Name,
					},
				},
			},
		})
		if err != nil {
			fmt.Println("failed to create workflow", err)
		}
	}
}
```

最后，执行命令：`go run .`

> 把上面的示例代码编译后，二进制文件大致在 60M+

## References
* [DevOps Practice Guide](https://github.com/LinuxSuRen/devops-practice-guide)
* [Argo CD Guide](https://github.com/LinuxSuRen/argo-cd-guide)
* [Argo Rollouts Guide](https://github.com/LinuxSuRen/argo-rollouts-guide)
