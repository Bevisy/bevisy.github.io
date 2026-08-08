---
title: "LLM 术语全景图（八）：量化方法——从 GPTQ 到 FP8"
date: 2026-07-27T10:45:00+08:00
slug: "llm-terminology-08-quantization"
draft: false
image: "08.png"
tags:
    - AI
    - LLM
    - Quantization
    - GPTQ
    - AWQ
    - FP8
    - GGUF
categories:
    - AI
    - LLM术语全景图
---

> 这是「大模型训练与推理术语全景图」系列的第 8 篇，覆盖量化方法的 6 个概念：GPTQ、AWQ、SmoothQuant、GGUF/llama.cpp、FP8 Quantization、QAT。
> 主要参考模型/框架：AutoGPTQ、AutoAWQ、vLLM、llama.cpp、Hopper GPU

---

## 开篇：量化的两条路线分野

量化在 LLM 时代分出两条路线。数据中心路线——GPTQ、AWQ、SmoothQuant、FP8——面向 GPU 高吞吐服务，在精度与算力间求平衡：权重压到 INT4 省显存，激活保高精度，或用 W8A8 享受 Tensor Core 算力翻倍。消费级路线——GGUF——面向 CPU 与 Apple Metal 本地部署，把权重、tokenizer、架构参数打包进单文件，让 MacBook 也能跑 70B 模型。两条路线工具链不重叠，场景互补。路线内部一个明确趋势：FP8 正在取代 INT8 成为数据中心推理首选。LLM 激活值有大量异常值，定点 INT8 难以同时覆盖，需 SmoothQuant 辅助；而 FP8 浮点格式的指数位天然适配异常值动态范围，几乎无损即可拿到 Hopper GPU 上 BF16 两倍的 Tensor Core 算力。本篇按此脉络梳理 6 个量化术语。


## 量化方法（Quantization Methods）

> 量化是将高精度模型参数降至低精度以减少显存和加速推理的技术。本篇展开具体方法。


### GPTQ

- **缩写**：GPTQ
- **中文名称**：GPTQ 量化
- **简短介绍**：基于二阶 Hessian 信息的训练后量化方法，逐层将权重量化到 INT4/INT8，利用校准数据补偿量化误差。生成式任务中几乎无损，是 INT4 权重量化的主流方案。
- **详细介绍**：GPTQ 的核心思想：

  1. 逐层量化：对每一层的权重矩阵 W 逐列量化
  2. Hessian 信息：用校准数据计算 H = X^T · X（输入的二阶统计量）
  3. 顺序量化：量化第 j 列时，用 Hessian 信息预测量化误差，并补偿到未量化的列
  4. 公式：quant 后的误差被投影到剩余权重上，最小化输出误差

  量化流程：
  - 准备 ~128 条校准数据（如 C4/WikiText 片段）
  - 逐层运行校准数据，收集输入激活
  - 对每层权重做 GPTQ 量化（~几分钟）
  - 保存量化后的权重 + scale/zp

  典型精度：
  - W4A16（权重 INT4，激活 FP16）：显存 ~25% BF16，精度损失 < 1%
  - W8A16：显存 ~50%，几乎无损

  支持框架：AutoGPTQ、vLLM（原生支持 GPTQ 推理）、HuggingFace transformers

  局限：主要量化权重，激活仍需 FP16（W4A16 模式）；对极小模型（< 1B）精度损失更大。

- **关联论文/模型**："GPTQ: Accurate Post-Training Quantization for GPT" (Frantar et al., 2022, arXiv:2210.17323)；AutoGPTQ；vLLM
- **参考分析链接**：
  - https://arxiv.org/abs/2210.17323
  - https://docs.vllm.ai/en/latest/features/quantization/gptqmodel/


### Activation-aware Weight Quantization

