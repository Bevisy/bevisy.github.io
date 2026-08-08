---
title: "LLM 术语全景图（十一）：术语关系总览与核心趋势"
date: 2026-07-27T11:30:00+08:00
slug: "llm-terminology-11-overview-trends"
draft: false
image: "11.png"
tags:
    - AI
    - LLM
    - Architecture
    - Trends
categories:
    - AI
    - LLM术语全景图
---


> 这是「大模型训练与推理术语全景图」系列的第 11 篇（系列收尾），不引入新概念，而是把前 10 篇的 64 个术语放回它们的关系网络中：注意力机制的演进谱系、模型与术语的对照、并行策略的组合、量化方法的选型、GPU 精度与算力。最后给出 10 条核心趋势和参考资源索引。
> 主要参考来源：前 10 篇引用的全部论文与技术报告

---

## 开篇：把 64 个术语放回关系网络

本篇是系列收尾，不引入新概念，而是把前 10 篇的 64 个术语放回它们的关系网络中：注意力机制的演进谱系、模型与术语的对照、并行策略的组合、量化方法的选型、GPU 精度与算力。最后给出 10 条核心趋势和参考资源索引。


## 注意力机制演进关系图

```
MHA (原始 Transformer)
├── MQA (极端共享 K/V，1 组)
├── GQA (分组共享 K/V，当前主流)
│   └── + QK-Norm (稳定性增强)
├── MLA (低维压缩 K/V，DeepSeek 首创)
│   ├── + DSA (稀疏选择，DeepSeek V3.2)
│   │   └── + IndexShare (复用 indexer，GLM-5.2)
│   └── + CSA/HCA (序列压缩，DeepSeek V4)
├── CCA (压缩空间直接计算，ZAYA1)
├── Lightning Attention (线性注意力，MiniMax)
├── KDA (改进线性注意力，Kimi Linear)
├── SWA (固定窗口)
│   └── + Global (混合，Gemma 3/Laguna)
├── NoPE (无位置编码层)
└── Hybrid (以上混合，2025-2026 趋势)
    ├── 3:1 线性+全局 (Kimi Linear, Qwen3-Next)
    ├── 5:1 SWA+全局 (Gemma 3)
    └── CSA+HCA 交替 (DeepSeek V4)
```

## 模型与术语对照表

| 模型 | 注意力 | 架构 | 位置编码 | 后训练 | 特殊技术 |
|------|--------|------|----------|--------|----------|
| DeepSeek V3 | MLA | MoE (256+1) | RoPE | SFT + RL | MTP, 无辅助损失路由 |
| DeepSeek R1 | MLA | MoE (256+1) | RoPE | SFT + GRPO | RLVR, 蒸馏, "Aha Moment" |
| DeepSeek V4 | CSA + HCA | MoE + mHC | RoPE | — | mHC, Muon, 序列压缩 |
| GLM-5 | MLA + DSA | MoE (256+1) | RoPE | SFT + RL | DSA |
| GLM-5.2 | MLA + DSA + IndexShare | MoE (256+1) | RoPE | SFT + RL | IndexShare (indexer 复用 4 层) |
| Kimi K2 | MLA | MoE (384+1) | RoPE | SFT + RL | MuonClip |
| Kimi Linear | KDA + MLA (3:1) | MoE | NoPE (线性层) | — | KDA, 线性注意力 |
| MiniMax-01/M1 | Lightning Attention + Full | MoE (32) | RoPE | RL (CISPO) | 闪电注意力, 1M 上下文 |
| MiniMax M2 | Full Attention (回归) | MoE | RoPE | RL | 放弃线性注意力 |
| Qwen3 | GQA + QK-Norm | Dense / MoE | RoPE | SFT + DPO/GRPO | — |
| Qwen3-Next | Gated DeltaNet + Full (3:1) | MoE | NoPE (线性层) | — | 线性注意力混合 |
| Llama 3/4 | GQA | Dense / MoE | RoPE | SFT + RLHF | Chunked attention (Llama 4) |
| Gemma 3 | GQA + SWA (5:1) + QK-Norm | Dense | RoPE | SFT + RLHF | — |
| Gemma 4 | MQA + Cross-Layer KV | Dense / MoE | RoPE | — | KV 共享, Per-Layer Emb |
| GPT-4/o1 | (闭源) | (闭源) | (闭源) | RLHF | 推理模型 |
| Claude 3.5/4 | (闭源) | (闭源) | (闭源) | Constitutional AI | — |

