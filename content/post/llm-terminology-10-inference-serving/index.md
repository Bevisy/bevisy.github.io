---
title: "LLM 术语全景图（十）：推理服务——Prefill Decode 与性能指标"
date: 2026-07-27T11:15:00+08:00
slug: "llm-terminology-10-inference-serving"
draft: false
image: "10.png"
tags:
    - AI
    - LLM
    - Inference Serving
    - TTFT
    - TPOT
    - vLLM
    - SGLang
categories:
    - AI
    - LLM术语全景图
---


> 这是「大模型训练与推理术语全景图」系列的第 10 篇，覆盖推理服务与性能指标的 6 个概念：Prefill/Decode、TTFT、TPOT、Prefix Caching / RadixAttention、Chunked Prefill、Disaggregated Inference。
> 主要参考框架/服务：vLLM、SGLang、TGI、DeepSeek 推理服务、Sarathi-Serve、DistServe、Splitwise

---

## 开篇：推理服务的两个阶段和两个指标

LLM 推理不是一次"模型前向"，而是两个特性截然不同的阶段。Prefill（预填充）一次性处理整个 prompt，构造 N×N 注意力矩阵、打满 GPU 算力，是 compute-bound；Decode（解码）逐 token 生成，每步都要读回全部模型权重与 KV Cache、算力利用率往往低于 5%，是 memory-bound。用户感知到的两个维度恰好对应这两阶段：TTFT（首 token 延迟）由 prefill 主导，决定"是不是即时响应"；TPOT（单 token 生成时间）由 decode 主导，决定"是不是流畅"。高效推理服务（vLLM、SGLang）的所有优化都围绕这两阶段展开——Chunked Prefill 把长 prompt 切块，避免 prefill 阻塞正在 decode 的请求；Prefix Caching 复用历史 KV，多轮对话 TTFT 降低 50-90%；Disaggregated Inference 干脆把两阶段分池部署，prefill 池高算力、decode 池高带宽，吞吐再提 30-60%。本篇按"两阶段 → 两指标 → 三类优化"的脉络梳理这 6 个术语。


## 推理服务与性能指标（Inference Serving & Metrics）

> LLM 推理不是简单的"模型前向"。高效推理服务需要理解 prefill/decode 两阶段、关键延迟指标和缓存策略。


### Prefill/Decode

- **缩写**：—
- **中文名称**：预填充与解码
- **简短介绍**：LLM 推理分两个阶段。Prefill（预填充）一次性处理整个 prompt，计算密集（compute-bound）；Decode（解码）逐 token 生成，内存密集（memory-bound）。两阶段特性不同，优化策略也不同。
- **详细介绍**：两阶段的本质区别：

  Prefill 阶段：
  - 输入：完整 prompt（N 个 token）
  - 计算：N × N 注意力矩阵 + N 层 FFN
  - 特性：compute-bound，GPU 算力打满
  - 耗时：与 prompt 长度 N 的平方成正比（注意力）+ 线性（FFN）

  Decode 阶段：
  - 输入：逐个 token（1 个/步）
  - 计算：1 × N_kv 注意力（与所有历史 KV）+ N 层 FFN
  - 特性：memory-bound，需读取全部权重 + KV Cache，但计算量小
  - 耗时：与模型大小成正比（读权重的时间），与 KV Cache 大小成正比

  为什么 decode 慢：
  - 每生成 1 个 token 需读取全部模型权重（如 70B 模型 = 140GB BF16）
  - H100 HBM 带宽 3.35 TB/s → 读 140GB 需 ~42ms → 仅 ~24 token/s
  - 算力远未打满（利用率可能 < 5%）

  优化策略差异：
  - Prefill：Flash Attention（减少 IO）、TP（并行计算）、Chunked Prefill
  - Decode：量化（减少权重大小 → 减少读取时间）、 batching（多请求分摊权重读取）、推测解码（减少 decode 步数）

  Continuous Batching 的关键：允许不同请求同时处于 prefill 和 decode 阶段，GPU 持续高利用率。

- **关联论文/模型**：vLLM、SGLang、TGI 等推理框架的基础概念
- **参考分析链接**：
  - https://huggingface.co/blog/tngtech/llm-performance-prefill-decode-concurrent-requests
  - https://docs.vllm.ai/en/v0.12.0/usage/v1_guide/


### Time To First Token

- **缩写**：TTFT
- **中文名称**：首 Token 延迟
- **简短介绍**：从发送请求到收到第一个生成 token 的时间。主要由 prefill 阶段决定，是用户体验的关键指标（尤其是对话场景）。
- **详细介绍**：TTFT 的构成：

  TTFT = 排队延迟 + 网络延迟 + Prefill 计算时间

  排队延迟：请求到达时 GPU 正忙，需等待当前 batch 处理完
  网络延迟：请求传输到推理服务器的时间（通常可忽略）
  Prefill 计算时间：处理 prompt 的时间

  Prefill 时间估算：
  - 短 prompt（100 token）：~10-50ms
  - 中等 prompt（1K token）：~100-500ms
  - 长 prompt（10K token）：~1-5s
  - 超长 prompt（100K token）：~10-60s（无优化时）

  优化 TTFT 的方法：
  - Chunked Prefill：将长 prompt 分块处理，避免长 prefill 阻塞 decode
  - Prefix Caching：如果多个请求共享相同前缀（如 system prompt），缓存其 KV
  - TP：多卡并行 prefill 加速
  - 量化：减少权重读取（对长 prompt 的 FFN 计算有帮助）

  典型目标：对话场景 TTFT < 200ms，用户感知"即时响应"。