- **缩写**：AWQ
- **中文名称**：激活感知权重量化
- **简短介绍**：通过分析激活值的分布识别"重要权重"（对应大激活的权重），对重要权重保持高精度、对非重要权重量化压缩。比 GPTQ 更快（无需反向传播补偿），精度相当。
- **详细介绍**：AWQ 的核心发现：

  不是所有权重同等重要——与大激活值对应的权重对量化误差更敏感。
  保持这些"重要权重"的高精度（或用 per-channel scale 缩放），其余权重可激进量化。

  AWQ 方法：
  1. 用校准数据统计每通道激活的幅度分布
  2. 对大激活通道的权重乘以 scale s（放大），量化后精度更高
  3. 对应的激活除以 s（反缩放），保持等价性
  4. 量化：W' = quant(W × s),  X' = X / s → W' × X' ≈ W × X

  与 GPTQ 对比：
  | 特性 | GPTQ | AWQ |
  |------|------|-----|
  | 方法 | Hessian 补偿 | 激活感知缩放 |
  | 校准 | 需要（二阶计算） | 需要（统计激活） |
  | 速度 | 较慢（逐列补偿） | 较快（仅需统计） |
  | 精度 | 略优（复杂模型） | 相当 |
  | 通用性 | 广 | 广 |

  支持：AutoAWQ、vLLM（原生支持）、HuggingFace

  W4A16 模式下 AWQ 和 GPTQ 是最常用的两种 INT4 量化方案。

- **关联论文/模型**："AWQ: Activation-aware Weight Quantization for LLM Compression" (Lin et al., 2023, arXiv:2306.00978)；AutoAWQ；vLLM
- **参考分析链接**：
  - https://arxiv.org/abs/2306.00978
  - https://docs.vllm.ai/en/latest/features/quantization/auto_awq/


### SmoothQuant

- **缩写**：—
- **中文名称**：平滑量化
- **简短介绍**：将量化难度从激活值转移到权重的训练后量化方法。通过 per-channel 缩放因子"平滑"激活的异常值，使权重和激活都能用 INT8 量化（W8A8），无需 FP16 激活。
- **详细介绍**：SmoothQuant 解决的问题：

  LLM 激活值有大量异常值（outlier），使 INT8 量化激活困难。GPTQ/AWQ 量化权重但保持激活 FP16（W4A16），仍需 FP16 计算。SmoothQuant 使 W8A8 成为可能——权重和激活都是 INT8，享受 Tensor Core INT8 算力加速。

  核心方法：将量化难度从激活值转移到权重，具体步骤如下：
  1. 统计每通道激活的最大值 s_j = max(|X_j|)^α / max(|W_j|)^(1-α)
  2. 平滑：W'_j = W_j / s_j,  X'_j = X_j × s_j
  3. 平滑后：W' 的通道更均匀（便于量化），X' 的异常值被抑制
  4. 量化：W'_int8 = quant(W'), X'_int8 = quant(X')

  α 参数（迁移强度）：α=0.5 是平衡点，典型值 0.5~0.8。

  优势：
  - W8A8：权重和激活都 INT8，推理用 INT8 GEMM（Tensor Core INT8 算力是 BF16 的 2x）
  - 无需 FP16 激活 → 减少内存带宽 → decode 加速
  - 精度损失小（< 1%）

  局限：INT8 精度不如 FP8（定点格式对异常值不友好），Hopper GPU 上 FP8 更优。

- **关联论文/模型**："SmoothQuant: Accurate and Efficient Post-Training Quantization for LLMs" (Xiao et al., 2022, arXiv:2211.10438)；MIT-Han-Lab
- **参考分析链接**：
  - https://arxiv.org/abs/2211.10438
  - https://github.com/mit-han-lab/smoothquant


### GGUF / llama.cpp Quantization

