---
title: "LLM 术语全景图（一）：注意力机制上——MHA 到 MLA 的压缩演进"
date: 2026-07-27T09:00:00+08:00
slug: "llm-terminology-01-attention-mha-mla"
draft: false
image: "01.png"
tags:
    - AI
    - LLM
    - Attention
    - MLA
    - GQA
    - KV Cache
categories:
    - AI
    - LLM术语全景图
---

> 这是「大模型训练与推理术语全景图」系列的第 1 篇，覆盖注意力机制前半部分的 8 个概念：MHA、MQA、GQA、MLA、SWA、DSA、CSA、HCA。
> 主要参考模型：DeepSeek、GLM、Kimi (Moonshot)、Llama、Gemma、MiniMax

---

## 开篇：为什么 2025-2026 年注意力机制在密集创新

2025-2026 年，注意力机制的迭代密度显著上升，其驱动力并非抽象的"模型变大"，而是一个具体的工程瓶颈：KV Cache 已成为长上下文推理的第一代价。在自回归解码阶段，每生成一个 token 都要对所有历史 token 的 K/V 做注意力，这意味着 KV Cache 的体积随上下文长度线性增长，而它在显存中的驻留又直接挤占了 batch size 与并发上限。当推理模型需要维持数万 token 的思维链、Agent 工作流需要在单次会话中携带工具调用历史与检索结果时，KV Cache 从"可优化项"变成了"硬约束"。MHA 到 MQA、GQA 的头维度压缩，MLA 的潜在维度压缩，SWA/DSA 的序列维度稀疏，CSA/HCA 的序列维度压缩，本质都在回答同一个问题：在不牺牲检索精度的情况下，如何让 KV Cache 不再是长上下文推理的瓶颈。本篇按这条主线梳理前 8 个注意力变体。


## 注意力机制（上）


### Multi-Head Attention

- **缩写**：MHA
- **中文名称**：多头注意力
- **简短介绍**：Transformer 原始论文提出的注意力机制，每个注意力头有独立的 Q/K/V 投影，能从不同子空间捕获不同的注意力模式。
- **详细介绍**：对于输入序列 X，每个头独立计算：

  Q_h = X·W_h^Q,  K_h = X·W_h^K,  V_h = X·W_h^V

  Attention(Q_h, K_h, V_h) = softmax(Q_h·K_h^T / sqrt(d_k)) · V_h

  多头输出拼接后经线性投影得到最终输出。头数 h 通常为 32~96，每头维度 d_k = d_model / h。MHA 的 KV Cache 占用最大：每 token 需存储 2 × n_layers × n_heads × d_head 维度。

  示例：GPT-2 XL（1.5B）有 48 层 × 25 头，每 token KV Cache 约 300 KiB（bf16）。

- **关联论文/模型**："Attention Is All You Need" (Vaswani et al., 2017, arXiv:1706.03762)；GPT-2、GPT-3、原始 Transformer；OLMo 2 仍保留 MHA
- **参考分析链接**：
  - https://sebastianraschka.com/llm-architecture-gallery/mha/
  - https://www.turingpost.com/p/attention-types


### Multi-Query Attention

- **缩写**：MQA
- **中文名称**：多查询注意力
- **简短介绍**：所有查询头共享同一组 K/V 投影，是 MHA 的极端压缩形式，大幅减少 KV Cache 但可能损失精度。
- **详细介绍**：MQA 只保留一个 K/V 头，所有 Q 头都共享这同一组 K/V：

  K = X·W^K (仅一份),  V = X·W^V (仅一份)

  每个 Q 头独立计算 attention，但 K/V 共享。KV Cache 减少到 MHA 的 1/n_heads。例如 32 头模型从 32 组 KV 降至 1 组，显存节省 ~96%。

  在 GQA 的统一框架下理解：当 GQA 的 KV 头数 n_kv_heads = 1 时即 MQA，当 n_kv_heads = n_heads 时即 MHA。MQA 是 GQA 的极端特例（仅 1 个 KV 头）。Gemma 4 E2B 使用 MQA + 跨层 KV 共享进一步压缩。

- **关联论文/模型**："Fast Transformer Decoding" (Shazeer, 2019, arXiv:1911.02150)；PaLM、Falcon、Gemma 4 E2B
- **参考分析链接**：
  - https://arxiv.org/abs/1911.02150
  - https://www.pythonalchemist.com/llm-architectures/attention-variants


### Grouped Query Attention

- **缩写**：GQA
- **中文名称**：分组查询注意力
- **简短介绍**：MHA 和 MQA 的折中方案，将查询头分组，每组共享一对 K/V 头，在精度和 KV Cache 压缩之间取得平衡。当前最主流的注意力机制。
- **详细介绍**：将 n_heads 个 Q 头分成 n_groups 组，每组共享一组 K/V。用 KV 头数 n_kv_heads 表达压缩程度：

  - n_kv_heads = 1 时即 MQA（极端压缩）
  - n_kv_heads = n_heads 时即 MHA（无压缩）
  - 介于两者之间即 GQA

  例如 Llama 3 8B：32 个 Q 头，8 个 KV 头（4:1 分组），KV Cache 降至 MHA 的 25%。

  GQA 是 2024-2026 年最广泛采用的注意力机制：
  - Llama 3/4：GQA + RoPE
  - Qwen3：GQA + QK-Norm
  - Mistral Small 3.1：标准 GQA
  - Kimi K2：GQA（部分层）

  KV Cache 每 token（bf16）= 2 × n_layers × n_kv_heads × d_head × 2 bytes

