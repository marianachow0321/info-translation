# 使用 IAM 简化 Amazon ElastiCache for Redis 集群的访问管理

作者：Maheedhar Gunturu、Andrey Belik、Christoph Koerner | 2022年11月29日

原文链接：https://aws.amazon.com/blogs/database/simplify-managing-access-to-amazon-elasticache-for-redis-clusters-with-iam/

---

## 概述

[Amazon ElastiCache for Redis](https://aws.amazon.com/elasticache/redis/) 是一项完全托管的、兼容 Redis 的内存缓存服务，提供微秒级速度以支持实时应用程序。ElastiCache for Redis 结合了开源 Redis 的速度、简洁性和多功能性，以及 AWS 提供的可靠性、可扩展性、可管理性和安全性，为媒体和娱乐、金融服务、电子商务、广告技术、游戏和医疗保健等领域最苛刻的实时应用程序提供支持。

我们很高兴地宣布，您现在可以使用 AWS Identity and Access Management (IAM) 身份连接到 ElastiCache for Redis 集群。当您使用启用了传输加密的 ElastiCache for Redis 7.0 或更高版本时，此功能无需额外费用。使用 IAM 等集中式身份存储管理 Redis 集群的访问权限，使您能够轻松遵循安全最佳实践和指南。您现在可以通过 IAM 策略语句定义哪些 IAM 身份可以访问您的 ElastiCache for Redis 集群。它还为您提供了一种安全且一致的方式来管理所有 AWS 资源的访问权限。

应用程序现在可以生成临时安全凭证，用于建立其身份并验证对 Redis 集群的访问权限。这种新的身份验证方法旨在与基于角色的访问控制 (RBAC) 结合使用，管理员可以指定访问字符串，进一步定义每个 ElastiCache 用户可以访问的命令和键。AWS 客户现在可以通过单点登录 (SSO) 将身份验证（SAML、LDAP、OpenID 或 OAuth）与自己的目录联合，直接连接到 ElastiCache for Redis。

在本文中，我们将向您展示如何使用 IAM 身份对 ElastiCache for Redis 集群进行身份验证和访问。

## IAM 身份验证的优势

IAM 为多种 AWS 服务和资源（包括 ElastiCache for Redis）提供细粒度的访问控制。使用 IAM，您可以通过提供必要的策略来指定谁可以访问哪些服务和资源。默认情况下拒绝访问，只有在策略明确授予访问权限时才允许访问。通过 IAM 策略，您可以管理员工和系统的权限，以确保最小权限原则。

以前，您需要使用 Redis 用户密码或将密码存储在 AWS Secrets Manager 或第三方密钥管理工具中来设置 ElastiCache for Redis 集群的身份验证。然而，在托管许多应用程序的大型组织中，密码在轮换时经常会不同步。IAM 身份验证通过允许从集中式服务进行访问管理来提供简化的安全态势。通过 IAM 身份验证，ElastiCache 用户可以在连接到 Redis 集群时使用其 IAM 身份。IAM 身份验证对 ElastiCache 用户的好处包括：

- **集中管理凭证**：使用 IAM 集中管理凭证，与管理其他 AWS 服务和资源的方式一致
- **验证临时安全凭证**：验证应用程序生成的临时安全凭证以认证对 Redis 集群的访问，并在令牌失效时进行密码轮换
- **与 RBAC 协同工作**：旨在与 RBAC 结合使用，您可以进一步定义每个 ElastiCache 用户可以访问的命令和键

## 解决方案概述

以下图表说明了解决方案架构。

![解决方案架构图](https://d2908q01vomqb2.cloudfront.net/887309d048beef83ad3eabf2a79a64a389ab1c9f/2022/11/28/DBBLOG-2711_img1-1024x454.png)

IAM 身份验证通过生成短期（15 分钟）临时安全凭证来工作，您的应用程序在连接到 ElastiCache 集群时将此凭证提供给 Redis 客户端。然后使用此凭证建立您的身份并应用相关策略，确定可以连接到 Redis 集群的资源。要将 IAM 身份与 ElastiCache for Redis 一起使用，您需要通过 ElastiCache API、AWS CLI 或 AWS 管理控制台创建启用 IAM 的 ElastiCache 用户，然后附加允许新 `elasticache:Connect` 操作的策略。附加必要的策略后，将 ElastiCache 用户添加到相应的用户组和复制组。然后应用程序生成临时安全凭证，用于建立 IAM 身份，将其映射到 ElastiCache for Redis 用户，并授予应用程序对 ElastiCache 集群的访问权限。我们将在本文的其余部分详细介绍这些步骤。

## 前提条件

此功能适用于启用了传输加密的 ElastiCache for Redis 集群（7.0 及以上版本）。开始之前，我们假设您已设置以下基础设施：

1. 一个 Amazon VPC，包含公有子网和私有子网，用于托管 Amazon EC2 实例和 ElastiCache for Redis 集群
2. 一个 EC2 实例，用于连接到 ElastiCache for Redis 集群（注意：只能从同一 VPC 内访问 ElastiCache）
3. 允许从 EC2 实例访问 ElastiCache for Redis 集群的安全组
4. 具有修改 IAM 权限和策略的适当权限
5. 具有修改和创建 EC2 实例和 ElastiCache for Redis 集群的适当权限
6. 最新版本的 AWS CLI

验证您的 AWS CLI 版本（2.9.0 或更高版本）：

```bash
aws --version
```

## 使用 IAM 设置 ElastiCache 身份验证

在本节中，我们将引导您完成使用 IAM 身份设置 ElastiCache 身份验证的步骤。以下图表说明了本文其余部分的工作流程。

![工作流程图](https://d2908q01vomqb2.cloudfront.net/887309d048beef83ad3eabf2a79a64a389ab1c9f/2022/11/28/DBBLOG-2711_img2-1024x131.png)

您创建一个 IAM 角色，然后附加一个策略，允许该 IAM 角色作为 ElastiCache 用户连接到 ElastiCache 集群。然后创建一个 ElastiCache 用户组，将默认用户和新创建的启用 IAM 的 ElastiCache 用户添加到其中，然后将其附加到复制组。

### 设置 IAM 角色

本节说明如何创建允许您连接到 ElastiCache 集群的策略，并将其与 IAM 角色关联。然后将此角色附加到具有本节中概述的策略的 Amazon EC2 实例配置文件。作为一般最佳实践，您不应使用根账户来测试 ElastiCache for Redis 集群的 IAM 身份验证，因为根账户会自动获得对所有资源的访问权限。

开始之前，确认您的 AWS CLI 使用的是正确的账户详细信息：

```bash
aws sts get-caller-identity
```

```json
{
    "Account": "<your-account-id>",
    "UserId": "<your-user-id>",
    "Arn": "<your-arn>"
}
```

创建名为 `trust-policy.json` 的文件，复制以下配置，允许您的 EC2 实例承担新角色：

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "sts:AssumeRole"
            ],
            "Principal": {
                "Service": [
                    "ec2.amazonaws.com"
                ]
            }
        }
    ]
}
```

创建名为 `policy.json` 的文件，复制以下配置：

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "elasticache:Connect"
      ],
      "Resource": [
        "arn:aws:elasticache:us-east-1:<your-account-id>:replicationgroup:ec-cluster-iam-tutorial",
        "arn:aws:elasticache:us-east-1:<your-account-id>:user:iam-test-user-01"
      ]
    }
  ]
}
```