- **缩写**：GGUF
- **中文名称**：GGUF 量化格式
- **简短介绍**：llama.cpp 项目定义的模型文件格式，支持 2~8 bit 多种量化方案，专为 CPU/消费级 GPU 推理优化。可在普通笔记本上运行大模型，是本地部署最流行的格式。
- **详细介绍**：GGUF（GPT-Generated Unified Format）替代了旧的 GGML 格式：

  支持的量化方案（按精度从高到低）：
  | 量化名 | bit/权重 | 说明 |
  |--------|---------|------|
  | Q8_0 | 8 | INT8 + FP16 scale，几乎无损 |
  | Q6_K | 6 | 6-bit 混合，精度好 |
  | Q5_K_M | 5.5 | 5-bit 混合，精度较好 |
  | Q4_K_M | 4.5 | 4-bit 混合，主流选择 |
  | Q4_0 | 4 | 基础 4-bit，速度快 |
  | Q3_K_M | 3.5 | 3-bit 混合，精度有损 |
  | Q2_K | 2 | 极限压缩，明显有损 |

  特点：
  - 单文件存储权重 + 元数据（tokenizer、架构参数）
  - 支持 CPU (AVX2/NEON) + GPU (CUDA/Metal) 混合推理
  - 部分层可在 GPU 计算，其余在 CPU（offload 层数可控）
  - 内存映射加载，启动快

  使用场景：
  - MacBook (Metal) 上跑 70B 模型（Q4 需 ~35GB 内存）
  - 没有数据中心 GPU 的本地开发和测试
  - 边缘设备部署

  与 vLLM/FP8 的关系：GGUF 面向消费级硬件和 CPU 推理，vLLM/FP8 面向数据中心 GPU 高吞吐服务。两者互补，不竞争。

- **关联论文/模型**：llama.cpp (Georgi Gerganov)；GGUF 规范
- **参考分析链接**：
  - https://github.com/ggerganov/llama.cpp
  - https://docs.vllm.ai/en/latest/features/quantization/gguf/
  - https://github.com/ggml-org/llama.cpp/tree/master/gguf-py


### FP8 Quantization

- **缩写**：FP8
- **中文名称**：FP8 量化
- **简短介绍**：利用 Hopper GPU 原生 FP8 Tensor Core 的量化方案。相比 INT4/INT8 定点量化，FP8 浮点格式天然适配 LLM 激活的异常值分布，几乎无损且算力翻倍。正在取代 INT8 成为推理量化的首选。
- **详细介绍**：FP8 量化的两种模式：

  1. W8A8（权重+激活 FP8）：
     - 权重和激活都用 FP8（E4M3）
     - GEMM 用 FP8 Tensor Core（H100 算力 1979 TFLOPS，BF16 的 2x）
     - 需要校准确定 scale factor（per-tensor 或 per-channel）
     - 精度损失通常 < 0.5%

  2. 仅权重量化（W8A16）：
     - 权重 FP8，激活 BF16
     - 用于不支持 FP8 计算的 GPU（如 A100）
     - 显存减半，但计算无加速

  两种 FP8 编码格式（由 "FP8 Formats for Deep Learning" 定义）：
  - E4M3（4 位指数 + 3 位尾数）：更多尾数位，精度更高，用于权重/激活与前向推理
  - E5M2（5 位指数 + 2 位尾数）：更多指数位，动态范围更大，用于梯度/反向传播
  推理场景主要使用 E4M3；E5M2 因动态范围大，在训练的反向梯度中更常见。

  FP8 vs INT8：
  | 特性 | FP8 (E4M3) | INT8 |
  |------|-----------|------|
  | 格式 | 浮点（指数+尾数） | 定点 |
  | 异常值 | 天然适配（浮点有指数） | 需要缩放/校准 |
  | 精度 | 更好（LLM 激活） | 需 SmoothQuant 等辅助 |
  | 算力 | H100 原生 1979T | H100 原生 1979T |
  | 门槛 | 需 Hopper+ | A100 即可 |

  实践用法（vLLM）：
  # FP8 权重量化（无需校准）
  --quantization fp8
  # FP8 KV Cache（长上下文减半 KV 内存）
  --kv-cache-dtype fp8

  局限：需 Hopper (H100/H200) 或 Ada (RTX 4090/6000) 架构。A100 不支持 FP8 Tensor Core。

