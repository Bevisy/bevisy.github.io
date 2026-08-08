---
title: "LLM 术语全景图（五）：推理优化——从推测解码到 Flash Attention"
date: 2026-07-27T10:00:00+08:00
slug: "llm-terminology-05-inference-optimization"
draft: false
image: "05.png"
tags:
    - AI
    - LLM
    - Speculative Decoding
    - PagedAttention
    - Flash Attention
    - vLLM
categories:
    - AI
    - LLM术语全景图
---

> 这是「大模型训练与推理术语全景图」系列的第 5 篇，覆盖推理优化的 5 个概念：Speculative Decoding、PagedAttention、Continuous Batching、Quantization（总览）、Flash Attention。
> 主要参考框架/论文：vLLM、FlashAttention、Orca、DeepSeek V3 MTP

---

## 开篇：推理优化的两个方向

推理优化沿两个方向展开。第一是减少单位计算的代价：Flash Attention 通过 tiling 重排注意力计算的内存访问模式，把 HBM 与 SRAM 之间的读写复杂度从 O(n²) 降到 O(n²/M)，不改变注意力的数学结果；Quantization 把权重/激活从 BF16 降到 INT8/INT4/FP8，直接压低显存占用和每步访存量。第二是提高 GPU 利用率：PagedAttention 把 KV Cache 切成固定 block 按需分配，消除碎片让显存利用率从 ~20% 提升到 ~96%；Continuous Batching 在每个 iteration 动态进出请求，避免短序列空等长序列；Speculative Decoding 用小模型先起草、大模型并行验证，减少 decode 步数。本篇按这两个方向梳理 5 个推理优化术语。


## 推理优化（Inference Optimization）


### Speculative Decoding

- **缩写**：SpecDec / 推测解码
- **中文名称**：推测解码
- **简短介绍**：用小模型（draft model）快速生成候选 token，大模型（target model）并行验证，接受正确的候选 token 以加速推理。典型加速 2-3 倍，输出完全无损。
- **详细介绍**：推测解码的工作流程：

  1. Draft：小模型快速生成 k 个候选 token {t_1, t_2, ..., t_k}
  2. Verify：大模型一次前向传播，并行计算这 k 个 token 的概率
  3. Accept/Reject：对每个候选 token，如果大模型也认为正确则接受，否则从拒绝点重新采样

  关键性质：输出分布与纯大模型生成完全一致（无损），通过拒绝采样保证数学等价。

  变体：
  - 标准 SpecDec：独立的小模型作为 draft
  - MTP 推测：DeepSeek V3/R1 用训练时 MTP 模块作为内置 draft model
  - Medusa：多头并行预测
  - EAGLE：在隐空间而非 token 空间做推测

  vLLM 中的使用：
  --speculative-config='{"method": "deepseek_mtp", "num_speculative_tokens": 1}'

  性能：典型场景加速 2-3x，但加速比取决于 draft 模型的接受率（accept rate）。

  注意：推测解码目前与 Pipeline Parallelism 不兼容（vLLM V1 已部分解决）。

- **关联论文/模型**："Speculative Decoding" (Leviathan et al., 2023, arXiv:2211.17192)；DeepSeek V3 MTP (技术报告)；EAGLE、Medusa
- **参考分析链接**：
  - https://rocm.docs.amd.com/projects/ai-developer-hub/en/latest/notebooks/inference/mtp.html
  - https://docs.vllm.ai/en/latest/features/speculative_decoding/


### PagedAttention

- **缩写**：—
- **中文名称**：分页注意力
- **简短介绍**：vLLM 提出的 KV Cache 管理方法，将 KV Cache 分为固定大小的块（page），按需分配，类似操作系统的虚拟内存分页机制，大幅减少显存碎片和浪费。
- **详细介绍**：PagedAttention 的设计：

  传统方式：为每个序列预分配最大长度的连续 KV Cache 空间
  问题：实际序列通常远短于最大长度，造成大量显存浪费和碎片

  PagedAttention：
  - 将 KV Cache 分为固定大小的 block（如 16 token/block）
  - 每个序列维护一个 block table（类似页表），映射逻辑位置到物理 block
  - 按需分配 block，不需要预分配最大长度
  - 显存利用率从 ~20% 提升到 ~96%

  优势：
  - 支持更大的 batch size（更多并发请求）
  - 允许变长序列（无需预知长度）
  - 支持共享 KV（如 beam search 共享 prefix）

  PagedAttention 是 vLLM 高吞吐推理的核心技术，已被广泛集成到主流推理框架。

- **关联论文/模型**："Efficient Memory Management for LLMs with PagedAttention" (Kwon et al., 2023, arXiv:2309.06180)；vLLM
- **参考分析链接**：
  - https://www.runpod.io/articles/guides/vllm-pagedattention-continuous-batching


### Continuous Batching

- **缩写**：—
- **中文名称**：连续批处理 / 动态批处理
- **简短介绍**：在推理过程中动态插入新请求和移除已完成请求，而非等待整个 batch 全部完成。与 PagedAttention 配合实现高吞吐推理。
- **详细介绍**：传统静态批处理 vs 连续批处理：

  静态批处理：batch 中最长的序列决定整个 batch 的等待时间，短序列 GPU 空闲
  连续批处理：每个 iteration 检查 batch，完成的请求移出，新请求加入

  配合 PagedAttention，不同序列可以不同长度、不同阶段（prefill/decode 混合），实现 GPU 持续高利用率。

  vLLM、TGI、SGLang 等推理框架均采用连续批处理作为基础调度策略。