此策略仅允许访问本文中使用的资源。您可以通过在 ARN 路径中添加通配符将其扩展到其他资源。

创建 EC2 实例在连接到 ElastiCache 集群时承担的 IAM 角色：

```bash
aws iam create-role \
  --role-name "elasticache-iam-auth-app" \
  --assume-role-policy-document file://trust-policy.json
```

使用 `policy.json` 文件创建允许访问 ElastiCache 集群的 IAM 策略：

```bash
aws iam create-policy \
  --policy-name "elasticache-iam-policy" \
  --policy-document file://policy.json
```

将 IAM 策略附加到角色：

```bash
aws iam attach-role-policy \
  --role-name "elasticache-iam-auth-app" \
  --policy-arn "arn:aws:iam::<your-account-id>:policy/elasticache-iam-policy"
```

验证策略已附加：

```bash
aws iam list-attached-role-policies \
  --role-name "elasticache-iam-auth-app"
```

输出列出已附加的策略：

```json
{
    "AttachedPolicies": [
        {
            "PolicyName": "elasticache-iam-policy",
            "PolicyArn": "arn:aws:iam::<your-account-id>:policy/elasticache-iam-policy"
        }
    ]
}
```

创建实例配置文件，稍后将其附加到用于连接 ElastiCache 集群的 EC2 实例：

