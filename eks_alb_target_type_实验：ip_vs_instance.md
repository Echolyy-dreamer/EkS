# EKS 中 ALB Target Group: IP 模式 vs Instance 模式实验

## 一、项目背景
在 Amazon EKS 中暴露HTTP/HTTPS服务时，AWS Load Balancer Controller（ALB Controller）是官方推荐的Ingress实现方式。但在实际架构设计中，一个经常被忽略、却对网络路径、延迟、可观测性和安全模型产生实质影响的关键点是：
> ALB Target Group 选择 IP 模式，还是 Instance 模式？
本实验以工程和架构视角，对比这两种模式在EKS中的配置要求、流量路径、网络行为以及隐含的系统设计差异，并给出实践结论。


## 二、实验目标

核心目标：

> 在 EKS 环境中，对比ALB Target Group 的**IP 模式** 与 **Instance 模式** 在真实流量路径和组件参与度上的差异，得出适用于生产环境的最佳实践。

具体关注点包括：
- Ingress/Service 的配置差异
- 流量在AWS与Kubernetes中的实际转发路径
- kube-proxy 是否参与转发
- SNAT行为是否发生
- 对客户端真实IP的影响
- 对IAM/控制面的依赖差异

## 三、实验环境

- Amazon EKS（VPC CNI）
- AWS Load Balancer Controller
- ALB（Application Load Balancer）
- Ingress 资源
- 对比两种 Target Group 类型：
  - `target-type: ip`
  - `target-type: instance`

## 四、模式一：ALB IP 模式（推荐实践）

### 1. 关键配置

- **IngressClass**：`alb`
- **Ingress Controller**：AWS Load Balancer Controller
- **Service 类型**：`ClusterIP`
- **关键注解**：

```yaml
alb.ingress.kubernetes.io/target-type: ip
```

### 2. 架构与流量路径说明（文字版流向图）

```
Client
  │
  ▼
ALB (Application Load Balancer)
  │  （Target Group: IP）
  ▼
Pod IP（VPC Secondary IP）
```

**关键说明**：

- ALB 直接将 **Pod IP** 注册到 Target Group
- Pod IP 来自 **VPC CNI** 分配的 ENI Secondary IP
- 流量**不经过 NodePort，不经过 kube-proxy**



### 3. 网络与系统行为分析

- **单跳转发**：ALB → Pod
- **kube-proxy 完全绕过**
- **无额外 SNAT**（ALB 层已完成转发）
- **客户端真实 IP 保留**：
  - 通过 `X-Forwarded-For` Header 传递给 Pod


### 4. 架构优势总结

- 网络路径最短，延迟最低
- 流量模型清晰、可预测
- 适合微服务、高并发、对延迟敏感的业务
- 更符合 AWS 对 EKS + ALB 的云原生现代设计方向

---

## 五、模式二：ALB Instance 模式（兼容但不推荐）

### 1. 关键配置

- **IngressClass**：`alb`
- **Ingress Controller**：AWS Load Balancer Controller
- **Service 类型**：`NodePort`
- **关键注解**：

```yaml
alb.ingress.kubernetes.io/target-type: instance
```

---

### 2. 架构与流量路径说明（文字版流向图）

```
Client
  │
  ▼
ALB
  │  （Target Group: Instance）
  ▼
Worker Node IP : NodePort
  │
  ▼
kube-proxy
  │
  ▼
Pod
```

---

### 3. 网络与系统行为分析

- **多跳转发**：ALB → Node → kube-proxy → Pod
- **kube-proxy 必须参与转发**
- **SNAT 行为发生**：
  - kube-proxy 在转发至 Pod 前，通常会将源 IP 转换为 Node IP
- **客户端真实 IP 丢失**（除非额外配置）

---

### 4. 架构代价

- 增加网络跳点与转发复杂度
- 排错成本更高
- 不符合云原生“直达后端”的设计理念
- 更多历史兼容意义，而非新集群首选

---

## 六、对比总结

| 维度 | IP 模式 | Instance 模式 |
|----|----|----|
| Service 类型 | ClusterIP | NodePort |
| kube-proxy 参与 | 否 | 是 |
| 网络跳数 | 少 | 多 |
| SNAT | 无 | 有 |
| 客户端 IP | 保留（Header） | 通常丢失 |


---

## 七、实验中遇到的关键问题（IAM）

在实验过程中，ALB Controller 在 Reconcile Ingress 时多次出现 403 错误：

```
elasticloadbalancing:DescribeListenerAttributes
```

### 根因分析

- ALB Controller **高度依赖 ELBv2 API 的 Describe 系列权限**
- 不同来源的 IAM Policy（旧博客 / 旧文档 / 自定义裁剪）可能缺少新 API
- Controller 会在调谐过程中频繁调用这些 API，一旦缺失即导致 Ingress 无法生效

### 结论

> **必须使用官方最新版本的 AWSLoadBalancerControllerIAMPolicy，或完整同步其 Action 列表。**

---
## 八、最终结论（最佳实践）

- **新建 EKS 集群 + ALB Ingress 场景**：
  - 默认选择 **IP 模式**
- **仅在特殊兼容需求下使用 Instance 模式**
- 确保 ALB Controller 的 IAM Policy 与官方版本保持同步

---

## 九、项目价值

本实验验证配置差异，从 **网络路径、数据平面、控制平面和 IAM 依赖** 四个层面理解了 EKS + ALB 的真实工作方式。



