---
title: "LLM 术语全景图（三）：模型架构——MoE 与残差连接的演进"
date: 2026-07-27T09:30:00+08:00
slug: "llm-terminology-03-moe-residual"
draft: false
image: "03.png"
tags:
    - AI
    - LLM
    - MoE
    - mHC
    - RoPE
    - DeepSeek
categories:
    - AI
    - LLM术语全景图
---

> 这是「大模型训练与推理术语全景图」系列的第 3 篇，覆盖模型架构的 8 个概念：MoE、MTP、mHC、RoPE、YaRN、RMSNorm、SwiGLU、KV Cache。
> 主要参考模型：DeepSeek V3/V4、GLM-5、Kimi K2、Llama 3、Mixtral

---

## 开篇：模型架构的三条演进线

把 2023 年到 2026 年的主流 LLM 架构按时间铺开，能清晰看到三条并行的演进线，本篇的 8 个概念分别落在其中。

**第一条线是 FFN 的 MoE 化。** 稠密 FFN 被替换为多个并行专家，路由器为每个 token 只激活少量专家。结果是模型总参数量大幅膨胀（DeepSeek V3 达 671B、Kimi K2 达 1T），但单次前向的激活参数仍保持在 30B-40B 量级，激活比压到 5% 上下。MoE 让"大参数量"与"低激活成本"第一次解耦。

**第二条线是残差连接从单流到多流。** 标准 ResNet 残差是 `y = x + F(x)` 的单残差通路；Hyper-Connections 把它扩展为 n 个并行残差流，并用 Pre/Res/Post 三类映射在层间混合；mHC 进一步把 Res Mapping 约束到双随机矩阵流形，避免深层堆叠时信号不可预测地放大或缩小。残差通路变宽，但 FLOPs 几乎不增。

**第三条线是基础组件趋于稳定。** 位置编码收敛到 RoPE（长上下文用 YaRN 扩展），归一化收敛到 RMSNorm 的 Pre-Norm 用法，FFN 激活收敛到 SwiGLU，KV Cache 则成为所有自回归推理绕不开的物理瓶颈。这三类组件已没有太多设计争议，新一代模型主要在 MoE 和残差结构上做差异化。

下面逐一展开。


## 模型架构（Architecture Components）


### Mixture of Experts

- **缩写**：MoE
- **中文名称**：混合专家模型
- **简短介绍**：将 FFN 层替换为多个并行的"专家"网络，通过路由器（router/gate）为每个 token 选择少量专家激活，实现大参数量但低激活成本的模型。
- **详细介绍**：MoE 的核心计算：

  1. Router 计算：g(x) = softmax(W_gate · x)，为每个专家打分
  2. Top-k 选择：选出得分最高的 k 个专家
  3. 加权输出：y = Σ g_i(x) · Expert_i(x)，仅对选中的 k 个专家计算

  关键设计选择：
  - 专家数量：DeepSeek V3 有 256 个路由专家 + 1 个共享专家；Qwen3-235B 有 128 个专家
  - Top-k：通常 k=1~8，DeepSeek V3 激活 8 个路由专家 + 1 个共享专家
  - 共享专家（Shared Expert）：所有 token 都经过的专家，提供通用知识，DeepSeek MoE 首创

  代表模型的 MoE 配置：
  | 模型 | 总参数 | 激活参数 | 激活比 | 专家数 |
  |------|--------|----------|--------|--------|
  | DeepSeek V3 | 671B | 37B | 5.5% | 256+1 |
  | DeepSeek V4-Pro | ~700B+ | ~30B | ~4% | 256+1 |
  | Kimi K2 | 1T | 32B | 3.2% | 384+1 |
  | Qwen3-235B | 235B | 22B | 9.4% | 128 |
  | MiniMax-01 | 456B | 46B | 10% | 32 |
  | GLM-5 | 744B | 40B | 5.4% | 256+1 |

  注：GLM-5 的 744B 总参/40B 激活来自 GLM-5 技术报告（arXiv:2602.15763），激活比 40/744 ≈ 5.4%。DeepSeek V3 的 671B/37B/5.5%/256+1 来自 DeepSeek V3 技术报告（arXiv:2412.19437）。

  负载均衡：DeepSeek V3 首创无辅助损失（auxiliary-loss-free）策略，避免传统负载均衡损失损害模型质量。

- **关联论文/模型**：DeepSeekMoE (arXiv:2401.06066)；DeepSeek V3 (技术报告 arXiv:2412.19437)；Qwen3 (技术报告)；Kimi K2 技术报告；GLM-5 (技术报告 arXiv:2602.15763)；Mixtral
- **参考分析链接**：
  - https://sebastianraschka.com/llm-architecture-gallery/moe/
  - https://magazine.sebastianraschka.com/p/the-big-llm-architecture-comparison


