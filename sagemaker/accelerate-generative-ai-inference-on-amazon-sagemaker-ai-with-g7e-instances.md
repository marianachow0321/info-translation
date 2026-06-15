# 使用 G7e 实例在 Amazon SageMaker AI 上加速生成式 AI 推理

作者：Hazim Qudah、Dmitry Soldatkin、Venu Kanamatareddy、Sanghwa Na | 2026年4月20日

原文链接：https://aws.amazon.com/blogs/machine-learning/accelerate-generative-ai-inference-on-amazon-sagemaker-ai-with-g7e-instances/

---

随着生成式 AI 需求持续增长，开发者和企业正在寻找更灵活、更具成本效益且更强大的加速器来满足其需求。今天，我们非常高兴地宣布，搭载 [NVIDIA RTX PRO 6000](https://www.nvidia.com/en-us/products/workstations/professional-desktop-gpus/rtx-pro-6000-family/) Blackwell Server Edition GPU 的 G7e 实例现已在 [Amazon SageMaker AI](https://aws.amazon.com/sagemaker/ai/) 上可用。

您可以配置具有 1、2、4 和 8 个 RTX PRO 6000 GPU 的实例节点，每个 GPU 提供 96 GB GDDR7 显存。此次发布提供了使用单节点 GPU（G7e.2xlarge 实例）来托管强大的开源基础模型（FM）的能力，例如 GPT-OSS-120B、Nemotron-3-Super-120B-A12B（NVFP4 变体）和 Qwen3.5-35B-A3B，为组织提供了经济高效且高性能的选择。这使其非常适合那些希望在保持推理工作负载高性能的同时降低成本的用户。G7e 实例的主要亮点包括：

- 与 G6e 实例相比，GPU 显存翻倍，支持以 FP16 精度部署大语言模型（LLM）：
  - 单 GPU 节点（G7e.2xlarge）可部署 350 亿参数模型
  - 4 GPU 节点（G7e.24xlarge）可部署 1500 亿参数模型
  - 8 GPU 节点（G7e.48xlarge）可部署 3000 亿参数模型
- 高达 1600 Gbps 的网络吞吐量
- G7e.48xlarge 上高达 768 GB GPU 显存

Amazon EC2 G7e 实例代表了云端 GPU 加速推理的重大飞跃，与上一代 G6e 实例相比，推理性能提升高达 2.3 倍。每个 G7e GPU 提供 1,597 GB/s 带宽，单 GPU 显存是 G6e 的 2 倍、G5 的 4 倍。网络带宽在最大规格的 G7e 上通过 EFA 可扩展至 1,600 Gbps——是 G6e 的 4 倍、G5 的 16 倍——解锁了之前在 G 系列实例上不切实际的低延迟多节点推理和微调场景。下表总结了 8-GPU 层级的代际演进：

| 规格 | G5 (g5.48xlarge) | G6e (g6e.48xlarge) | G7e (g7e.48xlarge) |
|------|-----------------|-------------------|-------------------|
| GPU | 8x NVIDIA A10G | 8x NVIDIA L40S | 8x NVIDIA RTX PRO 6000 Blackwell |
| 单 GPU 显存 | 24 GB GDDR6 | 48 GB GDDR6 | 96 GB GDDR7 |
| 总 GPU 显存 | 192 GB | 384 GB | 768 GB |
| GPU 显存带宽 | 600 GB/s | 864 GB/s | 1,597 GB/s |
| vCPU | 192 | 192 | 192 |
| 系统内存 | 768 GiB | 1,536 GiB | 2,048 GiB |
| 网络带宽 | 100 Gbps | 400 Gbps | 1,600 Gbps (EFA) |
| 本地 NVMe 存储 | 7.6 TB | 7.6 TB | 15.2 TB |
| 相对 G6e 推理性能 | 基准 | ~1x | 高达 2.3x |

凭借单实例 768 GB 的聚合 GPU 显存，G7e 可以托管之前在 G5 或 G6e 上需要多节点配置的模型，降低了运维复杂性和节点间延迟。结合使用第五代 Tensor Core 的 FP4 精度支持以及通过 EFAv4 的 NVIDIA GPUDirect RDMA，G7e 实例定位为在 AWS 上部署 LLM、多模态 AI 和智能体推理工作负载的首选。

## 适合 G7e 的使用场景

G7e 的显存密度、带宽和网络能力组合使其非常适合广泛的现代生成式 AI 工作负载：

- **聊天机器人和对话式 AI** — G7e 的低首字延迟（TTFT）和高吞吐量即使在高并发负载下也能保持交互体验的响应性。
- **智能体和工具调用工作流** — CPU 到 GPU 带宽提升 4 倍，使 G7e 在检索增强生成（RAG）流水线和智能体工作流中特别有效，这些场景中从检索存储快速注入上下文至关重要。
- **文本生成、摘要和长上下文推理** — G7e 的 96 GB 单 GPU 显存可容纳大型 KV 缓存用于扩展文档上下文——减少截断并支持对长文本输入进行更丰富的推理。
- **图像生成和视觉模型** — 之前的实例在较大的多模态模型上会遇到显存不足错误，G7e 翻倍的显存干净利落地解决了这些限制。
- **物理 AI 和科学计算** — G7e 的 Blackwell 架构计算能力、FP4 支持和空间计算功能（DLSS 4.0、第四代 RT 核心）将其适用范围扩展到数字孪生、3D 模拟和物理 AI 模型推理。

## 部署演练

### 前提条件

要使用 SageMaker AI 尝试此解决方案，您需要以下前提条件：

- 一个包含所有 AWS 资源的 [AWS 账户](https://console.aws.amazon.com/console/home)。
- 一个用于访问 Amazon SageMaker AI 的 [AWS IAM](https://aws.amazon.com/iam/) 角色。要了解更多关于 IAM 如何与 SageMaker AI 配合使用的信息，请参阅 [Amazon SageMaker AI 的身份和访问管理](https://docs.aws.amazon.com/sagemaker/latest/dg/security-iam.html)。
- 访问 [Amazon SageMaker Studio](https://aws.amazon.com/sagemaker/studio/) 或 SageMaker 笔记本实例，或集成开发环境（IDE）如 PyCharm 或 Visual Studio Code。我们建议使用 Amazon SageMaker Studio 进行简便的部署和推理。
- **ml.g7e.2xlarge**（或更大规格）用于 Amazon SageMaker AI **端点使用**的配额。您可以通过 [Service Quotas 控制台](https://us-east-1.console.aws.amazon.com/servicequotas/home/services?region=us-east-1) 请求配额提升。

### 部署

您可以克隆代码库并使用[此处](https://github.com/aws-samples/sagemaker-genai-hosting-examples/tree/main/03-features/instances/g7e)提供的示例笔记本。

## 性能基准测试

为了量化代际提升，我们在 G6e 和 G7e 实例上使用相同的工作负载对 Qwen3-32B（BF16）进行了基准测试：每个请求约 1,000 个输入 token 和约 560 个输出 token。这代表了文档摘要或修正任务的典型场景。两种配置均使用原生 [vLLM](https://github.com/vllm-project/vllm) 容器并启用前缀缓存。

基准测试套件可在示例 Jupyter 笔记本中获取。它遵循三步流程：（1）使用原生 vLLM 容器在 SageMaker AI 端点上部署模型，（2）在 1-32 个并发请求的并发级别进行负载测试，（3）分析结果生成以下性能表。

**G6e 基准：ml.g6e.12xlarge [4x L40S，$13.12/小时]**

使用 4x L40S GPU 和张量并行度 4，G6e 提供强劲的单请求吞吐量：单并发 37.1 tok/s，C=32 时 21.5 tok/s。

| 并发 | 成功率 | p50 (s) | p99 (s) | tok/s | RPS | 聚合 tok/s | $/百万 token |
|------|--------|---------|---------|-------|-----|------------|-------------|
| 1 | 100% | 16.1 | 16.3 | 37.1 | 0.07 | 37 | $38.09 |
| 8 | 100% | 19.8 | 20.2 | 30.3 | 0.42 | 242 | $5.85 |
| 16 | 100% | 23.1 | 23.5 | 26.0 | 0.73 | 416 | $3.41 |
| 32 | 100% | 26.0 | 29.2 | 21.5 | 1.21 | 686 | $2.06 |

**G7e：ml.g7e.2xlarge [1x RTX PRO 6000 Blackwell，$4.20/小时]**

G7e 在单个 GPU 上以张量并行度 1 运行相同的 320 亿参数模型。虽然单请求 tok/s 低于 G6e 的 4-GPU 配置，但成本表现截然不同。

| 并发 | 成功率 | p50 (s) | p99 (s) | tok/s | RPS | 聚合 tok/s | $/百万 token |
|------|--------|---------|---------|-------|-----|------------|-------------|
| 1 | 100% | 27.2 | 27.5 | 22.0 | 0.04 | 22 | $21.32 |
| 8 | 100% | 28.7 | 28.9 | 20.9 | 0.28 | 167 | $2.81 |
| 16 | 100% | 30.3 | 30.6 | 19.9 | 0.53 | 318 | $1.48 |
| 32 | 100% | 33.2 | 33.3 | 18.5 | 0.99 | 592 | $0.79 |

**数据解读**

在生产并发级别（C=32）下，G7e 实现了每百万输出 token $0.79 的成本，相比 G6e 的 $2.06 降低了 2.6 倍。这是由两个因素驱动的：G7e 显著更低的每小时费率（$4.20 vs $13.12）及其在负载下保持一致吞吐量的能力。

G7e 的单 GPU 架构也能更优雅地扩展。延迟从 C=1 到 C=32 增加 22%（27.2s 到 33.2s），而 G6e 增加 62%（16.1s 到 26.0s）。使用张量并行度 1 意味着：

- 无 GPU 间同步开销
- 无每个 Transformer 层的 all-reduce 操作
- 无跨 GPU KV 缓存碎片化
- 无 NVLink 通信瓶颈

随着并发增加和 GPU 饱和度提高，这种协调开销的缺失使延迟保持可预测。对于低并发下延迟敏感的工作负载，G6e 的 4-GPU 并行性仍能提供更快的单请求响应。对于优化大规模每 token 成本的生产部署，G7e 是明确的选择，而且正如下一节所示，将 G7e 与 EAGLE 推测解码结合使用会进一步扩大优势。

## 联合基准测试：G7e + EAGLE 推测解码

G7e 的硬件改进本身已经很显著，但将其与 EAGLE（Extrapolation Algorithm for Greater Language-model Efficiency）推测解码结合使用会产生复合增益。EAGLE 通过从模型自身的隐藏表示预测多个未来 token，然后在单次前向传递中验证它们来加速 LLM 解码。这在每步生成多个 token 的同时产生完全相同的输出质量。有关 EAGLE 在 SageMaker AI 上的详细演练，包括优化作业设置和基础 vs 训练 EAGLE 工作流，请参阅 [Amazon SageMaker AI 引入基于 EAGLE 的自适应推测解码以加速生成式 AI 推理](https://aws.amazon.com/blogs/machine-learning/amazon-sagemaker-ai-introduces-eagle-based-adaptive-speculative-decoding-to-accelerate-generative-ai-inference/)。

在本节中，我们使用 Qwen3-32B（BF16）测量从基准到 G7e + EAGLE3 的叠加改进。基准工作负载使用每请求约 1,000 个输入 token 和约 560 个输出 token，代表文档摘要或修正任务。EAGLE3 使用[社区训练的推测器](https://huggingface.co/RedHatAI/Qwen3-32B-speculator.eagle3)（约 1.56 GB）启用，设置 `num_speculative_tokens=4`。

G7e + EAGLE3 相比上一代基准实现了 2.4 倍的吞吐量提升和 75% 的成本降低。每百万输出 token $0.41 的价格也比 G6e + EAGLE3（$1.72）便宜 4 倍，同时提供更高的吞吐量。

**启用 EAGLE3**

对于使用微调模型的生产部署，SageMaker AI 的 [EAGLE 优化工具包](https://aws.amazon.com/blogs/machine-learning/amazon-sagemaker-ai-introduces-eagle-based-adaptive-speculative-decoding-to-accelerate-generative-ai-inference/) 可以在您自己的数据上训练自定义 EAGLE 头，进一步提高推测接受率和吞吐量，超越社区推测器所能提供的水平。

## 定价

Amazon SageMaker AI 上的 G7e 实例按所选实例类型和使用时长的标准 SageMaker AI 推理定价计费。在 G7e 上提供服务没有额外的按 token 或按请求收费。

EAGLE 优化作业在 SageMaker AI 训练实例上运行，按作业持续时间的标准 SageMaker 训练实例费率计费。生成的改进模型工件以标准存储费率存储在 [Amazon S3](https://aws.amazon.com/s3/) 中。部署改进模型后，EAGLE 加速推理没有额外收费——您只需支付标准端点实例费用。

下表显示了美国东部（弗吉尼亚北部）关键 G7e、G6e 和 G5 实例规格的按需定价供参考。G7e 行已突出显示。

| 实例 | GPU 数量 | GPU 显存 | 典型使用场景 |
|------|---------|---------|------------|
| ml.g5.2xlarge | 1 | 24 GB | 小型 LLM（≤7B FP16）；开发和测试 |
| ml.g5.48xlarge | 8 | 192 GB | G5 上的大型多 GPU LLM 服务 |
| ml.g6e.2xlarge | 1 | 48 GB | 中型 LLM（≤14B FP16） |
| ml.g6e.12xlarge | 2 | 96 GB | 大型 LLM（≤36B FP16）；上代基准 |
| ml.g6e.48xlarge | 8 | 384 GB | 超大型 LLM（≤90B FP16） |
| **ml.g7e.2xlarge** | **1** | **96 GB** | **大型 LLM（≤70B FP8）单 GPU** |
| **ml.g7e.24xlarge** | **4** | **384 GB** | **超大型 LLM；高吞吐量服务** |
| **ml.g7e.48xlarge** | **8** | **768 GB** | **最大吞吐量；最大模型** |

您还可以通过 [Amazon SageMaker Savings Plans](https://aws.amazon.com/savingsplans/ml-pricing/) 降低推理成本，该计划以承诺一致的使用量为交换条件提供高达 64% 的折扣。这非常适合具有可预测流量的生产推理端点。

## 清理资源

为避免完成测试后产生不必要的费用，请[删除演练过程中创建的 SageMaker 端点](https://docs.aws.amazon.com/sagemaker/latest/dg/realtime-endpoints-delete-resources.html)。您可以通过 SageMaker AI 控制台或使用 Python SDK 完成此操作，如 [Amazon SageMaker AI 开发者指南](https://docs.aws.amazon.com/sagemaker/latest/dg/realtime-endpoints-delete-resources.html) 所示。

如果您运行了 EAGLE 优化作业，还请删除 Amazon S3 中的输出工件以避免持续的存储费用。

## 结论

Amazon SageMaker AI 上的 G7e 实例代表了具成本效益的生成式 AI 推理的下一个重大飞跃。Blackwell GPU 架构提供了 2 倍的单 GPU 显存、1.85 倍的显存带宽以及高达 2.3 倍的推理性能提升（相比 G6e）。这使得之前需要多 GPU 的工作负载能够在单个 GPU 上高效运行，并提高了每种 GPU 配置的吞吐量上限。结合 SageMaker AI 的 EAGLE 推测解码，这些改进进一步复合。EAGLE 的显存带宽受限加速直接受益于 G7e 增加的带宽，而 G7e 更大的显存容量允许 EAGLE 草稿头与更大的模型共存而无显存压力。硬件和软件的改进共同带来了吞吐量增益，直接转化为大规模每输出 token 更低的成本。

从 G5 到 G6e 再到 G7e 的演进，叠加 EAGLE 优化，代表了一条近乎连续的硬件-软件协同优化路径——随着模型演进以及生产流量数据被捕获并回馈到 EAGLE 重训练中，这条路径将持续改进。