```bash
aws iam create-instance-profile --instance-profile-name ElastiCacheIAMConnect
```

将启用 IAM 的 ElastiCache 角色添加到此实例配置文件：

```bash
aws iam add-role-to-instance-profile \
  --role-name "elasticache-iam-auth-app" \
  --instance-profile-name "ElastiCacheIAMConnect"
```

现在转到 Amazon EC2 控制面板，使用 **操作 > 安全 > 修改 IAM 角色** 选项，选择 `ElastiCacheIAMConnect`，并更新附加到 EC2 实例的 IAM 角色。

SSH 进入 EC2 实例，验证实例角色已生效：

```bash
aws sts get-caller-identity
```

```json
{
    "Account": "<your-account-id>",
    "UserId": "<your-user-id>",
    "Arn": "arn:aws:sts::<your-account-id>:assumed-role/elasticache-iam-auth-app/<your-instance-id>"
}
```

稍后您将使用此 EC2 实例连接到 ElastiCache 集群。

### 设置启用 IAM 的 ElastiCache 用户

现在您可以使用 IAM 进行身份验证，让我们创建启用 IAM 的 ElastiCache 用户，并收紧内置默认用户的权限，确保最小权限原则。

创建新的默认用户并提供密码：

```bash
aws elasticache create-user \
  --user-id "restricted-default-user" \
  --user-name "default" \
  --engine "redis" \
  --authentication-mode "Type=password,Passwords=<your-password>" \
  --access-string "off -@all"
```

我们通过提供 `off -@all` 作为访问字符串来限制此用户的权限。

创建新的启用 IAM 的 ElastiCache 用户：

```bash
aws elasticache create-user \
  --user-name "iam-test-user-01" \
  --user-id "iam-test-user-01" \
  --authentication-mode "Type=iam" \
  --engine "redis" \
  --access-string "on ~* +@all"
```

输出类似如下：

```json
{
    "UserId": "iam-test-user-01",
    "UserName": "iam-test-user-01",
    "Status": "active",
    "Engine": "redis",
    "MinimumEngineVersion": "7.0",
    "AccessString": "on ~* +@all",
    "UserGroupIds": [],
    "Authentication": {
        "Type": "iam"
    },
    "ARN": "arn:aws:elasticache:us-east-1:<your-account-id>:user:iam-test-user-01"
}
```

创建用户组，添加新的默认用户 `restricted-default-user` 和启用 IAM 的 ElastiCache 用户 `iam-test-user-01`（每个 ElastiCache 用户组必须包含一个默认用户）：

```bash
aws elasticache create-user-group \
  --user-group-id "iam-test-ug-01" \
  --engine "redis" \
  --user-ids "restricted-default-user" "iam-test-user-01"
```

输出类似如下：

```json
{
    "UserGroupId": "iam-test-ug-01",
    "Status": "creating",
    "Engine": "redis",
    "UserIds": [
        "restricted-default-user",
        "iam-test-user-01"
    ],
    "MinimumEngineVersion": "7.0",
    "ReplicationGroups": [],
    "ARN": "arn:aws:elasticache:us-east-1:<your-account-id>:usergroup:iam-test-user-group-01"
}
```

创建复制组并将用户组附加到其中（注意：一个复制组仅支持一个用户组）：

```bash
aws elasticache create-replication-group \
  --replication-group-id "ec-cluster-iam-tutorial" \
  --replication-group-description "test for ElastiCache IAM Authentication" \
  --engine "redis" \
  --engine-version "7.0" \
  --cache-node-type "cache.m6g.large" \
  --transit-encryption-enabled \
  --cache-subnet-group "<insert-subnet-group>" \
  --security-group-ids "<insert-security-group>" \
  --tags Key=env,Value=test Key=task,Value=ec-iam-tutorial \
  --user-group-ids "iam-test-ug-01"
```

如果您有子网组名称和安全组 ID，请在上述命令中提供相应的值。

## 测试 IAM 身份验证

对于测试，您可以使用 redis-cli 或 AWS 提供的基于 Java 的演示应用程序连接到 ElastiCache 集群。演示应用程序使用 Redis Lettuce 客户端并实现 Redis 凭证提供程序，该提供程序使用 Signature V4 签名过程生成临时安全凭证。请按照 README 中的说明在 EC2 实例上克隆和设置演示应用程序。注意：只能从同一 VPC 内访问 ElastiCache 集群。

