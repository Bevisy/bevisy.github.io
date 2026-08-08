---
title: "LLM 术语全景图（二）：注意力机制下——线性注意力与混合架构"
date: 2026-07-27T09:15:00+08:00
slug: "llm-terminology-02-linear-hybrid-attention"
draft: false
image: "02.png"
tags:
    - AI
    - LLM
    - Linear Attention
    - KDA
    - QK-Norm
    - MiniMax
categories:
    - AI
    - LLM术语全景图
---

> 这是「大模型训练与推理术语全景图」系列的第 2 篇，覆盖注意力机制后半部分的 6 个概念：Lightning Attention、KDA、NoPE、QK-Norm、CCA、Hybrid Attention。
> 主要参考模型：MiniMax、Kimi (Moonshot)、ZAYA、OLMo、Qwen

---

## 开篇：线性注意力从理论到生产的曲折路径

线性注意力从理论到生产的路径比预想中曲折。Lightning Attention 在 MiniMax-01 上把注意力复杂度从 O(n^2) 降到 O(n)，纸面效率可观；但 MiniMax 在 M2 中放弃了 Lightning Attention，回归完整 softmax attention。这不是个别工程取舍，而是一个清晰的技术信号：在推理和多轮任务中，线性注意力的精度无法满足生产要求，且其低精度 KV 状态对 prefix caching 支持偏弱，使得长上下文复用这一关键推理优化失去杠杆。生产环境的精度门槛比纸面效率更重要，这是 2025-2026 年线性注意力落地必须正视的现实。本篇沿着这条主线，展开 Lightning Attention、KDA、NoPE、QK-Norm、CCA 与 Hybrid Attention 六个概念。


### Lightning Attention

- **缩写**：—
- **中文名称**：闪电注意力
- **简短介绍**：MiniMax 提出的线性注意力实现，将注意力计算从 O(n^2) 降至 O(n)，通过将 softmax 近似为线性形式并利用矩阵乘法结合律重排计算顺序。
- **详细介绍**：Lightning Attention 基于线性注意力理论：

  标准 softmax attention: O(Q·K^T)·V — 复杂度 O(n^2 × d)
  线性注意力: φ(Q)·(φ(K)^T·V) — 复杂度 O(n × d^2)

  关键技巧：利用 (Q·K^T)·V = Q·(K^T·V) 的结合律，先算 K^T·V（大小 d×d，与序列长度无关），再乘 Q，将序列维度从二次降为线性。

  MiniMax-01 采用混合架构：大部分层使用 Lightning Attention（线性），少量层使用完整 softmax attention（保证精度）。

  MiniMax M2 的转折：MiniMax 在 M2 中放弃了 Lightning Attention，回归完整注意力，原因是线性注意力在推理和多轮对话场景中精度不足。这表明线性注意力在生产环境中的成熟度仍有争议。

  对比：Kimi Linear 的 KDA 是另一种线性注意力变体，通过 channel-wise gating 改善精度。

- **关联论文/模型**：MiniMax-01 (arXiv:2501.08313)；MiniMax-M1 (arXiv:2506.13585)；MiniMax M2 (回归完整注意力)
- **参考分析链接**：
  - https://magazine.sebastianraschka.com/p/a-dream-of-spring-for-open-weight (MiniMax M2 分析)
  - https://kaitchup.substack.com/p/minimax-m2-and-kimi-linear-why-full


### Kimi Delta Attention

- **缩写**：KDA
- **中文名称**：Kimi 三角注意力 / Kimi Delta 注意力
- **简短介绍**：Moonshot AI 在 Kimi Linear 中提出的线性注意力变体，是对 Gated DeltaNet 的改进，引入 channel-wise（逐通道）衰减门控，实现更精细的内存遗忘控制。
- **详细介绍**：KDA 的技术演进路径：

  DeltaNet → Gated DeltaNet → KDA

  核心机制：
  1. Delta 规则：将 attention state 的更新视为在线梯度下降，新 key-value 对通过 delta 规则更新状态矩阵 S
  2. Gated DeltaNet 在此基础上添加标量衰减门控（scalar decay gate）
  3. KDA 将标量衰减升级为对角化逐通道门控 Diag(α_t)，每个通道独立控制遗忘率

  优势：
  - 逐通道门控使模型能对不同信息维度采用不同的记忆/遗忘策略
  - 增强位置感知能力
  - 最高 75% KV Cache 压缩，1M token 上下文下解码吞吐最高 6× 提升（据 Moonshot 官方 GitHub，指标限定 1M token 上下文）

  Kimi Linear 架构：3:1 混合（3 层 KDA + 1 层全局 MLA），48B 总参数 / 3B 激活，1M 上下文。

