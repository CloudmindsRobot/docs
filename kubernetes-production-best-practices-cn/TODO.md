### 为Kubernetes资源保留优先权和空间

可以指示Kubelet为系统和Kubernetes组件（Kubelet本身和Docker等）预留一定数量的资源。

保留资源从节点的可分配资源中减去。这样可以改进调度，并使资源分配/使用更加透明。

您可以[探索如何在官方文档中保留资源](https://kubernetes.io/docs/tasks/administer-cluster/reserve-compute-resources/#node-allocatable）。

### 处理长连接

<https://itnext.io/on-grpc-load-balancing-683257c5b7b3>
<https://kubernetes.io/blog/2018/11/07/grpc-load-balancing-on-kubernetes-without-tears/>

* * *

## 构建容器镜像

### 只使用可信的镜像

### 最小化镜像大小/构建 "scratch" 镜像

### 固定镜像标签

如果镜像的内容改变了，对应的镜像标签也必须改变。

通用的策略是使用Git的代码提交hash值或者CI的构建ID来作为镜像标签的一部分。这样可以与语义版本控制结合使用。

例如: `v1.0.1-bfeda01f`

**避免使用 `latest` 标签**

* * *

## 部署应用程序

### 使用一个进入控制来路由您的应用请求。

即使是简单的应用程序。

需要安装[Ingress controller](https://kubernetes.io/docs/concepts/services-networking/ingress-controllers/).

### 为你的应用程序设置一个Pod 破坏性预算

TODO:与“容错”（应用程序开发）中当前的“设置Pod中断预算”集成

### 所有不需要从集群外部访问的服务都应该是ClusterIP

### 使用TLS保护入口端点

* * *

## 用户管理

### 使用外部身份系统进行用户管理

例如，Azure Active Directory、AWS IAM。

使用持有令牌和API服务器的身份验证来验证外部服务令牌

* * *

##附加服务

### 运行您自己的容器仓库

TODO：这是最佳实践吗？

### 如果您使用Helm，请运行您自己的Helm chart 仓库

Chartmuseum, Artifactory

* * *

## Pod网络

TODO:集成当前的“网络策略”

### 使用NetworkPolicy限制pod之间的通信

### 在所有命名空间中创建拒绝所有网络策略

### 避免跨名称空间的机架间通信

* * *

## Pod 安全

TODO: 集成当前的"Pod安全策略" (治理)

### 使用PodSecurityPolicy在所有Pods中实施安全功能

- 使用PodSecurityPolicy需要启用PodSecurityPolicy许可控制器，但在大多数Kubernetes部署中，它没有启用。一旦启用了PodSecurityPolicy许可控制器，就需要适当的PodSecurityPolicy资源来允许创建任何Pods。

- 您还需要将对创建的podsecuritypolicy的“use”访问权授予工作负载的服务帐户或工作负载的控制器（您可以使用`system:serviceaccounts`访问所有控制器服务帐户的组）。

* * *

## 一般政策

TODO:集成当前的“自定义策略”（治理）

### 使用开放策略代理（OPA）和网关守卫器对所有资源强制执行自定义策略

只允许兼容的Kubernetes资源（任何类型）应用于集群（符合定义的策略）。

- 开放策略代理（OPA）：策略引擎
- 网关守卫器
- 正在验证许可控制webhook
- 用于安装、配置和管理开放策略代理策略的Kubernetes运算符

可以使用以下示例实施网关守卫策略：

- 不得在互联网上公开服务
- 只允许来自受信任容器仓库的容器
- 所有容器都必须有资源限制
- 入口主机名不能重叠
- 入口只能使用HTTPs

* * *
## 有状态的应用程序

### 如果可能的话，避免在Kubernetes中管理状态

使用集群外的存储服务（例如Amazon DynamodDB、Amazon S3）。

### 创建一个名为“default”的默认存储类

因为这个名称通常是Helm charts中的默认名称。

### 对于运行有状态应用程序使用operators比我们自己管理会更好

许多有状态的应用程序（例如数据库）都有操作程序，可以更容易地可靠地运行和管理它们。

* * *

## 准入控制器

### 使用官方推荐的准入控制器

建议启用准入控制器集:`NamespaceLifecycle, LimitRanger, ServiceAccount, DefaultStorageClass, DefaultTolerationSeconds, MutatingAdmissionWebhook, ValidatingAdmissionWebhook, Priority, ResourceQuota, PodSecurityPolicy`

###如果您使用多个转变过的准入控制webhook，请确保它们不会修改资源的相同字段

他们会互相干扰。此外，准入控制webhooks的运行顺序是未定义的。

### 如果您使用create a mutating admission webhook，那么也要创建一个验证这些突变的验证webhook

### 将发送到准入控制webhook的请求限制到最小

### 作用域准入控制webhooks到特定的名称空间

使用MutatingWebhookConfiguration/ValidatingWebhookConfiguration资源中的“namespaceSelector”字段。

### 始终将kube-system名称空间从自定义准入控制webhook的范围中排除

### 限制用于创建准入控制webhook配置的RBAC规则

限制对MutatingWebhookConfiguration/ValidatingWebhookConfiguration资源的“创建”。
* * * 

## 管理YAML资源清单

### 如果部署到多个环境，请使用模板系统

不要为不同的环境手动维护相同YAML文件的副本。使用像Helm，kustomize，Kapitan，ytt这样的模板系统。

通过定义一次YAML文件的公共结构并定义每个环境的替代值来工作。

* * *

## 日志

TODO：集成当前的“日志”（应用程序开发）和“日志设置”（集群配置）。

### 应用程序将日志写入stdout或stderr，而不是写入文件

这允许使用基于节点代理的日志聚合系统，而不是基于边车容器的系统。

容器写入stdout或stderr的所有内容都由容器运行时保存在已知位置。这意味着，在每个工作节点上运行的代理可以收集该节点上所有容器的日志并将其发送到中心日志存储。

如果容器将日志写入文件，则日志文件是容器的本地文件，节点代理无法访问日志文件。在这种情况下，每个Pod都需要一个边车容器，该容器收集这些日志并将其发送到中央日志存储区。

节点代理（例如，部署为守护进程）比每个Pod的边车容器更高效和更易于管理。

### 使用日志聚合系统

集群组件和应用程序将日志记录到它们的容器中，这意味着日志分布在整个集群中。这使得很难检查日志（可能是多个组件的日志）来解决问题。

日志聚合系统收集所有这些日志并将它们保存在集群中的中心位置，以便您可以在单个位置检查它们。

一些日志聚合工具包括：EFK堆栈（Elasticsearch、Fluentd、Kibana）、Loki、DataDog、Sumo Logic、Sysdig、GCP Stackdriver、Azure Monitor、AWS CloudWatch

### 使用托管日志聚合系统，而不是自托管系统（如果可能）

运行自己的日志聚合系统的操作开销可能相当大。这是因为对于运行日志聚合系统，您需要处理诸如持久性存储、备份、归档、日志轮换等事项。

所有的事情都可以用托管日志解决。

一些托管日志聚合系统包括：DataDog、Sumo Logic、Sysdig、GCP Stackdriver、Azure Monitor、AWS CloudWatch

### 定义日志保留和存档策略

日志在日志存储中保留多长时间以及以后如何处理它们？

一般来说，在日志储存库中保留30-45天的日志是一个合理的值。之后，如果日志仍然可用，您可以将它们移动到一个经济高效的归档存储（例如[Amazon Glacier](https://aws.amazon.com/glacier/)).

考虑到您所在组织的任何法规和内部合规性。

###从集群组件和应用程序收集日志

您的应用程序并不是在集群中生成日志的唯一组件—所有集群组件也会这样做，您也应该收集它们的日志。

以下是一些集群组件，您应该在日志聚合系统中包含它们的日志：

- 所有节点：kubelet、容器运行时
- 主节点：API服务器、调度程序、控制器管理器
- （Kubernetes审计（对API服务器的所有请求）

* * *

## 监控

### 在集群中设置广泛的监视

TODO：在这件事上有什么建议？

监视是从集群的不同组件收集和聚合度量。这对于深入了解集群的内部结构、评估其运行状况、检测和排除问题，甚至在问题发生之前加以预防，都是极其重要的。

监控包括两部分：

1. 组件进行度量并将其作为指标暴露

2. 监控系统定期收集这些指标

一些监控系统包括：

- 自托管：Prometheus
- 托管：DataDog、Sumo Logic、Sysdig、Google Stackdriver、Azure Monitor、Azure Monitor for Containers、AWS CloudWatch、AWS Container Insights

_你应该监控什么？_

确切地说，要收集哪些指标取决于集群中的组件（即它们公开了哪些指标）。

对于要收集的指标类型，有一些一般性的指导原则：

- 基础设施（例如节点）：使用指标-使用率、饱和度、错误
- 应用：红色指标-速率，错误，持续时间

### 如果您使用自托管监控系统，请在专用管理集群中运行它

您可以在生产集群中运行监视系统。但是，如果集群出现问题，监控系统也可能受到影响（您可能需要监控系统来解决集群的问题）。

为了避免这种情况，您可以创建一个专用集群，该集群只为所有其他集群运行管理工具（如监控系统）。

* * * 

## 告警

### 只对需要立即人工干预的事件发出告警

忽略不需要人立即采取行动的告警。太多不重要的告警会导致“告警疲劳”，并导致重要告警被忽略。

### 关注影响服务级别目标（SLO）的告警

告警的核心应该是那些对您向客户承诺的服务级别产生负面影响的事件。

### 自动修复所有非关键告警

所有不需要告警的事件（因为它们不需要立即的人为干预，或者它们不会影响客户体验）都应该自动处理（投资于自动化）。

* * *

## 配置和机密（Secret）

TODO:集成当前的“配置和机密（Secret）”

### 将所有配置与应用程序代码分开

配置应该在应用程序代码之外维护。

这有几个好处。首先，更改配置不需要重新编译应用程序。其次，可以在应用程序运行时更新配置。第三，相同的代码可以在不同的环境中使用。

在Kubernetes中，配置可以保存在ConfigMaps中，当卷作为环境变量传入时，可以将其装入容器中。

仅在ConfigMaps中保存非敏感配置。对于敏感信息（如凭据），请使用机密（Secret）资源。

### 在ConfigMaps中保存非关键配置，在Secrets中保存关键配置

机密类似于ConfigMaps，但是有一些特殊的语义来保护它们的内容（例如，机密的内容不会显示在某些kubectl输出中）

### 将机密（Secret）装载为卷，而不是环境变量

秘密（Secret）资源的内容应该作为卷装入容器中，而不是作为环境变量传入。

这是为了防止机密值出现在用于启动容器的命令中，而该容器可能由不应访问机密值的个人检查。

- 注入的环境变量始终存在，并且可能成为整个系统日志中的构件。
- 基于机密的环境变量应作为卷装入（而不是环境变量）。这样，它们只对所需的进程/容器可用。不是整个Pod。

见[推特](https://twitter.com/jeyfelbrandauer/status/1194752211366088704)

### 使用PodPresets自动将配置映射或机密装载到容器中

[PodPresets](https://kubernetes.io/docs/concepts/workloads/pods/podpreset/)

### 在ConfigMap和Secret中包含版本，以确保重新加载配置更改

例如，不要将配置映射命名为“config”，而只需将其命名为“config-v1”。每当您更新配置映射中的配置时，也要更新版本（例如，“config-v2”）。然后，将对ConfigMap的所有引用（例如，在部署中）更新为新版本（例如，“config-v2”）。

这将导致所有pod重新启动，从而确保新配置确实加载到Pods中，而不管ConfigMap是作为卷还是作为环境变量装载的，在后一种情况下，也不管应用程序是否监视配置文件以获取更改。

你可以用一个CD系统来自动化这个过程。

如果不删除ConfigMap的以前版本（例如“config-v1”），则可以轻松地回滚到以前的配置。

* * *

## 基于角色的访问控制（RBAC）

TODO:集成当前的“基于角色的访问控制（RBAC）策略”（治理）

### 角色遵循“RBAC最小特权原则”

### 不要对Pods使用“catch all”服务帐户

如果一个Pod需要访问Kubernetes API，那么定制一个RBAC角色，该角色允许Pod必须执行的操作（仅此而已），将其分配给一个新的服务帐户，并将此服务帐户分配给Pod。

不要为Pod使用现有服务帐户，因为该帐户可能具有比Pod所需权限更多的关联角色。

* * *

## 标记资源

TODO:集成当前的“标记资源”

### 所有资源都有一套通用的推荐标签

**是什么？**

[Kubernetes文档](https://kubernetes.io/docs/concepts/overview/working-with-objects/common-labels/)建议将一组公用标签应用于所有资源对象。

**为什么？**

拥有一组公共标签允许第三方工具与集群中的资源进行互操作。此外，一组通用的标签有助于并标准化资源的手动管理。

**怎么做的？**

建议的标签有：

- `app.kubernetes.io/name`：应用程序的名称
- `app.kubernetes.io/instance`：标识应用程序实例的唯一名称
- `app.kubernetes.io/version`：应用程序的当前版本（例如，语义版本、修订哈希等）
- `app.kubernetes.io/component`：体系结构中的组件
- `app.kubernetes.io/part-of`：此应用程序所属的更高级别应用程序的名称
- `app.kubernetes.io/managed-by`：用于管理应用程序操作的工具

以下是将这些标签应用于StatefulSet资源的示例：

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mysql-abcxyz
  labels:**
    app.kubernetes.io/name: mysql
    app.kubernetes.io/instance: mysql-abcxzy
    app.kubernetes.io/version: "5.7.21"
    app.kubernetes.io/component: database
    app.kubernetes.io/part-of: wordpress
    app.kubernetes.io/managed-by: helm
spec:
  # ...
  template:
    metadata:
      labels:
        app.kubernetes.io/name: mysql
        app.kubernetes.io/instance: mysql-abcxzy
        app.kubernetes.io/version: "5.7.21"
        app.kubernetes.io/component: database
        app.kubernetes.io/part-of: wordpress
        app.kubernetes.io/managed-by: helm
    spec:
      # ...
```

请注意，根据应用程序的不同，只能定义其中一些标签（例如`app.kubernetes.io/name`以及`app.kubernetes.io/instance`)，而它们应应用于所有资源对象。

**References**

- [Kubernetes documentation: Recommended Labels](https://kubernetes.io/docs/concepts/overview/working-with-objects/common-labels/)

### Use independent versions for containers, Pods, and apps

E.g. if Pod specification changes, update only the Pod and app version, but not the container image version, etc.

### Maintain a "release" for top-level resources

Resources that compromise an entire app (e.g. Deployment, StatefulSet) should have a `release` label.

This label should change every time the application is deployed (the release should be updated even if the version of the app stays the same).
**参考文献**

-[Kubernetes文档：推荐标签](https://kubernetes.io/docs/concepts/overview/working-with-objects/common-labels/)

### 对容器、pod和应用程序使用独立版本

例如，如果Pod规格发生变化，只更新Pod和app版本，不更新容器镜像版本等。

### 维护顶级资源的“释放”

危害整个应用程序的资源（例如Deployment、StatefulSet）应该有一个“release”标签。

每次部署应用程序时，这个标签都应该更改（即使应用程序的版本保持不变，也应该更新版本）。

* * *

##资源管理

TODO：集成当前的“资源利用”

###定义所有容器的资源请求

**是什么？**

您可以为Pod规范中的每个Pod容器设置资源请求（CPU和内存）。

**为什么？**

**请求**资源是容器运行所需的最小资源量。它影响Kubernetes调度程序的调度决策（调度程序只将pod调度到一个具有足够的空闲资源以容纳pod所有容器的请求的节点）。

默认情况下，不会设置资源请求，这意味着调度程序将pod调度到任何节点，无论它有多少空闲资源。

为Pod的每个容器设置内存和CPU请求，确保Pod能够在计划的节点上正常运行。

**怎么做到的？**

设置[`pod.spec.containers[].resources.requests`](https://kubernetes.io/docs/reference/generated/kubernetes api/v1.16/#资源需求-v1芯）Pod规范字段：

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: test
spec:
  # ...
    spec:
      containers:
      - image: nginx
        name: nginx
        resources:
          requests:
            memory: 100Mi
            cpu: 100m
```

**参考文献**

- [KKubernetes文档：管理容器的计算资源](https://kubernetes.io/docs/concepts/configuration/manage-compute-resources-container/)

###根据所需的服务质量（QoS）类定义资源限制

资源的**限制**是允许容器使用的最大资源量（可以将其视为资源使用突发的上限）。当容器达到限制时会发生什么，取决于资源的类型：

- 为了内存，容器被终止并重新启动
- 对于CPU，容器是被限制的（TODO：“throttled”到底意味着什么？）

默认情况下，没有设置资源限制，这意味着没有限制一个pod可以在一个节点上使用多少资源。

设置资源限制可以防止Pods独占节点上的所有资源。

给定资源请求的值，resoruce limit的值确定Pod的服务质量（QoS）类：

- Best-effort：没有请求和限制，最低优先级，Pod先被杀死
- Burstable：限制高于请求，中等优先级，如果不存在尽力而为的Pods，则终止
- Guaranteed：限制相等的请求，最高优先级，仅在没有Best-effort和Burstable Pods存在的情况下终止

### 不定义CPU限制

TODO:这对QoS类有何影响？这是最佳实践的来源在哪里？

### 为每个命名空间创建一个LimitRange

### 为每个命名空间创建ResourceQuota

### 使用PriorityClass定义调度和逐出pod的顺序

* * *

## 健康检查

TODO:集成当前的“健康检查”

### 将initialDelaySeconds设置为所有存活探针和就绪探针的适当值

* * *

## 高级调度

### 如果某些Pod应该并置，则使用Pod亲和力

### 如果某些Pod不应并置，请使用Pod Anti-affinity

### 如果Pod应该只在节点子集上运行，请使用节点关联或节点选择器

### 使用污点和容忍度为某些Pod保留节点

* * *

## 应用程序生命周期

TODO:集成当前的“平滑关闭”（重命名为“应用程序生命周期”）

### 容器监听SIGTERM信号并优雅地关闭

### 容器实现了preStop生命周期钩子以优雅地关闭

这是监听SIGTERM信号的另一种选择。

### 如果正常关闭需要很长时间，则设置终止宽限期

### 容器实现启动后生命周期钩子来执行任何启动任务

### 只对postStart生命周期钩子使用exec方法

* * *

## 亚马逊网络服务（AWS）

### 使用kube2iam将Kubernetes权限与AWS IAM集成

作为使用这个的最佳实践，我建议使用[kube2iam](https://github.com/jtblin/kube2iam)--这允许您根据命名空间控制允许pod承担哪些IAM角色，但也允许pod将自己视为拥有角色而不是要求它承担这样的角色。

在我们的设置中，我们使用kube2iam并为节点分配他们的IAM角色-这个IAM角色可以承担其他角色。对于kube2iam，我们通过pod注释来分配角色，因此pod认为自己拥有该角色。此外，我们使用名称空间限制来控制pod可以承担哪些角色的访问。

见 #7
