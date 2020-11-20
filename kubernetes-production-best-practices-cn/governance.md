# 治理

对于创建、管理和治理命名空间的最佳实践

## 命名空间限制

当您决定在集群中使用名称空间来做隔离时，您应该防止资源的误用。

你不应该允许你的用户使用超过你事先确定的资源。

集群管理员可以设置限制条件，以限制项目中使用的对象数或计算资源量，并设置配额和限制范围

如果你需要学习极限范围，你应该查阅官方文件 [limit ranges](https://kubernetes.io/docs/concepts/policy/limit-range/)

### 命名空间具有LimitRange

没有限制的容器会导致与其他容器的资源争用和计算资源的意外消耗。

Kubernetes约束资源利用有两个特点：资源配额和限制范围

使用LimitRange对象，您可以定义资源请求的默认值和名称空间中单个容器的限制。

在该命名空间内创建的任何容器，如果未显式指定请求和限制值，则将为其分配默认值。

如果需要了解资源配额，则应查看官方文档 [resource quotas](https://kubernetes.io/docs/concepts/policy/resource-quotas/).

### 命名空间具有资源配额

使用资源配额，可以限制命名空间内所有容器的总资源消耗。

为命名空间定义资源配额会限制属于该命名空间的所有容器可以使用的CPU、内存或存储资源总量。

您还可以为其他Kubernetes对象设置配额，例如当前名称空间中的pod数量。

如果您认为有人可以利用您的集群并创建20000个ConfigMaps，那么可以使用LimitRange来防止这种情况的发生。

## Pod安全策略

当一个Pod部署到集群中时，您应该防止:

- 容器受损
- 容器在节点上使用不允许的资源（如进程、网络或文件系统）

一般来说，你应该把Pod的功能限制在最小限度。

### 启用Pod安全策略

例如，您可以使用Kubernetes Pod安全策略来做如下限制：

- 限制访问主机进程或网络命名空间
- 限制运行特权容器
- 设置容器运行用户
- 限制访问主机文件系统
- 设置Linux功能、Seccomp或SELinux配置文件

选择正确的策略取决于集群的性质。

下面的文章解释了kubernetes pod安全策略的一些最佳实践 [Kubernetes Pod Security Policy best practices](https://resources.whitesourcesoftware.com/blog-whitesource/kubernetes-pod-security-policy)

### 禁用特权容器

在Pod中，容器可以以“特权”模式运行，并且几乎不受限制地访问主机系统上的资源。

虽然有特定的用例需要这种访问级别，但通常，让容器这样做会带来安全风险。

特权Pod的有效用例包括在节点上使用硬件，例如GPU。

您可以了解更多关于安全上下文和特权容器的信息[learn more about security contexts and privileges containers from this article](https://kubernetes.io/docs/tasks/configure-pod-container/security-context/)。

### 在容器中使用只读文件系统

在容器中运行只读文件系统会强制容器保持不变。

这不仅减轻了一些传统的（和有风险的）做法，例如热修补，而且有助于防止恶意进程在容器中存储或操纵数据的风险。

使用只读文件系统运行容器听起来可能很简单，但可能会带来一些复杂性。

_如果您需要在临时文件夹中写入日志或存储文件，该怎么办？_

您可以了解如何在生产中安全地运行容器 [running containers securely in production](https://medium.com/@axbaretto/running-docker-containers-securely-in-production-98b8104ef68).

### 防止容器以root用户身份运行

在容器中运行的进程与主机上的任何其他进程没有什么不同，只是它有一小块元数据声明它在容器中。

因此，容器中的root与主机上的root（uid0）相同。

如果用户成功地从容器中以root用户身份运行的应用程序中脱离，那么他们可能能够使用同一root用户访问主机。

将容器配置为使用非特权用户，是防止权限提升阻止攻击的最佳方法。

如果您想了解更多信息，请阅读 [article offers some detailed explanation examples of what happens when you run your containers as root](https://medium.com/@mccode/processes-in-containers-should-not-run-as-root-2feae3f0df3b).

### 权限限制

Linux功能使进程能够执行一些默认情况下只有root用户可以执行的特权操作。

例如，`CAP_CHOWN`允许进程“对文件uid和gid进行任意更改”。

即使您的进程不是以root用户身份运行，也有可能通过提升权限来使用这些类似于root的功能。

换句话说，如果您不想受到损害，您应该只启用所需的功能。

_但是应该启用哪些功能？为什么？_

以下两篇文章深入探讨了有关Linux内核功能的理论和实践最佳实践:

- [Linux Capabilities: Why They Exist and How They Work](https://blog.container-solutions.com/linux-capabilities-why-they-exist-and-how-they-work)
- [Linux Capabilities In Practice](https://blog.container-solutions.com/linux-capabilities-in-practice)

### 防止权限升级

您应该在关闭权限升级的情况下运行容器，以防止使用setuid或setgid二进制文件提升权限。

## 网络策略

Kubernetes集群默认网络策略：

1. 容器可以与网络中的任何其他容器通信，并且在这个过程中没有地址的转换，也就是说，不涉及NAT
1. 集群中的节点可以与网络中的任何其他容器通信，反之亦然。即使在这种情况下，也没有地址的解析-也就是说，没有NAT
1. 一个容器的IP地址总是相同的，如果从另一个容器或它本身看，它是独立的。

如果您计划将集群划分为更小的块，并且在名称空间之间进行隔离，那么第一条规则就没有帮助了。

_想象一下，如果集群中的用户能够使用集群中的任何其他服务_

现在，想象一下，如果集群中的恶意用户获得对集群的访问权，他们可以向整个集群发出请求。

为了解决这个问题，您可以定义如何允许pod在当前名称空间和使用网络策略进行跨命名空间通信。

### 启用网络策略

Kubernetes网络策略指定pod组的访问权限，就像云中的安全组用于控制对VM实例的访问一样。

换句话说，它在Kubernetes集群上运行的pod之间创建防火墙。

如果您不熟悉网络安全策略，请阅读 [Securing Kubernetes Cluster Networking](https://ahmet.im/blog/kubernetes-network-policy/).

### 在每个命名空间中都有一个默认的网络策略

这个存储库包含Kubernetes网络策略的各种用例和YAML文件示例，以便在您的设置中使用。如果您想知道如何删除/限制运行在Kubernetes上的应用程序的流量，请继续阅读 [how to drop/restrict traffic to applications running on Kubernetes](https://github.com/ahmetb/kubernetes-network-policy-recipes).

## 基于角色的访问控制（RBAC）策略

基于角色的访问控制允许您在集群中定义一些资源访问策略。

_通常的做法是放弃所需的最少的许可，但实际是什么？如何量化最小的特权？_

细粒度策略提供了更高的安全性，但需要更多的精力来管理。

更广泛的授权可以为服务帐户提供不必要的API访问，但更容易控制。

_是否应该为每个命名空间创建一个策略并共享它？_

_或者最好是把它们放在一个更细粒度的基础上？_

没有一刀切的方法，你应该根据具体情况来判断你的需求。

_但是你从哪里开始呢？_

如果你从一个有空规则的角色开始，你可以一个接一个地添加你需要的所有资源，但仍然要确保你没有放弃太多。

### 禁用默认ServiceAccount的自动挂载

请注意，默认的ServiceAccount会自动挂载到所有pod的文件系统中 [the default ServiceAccount is automatically mounted into the file system of all Pods](https://kubernetes.io/docs/tasks/configure-pod-container/configure-service-account/#use-the-default-service-account-to-access-the-api-server).

您可能希望禁用它并提供更细粒度的策略。

### RBAC策略设置为所需的最少权限

在如何设置RBAC规则方面找到好的建议是很有挑战性的。在Kubernetes RBAC的3个实际方法中，您可以找到三个实际场景和关于如何开始的实用建议 [3 realistic approaches to Kubernetes RBAC](https://thenewstack.io/three-realistic-approaches-to-kubernetes-rbac/)

### RBAC策略是细粒度的，不共享

Zalando有一个简洁的策略来定义角色和服务帐户。

首先，他们描述了他们的需求：

- 用户应该能够部署，但不应该允许他们读秘密
- 管理员应该获得对所有资源的完全访问权限
- 默认情况下，应用程序不应获得对kubernetesapi的写访问权限
- 对于某些用途，应该可以写入kubernetesapi。

这四项要求转化为五个不同的角色：

- 只读
- 高级用户
- 操作员
- 控制者
- 管理员

你可以在这个链接中了解他们的决定 [their decision in this link](https://kubernetes-on-aws.readthedocs.io/en/latest/dev-guide/arch/access-control/adr-004-roles-and-service-accounts.html).

## 自定义策略

即使您能够将集群中的策略分配给诸如机密和Pods之类的资源，也存在Pod安全策略（psp）、基于角色的访问控制（RBAC）和网络策略的不足。

例如，您可能希望避免从公共internet下载容器镜像，而这些容器镜像第一次是允许下载的。

也许您有一个内部镜像仓库，并且只有这个镜像仓库中的映像才能部署到集群中。

_如何强制在集群中只能部署受信任的容器？_

RBAC没有这方面的政策。

网络策略不起作用。

_你该怎么办？_

您可以使用许可控制器 [Admission controller](https://kubernetes.io/docs/reference/access-authn-authz/admission-controllers/) 来检查提交到集群的资源。

### 只允许从已知镜像仓库中部署容器

您可能需要考虑的最常见的自定义策略之一是限制镜像在集群中部署。

[The following tutorial explains how you can use the Open Policy Agent to restrict not approved images](https://blog.openpolicyagent.org/securing-the-kubernetes-api-with-open-policy-agent-ce93af0552c3#3c6e).
该教程解释了代理如何使用以下策略限制未经批准的镜像。

### 强制入口主机名的唯一性

当用户创建入口清单时，他们可以使用其中的任何主机名。

```yaml|highlight=7|title=ingress.yaml
apiVersion: networking.k8s.io/v1beta1
kind: Ingress
metadata:
  name: example-ingress
spec:
  rules:
    - host: first.example.com
      http:
        paths:
          - backend:
              serviceName: service
              servicePort: 80
```

但是，您可能希望防止用户多次使用同一主机名并相互覆盖。

Open Policy Agent的官方文档中有一个关于如何检查入口资源作为验证webhook的一部分的教程 [a tutorial on how to check Ingress resources as part of the validation webhook](https://www.openpolicyagent.org/docs/latest/kubernetes-tutorial/#4-define-a-policy-and-load-it-into-opa-via-kubernetes).

### 仅在入口主机名中使用经批准的域名

当用户创建入口清单时，他们可以使用其中的任何主机名。

```yaml|highlight=7|title=ingress.yaml
apiVersion: networking.k8s.io/v1beta1
kind: Ingress
metadata:
  name: example-ingress
spec:
  rules:
    - host: first.example.com
      http:
        paths:
          - backend:
              serviceName: service
              servicePort: 80
```

但是，您可能希望防止用户使用无效主机名。

Open Policy Agent的官方文档中有一个关于如何检查入口资源作为验证webhook的一部分的教程[a tutorial on how to check Ingress resources as part of the validation webhook](https://www.openpolicyagent.org/docs/latest/kubernetes-tutorial/#4-define-a-policy-and-load-it-into-opa-via-kubernetes).
