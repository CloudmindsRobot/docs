# 集群配置

集群配置最佳实践。

## 认可的Kubernetes配置

Kubernetes 是非常灵活的，可以通过几种不同的方式进行配置。

但是您如何知道您的集群的推荐配置是什么呢？

最好的选择是将集群与标准引用进行比较。

以Kubernetes为例，参考的是互联网安全中心（CIS）基准。

### 集群通过CIS基准

CIS 为您的代码安全的最佳实践提供了一些指导原则和基准测试

他们还为Kubernetes维护了一个基准，您可以从官方网站下载[download from the official website](https://www.cisecurity.org/benchmark/kubernetes/).

虽然您可以阅读冗长的指南并手动检查集群是否兼容，但更简单的方法是下载并执行 [`kube-bench`](https://github.com/aquasecurity/kube-bench).

[`kube-bench`](https://github.com/aquasecurity/kube-bench) 是一个旨在自动化CIS-Kubernetes基准测试并报告集群中错误配置的工具。

输出示例：

```terminal|title=bash
[INFO] 1 Master Node Security Configuration
[INFO] 1.1 API Server
[WARN] 1.1.1 Ensure that the --anonymous-auth argument is set to false (Not Scored)
[PASS] 1.1.2 Ensure that the --basic-auth-file argument is not set (Scored)
[PASS] 1.1.3 Ensure that the --insecure-allow-any-token argument is not set (Not Scored)
[PASS] 1.1.4 Ensure that the --kubelet-https argument is set to true (Scored)
[PASS] 1.1.5 Ensure that the --insecure-bind-address argument is not set (Scored)
[PASS] 1.1.6 Ensure that the --insecure-port argument is set to 0 (Scored)
[PASS] 1.1.7 Ensure that the --secure-port argument is not set to 0 (Scored)
[FAIL] 1.1.8 Ensure that the --profiling argument is set to false (Scored)
```

> 请注意，使用kube bench无法检查受管集群（如GKE、EKS和AKS）的主节点。主节点由云提供商控制和管理。

### 禁用云提供程序元数据API

云平台（AWS、Azure、GCE等）经常在本地向实例公开元数据服务。

默认情况下，运行在实例上的pod可以访问这些api，并且可以包含该节点的云凭据，或者提供kubelet凭据之类的数据。

这些凭据可用于在群集中升级或升级到同一帐户下的其他云服务。

### 限制访问alpha或beta功能

Kubernetes的Alpha和beta特性正在积极开发中，可能存在导致安全漏洞的限制或错误。

总是要评估alpha或beta功能的价值对您的安全可能带来的风险。

如果有疑问，请禁用该功能。

## 身份验证

使用“kubectl”时，您将根据kubeapi服务器组件对自己进行身份验证。

Kubernetes支持不同的身份验证策略：

- **静态令牌**：很难使其无效，应避免使用
- **引导令牌**：与上面的静态令牌相同
- **基本身份验证**以明文形式通过网络传输凭据
- **X509客户端证书**需要定期更新和重新分发客户端证书
- **服务帐户令牌**是集群中运行的应用程序和工作负载的首选身份验证策略
- **OpenID Connect（OIDC）令牌**：针对最终用户的最佳身份验证策略是使用OIDC与您的身份提供商（如AD、AWS IAM、GCP IAM等）进行集成。

您可以更详细地了解这些策略 [in the official documentation](https://kubernetes.io/docs/reference/access-authn-authz/authentication/)。

### 使用OpenID（OIDC）令牌作为用户身份验证策略

Kubernetes支持各种身份验证方法，包括OpenID Connect（OIDC）。

OpenID Connect允许单点登录（SSO），比如Google Identity 连接到Kubernetes集群和其他开发工具。

您不需要分别记住或管理凭据。

您可以将多个集群连接到同一个OpenID提供程序。

您可以在[learn more about the OpenID connect in Kubernetes](https://thenewstack.io/kubernetes-single-sign-one-less-identity/)中了解更多关于Kubernetes中OpenID连接的信息。

## 基于角色的访问控制（RBAC）

基于角色的访问控制（RBAC）允许您定义如何访问集群中的资源的策略。

### ServiceAccount令牌仅用于应用程序和控制器

服务帐户令牌不应用于尝试与Kubernetes集群交互的最终用户，但它们是Kubernetes上运行的应用程序和工作负载的首选身份验证策略。

## 日志记录设置

您应该收集并集中存储集群中运行的所有工作负载和集群组件本身的日志。

### 日志有一个保留和归档策略

你应该保留30-45天的历史日志。

### 日志收集来自节点、控制台和审计中

从何处收集日志：

- 节点（kubelet，容器运行时）
- 控制台（API服务器、调度程序、控制器管理器）
- Kubernetes审计（对API服务器的所有请求）

你应该收集什么：

- 应用程序名称。从元数据标签中获取。
- 应用程序实例。从元数据标签中获取。
- 应用程序版本。从元数据标签中获取。
- 群集ID。从Kubernetes群集中获取。
- 容器名称。从Kubernetes API中获取。
- 运行此容器的群集节点。从Kubernetes群集中获取。
- 运行容器的Pod名称。从Kubernetes群集中获取。
- 命名空间。从Kubernetes群集中获取。

### 希望每个节点上都有一个守护进程来收集日志，而不是边车

应用程序日志应该设成stdout而不是文件。

每个节点上的守护进程可以从容器运行时收集日志[A daemon on each node can collect the logs from the container runtime](https://rclayton.silvrback.com/container-services-logging-with-docker#effective-logging-infrastructure)（如果要记录到文件，则可能需要为每个pod提供一个边车容器）.

### 设置日志聚合工具

使用EFK stack (Elasticsearch, Fluentd, Kibana), DataDog, Sumo Logic, Sysdig, GCP Stackdriver, Azure Monitor, AWS CloudWatch等日志聚合工具。