- **关联论文/模型**：vLLM (arXiv:2309.06180)；Orca (OSDI 2022)
- **参考分析链接**：
  - https://mbrenndoerfer.com/writing/continuous-batching


### Quantization

- **缩写**：Quant / 量化
- **中文名称**：模型量化
- **简短介绍**：将模型参数从高精度（FP32/FP16/BF16）降至低精度（INT8/INT4/FP8），减少显存占用和推理延迟。分为训练后量化（PTQ）和量化感知训练（QAT）。
- **详细介绍**：主要量化方案：

  按精度：
  - FP8：Hopper GPU 原生支持，几乎无损
  - INT8：W8A8（权重和激活都 INT8），精度损失较小
  - INT4：W4A16（权重 INT4，激活 FP16），显存大幅减少但需要校正
  - GGUF：llama.cpp 格式，支持 2-8 bit 量化

  按方法：
  - PTQ（Post-Training Quantization）：训练后直接量化，简单快速
    - GPTQ：基于二阶信息的逐层量化
    - AWQ：保护重要权重的激活感知量化
    - SmoothQuant：将量化难度从激活转移到权重
  - QAT（Quantization-Aware Training）：训练时模拟量化，精度更高但需训练

  实际效果（以 70B 模型为例）：
  - BF16：~140 GB（需 2×A100 80GB）
  - INT8：~70 GB（1×A100 80GB）
  - INT4：~35 GB（1×A100 40GB 可行）

  趋势：FP8 量化成为 Hopper/Blackwell GPU 上的主流选择，近乎无损且硬件原生支持。

  本条为量化总览，按精度和按方法的分类到此为止；具体量化方法（GPTQ / AWQ / SmoothQuant / FP8 详析等）见篇 8。

- **关联论文/模型**：GPTQ (arXiv:2210.17323)；AWQ (arXiv:2306.00978)；SmoothQuant (arXiv:2211.10438)；llama.cpp GGUF
- **参考分析链接**：
  - https://docs.vllm.ai/en/latest/features/quantization/


### Flash Attention

- **缩写**：FA / FlashAttn
- **中文名称**：闪存注意力
- **简短介绍**：通过重新组织注意力计算的内存访问模式，减少 GPU HBM 和 SRAM 之间的读写次数，在不改变注意力数学结果的情况下加速计算 2-4 倍。
- **详细介绍**：Flash Attention 的核心思想：

  问题：标准注意力的中间矩阵 (Q·K^T) 大小为 n×n，远超 GPU SRAM 容量，需频繁读写 HBM
  方案：将 Q/K/V 分块加载到 SRAM，计算局部注意力并累积结果，避免实例化完整 n×n 矩阵

  关键技术：
  1. Tiling：将 Q/K/V 分块，每次处理一个块
  2. Recomputation：反向传播时不存储中间矩阵，重新计算（省显存但略增计算）
  3. Online Softmax：分块计算 softmax 的数值稳定算法

  性能：
  - 前向传播：减少 HBM 读写，IO 复杂度从 O(n^2) 降至 O(n^2 / M)，M 为 SRAM 大小
  - 反向传播：不存储 attention matrix，显存从 O(n^2) 降至 O(n)
  - 实测加速：2-4x（取决于序列长度和硬件）

  Flash Attention 已成为 LLM 训练和推理的标准组件，PyTorch 2.0+ 内置支持。

  版本演进：FA1 (2022) → FA2 (2023, 更高效) → FA3 (2024, 支持 FP8)

  注意：Flash Attention 是计算优化，不改变注意力的数学结果。与稀疏注意力（DSA/SWA）是正交概念——前者优化"怎么算"，后者改变"算什么"。

- **关联论文/模型**："FlashAttention: Fast and Memory-Efficient Exact Attention" (Dao et al., 2022, arXiv:2205.14135)；FlashAttention-2 (Dao, 2023)；FlashAttention-3 (2024)
- **参考分析链接**：
  - https://github.com/Dao-AILab/flash-attention


## 本篇小结

- **Speculative Decoding（推测解码）** 用 draft model 起草、target model 并行验证，输出分布通过拒绝采样与纯大模型生成数学等价（无损），典型加速 2-3x；与 Pipeline Parallelism 不兼容，vLLM V1 已部分解决。
- **PagedAttention** 把 KV Cache 切成固定 block 按需分配，配 block table 类比操作系统的虚拟内存分页，显存利用率从 ~20% 提升到 ~96%，是 vLLM 高吞吐的核心。
- **Continuous Batching** 在每个 iteration 动态进出请求，配合 PagedAttention 让 prefill/decode 混合、变长序列共存，GPU 持续高利用率；源自 Orca (OSDI 2022)。
- **Quantization** 按精度（FP8/INT8/INT4/GGUF）和按方法（PTQ vs QAT）两条轴分类；70B 模型 BF16 ~140GB / INT8 ~70GB / INT4 ~35GB；具体量化方法见篇 8。
- **Flash Attention** 用 tiling + online softmax 把 IO 复杂度从 O(n²) 降到 O(n²/M)（M 为 SRAM 大小），是计算优化而非改变注意力数学；与稀疏注意力（DSA/SWA）正交——前者优化"怎么算"，后者改变"算什么"。


本系列共 11 篇，本文是第 5 篇。
