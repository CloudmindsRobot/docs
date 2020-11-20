# Kubernetes 最佳实践

O'Reilly, 2019.

## 第一章。建立基本服务

### 管理资源清单

_资源清单是应用程序的声明性状态_

- 在Git中存储资源清单（版本控制）
- 在GitHub上管理Git存储库（代码复查）
- 对应用程序的所有清单使用单个顶层目录
- 将子目录用于应用程序的子组件（例如服务）
- 使用GitOps：确保集群的内容与Git存储库的内容匹配
- 使用某种自动化（例如CI/CD管道）仅从特定的Git分支部署到生产环境
- 从一开始就使用CI/CD（很难去改造现有的应用程序）

### 管理容器镜像

- 基础镜像使用知名和可信的镜像提供商
  - 或者，从零开始构建所有镜像（例如使用Go）
- 镜像的标记是不可变的（即具有不同内容的镜像具有不同的标记）
  - 将语义版本控制与相应Git commit的散列相结合（例如，v1.0.1-bfeda01f）
- 避免使用“最新”标记（不是不可变的）

### 部署应用程序

- 将资源请求值与限制值保持一致：以最大化资源利用率为代价提供可预测性（应用程序无法利用多余的空闲资源）
  - 只有当你有更多的经验时，才能将请求和限制设置为不同的值
- 始终使用入口控制来暴露服务，即使是对于简单的应用程序（对于生产，您不需要使用入口控制进行实验）
- 管理可能在运行时更新的配置数据，这些数据与ConfigMap中的代码分开管理
- 在每个配置映射的名称中输入版本号（例如“myconfig-v1”）。更新配置时，请使用增加的版本号创建一个新的ConfigMap（例如'myconfig-v2`），然后更新应用程序以使用新的ConfigMap。这可以确保新的配置被加载到应用程序中，而不管ConfigMap是如何装载到应用程序中的（作为env var或文件，在后一种情况下，如果应用程序监视文件的更改）
  - 不要删除以前的配置映射（例如“myconfig-v1”）。这允许随时回滚到以前的配置。
- 部署多个应用程序清单（如果不部署多个应用程序清单，则不要使用多个资源清单）
  - Helm, kustomize, Kapitan, ...

## 第二章。开发人员工作流

_创建一个开发集群（开发人员可以在这里部署和测试他们正在工作的应用程序）_

- 每个组织或团队一个集群（10-20人）
- 每个开发人员一个命名空间
- 使用外部标识系统进行群集用户管理（Azure Active Directory、AWS IAM）
  - 使用持有令牌和API服务器的身份验证通过外部服务验证令牌
- 使用RoleBinding（而不是ClusterRoleBinding）为开发人员的命名空间授予“edit” ClusterRole
- 使用ClusterRoleBinding（而不是RoleBinding）为整个集群授予“view” ClusterRole
- 为每个命名空间分配一个ResourceQuota
- 通过为开发人员命名空间分配生存时间（TTL）使其暂时化，然后自动删除它们
  - 这样就不会在集群中累积未使用的资源
- 为每个命名空间分配以下注释：TTL、assignee、resource quota、team、purpose
  - 您可以定义一个CRD，它用所有这些元数据创建一个命名空间

## Chapter 3. Monitoring and Logging in Kubernetes

### Monitoring

- Run monitoring system in a dedicated "utility cluster" (to avoid problems with the target cluster affecting the monitoring system)

### Logging

_Collect and centrally store logs from all the workloads running in the cluster and from the cluster components themselves._

- Implement a retention and archival strategy for logs (retain 30-45 days of historical logs)
- What to collect logs from:
  - Nodes (kubelet, container runtime)
  - Control plane (API server, scheduler, controller mananger)
  - Kubernetes auditing (all requests to the API server)
- Applications should log to stdout rather than to files
  - Allows a daemon on each node to collect the logs from the container runtime (if logging to files, a sidecar container for each pod might be necessary)
- Some log aggregation tools: EFK stack (Elasticsearch, Fluentd, Kibana), DataDog, Sumo Logic, Sysdig, GCP Stackdriver, Azure Monitor, AWS CloudWatch
- Use a hosted logging solution (e.g. DataDog, Stackdriver) rather than a self-hosted one (e.g. EFK stack)
## 第三章。Kubernetes的监测和日志

### 监控

- 在专用的“实用程序群集”中运行监视系统（以避免目标群集的问题影响监视系统）

### 日志

_收集并集中存储集群中运行的所有工作负载和集群组件本身的日志_