### 使用演示应用程序生成临时安全凭证

在 EC2 实例上使用 GitHub 仓库中提供的说明设置演示应用程序后，使用以下命令生成令牌：

```bash
cd elasticache-iam-auth-demo-app/
java -cp target/ElastiCacheIAMAuthDemoApp-1.0-SNAPSHOT.jar \
  com.amazon.elasticache.IAMAuthTokenGeneratorApp \
  --region us-east-1 \
  --replication-group-id ec-cluster-iam-tutorial \
  --user-id iam-test-user-01
```

此命令使用您的 EC2 实例角色生成临时安全凭证。输出类似如下：

```
ec-cluster-iam-tutorial/?Action=connect&User=iam-test-user-01&X-Amz-Algorithm=AWS4-HMAC-SHA256&X-Amz-Date=20221104T231633Z&X-Amz-SignedHeaders=host&X-Amz-Expires=900&X-Amz-Credential=<redacted>us-east-1%2Felasticache%2Faws4_request&X-Amz-Signature=<redacted>
```

这是临时安全凭证，从创建时起有效期为 15 分钟。

### 选项 A：使用临时安全凭证和 redis-cli 连接到 ElastiCache 集群

您现在可以通过在终端运行以下命令来连接 redis-cli 会话：

```bash
redis-cli -h <host> -p 6379 --tls
> AUTH iam-test-user-01 <insert-token>
```

### 选项 B：使用演示应用程序连接到 ElastiCache 集群

演示应用程序使用默认的 AWS 凭证提供程序链，通过您的 AWS 调用者身份生成临时安全凭证：

```bash
java -jar target/ElastiCacheIAMAuthDemoApp-1.0-SNAPSHOT.jar \
  --redis-host <elasticache-cluster-configuration-endpoint> \
  --region us-east-1 \
  --replication-group-id ec-cluster-iam-tutorial \
  --user-id iam-test-user-01 \
  --tls
```

对于启用集群模式的复制组，请添加 `--cluster-mode` 标志。输出类似如下：

```
=> Connected clients: 1
Using credentials: iam-test-user-01, ec-cluster-iam-tutorial/?Action=connect&User=iam-test-user-01&X-Amz-Algorithm=AWS4-HMAC-SHA256&X-Amz-Date=20221104T231633Z&X-Amz-SignedHeaders=host&X-Amz-Expires=900&X-Amz-Credential=<redacted>us-east-1%2Felasticache%2Faws4_request&X-Amz-Signature=<redacted>
```

输出包括已连接客户端的数量、使用的临时安全凭证及其从创建时起的过期时间。演示应用程序还会缓存和重用凭证直到其过期（从创建时起 15 分钟）。凭证过期后，令牌将失效，应用程序会自动创建新的临时安全凭证。

## 清理

为了保持最小权限原则并避免产生未来费用，请删除您在本文中创建的资源。删除 ElastiCache 集群、EC2 实例、IAM 策略和角色。

## 总结

ElastiCache for Redis 用户现在可以使用 IAM 身份对其 Redis 集群进行身份验证和连接。这提供了一种与管理所有 AWS 资源一致的方式来管理对 ElastiCache for Redis 集群的访问。应用程序可以生成临时安全凭证来建立其 IAM 身份，并无缝连接到 ElastiCache for Redis 集群。临时安全凭证在过期时失效，并由应用程序自动轮换。与使用特定于服务的凭证相比，这简化了整体管理开销，并改善了应用程序基础设施的安全态势。

在本文中，您学习了如何为 ElastiCache for Redis 集群启用 IAM 身份验证。您创建了一个具有必要策略的 IAM 角色，指定可以承担该角色并连接到 ElastiCache for Redis 集群的主体。然后将此角色附加到 EC2 实例，并使用演示应用程序生成临时安全凭证。应用程序随后使用此凭证作为启用 IAM 的 ElastiCache 用户连接到 ElastiCache for Redis 集群，而不是使用 Redis 用户密码或通过 Secrets Manager 的长期安全凭证。演示应用程序还演示了如何在失效时自动轮换临时安全凭证，使遵循安全最佳实践变得简单。