- **关联论文/模型**：Kimi Linear (arXiv:2510.26692)；Gated DeltaNet (arXiv:2504.12236)；DeltaNet (2024)
- **参考分析链接**：
  - https://github.com/MoonshotAI/Kimi-Linear
  - https://magazine.sebastianraschka.com/p/a-dream-of-spring-for-open-weight
  - https://www.youtube.com/watch?v=SH6Cyhunl8s (Hybrid Attention and Gated DeltaNet 视频)


### No Positional Encoding

- **缩写**：NoPE
- **中文名称**：无位置编码
- **简短介绍**：不在注意力计算中显式注入位置信息，依赖因果掩码（causal mask）隐式编码位置。部分模型在选定的层中省略 RoPE 以降低长上下文计算开销。
- **详细介绍**：NoPE 的原理：

  标准 Transformer 需要位置编码（绝对位置 APE / 旋转位置 RoPE）来感知 token 顺序。但研究发现，在 decoder-only 模型中，因果掩码（下三角掩码）本身已隐含了顺序信息。

  NoPE 的使用方式：
  - 完全 NoPE：所有层均不使用位置编码（少数实验性模型）
  - 交替 NoPE：大部分层用 RoPE，每隔几层用 NoPE（主流方式）

  采用 NoPE 的模型：
  - SmolLM3 (3B)：每 4 层有 1 层用 NoPE
  - Nemotron 3 / Soofi S：Mamba-2 层 + NoPE
  - Kimi Linear：线性注意力层使用 NoPE（线性注意力本身通过状态矩阵隐含位置信息）

  动机：RoPE 在长上下文外推时角度超出训练分布，NoPE 层不受此限制，有助于长上下文泛化。

- **关联论文/模型**：SmolLM3 (HuggingFaceTB, 2025-06)；Nemotron 3 (NVIDIA)；Kimi Linear (技术报告)
- **参考分析链接**：
  - https://sebastianraschka.com/llm-architecture-gallery/nope/
  - https://magazine.sebastianraschka.com/p/the-big-llm-architecture-comparison


### QK-Norm

- **缩写**：QK-Norm
- **中文名称**：查询-键归一化
- **简短介绍**：在注意力计算前对 Query 和 Key 应用 RMSNorm 归一化，防止注意力分数过大导致训练不稳定。被 OLMo 2、Qwen3 等广泛采用。
- **详细介绍**：QK-Norm 的操作：

  Q' = RMSNorm(Q),  K' = RMSNorm(K)
  Attention = softmax(Q' · K'^T / sqrt(d_k)) · V

  QK-Norm 在 RoPE 之前对 Q/K 应用 RMSNorm（Raschka 在架构对比中确认这一位置）。QK-Norm 替代或补充了传统的 1/sqrt(d_k) 缩放。RMSNorm 使 Q 和 K 的范数保持稳定，防止深层网络中 Q·K 值爆炸导致 softmax 饱和。

  采用 QK-Norm 的模型：
  - OLMo 2：MHA + QK-Norm
  - Qwen3 (235B/32B/4B/8B)：GQA + QK-Norm
  - MiniMax M2：per-head 变体的 QK-Norm

  效果：改善训练稳定性，特别是在大规模模型和长上下文训练中。

- **关联论文/模型**：OLMo 2 (arXiv:2501.00656)；Qwen3 (arXiv:2505.09388)；MiniMax M2 (技术报告)
- **参考分析链接**：
  - https://sebastianraschka.com/llm-architecture-gallery/qk-norm/
  - https://magazine.sebastianraschka.com/p/the-big-llm-architecture-comparison


### Compressed Convolutional Attention

- **缩写**：CCA
- **中文名称**：压缩卷积注意力
- **简短介绍**：Zyphra 在 ZAYA1-8B 中提出的注意力机制，将 Q/K/V 压缩到低维潜在空间后直接在压缩空间执行注意力计算，并用卷积混合增强压缩表示的表达力。
- **详细介绍**：CCA 与 MLA 的区别：

  MLA：压缩 KV 存储 → 解压 → 在原始空间计算注意力
  CCA：压缩 Q/K/V → 直接在压缩空间计算注意力 → 解压输出

  这意味着 CCA 不仅减少 KV Cache，还减少注意力 FLOPs（prefill 和训练阶段）。

  额外的卷积混合：在压缩后的 Q 和 K 上应用一维卷积，为压缩表示注入局部上下文信息（V 不做卷积，因为 V 是内容而非打分）。

  ZAYA1-8B 使用 CCA + 4:1 GQA + 极稀疏 MoE（每 token 仅激活 1 个专家）。

  据 CCA 论文报告，在同等压缩设置下 CCA 优于 MLA，但目前仅在 ZAYA1-8B 验证。

