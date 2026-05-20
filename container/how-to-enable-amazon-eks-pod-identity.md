# 如何启用 Amazon EKS Pod Identity 并为运行工作负载的 Service Account 分配角色

作者：[Arun Nagpal](https://repost.aws/community/users/USCNhMe9CQRR-rRe4mavyFXw) | 2024年1月12日

原文链接：https://repost.aws/articles/AR5ogohRSfRzCFh8ooFeq8kg/how-to-enable-amazon-eks-pod-identity-and-assign-role-to-service-account-running-workloads

阅读时间：3 分钟 | 内容级别：高级

---

## 概述

EKS Pod Identity 是 Amazon EKS（Elastic Kubernetes Service）推出的一项功能，它简化了集群管理员为 Kubernetes 应用程序配置 AWS IAM（Identity and Access Management）权限的方式。

## EKS Pod Identity 的优势

1. **简化 IAM 权限配置**：EKS Pod Identity 使 Kubernetes 应用程序的 IAM 权限配置更加简单。它通过 EKS 控制台、API 和 CLI 提供更简洁的流程，减少了配置 IAM 权限所需的步骤。
2. **权限策略复用**：EKS Pod Identity 支持跨 IAM 角色复用权限策略。这简化了策略管理，并允许在多个集群中轻松使用同一个 IAM 角色。
3. **细粒度 IAM 管理**：通过 EKS Pod Identity，您可以为运行在 AWS 上的 Kubernetes 应用程序配置细粒度的 AWS IAM 权限，从而对应用程序的 IAM 权限进行更精细的控制。

这些优势使 EKS Pod Identity 成为简化 IAM 权限管理、增强运行在 Amazon EKS 集群上的云原生应用程序安全性和效率的重要功能。

---

## 步骤 1：创建具有所需权限的 IAM 角色并更新信任策略

创建一个 IAM 角色，并将信任策略中的 Principal 设置为 `pods.eks.amazonaws.com`。

![创建 IAM 角色](https://repost.aws/media/postImages/original/IMfyEyXBP1S72Yp4U2gzc6EA)

![信任策略配置](https://repost.aws/media/postImages/original/IMgUEeocpPS_m8BC1DxaLhCw)

## 步骤 2：在集群上安装 "Amazon EKS Pod Identity Agent" 插件并验证 DaemonSet 运行状态

在 EKS 控制台中：

![EKS 集群插件页面](https://repost.aws/media/postImages/original/IMTOKa1HEaSFKD7JMYL7V6VA)

点击 **获取更多插件（Get more Add-ons）**

![获取更多插件](https://repost.aws/media/postImages/original/IME7wOaH0xRDik9CGPQUqiQA)

选中 **EKS Pod Identity Agent** 并点击下一步

![选择 EKS Pod Identity Agent](https://repost.aws/media/postImages/original/IMQ7tk9v8PQCS1yc10r7obZA)

![插件配置](https://repost.aws/media/postImages/original/IMtJY9to7qQSOBgKyoEK3z3g)

![确认安装](https://repost.aws/media/postImages/original/IMHIyiqYPDQBK68taDSZQr9A)

点击 **创建（Create）**：

![创建插件](https://repost.aws/media/postImages/original/IM0PK8XxuyTxuUFR0GKKWhhA)

这将在 `kube-system` 命名空间中启动 EKS Pod Identity DaemonSet。

![DaemonSet 运行状态](https://repost.aws/media/postImages/original/IMeYATpsj9Qpir9pT-Ww50sg)

**命令行方式：**

```bash
aws eks create-addon \
  --cluster-name <替换为您的集群名称> \
  --addon-name eks-pod-identity-agent \
  --addon-version v1.0.0-eksbuild.1
```

## 步骤 3：创建 Pod Identity 关联

1. 确认 Pod 使用的 Service Account

![确认 Service Account](https://repost.aws/media/postImages/original/IMMA7zVLpdTfO4IDrLzMKAow)

2. 在 EKS 控制台中，选择集群并进入 **访问（Access）** 选项卡，在 "Pod Identity Associations" 部分点击 **创建 Pod Identity 关联（Create Pod Identity association）**

![创建 Pod Identity 关联](https://repost.aws/media/postImages/original/IM40PjfWvoRGehxI9l9CThbw)

3. 选择 IAM 角色、命名空间和 Service Account，然后点击 **创建（Create）**

![选择角色和命名空间](https://repost.aws/media/postImages/original/IMyitrwI2oSV-BFhrJzK6GKg)

4. 创建完成后，EKS 控制台的访问选项卡中将显示该关联条目

![关联条目](https://repost.aws/media/postImages/original/IMrvM8y8dRRfyGAMFhegm5QQ)

## 步骤 4：使用 Service Account 验证访问权限

1. 创建 Pod YAML 配置文件：

```yaml
# aws-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  labels:
    run: aws-pod
  name: aws-pod
  namespace: demo
spec:
  serviceAccountName: demo-sa
  containers:
  - image: public.ecr.aws/aws-cli/aws-cli
    command:
      - "aws"
      - "s3"
      - "ls"
    name: aws-pod
    resources: {}
  dnsPolicy: ClusterFirst
  restartPolicy: Always
```

2. 使用 kubectl 启动 Pod：

```bash
kubectl create -f aws-pod.yaml
```

3. 验证 Pod 是否有权限访问 S3：

```bash
kubectl logs -f aws-pod -n demo
```

**预期结果**：输出该账户中的 S3 存储桶列表。