- **关联论文/模型**：vLLM、SGLang 推理框架；LLM 推理服务标准指标
- **参考分析链接**：
  - https://docs.vllm.ai/en/latest/design/metrics.html


### Time Per Output Token

- **缩写**：TPOT
- **中文名称**：单 Token 生成时间
- **简短介绍**：decode 阶段平均生成一个 token 的时间。由 memory-bound 特性决定，是吞吐和延迟的关键指标。TPOT 的倒数即为单请求解码速度（token/s）。
- **详细介绍**：TPOT 的决定因素：

  TPOT ≈ (模型权重大小 + KV Cache 大小) / HBM 带宽

  示例（70B BF16, 8K 上下文, H100 80GB）：
  - 权重：140GB
  - KV Cache：~8GB
  - HBM 带宽：3.35 TB/s
  - TPOT ≈ 148GB / 3.35TB/s ≈ 44ms → ~23 token/s

  优化 TPOT 的方法：
  - 量化：INT4/FP8 权重 → 读取量减半/四分之一 → TPOT 减半/四分之一
  - Batching：多请求共享一次权重读取 → 每请求均摊 TPOT 降低
  - MLA/GQA：减少 KV Cache → 减少 KV 读取时间
  - 推测解码：一次验证多 token → 有效 TPOT 降低

  TPOT vs 吞吐：
  - 单请求 TPOT：1 个请求的解码速度（用户体验）
  - 批量吞吐：batch_size / TPOT（系统效率）
  - 增大 batch 可提高吞吐但不变 TPOT（权重只需读一次，多请求共享）

  典型目标：对话场景 TPOT < 50ms（20+ token/s），用户感知流畅。

- **关联论文/模型**：vLLM、SGLang 推理框架；LLM 推理服务标准指标
- **参考分析链接**：
  - https://docs.vllm.ai/en/latest/design/metrics.html


### Prefix Caching / RadixAttention

- **缩写**：—
- **中文名称**：前缀缓存 / 基数注意力
- **简短介绍**：缓存已计算 prompt 的 KV Cache，当新请求共享相同前缀时直接复用，跳过 prefill。SGLang 的 RadixAttention 用基数树管理缓存，实现自动 prefix 复用。
- **详细介绍**：Prefix Caching 的场景：

  - 多轮对话：每轮都包含完整历史 → 历史部分的 KV 可复用
  - System prompt：所有请求共享同一 system prompt → 缓存其 KV
  - Few-shot：相同示例前缀 → KV 复用
  - Agent 工作流：多次调用中共享上下文

  实现：
  1. 计算每个 prompt 的 hash（基于 token 序列）
  2. 新请求到达时，检查是否有前缀的 KV Cache 命中
  3. 命中部分跳过 prefill，只 prefill 不命中部分
  4. 新计算的 KV 也存入缓存

  vLLM 的实现：Automatic Prefix Caching，基于 PagedAttention 的 block 管理，自动检测前缀匹配。

  SGLang RadixAttention：
  - 用基数树（Radix Tree）管理 KV Cache
  - 树的每条路径代表一个 token 序列前缀
  - 新请求沿树匹配最长前缀，未命中部分分支新建
  - LRU 淘汰策略管理缓存容量

  效果：
  - 多轮对话 TTFT 降低 50-90%（历史部分跳过 prefill）
  - System prompt 场景几乎消除重复计算
  - 对 Agent 工作流（重复调用）效果显著

- **关联论文/模型**："SGLang RadixAttention" (Zheng et al., 2023, arXiv:2312.07104)；vLLM Automatic Prefix Caching
- **参考分析链接**：
  - https://lmsys.org/blog/2024-01-17-sglang/
  - https://docs.vllm.ai/en/latest/features/automatic_prefix_caching.html


### Chunked Prefill