### Multi-Token Prediction

- **缩写**：MTP
- **中文名称**：多 Token 预测
- **简短介绍**：训练时不仅预测下一个 token，还同时预测未来第 2、3...个 token，通过额外的预测头增加训练信号密度。推理时可复用 MTP 模块做推测解码。
- **详细介绍**：MTP 的训练目标：

  标准 next-token：L = -log P(x_{t+1} | x_{<=t})
  MTP-k：L = -Σ_{i=1}^{k+1} log P(x_{t+i} | x_{<=t}, x_{t+1}, ..., x_{t+i-1})

  DeepSeek V3 使用 MTP-1（预测 1 个额外 token）：
  - 主解码器预测 x_{t+1}
  - MTP 模块以主解码器的隐状态 + x_{t+1} 为输入，预测 x_{t+2}
  - MTP 损失权重通常为 0.1（主损失 + 0.1 × MTP 损失）

  推理时的两种用法：
  1. 标准：忽略 MTP 头，逐 token 生成（不影响推理）
  2. 推测解码：MTP 头作为内置的 draft model，一次预测多个 token，经验证后批量接受

  在 vLLM 中启用：`--speculative-config='{"method": "deepseek_mtp", "num_speculative_tokens": 1}'`

  采用 MTP 的模型：DeepSeek V3/R1、DeepSeek V4、GLM-5 系列、Inkling

- **关联论文/模型**：DeepSeek V3 (arXiv:2412.19437, Section 4.4)；DeepSeek R1 (技术报告)
- **参考分析链接**：
  - https://sebastianraschka.com/llm-architecture-gallery/mtp/
  - https://docs.nvidia.com/nemo/megatron-bridge/nightly/training/multi-token-prediction.html
  - https://medium.com/data-science-collective/deepseek-explained-4-multi-token-prediction-33f11fe2b868


### Manifold-Constrained Hyper-Connections

- **缩写**：mHC
- **中文名称**：流形约束超连接
- **简短介绍**：DeepSeek V4 引入的残差连接改进，将 Transformer 块中的单残差流扩展为多个并行残差流，并通过双随机矩阵约束确保混合稳定。
- **详细介绍**：mHC 的演进路径：

  标准残差连接 → Hyper-Connections (HC) → mHC

  1. 标准 ResNet 残差：y = x + F(x)，单残差流
  2. Hyper-Connections：将单残差流扩展为 n 个并行残差流
     - Pre Mapping：将 n 个流合并为 1 个隐向量供 Attention/MoE 处理
     - Res Mapping：层间残差流混合（学习矩阵）
     - Post Mapping：将层输出分发回 n 个流
  3. mHC 关键约束：
     - Res Mapping 投影到双随机矩阵流形（行和=1、列和=1、非负）
     - Pre/Post Mapping 也约束为非负有界
     - 避免深层堆叠时信号放大/缩小的不可预测行为

  DeepSeek V4 使用 n=4 个并行残差流。实测仅增加 6.7% 训练时间开销，但指标可比基线少用一半训练 token 达到同等性能。

  核心价值：在几乎不增加 FLOPs 的情况下，使残差通路更宽更表达，且约束保证深层模型稳定性。

- **关联论文/模型**："mHC: Manifold-Constrained Hyper-Connections" (arXiv:2512.24880, DeepSeek, 2025-12-31)；Hyper-Connections 原始论文 (arXiv:2409.19606, Zhu et al., 2024)；DeepSeek V4 (技术报告)
- **参考分析链接**：
  - https://magazine.sebastianraschka.com/p/recent-developments-in-llm-architectures (Section 5.1)


### Rotary Position Embedding

- **缩写**：RoPE
- **中文名称**：旋转位置编码
- **简短介绍**：通过旋转矩阵编码 token 间的相对位置关系，将位置信息融入 Q 和 K 的旋转中，使注意力分数自然反映相对距离。当前绝大多数 LLM 的默认位置编码方案。
- **详细介绍**：RoPE 的数学形式：

  对位置 m 的向量 x，将其每两个维度分为一组，旋转角度 θ_i = m · base^(-2i/d)：

  R(x, m) = x · R_m，其中 R_m 是分块旋转矩阵

  关键性质：
  - <R(Q, m), R(K, n)> = <Q, K> 旋转后内积仅依赖相对位置 (m-n)
  - base 通常为 10000（LLaMA、DeepSeek V3 等多数模型）；部分长上下文模型（如 Qwen2、GLM-4-1M）调高到 500000~1000000 以利于长上下文外推

  外推问题：训练时最大位置 m_max 有限（如 4K），推理时超出范围的角度超出训练分布，质量退化。

  采用 RoPE 的模型：LLaMA 全系列、Qwen 全系列、DeepSeek V2/V3、GLM-4、Mistral、Kimi K2 等。

  注意：MLA 需要将 RoPE 部分独立处理（RoPE 不兼容低秩压缩），这是 MLA 实现复杂度的来源之一。