- 对日志实施保留和归档策略（保留30-45天的历史日志）
- 从何处收集日志：
  - 节点（kubelet，容器运行时）
  - 控制平面（API服务器、调度程序、控制器管理器）
  - Kubernetes审计（对API服务器的所有请求）
- 应用程序应该记录到stdout而不是文件
  - 允许每个节点上的守护进程从容器运行时收集日志（如果记录到文件，则可能需要为每个pod提供一个sidecar容器）
- 一些日志聚合工具：EFK堆栈（Elasticsearch、Fluentd、Kibana）、DataDog、Sumo Logic、Sysdig、GCP Stackdriver、Azure Monitor、AWS CloudWatch
- 使用托管日志解决方案（如DataDog、Stackdriver）而不是自托管解决方案（如EFK stack）

### 告警

- 仅对影响服务级别目标（SLO）的事件发出警报
- 只对需要立即人工干预的事件发出警报
- 自动修复不需要立即人工干预的事件
- 在警报通知中包括相关信息（例如故障排除行动手册的链接、上下文信息）

## 第四章。配置、机密和RBAC

### 配置映射和机密

- 使用ConfigMaps和Secrets将配置注入pod
- PodPresets：根据注释自动将ConfigMap或Secret挂载到pod
- 在应用程序中，监视配置文件的更改，以便可以在运行时通过更新ConfigMap或Secret来更改配置
- 使用ConfigMap/Secret中的值作为环境变量时，更新ConfigMap/Secret时不会更新容器中的环境变量
- 使用CI/CD管道，每当ConfigMap/Secret被更新时，该管道将重新启动POD（这可确保POD正在使用新数据，即使应用程序不监视配置文件中的更改，或者配置数据作为环境变量装载）
  - 或者，在ConfigMap的名称中包含一个版本名，当配置发生变化时，创建一个新的ConfigMap并更新应用程序以使用新的ConfigMap（参见第1章）。
- 始终将机密作为卷（文件）装入，而不是使用环境变量装入
- 避免在bernkuetes中使用有状态的应用程序
  - 将SaaS/云服务产品用于有状态服务
  - 如果不能选择运行内部部署和公共SaaS，那么应该有一个专门的团队，为组织的其他成员提供内部有状态的SaaS

### RBAC

- 为Kubernetes API的所有“用户”使用特定的服务帐户，这些用户被分配了具有最少特权的定制角色

##第五章。持续集成、测试和部署

_CI/CD管道的常见步骤是：（1）将代码推送到Git存储库；（2）构建整个应用程序代码；（3）针对构建的代码运行测试；（4）构建容器镜像；（5）将容器镜像推送到容器仓库；（6）将应用程序部署到Kubernetes（使用多种部署策略之一，例如滚动更新，蓝/绿部署、金丝雀部署或A/B部署），（7）针对已部署的应用程序运行测试（例如混沌实验）_

- 将生产代码保存在主分支中
- 保持容器镜像的小尺寸（使用带有多级构建的初始镜像、无发行版的基础镜像或优化的基础镜像，例如Alpine、Debian Slim）
- 使用镜像标记策略：由CI系统构建的每个镜像都应该有一个唯一的标记（镜像标记应该是不可变的，也就是说，如果两个镜像具有不同的内容，则它们不能具有相同的标记，请参见第1章）
  - 使用生成ID作为标记的一部分
  - 使用Git提交哈希作为标记的一部分
- 最小化CI构建时间
- 在CI中包含大量测试（如果任何测试失败，则构建应失败）
- 在生产环境中建立广泛的监控

## Chapter 6. Versioning, Releases, and Rollouts

_The true declarative nature of Kubernetes really shines when planning the proper use of labels._

_By properly identifying the operational and development states by the means of labels in the resource manifests, it becomes possible to tie in tooling and automation to more easily manage the complex processes of upgrades, rollouts, and rollbacks._