- **关联论文/模型**："FP8 Formats for Deep Learning" (2022)；vLLM FP8；TensorRT-LLM FP8；FlashAttention-3 FP8
- **参考分析链接**：
  - https://docs.vllm.ai/en/latest/features/quantization/fp8/
  - https://nvidia.github.io/TensorRT-LLM/reference/precision.html


### Quantization-Aware Training

- **缩写**：QAT
- **中文名称**：量化感知训练
- **简短介绍**：在训练过程中模拟量化效应（假量化），让模型在训练时就适应低精度表示。精度高于训练后量化（PTQ），但需要训练成本。
- **详细介绍**：QAT vs PTQ：

  PTQ（训练后量化）：训练完的 FP16/BF16 模型直接量化，无需再训练
  - 优点：快速，无需训练数据和资源
  - 缺点：低 bit（INT4 以下）精度损失明显

  QAT（量化感知训练）：在微调阶段插入假量化节点
  - 前向传播时模拟量化：x_fake = dequant(quant(x))，让模型"看到"量化误差
  - 反向传播用 STE（Straight-Through Estimator）绕过量化不可导问题
  - 模型学会调整权重以适应量化噪声

  QAT 流程：
  1. 从预训练 FP16 模型出发
  2. 插入假量化节点（目标精度如 INT4/INT8）
  3. 用训练数据继续训练几个 epoch
  4. 导出真正的量化模型

  适用场景：
  - 需要 INT4 以下极端量化且 PTQ 精度不够时
  - 部署在边缘设备（手机、嵌入式）有严格精度约束
  - 对精度要求极高的生成任务

  实践中：LLM 很少用 QAT（训练成本高），大多用 PTQ（GPTQ/AWQ/FP8）已足够。QAT 更多用于 CV 模型和移动端部署。

- **关联论文/模型**："Quantization and Training of Neural Networks for Efficient Integer-Arithmetic-Only Inference" (Jacob et al., 2018, arXiv:1712.05877)；LLM-QAT (arXiv:2207.04265)
- **参考分析链接**：
  - https://arxiv.org/abs/1712.05877


## 本篇小结

本篇 6 个量化术语沿"两条路线 + 一个趋势"归位：

- **GPTQ** 基于 Hessian 信息的逐列补偿，W4A16 模式下显存 ~25% BF16、精度损失 < 1%，局限是仅量化权重、激活仍需 FP16。
- **AWQ** 以"与大激活值对应的权重更敏感"为出发点，用 per-channel scale 缩放而非二阶补偿，比 GPTQ 更快、精度相当；W4A16 下与 GPTQ 并列最常用 INT4 方案。
- **SmoothQuant** 把量化难度从激活值转移到权重，使 W8A8 成为可能，享受 Tensor Core INT8 算力（BF16 的 2x）；但 INT8 定点格式对异常值不友好，Hopper GPU 上 FP8 更优。
- **GGUF** 是消费级路线的代表，Q2_K~Q8_0 单文件便携，MacBook Metal 上 Q4 跑 70B 约 35GB 内存；与 vLLM/FP8 数据中心路线互补而非竞争。
- **FP8 Quantization** 是数据中心推理的新首选：E4M3（前向/推理，更多尾数）与 E5M2（反向/梯度，更多指数）两种编码，浮点格式天然适配 LLM 激活异常值，H100 上算力 1979 TFLOPS（BF16 的 2x）；硬件门槛是 Hopper/Ada，A100 不支持 FP8 Tensor Core。
- **QAT** 精度高于 PTQ 但需训练成本，LLM 实践中很少用 QAT，PTQ（GPTQ/AWQ/FP8）已足够；QAT 主要用于 CV 模型与移动端部署。

两条路线的边界清晰：数据中心 GPU 上 FP8/INT8 量化服务高吞吐推理，消费级 CPU/Metal 上 GGUF 单文件服务本地部署；FP8 取代 INT8 的趋势由 LLM 激活异常值分布驱动，定点格式需 SmoothQuant 辅助，浮点格式原生适配。


本系列共 11 篇，本文是第 8 篇。
