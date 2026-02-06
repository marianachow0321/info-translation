# 使用容器网络可观测性监控 EKS 集群的网络性能和流量

作者：[Donnie Prakoso](https://aws.amazon.com/blogs/aws/author/donnie/) | 2025年11月19日

原文链接：https://aws.amazon.com/cn/blogs/aws/monitor-network-performance-and-traffic-across-your-eks-clusters-with-container-network-observability/

## 概述

随着组织不断扩展其 Kubernetes 规模，通过部署微服务来逐步创新并更快地交付业务价值，这种增长使得对网络的依赖性增加，给平台团队带来了监控 EKS 中网络性能和流量模式的复杂挑战。因此，随着容器环境的扩展，组织难以保持运营效率，往往会延迟应用程序交付并增加运营成本。

今天，我很高兴地宣布 **[Amazon Elastic Kubernetes Service (Amazon EKS)](https://aws.amazon.com/eks/) 中的容器网络可观测性**，这是 Amazon EKS 中的一套全面的网络可观测性功能，您可以使用它来更好地衡量系统中的网络性能，并动态可视化 EKS 中网络流量的格局和行为。

以下是 Amazon EKS 中容器网络可观测性的快速概览：

![容器网络可观测性概览](https://d2908q01vomqb2.cloudfront.net/da4b9237bacccdf19c0760cab7aec4a8359010b0/2025/11/18/2025-news-eks-container-network-observability-rev-6.png)

容器网络可观测性通过提供工作负载流量的增强可见性来解决可观测性挑战。它提供了集群内网络流以及与集群外部目标之间网络流的性能洞察。这使您的 EKS 集群网络环境更具可观测性，同时为更精确的故障排除和调查工作提供内置功能。

## 开始使用 EKS 中的容器网络可观测性

我可以为新的或现有的 EKS 集群启用此新功能。对于新的 EKS 集群，在**配置可观测性**设置期间，我导航到**配置网络可观测性**部分。在这里，我选择**编辑容器网络可观测性**。我可以看到包含三个功能：**服务地图**、**流量表**和**性能指标端点**，这些功能由 [Amazon CloudWatch Network Flow Monitor](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/CloudWatch-NetworkFlowMonitor.html) 启用。

![配置网络可观测性](https://d2908q01vomqb2.cloudfront.net/da4b9237bacccdf19c0760cab7aec4a8359010b0/2025/11/19/image-8-4.png)

在下一页，我需要安装 **AWS Network Flow Monitor Agent**。

![安装代理](https://d2908q01vomqb2.cloudfront.net/da4b9237bacccdf19c0760cab7aec4a8359010b0/2025/11/13/2025-news-eks-container-network-observability-2.png)

启用后，我可以导航到我的 EKS 集群并选择**监控集群**。

这将带我进入集群可观测性仪表板。然后，我选择网络选项卡。

![网络选项卡](https://d2908q01vomqb2.cloudfront.net/da4b9237bacccdf19c0760cab7aec4a8359010b0/2025/11/13/2025-news-eks-container-network-observability-3.png)

## 全面的可观测性功能

EKS 中的容器网络可观测性提供了几个关键功能，包括性能指标、服务地图和流量表，具有三个视图：AWS 服务视图、集群视图和外部视图。

### 性能指标

使用**性能指标**，您现在可以直接从 Network Flow Monitor 代理抓取 Pod 和工作节点的网络相关系统指标，并将它们发送到您首选的监控目标。可用指标包括入站/出站流量计数、数据包计数、传输字节数，以及带宽、每秒数据包数和连接跟踪限制的各种超限计数器。以下屏幕截图显示了如何使用 [Amazon Managed Grafana](https://aws.amazon.com/grafana/) 可视化使用 Prometheus 抓取的性能指标的示例。

![Grafana 性能指标](https://d2908q01vomqb2.cloudfront.net/da4b9237bacccdf19c0760cab7aec4a8359010b0/2025/11/13/2025-news-eks-container-network-observability-4.png)

### 服务地图

使用**服务地图**功能，您可以动态可视化集群中工作负载之间的相互通信，使您能够一目了然地了解应用程序拓扑。服务地图通过突出显示关键指标（如重传、重传超时和通信 Pod 之间网络流的数据传输）来帮助您快速识别性能问题。

让我用一个示例电子商务应用程序向您展示这是如何工作的。服务地图提供了微服务架构的高级和详细视图。在这个电子商务示例中，我们可以看到三个核心微服务协同工作：**GraphQL 服务**充当 API 网关，协调前端和后端服务之间的请求。

当客户浏览产品或下订单时，GraphQL 服务协调与**产品服务**（用于目录数据、定价和库存）和**订单服务**（用于订单处理和管理）的通信。这种架构允许每个服务独立扩展，同时保持清晰的关注点分离。

![服务地图高级视图](https://d2908q01vomqb2.cloudfront.net/da4b9237bacccdf19c0760cab7aec4a8359010b0/2025/11/13/2025-news-eks-container-network-observability-8.png)

为了进行更深入的故障排除，您可以展开视图以查看各个 Pod 实例及其通信模式。详细视图揭示了微服务通信的复杂性。在这里，您可以看到每个服务的多个 Pod 实例以及它们之间的连接网络。

![服务地图详细视图](https://d2908q01vomqb2.cloudfront.net/da4b9237bacccdf19c0760cab7aec4a8359010b0/2025/11/17/2025-news-eks-container-network-observability-rev-1.png)

这种细粒度的可见性对于识别问题至关重要，例如负载分配不均、Pod 到 Pod 通信瓶颈，或者特定 Pod 实例遇到更高延迟的情况。例如，如果一个 GraphQL Pod 对特定产品 Pod 进行的调用数量不成比例地多，您可以快速发现这种模式并调查潜在原因。

### 流量表

使用**流量表**从三个不同的角度监控集群中 Kubernetes 工作负载的主要通信者，每个角度都提供对网络流量模式的独特洞察：

* **AWS 服务视图**显示哪些工作负载向 AWS 服务（如 [Amazon DynamoDB](https://aws.amazon.com/dynamodb/) 和 [Amazon Simple Storage Service (Amazon S3)](https://aws.amazon.com/s3/)）生成最多流量，因此您可以优化数据访问模式并识别潜在的成本优化机会。
* **集群视图**揭示集群内最繁忙的通信者（东西向流量），这意味着您可以发现可能受益于优化或共置策略的频繁通信的微服务。
* **外部视图**识别与 AWS 外部目标（互联网或本地）流量最高的工作负载，这对于安全监控和带宽管理很有用。

流量表提供详细的指标和过滤功能来分析网络流量模式。在此示例中，我们可以看到流量表显示我们电子商务服务之间的集群视图流量。该表显示 `orders` Pod 正在与多个 `products` Pod 通信，传输大量数据。这种模式表明订单服务在订单处理期间频繁进行产品查找。

![流量表示例](https://d2908q01vomqb2.cloudfront.net/da4b9237bacccdf19c0760cab7aec4a8359010b0/2025/11/17/2025-news-eks-container-network-observability-rev-2.png)

过滤功能对于故障排除非常有用，例如，专注于来自特定订单 Pod 的流量。这种细粒度过滤可帮助您在调查性能问题时快速隔离通信模式。例如，如果客户遇到结账速度慢的问题，您可以过滤以查看订单服务是否对产品服务进行了太多调用，或者特定 Pod 实例之间是否存在网络瓶颈。

![流量表过滤](https://d2908q01vomqb2.cloudfront.net/da4b9237bacccdf19c0760cab7aec4a8359010b0/2025/11/17/2025-news-eks-container-network-observability-rev-5.png)

## 其他需要了解的事项

以下是关于 EKS 中容器网络可观测性需要注意的要点：

* **定价** – 对于网络监控，您需要支付标准的 Amazon CloudWatch Network Flow Monitor 定价。
* **可用性** – EKS 中的容器网络可观测性在 Amazon CloudWatch Network Flow Monitor 可用的所有商业 AWS 区域中可用。
* **将指标导出到您首选的监控解决方案** – 指标以 OpenMetrics 格式提供，与 Prometheus 和 Grafana 兼容。有关配置详细信息，请参阅 [Network Flow Monitor 文档](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/CloudWatch-NetworkFlowMonitor.html)。

立即开始使用 [Amazon EKS 中的容器网络可观测性](https://docs.aws.amazon.com/eks/latest/userguide/network-observability.html)，以改善集群中的网络可观测性。

祝您构建愉快！