注：DeepSeek V3/R1 技术报告未明确记载 QK-Norm，故注意力列只标 MLA。QK-Norm 的采用模型见篇 02。

## 并行策略关系图

```
训练并行
├── DP (数据并行) — 每卡完整模型，分数据
│   ├── ZeRO-1 (分片优化器状态)
│   ├── ZeRO-2 (+ 分片梯度)
│   ├── ZeRO-3 / FSDP (+ 分片参数)
│   │   └── ZeRO-Infinity (卸载到 CPU/NVMe)
│   └── 嵌套 FSDP (按层分片)
├── TP (张量并行) — 层内切分权重，NVLink 域
│   ├── Attention: Q/K/V 列并行 → 输出行并行
│   └── FFN: 第1层列并行 → 第2层行并行
├── PP (流水线并行) — 层间切分，可跨节点
│   ├── GPipe (微批次填充)
│   └── 1F1B (交替调度，省激活内存)
├── EP (专家并行) — MoE 专用，All-to-All 路由
├── CP (上下文并行) — 序列切分，超长上下文
│   ├── Ring Attention (环形 KV 传递)
│   └── Ulysses (All-to-All head 重排)
└── SP (序列并行) — TP 辅助，切分 LayerNorm/Dropout

推理并行（通常更简单）
├── TP (张量并行) — 多卡共服务一个请求
├── EP (专家并行) — MoE 推理标配
├── CP (上下文并行) — 1M token 超长上下文
└── 分离式 — Prefill 池 + Decode 池
```

## 量化方法对比

| 方法 | 精度 | 硬件要求 | 校准 | 精度损失 | 适用场景 |
|------|------|---------|------|---------|---------|
| GPTQ | INT4/INT8 | A100+ | 需要 | < 1% | 权重量化，通用 |
| AWQ | INT4/INT8 | A100+ | 需要 | < 1% | 权重量化，快于 GPTQ |
| SmoothQuant | INT8 | A100+ | 需要 | < 1% | W8A8，激活也量化 |
| FP8 | FP8 | Hopper+ | 可选 | < 0.5% | 推理首选，算力翻倍 |
| GGUF | 2-8 bit | CPU/Metal | 无需 | 可控 | 消费级硬件本地部署 |
| QAT | 任意 | 同训练 | 训练 | 最小 | 精度要求极高场景 |

## GPU 精度与算力对照

| 精度 | H100 算力 | B200 算力 | 用途 | 备注 |
|------|----------|----------|------|------|
| FP32 | 67 TFLOPS | — | 优化器状态 | 非 Tensor Core |
| TF32 | 495 TFLOPS | — | 兼容 FP32 | 自动降精度 |
| BF16 | 989 TFLOPS | 2.2 PFLOPS | 训练默认 | Tensor Core |
| FP16 | 989 TFLOPS | 2.2 PFLOPS | 推理/训练 | 需 loss scaling |
| FP8 | 1979 TFLOPS | 4.5 PFLOPS | 推理/训练 | Hopper 新增 |
| INT8 | 1979 TFLOPS | — | 推理 | 需校准 |
| FP4 | — | 9 PFLOPS | 推理 | Blackwell 新增 |


## 核心趋势总结

1. **KV Cache 是第一瓶颈**：MLA → DSA → CSA/HCA → 线性注意力，所有创新都围绕降低 KV Cache 成本
2. **混合架构成为主流**：纯完整注意力 → 3:1 / 5:1 混合，完整注意力成为稀缺资源
3. **MoE 普及**：大模型几乎全部 MoE 化，激活比持续下降（DeepSeek V4 ~4%）
4. **RL 驱动推理能力**：GRPO/RLVR 替代传统 RLHF，涌现推理行为
5. **训练优化器更新**：AdamW → Muon，收敛效率提升
6. **残差连接也在进化**：标准残差 → mHC 多残差流，信息通路更宽
7. **FP8 取代 INT8**：Hopper GPU 原生 FP8 支持，浮点量化几乎无损且算力翻倍
8. **推理分离式架构**：Prefill/Decode 分池部署，各阶段独立优化，吞吐提升 30-60%
9. **并行策略组合化**：大模型训练标配 TP+EP+PP+DP/ZeRO 多维并行，推理标配 TP+EP
10. **PEFT 成为微调标配**：LoRA/QLoRA 使单卡微调 70B 模型成为可能，多 LoRA 热切换服务