- **关联论文/模型**："GQA: Training Generalized Multi-Query Transformer Models" (Ainslie et al., 2023, arXiv:2305.13245)；Llama 3、Qwen3、Mistral、Kimi K2
- **参考分析链接**：
  - https://sebastianraschka.com/llm-architecture-gallery/gqa/
  - https://friendli.ai/blog/gqa-vs-mha


### Multi-head Latent Attention

- **缩写**：MLA
- **中文名称**：多头潜在注意力
- **简短介绍**：DeepSeek 提出的注意力机制，通过低秩投影将 K/V 压缩到低维潜在空间存储，推理时再上投影还原，大幅减少 KV Cache 同时保持精度。
- **详细介绍**：MLA 的核心思想是对 K/V 做低秩压缩：

  1. 压缩：c_KV = W_DKV · x  （将 d_model 压缩到 d_c，d_c << n_heads × d_head）
  2. 上投影：K = W_UK · c_KV,  V = W_UV · c_KV
  3. 查询同样压缩：c_Q = W_DQ · x,  Q = W_UQ · c_Q

  KV Cache 只需存储压缩后的 c_KV（d_c 维），而非完整的 K/V。以 DeepSeek V3 为例：KV Cache 每 token 仅 68.6 KiB（bf16），远低于同规模 MHA 模型。

  MLA 的关键创新：RoPE 需要独立处理。MLA 将 K 分为两部分——压缩部分（走低秩路径）和 RoPE 部分（独立计算），以兼容旋转位置编码。

  采用 MLA 的模型：
  - DeepSeek V2/V3/V4：MLA 是核心注意力
  - GLM-5/5.1/5.2：MLA + DSA
  - Kimi K2.5：MLA
  - Kimi Linear：3:1 混合 KDA + MLA（全局层用 MLA）
  - Mistral（部分新模型）

- **关联论文/模型**：DeepSeek-V2 (arXiv:2405.04434)；DeepSeek-V3 (arXiv:2412.19437)；GLM-5 (技术报告)；Kimi Linear (arXiv:2510.26692)
- **参考分析链接**：
  - https://sebastianraschka.com/llm-architecture-gallery/mla/
  - https://magazine.sebastianraschka.com/p/visual-attention-variants (Visual Guide to Attention Variants)


### Sliding Window Attention

- **缩写**：SWA
- **中文名称**：滑动窗口注意力
- **简短介绍**：每个 token 只关注固定大小窗口内的邻近 token（如 1024 或 4096），而非整个序列，将注意力复杂度从 O(n^2) 降至 O(n×w)。
- **详细介绍**：SWA 限制注意力范围到局部窗口 [t-w, t]：

  Attention(Q, K, V) 限制为仅计算 |i - j| <= w 的位置

  窗口大小 w 通常为 512~4096 token。SWA 可显著减少 KV Cache 和注意力计算量，但无法直接捕获长距离依赖。

  实践中通常混合使用 SWA + 全局注意力层：
  - Gemma 3 (27B)：52 层 SWA + 10 层全局（5:1 比例）
  - Mistral Small 3.1：5:1 SWA + 全局
  - Laguna XS.2：30 层 SWA + 10 层全局
  - Llama 4 Maverick：36 层 chunked + 12 层 full GQA

  信息传递机制：局部窗口层层传递，信息通过多层 SWA 逐步扩散到远距离（类似消息传递）。

- **关联论文/模型**："Longformer" & "BigBird" (2020)；Mistral 7B；Gemma 3；Llama 4；Laguna XS.2
- **参考分析链接**：
  - https://sebastianraschka.com/llm-architecture-gallery/swa/
  - https://magazine.sebastianraschka.com/p/the-big-llm-architecture-comparison


### DeepSeek Sparse Attention

- **缩写**：DSA
- **中文名称**：DeepSeek 稀疏注意力
- **简短介绍**：DeepSeek V3.2-Exp 引入的稀疏注意力机制，通过学习到的选择器（indexer）动态选择需要关注的 token 子集，在保持检索能力的同时大幅降低注意力计算量。
- **详细介绍**：DSA 的工作原理：

  1. Indexer 层：用一个轻量级网络对序列中的 token 打分，选出最重要的 top-k token
  2. Attention 仅在选中的 token 上计算，而非全部
  3. 保留局部窗口（近期 token 全部可见）+ 选中的稀疏 token

  与 SWA 的区别：SWA 是固定窗口，DSA 是动态选择。与标准稀疏注意力的区别：DSA 的选择是端到端学习的，不是预设模式。

  GLM-5 用 DSA；GLM-5.2 在 DSA 基础上引入 IndexShare 机制：一个 DSA indexer 的结果复用到后续 4 层，进一步降低 1M token 上下文的计算成本（见 GLM-5 GitHub：reuses the same indexer across every four sparse attention layers）。

  采用 DSA 的模型：
  - DeepSeek V3.2-Exp：MLA + DSA
  - GLM-5：MLA + DSA
  - GLM-5.1/5.2：MLA + DSA + IndexShare（5.2 引入 IndexShare）
  - DeepSeek V4：CSA（基于 DSA 选择器）+ HCA