- **关联论文/模型**："RoFormer: Enhanced Transformer with Rotary Position Embedding" (Su et al., 2021, arXiv:2104.09864)；LLaMA、Qwen、DeepSeek、Mistral 等
- **参考分析链接**：
  - https://sebastianraschka.com/llms-from-scratch/ch04/10_rope/
  - https://mbrenndoerfer.com/writing/llama-components-rmsnorm-swiglu-rope


### Yet Another RoPE Extension

- **缩写**：YaRN
- **中文名称**：YaRN 上下文扩展
- **简短介绍**：目前最主流的 RoPE 上下文扩展方法，通过分段插值 + 温度缩放，在少量微调（约 400 步）下将上下文从 4K 扩展到 128K+。
- **详细介绍**：YaRN 的核心方法：

  1. 分段策略：将 RoPE 的频率维度分为三段
     - 高频段：保持原样（局部信息不需要外推）
     - 中频段：插值 + 温度缩放（平滑过渡）
     - 低频段：纯插值（长距离信息需要外推）
  2. 温度缩放：调整 softmax 温度补偿插值带来的注意力分布变化
  3. 微调：仅需约 400 步微调即可适应新长度

  采用 YaRN 的模型：Qwen 系列（4K→128K）、DeepSeek（4K→128K）、LLaMA 系列、GLM 等。

  其他上下文扩展方法对比：
  - PI (Position Interpolation)：线性插值所有频率，简单但效果一般
  - NTK-aware：调整 base，不微调直接外推
  - YaRN：分段 + 温度，当前最优

- **关联论文/模型**："YaRN: Efficient Context Window Extension" (Peng et al., 2023, arXiv:2309.00071)；Qwen、DeepSeek、LLaMA 系列
- **参考分析链接**：
  - https://amaarora.github.io/posts/2025-09-21-rope-context-extension.html


### RMSNorm

- **缩写**：RMSNorm
- **中文名称**：均方根归一化
- **简短介绍**：LayerNorm 的简化版本，去除均值计算仅用均方根归一化，计算更快且在大规模训练中表现稳定。几乎所有现代 LLM 的默认归一化方案。
- **详细介绍**：RMSNorm vs LayerNorm：

  LayerNorm: y = (x - μ) / sqrt(σ^2 + ε) · γ + β
  RMSNorm: y = x / RMS(x) · γ,  其中 RMS(x) = sqrt(mean(x^2) + ε)

  RMSNorm 去掉了均值减法和偏置 β，计算量更小，且在大规模训练中更稳定。

  使用方式：
  - Pre-Norm（主流）：在 attention/FFN 之前做归一化，梯度更稳定
  - Post-Norm（GPT-2 风格）：在之后做，训练不稳定，已少用
  - Inside-Residual Post-Norm（OLMo 2）：在残差路径内部做 post-norm

  采用 RMSNorm 的模型：LLaMA 全系列、Qwen3、DeepSeek V3/V4、GLM-5、Mistral、Gemma 3、Kimi K2 等几乎所有现代模型。

- **关联论文/模型**："Root Mean Square Layer Normalization" (Zhang & Sennrich, 2019, arXiv:1910.07467)；几乎所有现代 LLM
- **参考分析链接**：
  - https://machinelearningmastery.com/layernorm-and-rms-norm-in-transformer-models/
  - https://mbrenndoerfer.com/writing/llama-components-rmsnorm-swiglu-rope


### SwiGLU

- **缩写**：SwiGLU
- **中文名称**：SwiGLU 激活函数
- **简短介绍**：GLU（Gated Linear Unit）的变体，使用 SiLU（Swish）作为门控激活函数，替代传统 ReLU/GELU。几乎所有现代 LLM FFN 层的标准配置。
- **详细介绍**：SwiGLU 的公式：

  SwiGLU(x) = SiLU(W·x) ⊙ (V·x)
  其中 SiLU(x) = x · sigmoid(x)（也称 Swish）

  对比传统 FFN：
  - 传统：FFN(x) = W2 · activation(W1 · x)，单一激活
  - SwiGLU：FFN(x) = W2 · [SiLU(W_gate · x) ⊙ (W_up · x)]，门控机制

  门控机制使网络能动态选择信息通过与否，比固定激活函数更有表达力。

  实现细节：SwiGLU FFN 有 3 个权重矩阵（W_gate, W_up, W_down），比传统 2 个多一个，因此隐藏维度通常调整为 2/3 × 4d 以保持参数量。

  采用 SwiGLU 的模型：LLaMA 全系列、Qwen3、DeepSeek V3/V4、GLM-5、Mistral、Kimi K2、PaLM 等。