- **缩写**：—
- **中文名称**：分块预填充
- **简短介绍**：将长 prompt 的 prefill 分成多个小块，与 decode 请求混合调度，避免长 prefill 阻塞正在 decode 的请求。vLLM V1 的核心调度策略之一。
- **详细介绍**：Chunked Prefill 解决的问题：

  无 Chunked Prefill 时：
  - 一个 10K token 的 prefill 请求到达
  - GPU 花 ~2s 处理这个 prefill
  - 正在 decode 的其他请求全部暂停 2s → TPOT 暴涨

  Chunked Prefill 方案：
  1. 将 10K token 的 prefill 分成 20 个 512-token 块
  2. 每个 iteration 只处理 1 个 prefill 块 + 当前所有 decode token
  3. prefill 和 decode 在同一 batch 混合执行
  4. 20 个 iteration 后 prefill 完成

  效果：
  - decode 请求每步都能执行 → TPOT 不受长 prefill 影响
  - prefill 总时间略增（分块效率略低），但整体延迟体验更好
  - GPU 持续高利用率（prefill 块 + decode 混合填满算力）

  vLLM V1 默认开启 Chunked Prefill，chunk_size 通常 512~2048 token。

  与 Continuous Batching 的关系：Chunked Prefill 是 Continuous Batching 的增强——传统 Continuous Batching 只混合不同 decode 请求，Chunked Prefill 进一步混合 prefill 和 decode。

- **关联论文/模型**：vLLM V1；SGLang；Sarathi-Serve (arXiv:2308.16369)
- **参考分析链接**：
  - https://docs.vllm.ai/en/v0.12.0/usage/v1_guide/
  - https://arxiv.org/abs/2308.16369


### Disaggregated Inference

- **缩写**：Disaggregated Inference / Prefill-Decode 分离
- **中文名称**：分离式推理
- **简短介绍**：将 prefill 和 decode 分配到不同的 GPU 池执行，各池针对各自阶段优化（prefill 池用高算力配置，decode 池用高带宽配置），通过 KV Cache 传输连接两阶段。
- **详细介绍**：分离式推理的动机：

  Prefill 和 Decode 的计算特性完全不同：
  - Prefill：compute-bound，需要高 FLOPS（TP 度数高）
  - Decode：memory-bound，需要高 HBM 带宽（batch 大更高效）

  传统推理（混合在同一 GPU 池）：
  - 同一 GPU 同时做 prefill 和 decode → 两阶段互相干扰
  - prefill 阻塞 decode（或 chunked prefill 增加 prefill 延迟）

  分离式推理：
  1. Prefill 池：专门处理 prefill，高 TP 度数，compute-optimized
  2. 计算完的 KV Cache 传输到 Decode 池
  3. Decode 池：专门做 decode，大 batch（多请求共享权重读取），memory-optimized

  KV Cache 传输：
  - 跨池传输 KV Cache 是主要开销
  - 通过高速互连（NVLink/InfiniBand）传输
  - 可在 prefill 最后几层时就启动传输（overlap）

  优势：
  - 两阶段各自独立优化，吞吐提升 30-60%
  - prefill 不会被 decode 拖慢，decode 不会被 prefill 阻塞
  - 可根据负载独立扩缩两池

  实践：DeepSeek 推理服务采用分离式架构；vLLM 正在支持；Moonshot/SGLang 也在探索。

  局限：KV Cache 传输增加延迟和复杂度，需要高速互连；对短序列收益小。

- **关联论文/模型**："DistServe: Disaggregating Prefill and Decoding" (Zhong et al., 2024, arXiv:2401.09670)；DeepSeek 推理服务；Splitwise (arXiv:2311.18677)
- **参考分析链接**：
  - https://arxiv.org/abs/2401.09670
  - https://bentoml.com/llm/inference-optimization/prefill-decode-disaggregation


## 本篇小结

本篇 6 个推理服务术语沿"两阶段 → 两指标 → 三类优化"的脉络归位：

- **Prefill/Decode** 是推理的两个阶段：前者一次性处理整个 prompt，compute-bound，GPU 算力打满；后者逐 token 生成，memory-bound，每步读全部权重但算力利用率常 < 5%。两阶段特性截然不同，优化策略也不同。
- **TTFT**（首 token 延迟）由 prefill 主导，决定"是不是即时响应"，对话场景尤其敏感。
- **TPOT**（单 token 生成时间）由 decode 主导，决定"是不是流畅"；其倒数即单请求解码速度（token/s）。增大 batch 能提高系统吞吐却不改变单请求 TPOT，因为权重只需读一次、多请求共享。
- **Prefix Caching / RadixAttention** 缓存已计算 prompt 的 KV，新请求共享前缀时直接复用跳过 prefill；SGLang 的 RadixAttention 用基数树管理缓存，多轮对话 TTFT 可降至 1/10。
- **Chunked Prefill** 把长 prompt 的 prefill 切块与 decode 混合调度，避免长 prefill 阻塞正在 decode 的请求；vLLM V1 的核心调度策略之一。
- **Disaggregated Inference** 把 prefill 和 decode 分到不同 GPU 池，各池针对各自阶段优化（prefill 池高算力、decode 池高带宽），DeepSeek 推理服务已落地，吞吐再提 30-60%，代价是 KV Cache 跨池传输。

三类优化层层递进：Prefix Caching 复用历史 KV → Chunked Prefill 让长 prompt 不再阻塞 decode → Disaggregated Inference 干脆分池部署。前两者在单池内调度，后者跨越池边界，代价是 KV Cache 传输开销。

本系列共 11 篇，本文是第 10 篇。系列导读见：[LLM 术语全景图：系列导读](/p/llm-terminology-series-guide/)