## 本篇小结

本篇是系列收尾，不引入新概念，而是把前 10 篇的 64 个术语放回关系网络：

- **演进谱系图**把注意力机制从 MHA 到 MLA、CSA/HCA、线性注意力、混合架构的分支与组合画在一棵树上，让前两篇的概念回到同一张图里。
- **模型与术语对照表**把 15 个代表模型按注意力、架构、位置编码、后训练、特殊技术五列对齐，读者可横向对照"谁用了什么"。
- **并行策略关系图**把训练侧的 DP/TP/PP/EP/CP/SP 与推理侧的 TP/EP/CP/分离式分别画出，呼应篇 6。
- **量化方法对比表**把 GPTQ/AWQ/SmoothQuant/FP8/GGUF/QAT 按精度、硬件、校准、精度损失、适用场景五列对齐，呼应篇 8。
- **GPU 精度与算力对照表**把 FP32 到 FP4 的算力与用途对齐，呼应篇 7。
- **核心趋势 10 条**浓缩全系列主线：KV Cache 是第一瓶颈、混合架构主流化、MoE 普及、RL 驱动推理、优化器更新、残差进化、FP8 取代 INT8、推理分离式架构、并行策略组合化、PEFT 成微调标配。

本篇的价值不在于新知识，而在于把分散在 10 篇里的术语重新编织成一张可查阅的关系网——单篇看细节，本篇看整体。


## 参考资源汇总

### 核心参考文章
- Sebastian Raschka "The Big LLM Architecture Comparison": https://magazine.sebastianraschka.com/p/the-big-llm-architecture-comparison
- Sebastian Raschka "A Visual Guide to Attention Variants": https://magazine.sebastianraschka.com/p/visual-attention-variants
- Sebastian Raschka "Recent Developments in LLM Architectures" (CSA/HCA/mHC): https://magazine.sebastianraschka.com/p/recent-developments-in-llm-architectures
- Sebastian Raschka "From DeepSeek V3 to V3.2": https://magazine.sebastianraschka.com/p/technical-deepseek
- Sebastian Raschka "GLM-5.2 IndexShare": https://sebastianraschka.com/blog/2026/glm-5-2-indexshare.html
- Sebastian Raschka "A Dream of Spring for Open-Weight LLMs" (MiniMax/Qwen/Ling): https://magazine.sebastianraschka.com/p/a-dream-of-spring-for-open-weight
- Sebastian Raschka "From GPT-2 to gpt-oss": https://magazine.sebastianraschka.com/p/from-gpt-2-to-gpt-oss-analyzing-the
- LLM Architecture Gallery: https://sebastianraschka.com/llm-architecture-gallery

### 核心视频
- "Hybrid Attention and Gated DeltaNet: How 2026 LLMs Actually Work": https://www.youtube.com/watch?v=SH6Cyhunl8s
- "LLM Architecture in 2026" (Sebastian Raschka 播客): https://www.youtube.com/watch?v=Y6APnyZT6XU
- Yannic Kilcher "GRPO Explained": https://www.youtube.com/watch?v=bAWV_yrqx4w

### 重要论文索引（arXiv 链接）
- "Attention Is All You Need" (Transformer 原始论文): https://arxiv.org/abs/1706.03762
- "RoPE / RoFormer" (旋转位置编码): https://arxiv.org/abs/2104.09864
- "YaRN" (上下文扩展): https://arxiv.org/abs/2309.00071
- "GQA" (分组查询注意力): https://arxiv.org/abs/2305.13245
- "FlashAttention" (闪存注意力): https://arxiv.org/abs/2205.14135
- "PagedAttention / vLLM" (分页注意力): https://arxiv.org/abs/2309.06180
- "Speculative Decoding" (推测解码): https://arxiv.org/abs/2211.17192
- "DPO" (直接偏好优化): https://arxiv.org/abs/2305.18290
- "DeepSeekMath / GRPO" (组相对策略优化): https://arxiv.org/abs/2402.03300
- "Hyper-Connections" (超连接，mHC 前身): https://arxiv.org/abs/2409.19606
- "CCA" (压缩卷积注意力): https://arxiv.org/abs/2510.04476

本系列共 11 篇，本文是第 11 篇。系列导读见：[LLM 术语全景图：系列导读](/p/llm-terminology-series-guide/)