- **关联论文/模型**："GLU Variants Improve Transformer" (Shazeer, 2020, arXiv:2002.05202)；LLaMA、Qwen、DeepSeek 等
- **参考分析链接**：
  - https://www.emergentmind.com/topics/swiglu-activation-function
  - https://mbrenndoerfer.com/writing/llama-components-rmsnorm-swiglu-rope


### KV Cache

- **缩写**：KV Cache
- **中文名称**：键值缓存
- **简短介绍**：自回归生成时，缓存之前 token 在每层注意力的 K/V 矩阵，避免重复计算。KV Cache 大小是长上下文推理最硬的物理瓶颈。
- **详细介绍**：KV Cache 的大小计算：

  KV Cache = 2(K+V) × n_layers × n_kv_heads × d_head × seq_len × batch × bytes_per_param

  示例（DeepSeek V3, 128K 上下文, bf16, batch=1）：
  - MLA 压缩后：约 68.6 KiB/token × 128K = ~8.6 GB
  - 对比 Llama 3 8B（GQA）：128 KiB/token × 128K = ~16 GB
  - 对比 OLMo 2 7B（MHA）：512 KiB/token × 128K = ~64 GB

  注：以上 MHA/GQA 数字为示意量级，便于横向对比 MLA 的压缩收益。

  KV Cache 是长上下文推理的核心瓶颈，催生了大量优化：
  - 压缩 KV 表示：MLA（低维压缩）、CCA（压缩空间计算）
  - 减少序列条目：CSA/HCA（序列压缩）、SWA（局部窗口）
  - 跨层共享：Gemma 4 的 Cross-Layer Attention
  - 线性注意力：Lightning Attention、KDA（O(1) 状态替代 O(n) 缓存）
  - 分页管理：vLLM PagedAttention

  趋势：随着推理模型（o1/R1）和 Agent 工作流需要更长的上下文，KV Cache 优化成为 2025-2026 年架构设计的最大驱动力。

- **关联论文/模型**：所有 Transformer 模型；MLA (DeepSeek V2)、PagedAttention (vLLM)
- **参考分析链接**：
  - https://magazine.sebastianraschka.com/p/coding-the-kv-cache-in-llms
  - https://sebastianraschka.com/llm-architecture-gallery/kv-cache-calculations/

## 本篇小结

本篇 8 个模型架构术语沿"稀疏激活 + 残差连接进化 + 位置编码/归一化/FFN 标准件 + KV Cache 瓶颈"四条线归位：

- **MoE** 把 FFN 替换为多个并行专家网络，路由器为每个 token 选少量专家，实现大参数量但低激活成本；2025-2026 年大模型几乎全部 MoE 化，激活比持续下降（DeepSeek V4 ~4%）。
- **MTP** 训练时同时预测未来多个 token 增加训练信号密度，推理时可复用 MTP 模块做推测解码；DeepSeek V3 用 MTP，是训练与推理的连接点。
- **mHC** 把 Transformer 块的单残差流扩展为多个并行残差流，用双随机矩阵约束保证混合稳定；DeepSeek V4 引入，信息通路更宽。
- **RoPE** 通过旋转矩阵把相对位置编入 Q/K 的旋转中，使注意力分数自然反映相对距离；当前绝大多数 LLM 的默认位置编码。
- **YaRN** 通过分段插值 + 温度缩放做 RoPE 上下文扩展，约 400 步微调即可从 4K 扩到 128K+；目前最主流的上下文扩展方法。
- **RMSNorm** 去掉 LayerNorm 的均值计算仅用均方根归一化，计算更快且大规模训练稳定；几乎所有现代 LLM 的默认归一化。
- **SwiGLU** 用 SiLU 作为 GLU 门控激活替代 ReLU/GELU；几乎所有现代 LLM FFN 层的标准配置。
- **KV Cache** 缓存历史 token 的 K/V 避免重复计算，但其体积随上下文线性增长，是长上下文推理最硬的物理瓶颈——催生篇 1 的 MLA/CSA/HCA 压缩、篇 5 的 PagedAttention 分页管理、篇 10 的 Prefix Caching 复用。

KV Cache 是本篇与篇 1（注意力压缩）、篇 5（推理优化）、篇 10（推理服务）的交叉点：架构设计层面的每一次创新，最终都要回到这个物理瓶颈上检验。


本系列共 11 篇，本文是第 3 篇。系列导航见：LLM 术语全景图系列导读。
