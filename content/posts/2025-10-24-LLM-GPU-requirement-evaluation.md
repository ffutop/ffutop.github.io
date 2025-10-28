---
title: 大语言模型显卡需求的定量评估
author: ffutop
date: 2025-10-24
tags:
  - LLM
  - GPU
  - Self Attention
categories:
  - Cheet Sheet
---

在部署和运行大语言模型 (LLM) 时，精确评估并配置合适的 GPU 资源是核心挑战。模型的性能与 GPU 的三大关键指标——显存容量、显存带宽、浮点计算能力——紧密相关。本文旨在提供一个从模型基础到硬件评估的完整分析框架，帮助您根据模型规模和精度要求，定量地进行 GPU 选型。

## 大语言模型基础

在评估硬件之前，我们必须先了解评估的对象。现代大语言模型普遍基于 Transformer 架构，其工作方式和规模是影响硬件需求的核心。

### 工作原理

**自注意力机制 (Self-Attention)**

自注意力机制是一种允许模型在处理序列数据时，动态评估序列中各个元素之间重要性关系的运算方式。通过该机制，模型能够为序列中的每个元素生成一个充分融合了全局上下文信息的新表示。这使得模型能有效捕捉长距离依赖关系，例如，准确地将代词关联到其所指代的名词，无论它们在文本中间隔多远。

**自回归模型 (Autoregressive Model)**

自回归模型是一种逐个元素地生成序列的模型。其核心特征是，每一步生成新元素时，都会将之前已生成的所有元素作为输入条件。这个“预测→拼接→循环”的过程确保了生成内容的连贯性。名称“自回归”（Auto-Regressive）的含义即“对自身进行回归”，因为它使用自己的历史输出来预测未来的输出。

### 参数量计算

模型的参数量（Parameters）是指模型中所有可学习权重和偏置的总数。它是衡量模型规模最核心的指标，通常用 `P` 表示。对于 Transformer 架构，绝大部分参数集中在以下几个关键部分：

1. 词嵌入层 (Token Embedding Layer)：负责将输入的词元（Token）映射为向量。其参数量为 $\text{词汇表大小 (Vocab Size)} \times \text{模型维度 (d\_model)}$。
2. Transformer 层 (Transformer Layers)：作为模型的主体，由 N 个相同的 Transformer 层堆叠而成。每个 Transformer 层主要包含以下模块：
    - 自注意力模块 (Self-Attention)：参数量约为 $4 \times \text{d\_model}^2$。
    - 前馈网络 (Feed-Forward Network, FFN)：参数量约为 $8 \times \text{d\_model}^2$。

**近似计算公式**：
基于以上分析，我们可以得出一个估算 LLM 参数量的近似公式：

$$P \approx \text{n\_layers} \times (12 \times \text{d\_model}^2) + (\text{Vocab Size} \times \text{d\_model})$$

通过这个公式，您可以根据模型的公开架构信息（层数、模型维度）快速估算出其参数规模 `P`。这个 `P` 值是后续所有硬件评估计算的基础，为后续的硬件选型提供了量化依据。

### 参数量计算示例

