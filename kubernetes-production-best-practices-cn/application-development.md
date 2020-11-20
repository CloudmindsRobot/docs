# 应用程序开发

基于Kubernetes的应用程序开发最佳实践。

## 健康检查

Kubernetes提供了两种机制来跟踪容器和pod的生命周期：活动性和就绪性探测

**就绪探针确定容器何时可以接收流量请求。**

kubelet执行检查并决定应用程序是否可以接收流量请求。

**存活探针确定何时应该重新启动容器。**

kubelet执行检查并决定是否应该重新启动容器。

**资源:**

- Kubernetes的官方文档提供了一些关于如何配置存活探针、就绪探针和启动探测的实用建议。 [configure Liveness, Readiness and Startup Probes](https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/).
- [Liveness probes are dangerous](https://srcco.de/posts/kubernetes-liveness-probes-are-dangerous.html) 有一些关于如何对就绪探针设置（或不设置）依赖关系的信息。

### 容器要有就绪探针

> 请注意，对于就绪探针和存活探针，没有默认值。

如果不设置就绪探测器，kubelet会假设应用程序在容器启动时就可以接收流量请求。

如果容器启动需要2分钟，那么在这2分钟内，对该容器的所有请求都将失败。

### 发生致命错误时容器会崩溃挂掉

如果应用程序遇到不可恢复的错误，应该让它崩溃挂掉。[you should let it crash](https://blog.colinbreck.com/kubernetes-liveness-and-readiness-probes-revisited-how-to-avoid-shooting-yourself-in-the-other-foot/#letitcrash).

像如下此类不可恢复的错误：

- 一个未被捕捉的异常
- 代码中的拼写错误（对于动态语言）
- 无法加载的头文件或依赖库

请注意，您不应向存活探针探测失败的容器发送信号。

相反，您应该让容器立即退出进程并让kubelet重新启动它。

### 配置一个检测容器存活的探针

存活探针设计的目的是用于在容器卡住（假死）时重新启动它。

考虑以下场景：如果应用程序正在处理一个无限循环，程序无法退出或未发出告警信号。

则当进程消耗100%的CPU时，它将没有时间响应（其他）就绪探针检查，它最终将从服务中删除。

但是，Pod仍然注册在当前部署的活动副本中。

如果没有存活探针，Pod将保持运行，但它与进程服务已经脱离。

换句话说，进程已经不仅不提供任何请求了，而且还在消耗资源。

_你应该怎么办呢?_

1. 从应用程序暴露一个请求端点
1. 请求端点总是以成功响应进行回复
1. 使用存活探针探测服务端点

请注意，您不应该使用存活探针来处理应用程序中的致命错误，并请求Kubernetes重新启动应用程序。

相反，你应该让应用崩溃挂掉。

只有在进程没有响应的情况下，才将存活探针用作程序恢复机制。

### 存活探针的配置值不能和就绪探针的值一致

当存活探针和就绪探测指向同一个端点时，两个探测的效果会相结合。

当应用程序发出信号说它还没有准备好或者还没有激活时，kubelet会将容器从服务中分离出来，同时删除它。

您可能会注意到断开连接，因为容器没有足够的时间来排空当前连接或处理完传入的连接。

您可以在阅读 [article that discussed graceful shutdown](https://freecontent.manning.com/handling-client-requests-properly-with-kubernetes/)进行更深入的研究。

## 应用程序是独立的

只有当所有依赖项（如数据库或后端API）准备就绪时，您可能会试图发出应用程序准备就绪的信号。

如果应用程序连接到数据库，您可能会认为在数据库准备就绪之前就绪探针探测返回失败是一个好主意，事实并非如此。

考虑以下场景：有一个前端应用程序依赖于后端API。

如果API是不稳定的（例如，由于一个错误，它有时不可用），就绪探针探测失败，使用就绪探针的前端应用程序也会失败。

那么你会有停服时间。

更进一步地说，下游依赖关系的失败可能会传播到上游的所有应用程序，最终会导致面向多层次前端的崩溃。

### 就绪探针是独立的

就绪探针不探测服务的依赖关系，例如：

- 数据库依赖
- 数据库迁移
- APIs
- 第三方服务

请可以阅读[explore what happens when there're dependencies in the readiness probes in this essay](https://blog.colinbreck.com/kubernetes-liveness-and-readiness-probes-how-to-avoid-shooting-yourself-in-the-foot/#shootingyourselfinthefootwithreadinessprobes).

### 应用程序将重试去连接到依赖的服务

当应用程序启动时，不应该因为数据库等依赖服务尚未准备好而崩溃。

相反，应用程序应该继续尝试连接到数据库，直到成功。

Kubernetes期望应用程序组件可以以任何顺序启动。

当你确保你的应用程序可以去重新连接到一个依赖服务，比如数据库，你就知道你可以提供一个更加健壮和有弹性的服务。

## 平滑关闭

你不应该直接关闭您的应用程序。

相反，您应该等待现有连接耗尽并停止处理新的连接后再关闭。

请注意，当Pod终止时，该Pod的端点将从服务中移除。

但是，在kube-proxy或入口控制器等组件收到更改通知之前，可能需要一些时间进行现有请求处理。

您可以在Kubernetes中找到关于平滑关闭如何正确处理客户机请求的详细说明 [handling client requests correctly with Kubernetes](https://freecontent.manning.com/handling-client-requests-properly-with-kubernetes/).

正确的平滑关闭顺序是：

1. 收到SIGTERM后
1. 服务器停止接受新连接
1. 完成所有活动请求
1. 然后立即终止所有keepalive连接并退出进程

你可以用这个工具测试你的应用程序是否正常关闭：[test that your app gracefully shuts down with this tool: kube-sigterm-test](https://github.com/mikkeloscar/kube-sigterm-test).

### 应用程序不应该在SIGTERM上关闭，而它应该在优雅地终止了连接后再进行关闭

当端点服务（pod）变更在发送变更信号给k8s组件（如：kube-proxy或者 Ingress controller）之前可能需要一些时间。

因此，尽管Pod被标记为已终止，但流量仍可能流向Pod。

应用程序应该停止接受所有剩余连接上的新请求，并在外部请求队列清空后关闭这些请求。

如果您需要了解端点服务是如何在集群中传播的，请阅读[read this article on how to handle client requests properly](https://freecontent.manning.com/handling-client-requests-properly-with-kubernetes/).

### 应用程序在宽限期内仍需处理传入的请求

您可能需要考虑使用容器生命周期事件[the preStop handler](https://kubernetes.io/docs/tasks/configure-pod-container/attach-handler-lifecycle-event/#define-poststart-and-prestop-handlers) 去定制一些事件来追踪Pod被删除之前发生的事情。

### Dockerfile中的CMD将SIGTERM转发给业务进程

当Pod即将终止时，可以通过在应用程序中捕获SIGTERM信号来通知您。

您可以关注[forwarding the signal to the right process in your container](https://pracucci.com/graceful-shutdown-of-kubernetes-pods.html).

### 关闭所有空闲的活动连接

如果主呼叫应用程序正在进行TCP连接（例如，使用TCP keep alive或一个连接池），它只连接到该服务的一个pod，而该服务的其他pod不会被使用。

_但是当Pod被删除时会发生什么呢？_

理想情况下，请求应该转到另一个Pod。

然而，主呼叫应用程序与即将终止的Pod有一个长连接，它将继续使用它。

您不应该突然关闭应用程序。

相反，您应该在关闭应用程序之前先终止长连接。

您可以阅读 [gracefully shutting down a Nodejs HTTP server](http://dillonbuchanan.com/programming/gracefully-shutting-down-a-nodejs-http-server/)了解更多。

## 容错

您的群集节点可能随时消失，原因如下：

- 物理机器的硬件故障
- 云提供器或虚拟机管理器故障
- 内核错误

部署在这些节点上的pod也会丢失。

此外，还有其他可能删除Pods的情况，如：

- 直接删除Pod（事故）
- 排空节点
- 从一个节点上移除一个Pod以允许另一个Pod固定在该节点上

以上任何情况都可能影响应用程序的可用性，并可能导致停服。

您应该避免出现所有的Pod都不可用的情况，导致无法提供实时流量请求。

### 服务部署运行多个副本

不要单独运行一个Pod。

相反，您应该可虑将Pod放到Deployment, DaemonSet, ReplicaSet or StatefulSet中进行部署。

运行多个Pods实例可以保证删除单个Pod不会导致停止服务，您可以阅读[Running more than one instance your of your Pods guarantees that deleting a single Pod won't cause downtime](https://cloudmark.github.io/Node-Management-In-GKE/#replicas).

### 避免同一个服务的pods分配到同一个节点服务器上运行

**即使运行多个Pods副本，也不能保证丢失一个节点不会导致服务中断。**

考虑以下场景：在一个单节点的集群上运行11个Pods副本。

如果节点不可用，这11个Pods副本将丢失，您将有停止服务时间。

您应该对应用部署使用反亲和性规则，这样pods就可以分布在集群的所有节点上，请看[You should apply anti-affinity rules to your Deployments so that Pods are spread in all the nodes of your cluster](https://cloudmark.github.io/Node-Management-In-GKE/#pod-anti-affinity-rules).

The [inter-pod affinity and anti-affinity](https://kubernetes.io/docs/concepts/configuration/assign-pod-node/#inter-pod-affinity-and-anti-affinity) 文档描述了如何将pod更改为调度（或不调度）到同一节点中。

### 做好Pod中断预算

当一个节点被排空时，该节点上的所有pod都将被删除并重新调度。

_但是如果你在重负载之下，你不能失去超过50%的Pods呢？_

节点排空事件可能会影响您的可用性。

为了保护部署服务不受意外事件的影响，这些意外事件可能同时导致多个Pod崩溃，您可以定义Pod中断预算。

想象一下这样说：“Kubernetes，请确保至少有5个Pods在为我的应用程序运行”。

如果最终状态导致部署的pod少于5个，Kubernetes将阻止节点排空事件。

请参考官方文档 [Pod Disruption Budgets](https://kubernetes.io/docs/concepts/workloads/pods/disruptions/).

## 资源利用

你可以把Kubernetes看作是一个熟练的俄罗斯方块游戏。

Docker容器是块；服务器是板，调度程序是玩家。
![Kubernetes is the best Tetris player](tetris.svg)

为了最大限度地提高调度程序的效率，您应该与Kubernetes共享诸如资源利用率、工作负载优先级和开销等详细信息。

### 为所有容器设置内存限制和请求

使用containerSpec的resources属性容器资源进行设置，用于限制容器可以使用的CPU和内存量。

调度器使用这些作为度量标准之一，以确定哪个节点最适合当前的Pod。

根据调度程序的默认规则，没有内存限制的容器的内存使用率为零。

如果可在任何节点上调度无限数量的Pod，则会导致节点资源超量使用甚至会导致节点（以及kubelet）崩溃。

这同样适用于CPU限制。

_但是你应该总是设置内存和CPU的限制和请求吗？_

是，也不是。

如果进程超过内存限制，进程将被终止。

CPU限制很难，由于CPU是可压缩的资源，因此如果容器超出限制，则会限制进程。

即使它可以使用一些当时可用的CPU。

**[CPU limits are hard.](https://www.reddit.com/r/kubernetes/comments/cmp7jj/multithreading_in_a_container_with_limited/ew52fcj/)**s

如果您希望深入了解CPU和内存限制，请参阅以下文章：

- [Understanding resource limits in kubernetes: memory](https://medium.com/@betz.mark/understanding-resource-limits-in-kubernetes-memory-6b41e9a955f9)
- [Understanding resource limits in kubernetes: cpu time](https://medium.com/@betz.mark/understanding-resource-limits-in-kubernetes-cpu-time-9eff74d3161b)

> 请注意，如果您不确定什么是正确的CPU或内存限制，您可以使用Kubernetes中打开推荐模式参考[Vertical Pod Autoscaler](https://github.com/kubernetes/autoscaler/tree/master/vertical-pod-autoscaler) 分析你的应用程序，并为它设置合理的限制。

### 将CPU请求设置为1 CPU或更低

除非你有密集型计算的工作，否则推荐将请求设置为1个CPU或更低，请参阅[it is recommended to set the request to 1 CPU or below](https://www.youtube.com/watch?v=xjpHggHKm78).

### 禁用CPU限制-除非你有一个好的用例

CPU是以每个时间单位的CPU时间单位来衡量的。

cpu:1表示每秒1个cpu秒。

如果你有一个线程，你不能消耗超过每秒1个CPU。

如果您有2个线程，您可以在0.5秒内消耗1cpu秒。

8个线程可以在0.125秒内消耗1cpu秒。

之后，您的进程将被限制。

如果你不确定应用程序的最佳设置是什么，最好不要设置CPU限制。

如果您想了解更多请阅读 [this article digs deeper in CPU requests and limits](https://medium.com/@betz.mark/understanding-resource-limits-in-kubernetes-cpu-time-9eff74d3161b).

### 命名空间具有LimitRange

如果您认为可能会忘记去设置内存和CPU的限制，那么应该考虑使用LimitRange来定义部署在当前命名空间中的容器的标准资源大小。

关于LimitRange的官方文档是一个很好的起点，请参阅[The official documentation about LimitRange](https://kubernetes.io/docs/concepts/policy/limit-range/) 。

### 为POD设置适当的服务质量（QoS）

当一个节点进入超负载状态（即使用过多的资源）时，Kubernetes会尝试驱逐该节点中的一些Pod。

Kubernetes按照一套定义好的逻辑对Pods进行排序和驱逐。

在官方文档您能了解更多 [configuring the quality of service for your Pods](https://kubernetes.io/docs/tasks/configure-pod-container/quality-service-pod/)。

## 标记资源

标签是没有任何预定义含义的键值对。

它们可以应用于集群中从pod到服务、入口清单、服务端点等的所有资源。

可以使用标签按用途、所有者、环境或其他条件对资源进行分类。

因此，您可以选择一个标签来标记一个Pod，比如：“这个Pod运行在生产环境”或“哪个团队拥有该部署”。

您也可以完全省略标签。

但是，您可能需要考虑使用标签来覆盖以下类别：

- 技术标签，如环境
- 自动化标签
- 与您的业务相关的标签，如成本中心分配
- 与安全相关的标签，如规则遵从要求

### 资源定义了技术标签

您可以对Pods打标记，例如：

- 名称，应用程序的名称，如“用户API”
- 实例，标识应用程序实例的唯一名称（可以使用容器镜像标记）
- 版本，appl的当前版本（增量计数器）
- 组件，体系结构中的组件，如“API”或“database”
- 属主，上级应用程序的名称，例如“支付网关”
- 管理者，用于管理应用程序操作的工具，如“kubectl”或“Helm”

下面是一个如何在部署中使用此类标签的示例：

```yaml|highlight=6-11,20-24|title=deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: deployment
  labels:
    app.kubernetes.io/name: user-api
    app.kubernetes.io/instance: user-api-5fa65d2
    app.kubernetes.io/version: "42"
    app.kubernetes.io/component: api
    app.kubernetes.io/part-of: payment-gateway
    app.kubernetes.io/managed-by: kubectl
spec:
  replicas: 3
  selector:
    matchLabels:
      application: my-app
  template:
    metadata:
      labels:
        app.kubernetes.io/name: user-api
        app.kubernetes.io/instance: user-api-5fa65d2
        app.kubernetes.io/version: "42"
        app.kubernetes.io/component: api
        app.kubernetes.io/part-of: payment-gateway
    spec:
      containers:
      - name: app
        image: myapp
```

这些标签是官方文件推荐的 [recommended by the official documentation](https://kubernetes.io/docs/concepts/overview/working-with-objects/common-labels/).

> 这不是推荐您标记所有资源。

### 资源已定义业务标签

您可以对Pods打标记，例如：

- 所有者，用于标识谁负责该资源
- 项目，用于确定资源所属的项目
- 业务单元，用于标识与资源相关联的成本中心或业务单元；通常用于成本分配和跟踪

下面是一个如何在部署中使用此类标签的示例：

```yaml|highlight=6-8,17-19|title=deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: deployment
  labels:
    owner: payment-team
    project: fraud-detection
    business-unit: "80432"
spec:
  replicas: 3
  selector:
    matchLabels:
      application: my-app
  template:
    metadata:
      labels:
        owner: payment-team
        project: fraud-detection
        business-unit: "80432"
    spec:
      containers:
      - name: app
        image: myapp
```

您可以在AWS标记策略页面上浏览资源的标签和标记[tagging for resources on the AWS tagging strategy page](https://aws.amazon.com/answers/account-management/aws-tagging-strategies/).

该文并不是专门针对Kubernetes的，而是探讨了一些最常见的标记资源的策略。

> 这不是推荐您标记所有资源。

### 资源已定义安全标签

您可以对Pods打标记，例如：

- 机密性，一些资源支持的特定数据机密性级别的标识符
- 规则遵从，工作负载的标识符，旨在遵守特定的规则遵从要求

下面是一个如何在部署中使用此类标签的示例：

```yaml|highlight=6-11,20-24|title=deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: deployment
  labels:
    confidentiality: official
    compliance: pci
spec:
  replicas: 3
  selector:
    matchLabels:
      application: my-app
  template:
    metadata:
      labels:
        confidentiality: official
        compliance: pci
    spec:
      containers:
      - name: app
        image: myapp
```

您可以在AWS标记策略页面上浏览资源的标签和标记 [tagging for resources on the AWS tagging strategy page](https://aws.amazon.com/answers/account-management/aws-tagging-strategies/).

该文并不是专门针对Kubernetes的，而是探讨了一些最常见的标记资源的策略。

> 这不是推荐您标记所有资源。

## 日志

应用程序日志可以帮助您了解应用程序内部发生的事情。

日志对于调试问题和监控应用程序活动特别有用。

### 应用程序沿stdout和stderr日志

有两种日志策略：_被动_和_主动_。

使用被动日志的应用程序不知道日志基础设施，并将日志记录到标准输出。

此最佳实践是十二要素应用程序的一部分[the twelve-factor app](https://12factor.net/logs)。

在主动日志中，应用程序与中间聚合器建立网络连接，向第三方日志服务发送数据，或直接写入数据库或索引。

主动日志记录被认为是一种反模式，应该避免。

### 避免边车日志（如果可以的话）

如果希望将日志转换应用于具有非标准日志事件模型的应用程序，则可能需要使用sidecar容器 [apply log transformations to an application with a non-standard log event model](https://rclayton.silvrback.com/container-services-logging-with-docker#effective-logging-infrastructure)。

使用边车容器，您可以在将日志条目运送到其他位置之前将其规范化。

例如，在将Apache日志发送到日志基础设施之前，您可能希望将Apache日志转换为Logstash JSON格式。

然而，如果您可以控制应用程序，则可以首先输出正确的格式。

您可以节省为集群中的每个机架运行额外的容器。

## 弹性伸缩

### 容器不在其本地文件系统中存储任何状态

容器有一个本地文件系统，您可能会想用它来保存数据。

然而，在容器的本地文件系统中存储持久性数据会阻止Pod的水平伸缩（即通过添加或删除Pod的副本）。

这是因为，通过使用本地文件系统，每个容器都保持自己的“状态”存放在本地，这意味着Pod副本的状态可能会随着时间的推移而发生变化。

从用户的角度来看，这会导致不一致的行为（例如，当请求命中一个Pod时，特定的用户信息是可用的，而当请求到达另一个Pod时则不可用）。

相反，任何持久性信息都应该保存在Pods外部的中心位置。例如，在集群中的存储卷中，或者更好的是在集群外的某个存储服务中。

### 使用水平自动伸缩的Pod对于需要使用动态模式的应用程序

[Horizontal Pod Autoscaler (HPA)](https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/) 水平自动伸缩器是Kubernetes内置的一个特性，它监控应用程序并根据当前使用情况自动添加或删除Pod副本。

配置HPA可以让你的应用程序在任何流量的情况下（包括意外的峰值）保持可用和响应。

要将HPA配置为可自动伸缩的应用程序，您必须创建一个水平自动伸缩器资源[HorizontalPodAutoscaler](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.16/#horizontalpodautoscaler-v1-autoscaling) ，该资源定义应用程序要监控的指标。

HPA可以监控内置的资源指标（Pods的CPU和内存使用情况）或自定义指标。在定制度量的情况下，您还负责收集和展示这些度量，您可以这样做，例如，使用[Prometheus](https://prometheus.io/) and the [Prometheus Adapter](https://github.com/DirectXMan12/k8s-prometheus-adapter)。

### 不要使用垂直自动伸缩器，因为它还处于测试阶段

与水平自动伸缩器 [Horizontal Pod Autoscaler (HPA)](https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/)类似，也存在垂直自动伸缩器 [Vertical Pod Autoscaler (VPA)](https://github.com/kubernetes/autoscaler/tree/master/vertical-pod-autoscaler).

VPA可以自动调整Pod的资源请求和限制，以便当Pod需要更多资源时，它可以获得它们（增加/减少单个Pod的资源称为垂直扩展，而不是水平扩展，这意味着增加/减少Pod的副本数量）。

这对于需要伸缩但不能使用水平伸缩的应用程序非常有用。

H然而，VPA目前还处于beta测试阶段，它有一些已知的限制 [some known limitations](https://github.com/kubernetes/autoscaler/tree/master/vertical-pod-autoscaler#limitations-of-beta-version)（例如，通过改变资源需求来扩展一个Pod，需要关闭并重新启动Pod）。

考虑到这些限制，以及Kubernetes上的大多数应用程序都可以水平扩展，因此建议不要在生产中使用VPA（至少在有稳定版本之前）。

### 如果工作负载变化很大，请使用集群自动伸缩器

集群自动伸缩器 [Cluster Autoscaler](https://github.com/kubernetes/autoscaler/tree/master/cluster-autoscaler) 是另一种“自动伸缩器” (除了 [Horizontal Pod Autoscaler](https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/) 和 [Vertical Pod Autoscaler](https://github.com/kubernetes/autoscaler/tree/master/vertical-pod-autoscaler))。

集群自动伸缩器可以通过添加或删除工作节点来自动伸缩集群的节点数量。

如果由于现有工作节点上的资源不足而无法正常调度Pod，则会触发集群扩容操作。在本例中，集群自动缩放器创建一个新的工作节点，以便可以调度Pod。类似地，当现有工作节点的利用率较低时，集群自动缩放器可以通过从其中一个工作节点中逐出所有工作负载并将其移除来缩小集群节点规模。

对于高度可变的工作负载，使用集群自动伸缩是有意义的，例如，当pod的数量可能在短时间内成倍增加，然后返回到以前的值。在这种情况下，集群自动伸缩器允许您在不浪费资源的情况下通过过度配置工作节点来满足需求高峰。

但是，如果您的工作负载变化不大，那么可能不值得设置集群自动伸缩器，因为它可能永远不会被触发。如果您的工作负载增长缓慢且单调，那么监控现有工作节点的利用率并在它们达到临界值时手动添加一个额外的工作节点就足够了。


## 配置和机密文件（Secret）

### 配置外设

配置应该在应用程序代码之外维护。

这有几个好处。首先，更改配置不需要重新编译应用程序。其次，可以在应用程序运行时更新配置。第三，相同的代码可以在不同的环境中使用。

在Kubernetes中，配置可以保存在ConfigMaps中，当卷作为环境变量传入时，可以将其装入容器中。

仅在ConfigMaps中保存非敏感配置。对于敏感信息（如凭据），请使用Secret资源。

### 挂载Secret资源，而不是使用环境变量

Secret资源的内容应该作为卷装入容器中，而不是作为环境变量传入。

这是为了防止机密值出现在用于启动容器的命令中，而该容器可能由不应访问机密值的个人检查。

