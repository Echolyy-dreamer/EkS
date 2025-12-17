# 🛠️ EKS ALB Target Group 实验步骤（IP Mode vs Instance Mode）

> 本 README **仅记录实验操作步骤**，用于在 Amazon EKS 中对比 ALB Target Group 的 **IP 模式** 与 **Instance 模式** 的行为差异。

---

## 一、基础环境准备

### 1. 创建 EKS 集群

```bash
eksctl create cluster \
  --name my-alb-test-cluster \
  --region us-east-1 \
  --node-type t3.medium \
  --nodes 2 \
  --with-oidc  # 自动创建并关联OIDC Provider（IRSA核心依赖）
```

确认节点就绪：

```bash
kubectl get nodes -o wide
```

---

### 2. 安装 AWS Load Balancer Controller

> 阶段 1：创建 IAM 策略和服务账户
下载 IAM Policy 文档：

```bash
Invoke-WebRequest -Uri "https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/main/docs/install/iam_policy.json" -OutFile "iam_policy.json"
```

创建 IAM 策略：
# 请替换 <YOUR_REGION> 为您的集群所在区域 (例如 us-east-1，根据您 cluster-info 的输出)
```bash
$policyArn = aws iam create-policy --policy-name AWSLoadBalancerControllerIAMPolicy --policy-document file://iam_policy.json --query 'Policy.Arn' --output text
Write-Host "IAM Policy ARN: $policyArn"
```bash

创建 Kubernetes IAM 服务账户 (IRSA)：
# 替换 <YOUR_CLUSTER_NAME> 和 <YOUR_REGION>
```bash
eksctl create iamserviceaccount `
    --cluster my-alb-test-cluster `
    --namespace kube-system `
    --name aws-load-balancer-controller `
    --role-name AmazonEKSLoadBalancerControllerRole `
    --attach-policy-arn $policyArn `
    --approve
```bash
> 阶段 2：通过 Helm 部署 Controller
添加 Helm 仓库：
```bash
helm repo add aws-load-balancer-controller https://aws.github.io/eks-charts
helm repo update
```bash

部署 Controller：
# 替换 <YOUR_CLUSTER_NAME>
```bash
helm install aws-load-balancer-controller aws-load-balancer-controller/aws-load-balancer-controller `
    --set clusterName=my-alb-test-cluster `
    --set serviceAccount.create=false `
    --set serviceAccount.name=aws-load-balancer-controller `
    --namespace kube-system
```bash

确认 Controller 正常运行：

```bash
kubectl -n kube-system get pods -l app.kubernetes.io/name=aws-load-balancer-controller
```

---

### 3. 部署基础应用（Nginx）

所有实验模式共用同一个 Deployment。

```bash
kubectl apply -f nginx-base.yaml
```

记录 Pod IP（用于后续验证）：

```bash
kubectl get pods -l app=nginx -o wide
```

---

## 二、实验一：IP Mode（ClusterIP + target-type: ip）

### 1. 部署 Service 与 Ingress

```bash
kubectl apply -f nginx-ip-mode.yaml
```

关键点：

* Service 类型：`ClusterIP`
* Ingress 注解：

  ```yaml
  alb.ingress.kubernetes.io/target-type: ip
  ```

---

### 2. 等待 ALB 创建

```bash
kubectl get ingress nginx-ingress-ip -o wide
```

确认 Ingress Event 无权限错误：

```bash
kubectl describe ingress nginx-ingress-ip
```

获取 ALB 地址：

```bash
kubectl get ingress nginx-ingress-ip -o jsonpath='{.status.loadBalancer.ingress[0].hostname}'
```

---

### 3. 验证 Target Group 与流量路径

* AWS 控制台 / CLI：

  * Target Group 中注册的目标应为 **Pod IP**

* 访问 ALB：

```bash
curl http://<ALB_IP_MODE_ADDRESS>
```

* 查看 Nginx Pod 日志：

```bash
kubectl logs <NGINX_POD_NAME>
```

预期：

* 流量直接到达 Pod
* 客户端真实 IP 通过 `X-Forwarded-For` Header 保留

---

## 三、实验二：Instance Mode（NodePort + target-type: instance）

### 1. 部署 Service 与 Ingress

```bash
kubectl apply -f nginx-instance-mode.yaml
```

关键点：

* Service 类型：`NodePort`
* Ingress 注解：

  ```yaml
  alb.ingress.kubernetes.io/target-type: instance
  ```

---

### 2. 验证 NodePort 与 Target Group

```bash
kubectl get service nginx-service-instance
```

记录分配的 NodePort（示例：30456）。

* AWS 控制台 / CLI：

  * Target Group 中注册的目标应为 **Worker Node IP : NodePort**

---

### 3. 验证 SNAT 行为

访问 ALB：

```bash
curl http://<ALB_INSTANCE_MODE_ADDRESS>
```

查看 Pod 日志：

```bash
kubectl logs <NGINX_POD_NAME>
```

预期：

* Pod 看到的源 IP 为 **Worker Node IP**
* 客户端真实 IP 丢失（默认 kube-proxy SNAT 行为）

---

## 四、实验完成

至此，可通过 Target Group 注册对象、Pod 日志与网络路径，对比 IP Mode 与 Instance Mode 的差异。