为了将理论付诸实践，我们以 [Llama2:7b](https://ollama.com/library/llama2:7b) 模型为例，验证其 `7B` 的标称参数量。

通过查阅其[模型定义文件](https://ollama.com/library/llama2:7b/blobs/8934d96d3f08)，我们可以获得以下关键架构信息：

![alt text](https://img.ffutop.com/39B26324-37EC-4852-A8DE-CFAE6705CACD1.png)

词嵌入层 (Embedding Layer): 词汇表大小 (Vocab Size) 为 32000，模型维度 (d_model) 为 4096。

![alt text](https://img.ffutop.com/39B26324-37EC-4852-A8DE-CFAE6705CACD3.png)
![alt text](https://img.ffutop.com/39B26324-37EC-4852-A8DE-CFAE6705CACD2.png)

Transformer 层: 共 32 层，每层的参数由自注意力模块和前馈网络 (FFN) 构成。

在分析 FFN 部分时，我们注意到其参数量与前述的 $8 \times \text{d\_model}^2$ 估算公式不完全相符。

$$4096 \times 11008 \times 3 \approx 12 \times 4096^2$$

这是因为 Llama 2 的 FFN 引入了额外的 `Gate` 权重矩阵，使其结构变为 `Up`、`Down`、`Gate` 三个矩阵。尽管矩阵数量增加，但其内部维度被调整为 11008（约为 $2.6875 \times 4096$），而非传统 FFN 的 $4 \times 4096$。这种设计上的权衡使得总参数量与 $12 \times \text{d\_model}^2$ 的估算值仍然非常接近。

综合计算，Llama 2 7B 的总参数量为：

$$32 \times (4096\times 4096 \times 4 + 4096 \times 11008 \times 3) + 32000 \times 4096 \approx 6.6B$$

计算结果约为 6.6B，这与行业内通常将其四舍五入后标称为 7B 的做法相符。

## 显卡需求定量评估框架

在明确了模型参数量 `P` 的计算方法后，我们就可以着手定量评估 GPU 的各项关键指标，从而为大语言模型的部署提供硬件选型依据。

### 显存容量 (VRAM Capacity)

显存容量是决定一个 GPU 能否承载特定模型的首要硬性指标。模型参数、KV 缓存和中间计算结果都需要被加载到 GPU 显存中。

**经验公式<sup>[1]</sup>**

对于初步的资源规划，可以使用以下简化公式快速估算所需显存：

$$M \approx P \times (Q/8) \times 1.2 \text{\ (GB)}$$

| 符号 | 说明 |
| :--- | :--- |
| M | 所需最小 GPU 显存（以 GB 为单位）。 |
| P | 模型参数量（以十亿字节为单位，即常见模型标称 `7B`, `13B` 中的 `B`-billion）。 |
| Q | 量化位数 (Quantization)，常见值为 16 (FP16) 或 8 (INT8)。 |
| 1.2 | 约 20% 的额外显存开销，用于存储激活值、KV 缓存等。 |

[1]: https://web.archive.org/web/20250308052856/https://www.substratus.ai/blog/calculating-gpu-memory-for-llm/ "Calculating GPU memory for serving LLMs"

**精确分析**

对于精细规划，需要额外单独计算 **KV 缓存** 的大小，因为它在长序列或大批量请求时会成为显存的主要消耗者。

$$\text{每个 Token 的 KV 缓存大小 (字节)} ≈ 2 \times \frac{\text{量化位数}}{8} \times \text{n\_layers} \times \text{d\_model}$$

在自注意力机制中，模型需要为序列中的每一个 Token 计算一个“查询 (Query)”向量、一个“键 (Key)”向量和一个“值 (Value)”向量。在生成下一个 Token 时，当前 Token 的 Key 和 Value 需要被缓存起来，以便在后续的生成步骤中作为上下文供模型参考。因此，每个 Token 的缓存都包含 K 和 V 两部分，这里的 2 就是计算这两部分的总大小。

### 显存带宽 (Memory Bandwidth)

Token 的生成速度，即模型的推理性能，主要由两大瓶颈决定：**显存带宽**和**浮点计算能力 (FLOPS)**。前者决定了数据加载的速度，后者决定了数据计算的速度。在不同的应用场景下，主导瓶颈会有所不同。

**访存密集型 (Memory-Bound)**

LLM 的自回归特性决定了它必须逐个生成 Token。在生成每一个 Token 时，GPU 都需要完整地读取一次模型的全部参数（数十亿个）到计算核心中进行运算。在小批量（尤其是批量为 1）的场景下，计算量相对较小，导致 GPU 的计算单元在大部分时间内处于空闲状态，等待数据从显存加载。

此时，显存带宽直接决定了模型参数的加载速度，从而决定了生成 Token 的速度。

**性能评估公式**

在访存密集型场景下，生成单个 Token 的理论时间主要由加载模型参数所需的时间决定。其计算公式如下：

$$T_{\text{token}} \approx \frac{M}{B}$$

| 符号 | 说明 |
| :--- | :--- |
| T<sub>token</sub> | 生成单个 Token 的理论时间（以秒为单位）。 |
| M | 模型参数、KV 缓存、激活值等占用的总显存大小（以 GB 为单位）。 |
| B | GPU 的显存带宽（GB/s）。 |

### 浮点计算能力 (FLOPS)

FLOPS (每秒浮点运算次数) 是衡量 GPU 核心计算能力的直接指标。

为了充分利用 GPU 的并行计算能力，可以让同一个模型同时处理多个请求，来避免每个请求都需要独立加载一次模型参数的问题。

**计算密集型 (Compute-Bound)**

通过增大批次大小，GPU 加载一次模型参数后可以并行处理大量数据。此时，模型加载的等待时间被分摊，计算单元得以持续饱和工作，性能瓶颈从显存带宽转移到纯粹的数学计算能力上。

在这种场景下，生成单个 Token 所需的理论计算时间由 FLOPS 决定。

**性能评估公式**：

$$T_{\text{token}} \approx \frac{2 \times P}{FLOPS}$$

| 符号 | 说明 |
| :--- | :--- |
| T<sub>token</sub> | 生成单个 Token 的理论时间（以秒为单位）。 |
| P | 模型参数量（以十亿字节为单位，即常见模型标称 `7B`, `13B` 中的 `B`-billion）。 |
| 2 | 估算的生成单个 Token 所需的浮点运算次数。系数 `2` 源于经验法则，即每个参数大致对应两次浮点运算（一次乘法和一次加法）。 |
| FLOPS | GPU 的浮点运算能力。 |

### 算力-带宽比 (Compute-to-Bandwidth Ratio)

从“访存密集型”切换到“计算密集型”的临界点，取决于 GPU 的**算力-带宽比 (Compute-to-Bandwidth Ratio)**。这个比率反映了 GPU 的架构特性：需要多大的计算量才能“喂饱”其强大的计算核心，从而不让其等待访存。

我们可以通过一个简化的思想实验来理解这一点：假设系统达到平衡时，计算一个批次所需的时间恰好等于从显存中加载这批次数据所需的时间。

以 [NVIDIA A100 80GB PCIe](https://www.nvidia.com/en-us/data-center/a100/) 为例：
- FP16 理论算力 (F): 312 TFLOPS (即 $312 \times 10^{12}$ FLOPS)
- 显存带宽 (B): 1,935 GB/s (即 $1.935 \times 10^{12}$ B/s)

对于一个 FP16 模型，每个参数占用 2 字节。我们可以估算，为了让计算和访存时间达到平衡，每个字节的内存访问需要对应多少次浮点运算：

$$\text{算力-带宽比} = \frac{F}{B} = \frac{312 \times 10^{12} \text{ FLOPS}}{1.935 \times 10^{12} \text{ B/s}} \approx 161 \text{ FLOPs/Byte}$$

这意味着，对于 A100 而言，平均每从显存中读取 1 字节的数据，GPU 需要执行约 161 次浮点运算，才能使计算单元和显存带宽同时饱和。在 LLM 推理中，这意味着我们需要足够大的批次大小（Batch Size），使得总计算量与总访存量的比值达到这个水平，才能将瓶颈从带宽转移至算力，从而最大化 GPU 的吞吐量。

## 以 [Llama3.3](https://ollama.com/library/llama3.3:70b) 为例定量评估

| 模型 | 显存需求 | GPU | 批次大小 | 每 Token 延迟 | 预估成本 | 
|---|---|---|---|---|---|
| 70b | 168GB | NVIDIA A100 80GB x2 | ~ 160 | 448 ms | ~ 35 万 |
| 70b-q4_K_M | 42GB | NVIDIA RTX 4090 x2 | ~ 1000 | 60 ms | ~ 5 万 |

*上述评估仅作为概念性参考，实际情况下还有更多的影响因素会导致出现偏差*

## 总结与选型策略

| 场景需求 | 核心瓶颈 | 优先考虑的 GPU 指标 | 典型应用 |
| :--- | :--- | :--- | :--- |
| 低延迟 (Low Latency) | 访存密集型 (Memory-Bound) | 1. 显存带宽 <br> 2. 显存容量 | 实时聊天机器人、在线编程助手 |
| 高吞吐 (High Throughput) | 计算密集型 (Compute-Bound) | 1. 浮点计算能力<br> 2. 显存容量 | 离线文档分析、批量翻译任务 |
| 运行超大模型 | 显存容量不足 | 1. 显存容量 <br> 2. 多卡互联带宽 (如 NVLink) | 前沿科研、运行千亿级以上模型 |

最终选型建议：

1. 容量优先：首先计算峰值显存需求，确保所选 GPU 的显存容量充足。
2. 场景匹配：
    - 交互式应用：在满足容量的前提下，优先选择显存带宽最高的 GPU。
    - 批处理应用：在满足容量的前提下，优先选择 FLOPS 最高的 GPU。
3. 成本考量：基于上述公式，估算不同 GPU 配置下的理论性能和成本，寻找最具性价比的方案。

## 参考资料

1. [LLM Parameter Counting](https://kipp.ly/transformer-param-count/)
1. [Transformer Inference Arithmetic](https://kipp.ly/transformer-inference-arithmetic/)
1. [Scaling Laws for Neural Language Models](https://arxiv.org/abs/2001.08361)