- **关联论文/模型**：DeepSeek V3.2-Exp 技术报告；GLM-5 (技术报告)；DeepSeek V4 (技术报告)
- **参考分析链接**：
  - https://sebastianraschka.com/llm-architecture-gallery/deepseek-sparse-attention/
  - https://magazine.sebastianraschka.com/p/technical-deepseek (From DeepSeek V3 to V3.2)
  - https://sebastianraschka.com/blog/2026/glm-5-2-indexshare.html (GLM-5.2 IndexShare)


### Compressed Sparse Attention

- **缩写**：CSA
- **中文名称**：压缩稀疏注意力
- **简短介绍**：DeepSeek V4 引入的注意力机制，结合序列维度压缩和 DSA 风格的稀疏选择，以较低的压缩率保留更多细节，适合需要精确检索的场景。
- **详细介绍**：CSA 的核心设计：

  1. 序列维度压缩：将连续的 token 组压缩为更少的 KV 条目（但压缩率比 HCA 温和）
  2. 稀疏选择：使用 DSA 风格的 indexer 选择最相关的压缩条目
  3. 局部窗口：保留近期 token 的未压缩 KV 用于精确注意力
  4. 共享 KV 注意力：压缩条目使用共享 K/V 进一步降低开销

  与 MLA 的关键区别：MLA 压缩的是单个 token 的 KV 表示维度（per-token compression），CSA 压缩的是序列长度维度（sequence-length compression），即将多个 token 的信息汇总到更少的条目中。其中"共享 KV 注意力"（shared-KV）的点在 Raschka 原文中有提及，此处保留。

  CSA 保留较多压缩条目（细节多），但需要稀疏选择来控制计算量。HCA 则是更激进的版本。

  在 DeepSeek V4 中，CSA 和 HCA 交替使用：CSA 负责精确检索，HCA 负责低成本全局覆盖。

- **关联论文/模型**：DeepSeek V4 (技术报告)；mHC 论文 (arXiv:2512.24880)
- **参考分析链接**：
  - https://magazine.sebastianraschka.com/p/recent-developments-in-llm-architectures (Section 5.2)
  - https://sebastianraschka.com/llm-architecture-gallery/csa-hca/


### Heavily Compressed Attention

- **缩写**：HCA
- **中文名称**：重度压缩注意力
- **简短介绍**：DeepSeek V4 引入的注意力机制，将每 128 个 token 激进压缩为一个 KV 条目，然后对这些少量压缩条目做密集注意力，以最低成本实现全局覆盖。
- **详细介绍**：HCA 的设计思路与 CSA 互补：

  1. 激进压缩：每 128 个 token 压缩为 1 个 KV 条目（压缩率 128:1）
  2. 密集注意力：压缩后条目数量极少，可以对所有条目做完整 attention
  3. 局部窗口：同样保留近期未压缩 token

  对比 CSA vs HCA：
  - CSA：温和压缩 + 稀疏选择 → 更多细节，但需要选择器开销
  - HCA：激进压缩 + 密集注意力 → 更少条目，无需选择器，但丢失细节

  DeepSeek V4 交替使用 CSA 和 HCA，使两者互补。

  效果指标（DeepSeek V4 论文，1M token 上下文，相对 V3.2）：
  - V4-Pro：27% FLOPs、10% KV Cache
  - V4-Flash：10% FLOPs、7% KV Cache

  注意：CSA/HCA 比 MLA 更复杂，目前仅在 DeepSeek V4 大规模验证，不一定普遍优于 MLA。

- **关联论文/模型**：DeepSeek V4 (技术报告)
- **参考分析链接**：
  - https://magazine.sebastianraschka.com/p/recent-developments-in-llm-architectures (Section 5.2 CSA/HCA)
  - https://sebastianraschka.com/llm-architecture-gallery/csa-hca/


## 本篇小结

本篇 8 个注意力变体可以沿三条压缩主线归位：

- **头维度压缩**：MHA → MQA → GQA，沿 n_kv_heads 轴从 n_heads 到 1 逐级压缩，GQA 的 n_kv_heads=1 即 MQA、n_kv_heads=n_heads 即 MHA。
- **潜在维度压缩**：MLA 把 per-token 的 K/V 压进低秩潜在空间，是另一种正交的压缩方向。
- **序列维度压缩 / 稀疏**：SWA 用固定窗口、DSA 用学习到的 indexer 做稀疏选择，CSA/HCA 则直接压缩序列长度本身，HCA 更激进、CSA 更温和，DeepSeek V4 让两者交替互补。


本系列共 11 篇，本文是第 1 篇。系列导航见：LLM 术语全景图系列导读。