- _Version: increments when the code specification changes_
- _Release: increments when the applicatoin is (re)-deployed (even if it's the same version of the app)_
- _Rollout: how a replicated app is put into production (this is taken care of automatically by the Deployment resource when there are changes to the `deployment.spec.template` field)_
- _Rollback: revert an application to the state of a previous release_
## 第6章 版本控制、发布和部署

_Kubernetes真正的声明性本质在计划标签的正确使用时非常闪耀_

_通过在资源清单中通过标签的方式正确识别运维和开发状态，可以将工具和自动化结合起来，以便更轻松地管理升级、部署和回滚的复杂过程_

- _Version:代码规范更改时递增_
- _Release：当应用程序被（重新）部署（即使它是相同版本的应用程序）时递增_
- _Rollout：如何将复制集的应用程序投入生产（当对`deployment.spec.template`字段）_
- _Rollback：将应用程序还原为以前版本的状态_

最佳实践：

- 至少为每个资源添加标签：“app”、“version”、“environment”`
  - pod还可以用“tier”标记，而顶级对象（如部署或作业）应标记为“release”和“release number”`
- 对容器映像、pod和部署使用独立的版本
  - 例如，如果Pod规范发生变化，只更新Pod和部署版本，而不更新容器映像版本
- 前端发布（例如，第1版）和第1版（第1版）
  - 如果再次部署相同版本的应用程序，则会生成新的版本号
  - 版本号由部署应用程序的CI/CD工具创建

_与官方[推荐标签]进行比较(https://kubernetes.io/docs/concepts/overview/working-with-objects/common-labels/)在Kubernetes文档中_

- `app.kubernetes.io/name`
- `app.kubernetes.io/instance`
- `app.kubernetes.io/version`
- `app.kubernetes.io/component`
- `app.kubernetes.io/part-of`
- `app.kubernetes.io/managed-by`

## Chapter 7. Worldwide Application Distribution and Staging

_Deploying app in multiple regions around the world (for scaling, reduced latency, etc.)._

_Distributing container images, load balancing, canary regions, testing..._
## 第七章 全局应用程序分发和部署

_在全球多个地区部署应用程序（用于扩展、减少延迟等）_

_分发容器镜像，负载均衡，金丝雀区域，测试_

## 第八章 资源管理

### 高级调度

- Pod 亲和性
- Pod 反亲和性
- 节点选择
- 污点和容忍

### Pod 资源管理

- 资源请求
- 资源限制
- 服务质量（由请求和限制的值自动确定）
- Pod 中断预算
- 资源配额
- 限制范围
- 集群节点自动伸缩
- Pod 水平自动伸缩
- Pod 垂直自动伸缩

## 第九章 网络、网络安全和服务网格

### 服务和入口控制器

_服务:_

- ClusterIP（headless服务没有标签选择器，但有一个明确分配的端点；不由kube-proxy管理；没有ClusterIP地址，但为端点中的每个Pod创建一个DNS条目）
- 节点端口
- 外部名称
- 负载均衡器

_入口控制器：_

提供HTTP应用程序级路由，而不是3/4级服务。

入口控制器允许使用入口资源（所有资源都是第三方）

- 所有不需要从集群外部访问的服务都应该是ClusterIP
- 对面向外部的HTTP服务使用入口，并选择适当的入口控制器

### 网络策略

_定义如何允许集群中的Pod彼此通信_

- _需要支持网络策略的CNI（Calico, Cilium, Weave Net）_
- 从限制进入开始，然后根据需要限制出口
- 在所有命名空间中创建“拒绝所有”策略
- 尝试将pod之间的通信限制在名称空间内（避免跨名称空间通信）

### 服务网格

_管理应用程序（或多个应用程序）服务之间的通信量_

-大多数可能只需要在有数百个服务和数千个端点的大型部署中使用

## 第十章 Pod和容器安全

### Pod安全策略

_集中执行pod规范中的安全敏感字段_

_PodSecurityPolicy的许多字段与Pod规范中securityContext的字段相匹配_

- 使用PodSecurityPolicy需要启用PodSecurityPolicy许可控制器，但在大多数Kubernetes部署中，它没有启用。一旦启用了PodSecurityPolicy许可控制器，就需要适当的PodSecurityPolicy资源来允许创建任何Pods。
  - 您还需要将对创建的podsecuritypolicy的“use”访问权授予工作负载的服务帐户或工作负载的控制器（您可以使用`system:serviceaccounts`损害所有控制器服务帐户的组）。
- 使用<https://github.com/sysdiglabs/kube-psp-advisor>基于现有的Pods自动生成podsecuritypolicy

### 运行时类

_允许根据这个Pod所需的容器之间的隔离量来指定将哪个容器运行时用于一个Pod（如果配置了多个容器）_

- 在pod规范中设置“runtimeClassName”字段
- 仅当您的工作负载需要在主机上隔离不同数量的工作负载（为了安全性或法规遵从性），才使用它

### 其他

- 使用DenyExecOnPrivileged或DenyEscalatingExec许可控制器作为PodSecurityPolicies的一个更简单的替代方法->**但是，这不是一个最佳实践，因为这些都是不推荐的，建议使用PodSecurityPolicies**
- 使用Falco在容器运行时中强制执行安全策略

## 第11章 集群的策略和治理

说明：

只允许兼容的Kubernetes资源（任何类型）应用于集群（符合定义的策略）。

-开放策略代理（OPA）：策略引擎
-网关守卫器
-正在验证许可控制webhook
-用于安装、配置和管理开放策略代理策略的Kubernetes运算符

可以使用以下示例实施网关守卫策略：

-不得在互联网上公开服务
-只允许来自受信任容器注册表的容器
-所有容器都必须有资源限制
-入口主机名不能重叠
-入口只能使用HTTPs

## 第12章 管理多个群集

_如何管理多个集群，使不同集群中的应用程序相互交互，一次将应用程序部署到多个集群，Kubernetes Federation_

## 第13章 整合外部服务和Kubernetes

- Kubernetes中使用集群外服务的应用程序_
- 在Kubernetes中使用服务的集群外的应用程序_
- Kubernetes中使用另一个Kubernetes集群服务的应用程序_

## 第14章 Kubernetes的运行机器学习

_显然，Kubernetes是“支持机器学习工作流和生命周期的完美环境”_

## 第15章 高层建筑中bernkuets模式的应用

_开发更高层次的抽象，以便在topof Kubernetes上提供更适合开发人员的原语_

## 第16章 管理状态和有状态的应用程序

### 基本卷

_将目录从主机装载到容器中_

- 使用“emptyDir”在同一个容器中的容器之间共享数据
- 如果节点上运行的代理也需要访问数据，请使用“hostPath”

###由Kubernetes管理的存储

_Kubernetes支持管理持久存储_

- _PersistentVolume：一个独立于集群中任何节点的“磁盘”，并拥有自己的Kubernetes资源_
- _PersistentVolumeClaim：对从Pod规范引用的PersistentVolume的请求。存在此请求是为了防止必须通过引用通用PersistentVolumeClaim从Pod规范引用特定的PersistentVolumeClaim（使Pod规范不可移植）_
- _StorageClass：定义一个供应器来创建支持PersistentVolume的磁盘，以自动创建PersistentVolume。从PersistenVolumeClaim引用了StorageClass名称_
- _Default StorageClass：由任何没有显式定义StorageClass名称的PersistentVolumeClaim使用。需要[DefaultStorageClass](https://kubernetes.io/docs/reference/access authn authz/admission controllers/#defaultstorageclass)要启用的许可控制器_

最佳实践：

- 如果可以，请避免在集群中管理状态：使用外部服务来持久化状态
  - 即使它涉及到修改应用程序变成无状态
- 定义一个名为“default”的默认存储类（因为这通常在Helm图表中默认使用）
- 如果集群分布在多个可用性区域，请确保PersistentVolumes和使用它们的Pod位于同一可用性区域中
  - 通过正确标记所有对象和使用节点亲和力等。

###运行有状态的应用程序

-检查应用程序类型是否存在维护程序，如果存在，则使用它

## 第十七章 准入控制和授权

### 准入控制

-建议启用的许可控制器集：`NamespaceLifecycle、LimitRanger、ServiceAccount、DefaultStorageClass、DefaultTolerationSeconds、MutatingAdministrationWebhook、ValidatingAdministrationWebHook、优先级、ResourceQuota、PodSecurityPolicy`
- 如果使用多个可变的许可控制webhook，不要修改相同资源的相同字段（调用许可控制webhook的顺序未定义）
- 如果您使用变异的准入webhook，那么还要创建一个验证准入webhook，以验证资源是否按预期的方式被修改
- 定义发送到许可webhook的最少请求量（避免使用`resources:[*]`，等等）
- 始终在MutatingWebhookConfiguration/ValidatingWebhookConfiguration中使用“namespaceSelector”字段，这将导致许可控制webhook仅应用于某些名称空间。选择所需的最少数量的命名空间。
- 始终通过“namespaceSelector”字段将“kube system”命名空间从admisson控件webhook的范围中排除
- 除非需要任何人创建WebHookingRules
- 如果不需要，不要将秘密资源发送到许可控制webhook（将传递给webhook的请求限制到最低限度）

### 授权

- 只使用默认的RBAC模式（还有ABAC和webhook，但不使用它们）
- 有关RBAC最佳实践，请参阅第4章
* * *

## Table of contents

- [1. Setting Up a Basic Service](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch01.html#setting_up_a_basic_service)
  1. [Application Overview](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch01.html#idm46116561250104)
  1. [Managing Configuration Files](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch01.html#idm46116561246824)
  1. [Creating a Replicated Service Using Deployments](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch01.html#idm46116561241144)
     1. [Best Practices for Image Management](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch01.html#idm46116561237176)
     1. [Creating a Replicated Application](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch01.html#idm46116561242120)
  1. [Setting Up an External Ingress for HTTP Traffic](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch01.html#idm46116561243528)
  1. [Configuring an Application with ConfigMaps](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch01.html#idm46116561197272)
  1. [Managing Authentication with Secrets](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch01.html#idm46116561203320)
  1. [Deploying a Simple Stateful Database](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch01.html#idm46116561204264)
  1. [Creating a TCP Load Balancer by Using Service](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch01.html#idm46116561173288)
  1. [Using Ingress to Route Traffic to a Static File Server](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch01.html#idm46116561167912)
  1. [Parameterizing Your Application by Using Helm](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch01.html#idm46116560979624)
  1. [Deploying Services Best Practices](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch01.html#idm46116560979048)
  1. [Summary](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch01.html#idm46116560953208)
- [2. Developer Workflows](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch02.html#developer_workflows)
  1. [Goals](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch02.html#idm46116560933368)
  1. [Building a Development Cluster](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch02.html#idm46116560937720)
  1. [Setting Up a Shared Cluster for Multiple Developers](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch02.html#idm46116560926536)
     1. [Onboarding Users](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch02.html#idm46116560944488)
     1. [Creating and Securing a Namespace](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch02.html#idm46116560934536)
     1. [Managing Namespaces](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch02.html#idm46116560895832)
     1. [Cluster-Level Services](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch02.html#idm46116560895576)
  1. [Enabling Developer Workflows](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch02.html#idm46116560904456)
  1. [Initial Setup](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch02.html#idm46116560891160)
  1. [Enabling Active Development](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch02.html#idm46116560881816)
  1. [Enabling Testing and Debugging](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch02.html#idm46116560881560)
  1. [Setting Up a Development Environment Best Practices](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch02.html#idm46116560869960)
  1. [Summary](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch02.html#idm46116560857912)
- [3. Monitoring and Logging in Kubernetes](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch03.html#monitoring_and_logging_in_kubernetes)
  1. [Metrics versus Logs](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch03.html#idm46116560853272)
  1. [Monitoring Techniques](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch03.html#idm46116560844312)
  1. [Monitoring Patterns](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch03.html#idm46116560837624)
  1. [Kubernetes Metrics Overview](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch03.html#idm46116560837048)
     1. [cAdvisor](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch03.html#idm46116560817480)
     1. [Metrics Server](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch03.html#idm46116560823304)
     1. [Kube-State-Metrics](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch03.html#idm46116560816504)
  1. [What Metrics Do I Monitor?](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch03.html#idm46116560793912)
  1. [Monitoring Tools](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch03.html#idm46116560793608)
  1. [Monitoring Kubernetes by Using Prometheus](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch03.html#idm46116560770200)
  1. [Logging Overview](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch03.html#idm46116560747672)
  1. [Tools for Logging](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch03.html#idm46116560703128)
  1. [Logging by Using an EFK Stack](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch03.html#idm46116560673528)
  1. [Alerting](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch03.html#idm46116560671768)
  1. [Best Practices for Monitoring, Logging, and Alerting](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch03.html#idm46116560628088)
     1. [Monitoring](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch03.html#idm46116560624744)
     1. [Logging](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch03.html#idm46116560635928)
     1. [Alerting](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch03.html#idm46116560631672)
  1. [Summary](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch03.html#idm46116560616104)
- [4. Configuration, Secrets, and RBAC](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch04.html#configuration_secrets_and_rbac)
  1. [Configuration through ConfigMaps and Secrets](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch04.html#idm46116560611272)
     1. [ConfigMaps](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch04.html#idm46116560599272)
     1. [Secrets](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch04.html#idm46116560602904)
  1. [RBAC](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch04.html#idm46116560540264)
     1. [RBAC Primer](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch04.html#idm46116560526056)
     1. [RBAC Best Practices](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch04.html#idm46116560515272)
  1. [Summary](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch04.html#idm46116560499352)
- [5. Continuous Integration, Testing, and Deployment](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch05.html#continuous_integration_testing_and_deployment)
  1. [Version Control](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch05.html#idm46116560474808)
  1. [Continuous Integration](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch05.html#idm46116560473352)
  1. [Testing](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch05.html#idm46116560471640)
  1. [Container Builds](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch05.html#idm46116560459496)
  1. [Container Image Tagging](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch05.html#idm46116560451208)
  1. [Continuous Deployment](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch05.html#idm46116560452440)
  1. [Deployment Strategies](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch05.html#idm46116560434776)
  1. [Testing in Production](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch05.html#idm46116560401128)
  1. [Setting Up a Pipeline and Performing a Chaos Experiment](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch05.html#idm46116560433704)
     1. [Setting Up CI](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch05.html#idm46116560374392)
     1. [Setting Up CD](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch05.html#idm46116560378168)
     1. [Performing a Rolling Upgrade](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch05.html#idm46116560341736)
     1. [A Simple Chaos Experiment](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch05.html#idm46116560336632)
  1. [Best Practices for CI/CD](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch05.html#idm46116560323736)
  1. [Summary](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch05.html#idm46116560313288)
- [6. Versioning, Releases, and Rollouts](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch06.html#versioning_releases_and_rollouts)
  1. [Versioning](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch06.html#idm46116560292760)
  1. [Releases](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch06.html#idm46116560308088)
  1. [Rollouts](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch06.html#idm46116560301944)
  1. [Putting It All Together](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch06.html#idm46116560280584)
     1. [Best Practices for Versioning, Releases, and Rollouts](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch06.html#idm46116560299720)
  1. [Summary](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch06.html#idm46116560295256)
- [7. Worldwide Application Distribution and Staging](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch07.html#worldwide_application_distribution_and_staging)
  1. [Distributing Your Image](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch07.html#idm46116560261480)
  1. [Parameterizing Your Deployment](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch07.html#idm46116560248616)
  1. [Load Balancing Traffic Around the World](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch07.html#idm46116560242456)
  1. [Reliably Rolling Out Software Around the World](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch07.html#idm46116560238584)
     1. [Pre-Rollout Validation](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch07.html#idm46116560222216)
     1. [Canary Region](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch07.html#idm46116560209464)
     1. [Identifying Region Types](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch07.html#idm46116560214664)
     1. [Constructing a Global Rollout](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch07.html#idm46116560203320)
  1. [When Something Goes Wrong](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch07.html#idm46116560240056)
  1. [Worldwide Rollout Best Practices](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch07.html#idm46116560239752)
  1. [Summary](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch07.html#idm46116560206392)
- [8. Resource Management](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch08.html#resource_management)
  1. [Kubernetes Scheduler](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch08.html#idm46116560193816)
     1. [Predicates](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch08.html#idm46116560181704)
     1. [Priorities](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch08.html#idm46116560186200)
  1. [Advanced Scheduling Techniques](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch08.html#idm46116560175928)
     1. [Pod Affinity and Anti-Affinity](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch08.html#idm46116560177992)
     1. [nodeSelector](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch08.html#idm46116560172360)
     1. [Taints and Tolerations](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch08.html#idm46116560162200)
  1. [Pod Resource Management](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch08.html#idm46116560179080)
     1. [Resource Request](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch08.html#idm46116560134984)
     1. [Resource Limits and Pod Quality of Service](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch08.html#idm46116560134408)
     1. [Pod Disruption Budgets](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch08.html#idm46116560129736)
     1. [Managing Resources by Using Namespaces](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch08.html#idm46116560096152)
     1. [ResourceQuota](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch08.html#idm46116560093512)
     1. [LimitRange](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch08.html#idm46116560077128)
     1. [Cluster Scaling](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch08.html#idm46116560047784)
     1. [Application Scaling](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch08.html#idm46116560031400)
     1. [HPA with Custom Metrics](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch08.html#idm46116560028392)
     1. [HPA with Custom Metrics](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch08.html#idm46116560015352)
  1. [Resource Management Best Practices](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch08.html#idm46116560142104)
  1. [Summary](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch08.html#idm46116559974184)
- [9. Networking, Network Security, and Service Mesh](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch09.html#networking_network_security_and_service_mesh)
  1. [Kubernetes Network Principles](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch09.html#idm46116559966824)
  1. [Network Plug-ins](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch09.html#idm46116559961016)
     1. [Kubenet](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch09.html#idm46116559949016)
  1. [Kubenet Best Practices](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch09.html#idm46116559947592)
     1. [The CNI Plug-in](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch09.html#idm46116559943624)
  1. [CNI Best Practices](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch09.html#idm46116559953512)
  1. [Services in Kubernetes](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch09.html#idm46116559921624)
     1. [Service Type ClusterIP](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch09.html#idm46116559916888)
     1. [Service Type NodePort](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch09.html#idm46116559911912)
     1. [Service Type ExternalName](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch09.html#idm46116559929032)
     1. [Service Type LoadBalancer](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch09.html#idm46116559924568)
     1. [Ingress and Ingress Controllers](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch09.html#idm46116559901736)
  1. [Services and Ingress Controllers Best Practices](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch09.html#idm46116559892984)
  1. [Network Security Policy](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch09.html#idm46116559881640)
     1. [Network Policy Best Practices](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch09.html#idm46116559861992)
  1. [Service Meshes](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch09.html#idm46116559877352)
  1. [Service Meshe Best Practices](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch09.html#idm46116559834840)
  1. [Summary](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch09.html#idm46116559826904)
- [10. Pod and Container Security](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch10.html#pod_and_container_security)
  1. [Pod Security Policies](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch10.html#idm46116559815192)
     1. [Enabling PodSecurityPolicy](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch10.html#idm46116559832184)
     1. [Anatomy of a PodSecurityPolicy](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch10.html#idm46116559833528)
     1. [PodSecurityPolicy challenges](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch10.html#idm46116559797704)
  1. [PodSecurityPolicy Best Practices](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch10.html#idm46116559739752)
  1. [Pod Security Policy Next Steps](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch10.html#idm46116559728008)
  1. [Workload Isolation and RuntimeClass](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch10.html#idm46116559716552)
     1. [Using the RuntimeClass](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch10.html#idm46116559707480)
     1. [Runtime Implementations](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch10.html#idm46116559704744)
  1. [Workload Isolation and RuntimeClass Best Practices](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch10.html#idm46116559729672)
  1. [Other Pod and Container Security Considerations](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch10.html#idm46116559687128)
     1. [Admission Controllers](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch10.html#idm46116559681304)
     1. [Intrusion and Anomaly Detection Tooling](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch10.html#idm46116559682856)
  1. [Summary](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch10.html#idm46116559677992)
- [11. Policy and Governance for Your Cluster](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch11.html#policy_and_governance_for_your_cluster)
  1. [Why Policy and Governance Is Important](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch11.html#idm46116559674616)
  1. [How Is This Policy Different?](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch11.html#idm46116559662808)
  1. [Cloud-Native Policy Engine](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch11.html#idm46116559661528)
  1. [Introducing Gatekeeper](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch11.html#idm46116559671928)
  1. [Example Policies](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch11.html#idm46116559670312)
  1. [Gatekeeper Terminology](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch11.html#idm46116559651288)
     1. [Constraint](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch11.html#idm46116559647656)
     1. [Rego](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch11.html#idm46116559639944)
     1. [Constraint Template](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch11.html#idm46116559638552)
  1. [Defining Constraint Templates](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch11.html#idm46116559644296)
  1. [Defining Constraints](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch11.html#idm46116559629816)
  1. [Data Replication](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch11.html#idm46116559610936)
  1. [UX](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch11.html#idm46116559607128)
  1. [Audit](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch11.html#idm46116559600456)
  1. [Becoming Familiar with Gatekeeper](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch11.html#idm46116559595192)
  1. [Gatekeeper Next Steps](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch11.html#idm46116559587256)
  1. [Policy and Governance Best Practices](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch11.html#idm46116559583208)
  1. [Summary](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch11.html#idm46116559574696)
- [12. Managing Multiple Clusters](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch12.html#managing_multiple_clusters)
  1. [Why Multiple Clusters](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch12.html#idm46116559565848)
  1. [Multicluster Design Concerns](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch12.html#idm46116559565432)
  1. [Managing Multiple Cluster Deployments](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch12.html#idm46116559539240)
     1. [Deployment and Management Patterns](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch12.html#idm46116559523784)
  1. [The GitOps Approach to Managing Clusters](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch12.html#idm46116559538984)
  1. [Multicluster Management Tools](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch12.html#idm46116559505816)
  1. [Kubernetes Federation](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch12.html#idm46116559470712)
  1. [Managing Multiple Clusters Best Practices](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch12.html#idm46116559470136)
  1. [Summary](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch12.html#idm46116559432024)
- [13. Integrating External Services and Kubernetes](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch13.html#integrating_external_services_and_kubernetes)
  1. [Importing Services into Kubernetes](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch13.html#idm46116559434680)
     1. [Selector-Less Services for Stable IP Addresses](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch13.html#idm46116559433048)
     1. [CNAME-Based Services for Stable DNS Names](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch13.html#idm46116559413272)
     1. [Active Controller-Based Approaches](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch13.html#idm46116559402936)
  1. [Exporting Services from Kubernetes](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch13.html#idm46116559395672)
     1. [Exporting Services by Using Internal Load Balancers](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch13.html#idm46116559380584)
     1. [Exporting Services on NodePorts](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch13.html#idm46116559387944)
     1. [Integrating External Machines and Kubernetes](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch13.html#idm46116559399400)
  1. [Sharing Services Between Kubernetes](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch13.html#idm46116559372280)
  1. [Third-Party Tools](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch13.html#idm46116559370024)
  1. [Connecting Cluster and External Services Best Practices](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch13.html#idm46116559358280)
  1. [Summary](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch13.html#idm46116559340328)
- [14. Running Machine Learning in Kubernetes](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch14.html#running_machine_learning_in_kubernetes)
  1. [Why Is Kubernetes Great for Machine Learning?](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch14.html#idm46116559349160)
  1. [Machine Learning Workflow](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch14.html#idm46116559351480)
  1. [Machine Learning for Kubernetes Cluster Admins](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch14.html#idm46116559325096)
  1. [Model Training on Kubernetes](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch14.html#idm46116559317832)
     1. [Training Your First Model on Kubernetes](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch14.html#idm46116559323528)
  1. [Distributed Training on Kubernetes](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch14.html#idm46116559299512)
     1. [Resource Constraints](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch14.html#idm46116559296552)
     1. [Specialized Hardware](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch14.html#idm46116559293624)
     1. [Libraries, Drivers, and Kernel Modules](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch14.html#idm46116559289560)
     1. [Storage](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch14.html#idm46116559274488)
     1. [Networking](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch14.html#idm46116559271336)
     1. [Specialized Protocols](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch14.html#idm46116559283944)
  1. [Data Scientist Concerns](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch14.html#idm46116559259240)
  1. [Machine Leaning on Kubernetes Best Practices](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch14.html#idm46116559255368)
  1. [Summary](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch14.html#idm46116559247176)
- [15. Building Higher-Level Application Patterns on Top of Kubernetes](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch15.html#building_higher_level_application_patterns_on_kubernetes)
  1. [Approaches to Developing Higher-Level Abstractions](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch15.html#idm46116559230936)
  1. [Extending Kubernetes](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch15.html#idm46116559224888)
     1. [Extending Kubernetes Clusters](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch15.html#idm46116559228840)
     1. [Extending the Kubernetes User Experience](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch15.html#idm46116559228584)
  1. [Design Considerations When Building Platforms](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch15.html#idm46116559204344)
     1. [Support Exporting to a Container Image](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch15.html#idm46116559210232)
     1. [Support Existing Mechanisms for Service and Service Discovery](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch15.html#idm46116559209816)
  1. [Building Application Platforms Best Practices](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch15.html#idm46116559199784)
  1. [Summary](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch15.html#idm46116559195672)
- [16. Managing State and Stateful Applications](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch16.html#managing_state_and_stateful_application)
  1. [Volumes and Volume Mounts](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch16.html#idm46116559179528)
     1. [Volume Best Practices](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch16.html#idm46116559171176)
  1. [Kubernetes Storage](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch16.html#idm46116559166920)
     1. [Persistent Volumes](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch16.html#idm46116559176808)
     1. [Persistent Volume Claim](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch16.html#idm46116559162232)
     1. [Storage Classes](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch16.html#idm46116559152200)
     1. [Kubernetes Storage Best Practices](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch16.html#idm46116559141272)
  1. [Stateful Applications](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch16.html#idm46116559166616)
     1. [StatefulSets](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch16.html#idm46116559116024)
     1. [Operators](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch16.html#idm46116559100040)
  1. [StatefulSet and Operator Best Practices](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch16.html#idm46116559132584)
  1. [Summary](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch16.html#idm46116559076840)
- [17. Admission Control and Authorization](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch17.html#admission_control_and_authorization)
  1. [Admission Control](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch17.html#idm46116559072168)
     1. [What Are They?](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch17.html#idm46116559069192)
     1. [Why Are They Important?](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch17.html#idm46116559059240)
     1. [Configuring Admission Webhooks](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch17.html#idm46116559045176)
  1. [Admission Control Best Practices](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch17.html#idm46116559026088)
  1. [Authorization](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch17.html#idm46116559003592)
     1. [Authorization Modules](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch17.html#idm46116558998344)
  1. [Authorization Best Practices](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch17.html#idm46116558959864)
  1. [Summary](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch17.html#idm46116558957880)
- [18. Conclusion](https://learning.oreilly.com/library/view/kubernetes-best-practices/9781492056461/ch18.html#conclusion)