- **关联论文/模型**："Compressed Convolutional Attention" (arXiv:2510.04476, 2025-10)；ZAYA1-8B (arXiv:2605.05365)
- **参考分析链接**：
  - https://magazine.sebastianraschka.com/p/recent-developments-in-llm-architectures (Section 4)


### Hybrid Attention

- **缩写**：—
- **中文名称**：混合注意力
- **简短介绍**：在同一个模型中交替使用不同类型的注意力层（如线性注意力 + 完整注意力），在效率和精度之间取得平衡。2025-2026 年的主流趋势之一。
- **详细介绍**：混合注意力的常见模式：

  1. 线性 + 全局（3:1）：3 层线性注意力 + 1 层完整注意力
     - Kimi Linear：3 层 KDA + 1 层 MLA
     - Qwen3-Next：3 层 Gated DeltaNet + 1 层 full attention

  2. 滑动窗口 + 全局（5:1）：5 层 SWA + 1 层全局注意力
     - Gemma 3 (27B)：52 层 SWA + 10 层全局
     - Laguna XS.2：30 层 SWA + 10 层全局

  3. Mamba-2 + GQA：SSM 层 + 少量注意力层
     - Nemotron 3：6 层 GQA + 23 层 Mamba-2
     - Soofi S：相同架构

  4. CSA + HCA：压缩稀疏 + 重度压缩交替
     - DeepSeek V4：CSA 和 HCA 交替

  核心思想：完整注意力是昂贵但精确的"稀缺资源"，放在关键层；廉价注意力放在大部分层以控制成本。

  反例：MiniMax M2 放弃混合注意力回归完整注意力。原因在于 linear attention 在推理和多轮任务中精度不足，且低精度 KV 状态对 prefix caching 支持偏弱，使长上下文复用失去杠杆（Raschka 在 M2 分析中明确指出这两点）。

- **关联论文/模型**：Kimi Linear (技术报告)；Qwen3-Next；Gemma 3；Nemotron 3；DeepSeek V4 (技术报告)
- **参考分析链接**：
  - https://sebastianraschka.com/llm-architecture-gallery/hybrid-attention/
  - https://www.youtube.com/watch?v=SH6Cyhunl8s (Hybrid Attention and Gated DeltaNet)
  - https://www.youtube.com/watch?v=Y6APnyZT6XU (LLM Architecture in 2026, 49:00 段)

## 本篇小结

本篇 6 个注意力相关术语沿"线性注意力从理论到生产 + 配套稳定性与混合架构"两条线归位：

- **Lightning Attention** 把 softmax 近似为线性形式并用结合律重排计算，将注意力复杂度从 O(n²) 降到 O(n)；但 MiniMax 在 M2 中放弃它回归完整注意力，原因是线性注意力在推理和多轮任务中精度不足，且低精度 KV 状态对 prefix caching 支持偏弱——生产环境的精度门槛比纸面效率更重要。
- **KDA** 是 Gated DeltaNet 的改进，把标量衰减门控升级为逐通道对角门控 Diag(α_t)，每个通道独立控制遗忘率；Kimi Linear 用 3:1 混合（3 层 KDA + 1 层全局 MLA），1M token 上下文下最高 75% KV 压缩与 6× 解码提升（Moonshot 官方指标，限定 1M 上下文）。
- **NoPE** 在选定层省略位置编码，依赖因果掩码隐式编码位置，降低长上下文计算开销；常见于线性注意力层（Kimi Linear、Qwen3-Next）。
- **QK-Norm** 在注意力计算前对 Q/K 做 RMSNorm，防止注意力分数过大导致训练不稳定；OLMo 2、Qwen3、Gemma 3 已广泛采用，是"免费的稳定性增强"。
- **CCA** 把 Q/K/V 压缩到低维潜在空间后直接在压缩空间执行注意力，并用卷积混合增强表达力；ZAYA1-8B 的实践，与 MLA 的区别是 MLA 上投影还原后再算注意力，CCA 在压缩空间直接算。
- **Hybrid Attention** 在同一模型中交替使用不同注意力层（线性 + 完整、SWA + 全局、CSA + HCA），完整注意力作为稀缺资源放在关键层；2025-2026 年的主流趋势，但 MiniMax M2 放弃混合回归完整注意力是重要反例。


本系列共 11 篇，本文是第 2 篇。系列导航见：LLM 术语全景图系列导读。
