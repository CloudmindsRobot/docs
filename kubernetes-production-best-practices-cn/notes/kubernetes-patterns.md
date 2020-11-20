# Kubernetes 模式

O'Reilly, 2019.

本书是基于容器的云原生环境和应用程序的“模式”集合，它展示了Kubernetes如何实现这些模式。

您可以学习如何最佳地使用Kubernetes提供给您的工具，以便最好地利用Kubernetes对这些模式的实现。

## 第二章：可预见的需求

集装箱应了解并声明其要求。这允许平台（Kubernetes）将pod放在集群中的正确位置，并对其进行适当的处理。

**需求类型：**

- 运行时依赖项
  - 持久存储、主机端口、配置（ConfigMaps、Secrets）：在Pod规范中定义。
- 所需资源
  - 请求
  - 限制
  - Pods的**QoS**（由请求和限制定义）：定义**kubelet按什么顺序杀死Pods**
    - 最大努力：没有请求和限制，最低优先级，吊舱先被杀死
    - Burstable：限制高于请求，中等优先级，如果不存在尽力而为的Pods，则终止
    - 保证：限制相等的请求，最高优先级，仅在没有最大努力和Burstable Pods存在的情况下终止

[**Pod优先级和抢占**](https://kubernetes.io/docs/concepts/configuration/pod-priority-preemption/):

[PriorityClass](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.16/#priorityclass-v1-scheduling-k8s-io）：一个Pod相对于其他Pods的优先级，由**调度器使用**

- 定义POD的计划顺序（如果多个POD处于挂起状态）
- 如果没有地方放置挂起的Pod，调度程序可能会以较低的优先级逐出正在运行的Pod

PriorityClass和QoS无关：**QoS由kubelet**用于终止，**PriorityClass由scheduler**用于调度（和逐出）

**最佳实践：**

- 进行测试以发现所有容器的资源需求（CPU和内存），并在Pod规格中设置请求和限制

- 对生产中的关键pod使用保证的QoS类（在dev中，可以使用最佳工作和Burstable）

- 如果使用_Burstable_uQoS，请确保限制范围内的限制/请求比率较低

  - 限制/请求比率越高，节点耗尽资源和必须杀死pod的风险就越高（如果许多容器同时接近其限制）

- 使用RBAC锁定对PriorityClass的访问，限制每个命名空间的ResourceQuota中的PriorityClass

## 第3章：声明性部署

## 第4章 健康调查

一种让平台（Kubernetes）了解容器化应用程序的内部状态（否则就是平台的黑匣子）并采取适当行动的方法。

应用程序应该实现这些接口来提供关于其内部状态的最大数量的信息。

Kubernetes（kubelet）执行三种定期健康检查：

1 **进程：**容器的主进程仍在运行吗？→重新启动容器
1 **存活探针：**通过运行应用程序实现？→重新启动容器
1 **就绪探针：**由应用程序实现，应用程序是否准备好服务请求？→停止使用吊舱

**最佳实践：**

- 为所有容器定义活动性和就绪性探测
- 为所有活跃度和就绪性探测设置initalDelaySeconds（进行测试以确定该值应该有多大）

## 第5章：管理生命周期

平台向应用程序发出有关应用程序生命周期（由平台管理）的信号。

应用程序应该监听这些信号来了解它自己的生命周期（否则它无法访问，因为它完全由平台管理），并相应地做出反应。

信号：

-**SIGTERM:**当平台决定关闭一个容器（无论出于什么原因）时发出信号到容器中。容器应该监听到这些信号并优雅地关闭（即清理并退出主容器进程）。
-**SIGKILL:**仅当容器进程在“terminationGracePeriodSeconds”之后仍在运行时，才会在SIGTERM之后发出。杀死容器（硬）。
-**postStart:**创建容器后立即发出。该实现必须由应用程序提供（与活跃度/就绪性探测相同，即_exec_，_httpGet_，_tcpSocket_）。与启动容器主进程无关，即调用postStart钩子和启动容器进程同时发生。
-**preStop:**当平台决定关闭容器时发出。在SIGTERM之前发出，而SIGTERM将仅在preStop处理程序完成后执行。可以用来替代对SIGTERM的反应。

**最佳实践：**

- 应用程序应该监听SIGTERM或实现一个preStop钩子并优雅地退出
  - 优雅地退出应该很快，至少在30秒的默认值“terminationGracePeriodSeconds”内`
- 如果优雅退出需要很长时间，请在Pod规范中设置“terminationGracePeriodSeconds”
- 如果要为容器执行任何启动任务，则实现postStart钩子
- 只对postStart钩子使用_exec_方法，因为不能保证在调用postStart钩子时容器进程已经在运行（因此，_httpGet_ and _tcpSocket_ 有可能不能以竞态条件方式执行）

## 第6章：自动布局
