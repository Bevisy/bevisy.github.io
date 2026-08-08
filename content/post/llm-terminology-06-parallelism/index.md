---
title: "LLM 术语全景图（六）：并行策略——多维并行的组合"
date: 2026-07-27T10:15:00+08:00
slug: "llm-terminology-06-parallelism"
draft: false
image: "06.png"
tags:
    - AI
    - LLM
    - Tensor Parallelism
    - Pipeline Parallelism
    - ZeRO
    - FSDP
    - MoE
categories:
    - AI
    - LLM术语全景图
---

> 这是「大模型训练与推理术语全景图」系列的第 6 篇，覆盖并行策略的 8 个概念：Data Parallelism、Tensor Parallelism、Pipeline Parallelism、Expert Parallelism、Context Parallelism、Sequence Parallelism、FSDP、ZeRO。
> 主要参考模型：DeepSeek V3、GLM-5、Llama 3、Megatron-LM、vLLM

---

## 开篇：并行策略是组合问题

大模型的多卡并行不是单一策略的选择题，而是多维度的组合题。各策略切分的对象不同：Data Parallelism（DP）切 batch，Tensor Parallelism（TP）在层内切权重矩阵，Pipeline Parallelism（PP）在层间切模型，Expert Parallelism（EP）切专家，Context/Sequence Parallelism（CP/SP）切序列。训练大 MoE 模型常同时叠加 TP+EP+PP+DP（或 FSDP/ZeRO 替代纯 DP），DeepSeek V3 即是 TP=8 + EP=32 + PP + DP 的组合；推理侧则以 TP+EP 为主，PP 较少使用（vLLM V1 才补上 PP 推理）。组合时关键约束是通信带宽层级——TP 每层都要 AllReduce，对带宽极敏感，必须放在 NVLink 域内（单节点，TP=2/4/8）；PP 的 stage 间只传隐状态、EP 的 All-to-All 通信量较小，可跨节点；DP 的 AllReduce 介于二者之间。一句话：高带宽通信配高频切分（TP 在节点内），低带宽链路配低频切分（PP/EP 跨节点）。本篇按切分对象逐条梳理这 8 个术语。


## 并行策略（Parallelism Strategies）

> LLM 训练和推理都需要多 GPU 协作。并行策略决定模型参数、梯度、优化器状态、激活值如何在多卡间分割与通信。


### Data Parallelism

- **缩写**：DP
- **中文名称**：数据并行
- **简短介绍**：每张 GPU 持有完整模型副本，将不同 batch 分配给不同 GPU 并行计算，反向传播后通过 AllReduce 同步梯度。最简单的并行方式，但要求单卡能放下完整模型。
- **详细介绍**：DP 的工作流程：

  1. 将 batch 均分到 N 张 GPU，每卡持有相同模型权重
  2. 各卡独立前向 + 反向计算，得到本地梯度
  3. AllReduce 通信：所有 GPU 梯度求和取平均
  4. 各卡用同步后的梯度更新本地权重

  通信模式：每步训练需 1 次 AllReduce（梯度同步），通信量 = 2 × model_size（ring-allreduce 发送+接收各一次）。

  纯 DP 的限制：单卡必须能放下完整模型 + 优化器状态 + 激活值。对 7B 模型，bf16 权重 14GB + AdamW 优化器状态 (fp32 m/v) 56GB + 激活值 → 需 80GB+ 显存。超过单卡容量就必须用 FSDP/ZeRO 或 TP/PP。

  与 FSDP/ZeRO 的关系：FSDP/ZeRO 是 DP 的改进版，将模型参数也分片到各卡，解决单卡放不下的问题。

- **关联论文/模型**：PyTorch DDP (DistributedDataParallel)；所有多卡训练的基础
- **参考分析链接**：
  - https://docs.pytorch.org/docs/stable/generated/torch.nn.parallel.DistributedDataParallel.html
  - https://docs.vllm.ai/en/latest/serving/parallelism_scaling.html


### Tensor Parallelism

- **缩写**：TP
- **中文名称**：张量并行
- **简短介绍**：将单个层的权重矩阵按行或列切分到多张 GPU 上，各卡并行计算矩阵乘法的不同部分，再通过 AllReduce/AllGather 拼接结果。实现层内并行，是 MoE 大模型推理的标配。
- **详细介绍**：TP 的两种切分方式：

  1. Column Parallelism（列并行）：W 按 output 维度切分
     - 各卡计算 Y_i = X · W_i（输入相同，权重不同）
     - 输出拼接需 AllGather
  2. Row Parallelism（行并行）：W 按 input 维度切分
     - 各卡计算 Y_i = X_i · W（输入不同，权重相同）
     - 输出求和需 AllReduce

  Transformer 层的 TP 策略（Megatron 方案）：
  - Attention：Q/K/V 投影用列并行（按 head 切分，天然并行），输出投影用行并行
  - FFN：第一个 Linear 列并行，第二个 Linear 行并行
  - 一层内只需 2 次 AllReduce（attention 后 + FFN 后）

  TP 的通信开销：每层 2 次 AllReduce，通信量 = 2 × hidden_size × seq_len × batch × 2 (send+recv)。

  TP 度数选择：通常 TP=2/4/8（受 NVLink 带宽限制，跨节点 TP 性能差）。推理时 TP=N 意味着 N 张卡共同服务一个请求。

  vLLM 推理：--tensor-parallel-size N

  与 EP 的关系：MoE 模型通常 TP + EP 组合——注意力用 TP，专家用 EP。

- **关联论文/模型**："Megatron-LM" (Shoeybi et al., 2019, arXiv:1909.08053)；DeepSeek V3 (TP=8)；GLM-5 (TP=8)；vLLM 推理标配
- **参考分析链接**：
  - https://docs.vllm.ai/en/latest/serving/parallelism_scaling.html
  - https://arxiv.org/abs/1909.08053


### Pipeline Parallelism

- **缩写**：PP
- **中文名称**：流水线并行
- **简短介绍**：将模型的不同层分配到不同 GPU 上，数据像流水线一样依次通过各卡。解决单卡放不下深层模型的问题，但有流水线气泡（bubble）开销。
- **详细介绍**：PP 的工作方式：

  将 L 层模型分到 N 个 stage（GPU），每个 stage 处理 L/N 层。
  GPU 0 (layer 0-15) → GPU 1 (layer 16-31) → ... → GPU N-1 (layer L-N*16 ~ L-1)

  问题：朴素 PP 下，GPU 0 计算 forward 时 GPU 1~N-1 空闲 → 气泡占比 1-1/N。

  GPipe 方案（微批次填充）：
  - 将一个 batch 分成 M 个 micro-batch
  - M 个 micro-batch 流水通过各 stage
  - 气泡占比 = (N-1) / (M+N-1)，M 越大气泡越小

  1F1B 调度（Megatron）：交替执行 1 个 forward + 1 个 backward，减少激活值内存峰值。

  PP 的通信：stage 间传递隐状态，通信量 = hidden_size × batch × seq_len，比 TP 小得多（适合跨节点）。

  PP 在推理中的应用：较少用于推理（推理无反向传播，气泡问题不同），主要用于训练。vLLM V1 已支持 PP 推理。

  注意：推测解码（Speculative Decoding）长期与 PP 不兼容，vLLM V1 近期才解决。

- **关联论文/模型**："GPipe" (Huang et al., 2019, arXiv:1811.06965)；Megatron-LM (1F1B 调度)；DeepSpeed
- **参考分析链接**：
  - https://docs.vllm.ai/en/latest/serving/parallelism_scaling.html
  - https://arxiv.org/abs/1811.06965


### Expert Parallelism

- **缩写**：EP
- **中文名称**：专家并行
- **简短介绍**：MoE 模型专用并行策略，将不同的专家（expert）分配到不同 GPU 上，每个 token 通过 All-to-All 通信路由到对应专家所在的卡上计算。
- **详细介绍**：EP 的核心流程：

  1. Router 在每卡本地计算路由（哪些 token 去哪些专家）
  2. All-to-All 通信：将 token 发送到目标专家所在 GPU
  3. 各卡上的专家计算 FFN
  4. All-to-All 通信：将结果送回原始卡
  5. Router 加权合并结果

  EP 的通信模式：2 次 All-to-All（dispatch + combine），通信量取决于 token 数和激活专家数。

  EP 度数：通常等于专家数或其因子。如 DeepSeek V3 有 256 个专家，EP=256 意味着每卡 1 个专家。实际中 EP=8/16/32 更常见（每卡多个专家）。

  TP + EP 组合（大 MoE 模型标配）：
  - Attention 层：TP 切分（按 head 分卡）
  - MoE 层：EP 切分（按 expert 分卡）
  - 示例：DeepSeek V3 训练用 TP=8 + EP=32 + DP=... + PP=...

  EP 的挑战：
  - 负载不均衡：某些专家被选中的 token 多，导致某些 GPU 过载
  - All-to-All 通信：跨节点延迟高，需要高效通信库（如 NCCL）
  - DeepSeek V3 的无辅助损失路由策略帮助均衡

- **关联论文/模型**："GShard" (Lepikhin et al., 2020, arXiv:2006.16668)；"Switch Transformer" (Fedus et al., 2021, arXiv:2101.03961)；DeepSeek V3 (技术报告)；GLM-5 (技术报告)
- **参考分析链接**：
  - https://docs.vllm.ai/en/latest/serving/parallelism_scaling.html
  - https://arxiv.org/abs/2101.03961


### Context Parallelism

- **缩写**：CP
- **中文名称**：上下文并行
- **简短介绍**：将输入序列按长度维度切分到多张 GPU 上，各卡并行处理序列的不同段落，通过环式通信交换 KV 实现完整注意力。用于超长上下文训练和推理。
- **详细介绍**：CP 解决的问题：

  长序列（128K~1M token）的注意力计算 O(n^2) 和 KV Cache 内存超出单卡容量。CP 将序列分段，多卡协作完成注意力。

  工作方式（Ring Attention / Ulysses 两种方案）：

  Ring Attention：
  1. 序列分成 N 段，每卡持有 1 段的 Q/K/V
  2. KV 在卡间环形传递，每卡依次与其他段的 K/V 做注意力
  3. N 步后完成完整注意力计算

  Ulysses（DeepSpeed Ulysses）：
  1. 序列按 head 维度分卡（不是按序列位置）
  2. 通过 All-to-All 将 Q/K/V 重排为每卡持有所有位置的部分 head
  3. 各卡独立计算部分 head 的完整注意力
  4. 再 All-to-All 重排回去

  使用 CP 的模型：DeepSeek V3（训练用 CP）、GLM-5（1M 上下文训练）、Kimi Linear（1M 上下文）。

  vLLM 推理中：CP 用于超长上下文推理（如 1M token），--context-parallel-size N。

- **关联论文/模型**："Ring Attention" (Liu et al., 2023, arXiv:2310.01889)；"DeepSpeed Ulysses" (Microsoft, 2023)；DeepSeek V3 训练
- **参考分析链接**：
  - https://docs.vllm.ai/en/latest/serving/parallelism_scaling.html
  - https://arxiv.org/abs/2310.01889


### Sequence Parallelism

- **缩写**：SP
- **中文名称**：序列并行
- **简短介绍**：将序列的激活值沿序列维度切分到多卡，主要用于减少 TP 中 LayerNorm 和 Dropout 的冗余计算和内存。与 CP 类似但侧重点不同——SP 是 TP 的辅助，CP 是独立的注意力并行。
- **详细介绍**：SP 的两种含义：

  1. Megatron SP（TP 辅助）：
     - TP 只切分 Q/K/V/FFN 矩阵，LayerNorm 和 Dropout 在所有 TP 卡上冗余计算
     - SP 将 LayerNorm/Dropout 的激活沿序列维度切分，各卡只处理部分 token
     - 在进入 TP 层前用 AllGather 恢复完整序列
     - 减少激活内存，但增加通信

  2. DeepSpeed Ulysses SP（= CP 的一种实现）：
     - 与上述 Megatron SP 不同，是独立的序列维度并行
     - 通过 All-to-All 重排 head 和 sequence 维度

  Megatron SP 与 CP 的区别：
  - SP：只切分非 TP 部分（LayerNorm/Dropout），注意力仍在单卡完整计算
  - CP：注意力本身也并行化，多卡协作完成完整注意力

  实践中 SP 通常与 TP 配合使用（TP+SP），而 CP 可独立使用。

- **关联论文/模型**："Megatron-LM" (arXiv:2104.04473, SP 部分)；DeepSpeed Ulysses
- **参考分析链接**：
  - https://github.com/NVIDIA/Megatron-LM


### Fully Sharded Data Parallelism

- **缩写**：FSDP
- **中文名称**：全分片数据并行
- **简短介绍**：DP 的进化版，将模型参数、梯度、优化器状态全部分片到各 GPU，计算时按需 AllGather 恢复完整参数，反向后 ReduceScatter 分片梯度。PyTorch 原生支持的大模型训练方案。
- **详细介绍**：FSDP 的工作流程：

  前向传播：
  1. 进入某层前，AllGather 从各卡收集该层完整参数
  2. 用完整参数计算前向
  3. 计算完后立即丢弃完整参数，只保留本卡分片（节省显存）

  反向传播：
  1. 再次 AllGather 恢复该层参数
  2. 计算梯度
  3. ReduceScatter 将梯度按分片同步到各卡
  4. 各卡用本地分片梯度更新本地分片的优化器状态

  显存节省：模型参数+梯度+优化器从全量 (16+16+32 = 64 bytes/param, fp32 AdamW) 降至 1/N。7B 模型在 N=8 卡上每卡仅需 ~8GB 而非 ~56GB。

  FSDP vs ZeRO-3：两者原理几乎相同，FSDP 是 PyTorch 原生实现，ZeRO-3 是 DeepSpeed 实现。FSDP 更易用，ZeRO 配置更灵活。

  FSDP 的分层分片（nested FSDP / per-layer sharding）：可按层粒度分片，进一步减少峰值显存（只需单层完整参数在内存中）。

  混合并行：FSDP + TP + PP 可组合使用。如 FSDP(dp=4) + TP(tp=8) + PP(pp=2) 用于超大模型。

- **关联论文/模型**："PyTorch FSDP" (Zhao et al., 2023, arXiv:2304.11277)；PyTorch 2.x 原生支持；Llama 3 训练
- **参考分析链接**：
  - https://docs.pytorch.org/docs/stable/fsdp.html
  - https://arxiv.org/abs/2304.11277


### Zero Redundancy Optimizer

- **缩写**：ZeRO
- **中文名称**：零冗余优化器
- **简短介绍**：DeepSpeed 提出的训练内存优化方案，逐步消除 DP 中的冗余：ZeRO-1 分片优化器状态，ZeRO-2 额外分片梯度，ZeRO-3 额外分片参数。FSDP 的理论基础。
- **详细介绍**：ZeRO 的三个阶段：

  基线（纯 DP）：每卡存储完整参数(P) + 梯度(G) + 优化器状态(O)
  - 以 7B 模型 fp32 AdamW 为例：P=28GB, G=28GB, O=56GB → 共 112GB/卡

  ZeRO-1：分片优化器状态 O → 每卡 1/N 的 O
  - 节省：56GB → 56/N GB。总显存 28+28+56/N

  ZeRO-2：额外分片梯度 G → 每卡 1/N 的 G 和 O
  - 节省：(28+56)/N。总显存 28+84/N
  - 反向后只同步本卡负责的参数的梯度

  ZeRO-3：额外分片参数 P → 每卡 1/N 的 P、G、O
  - 节省：全部 112/N。7B 模型 8 卡只需 14GB/卡
  - 前向/反向需 AllGather 恢复完整参数（= FSDP）

  通信代价：
  - ZeRO-1/2：与纯 DP 相同（1 次 AllReduce 梯度），无额外通信
  - ZeRO-3：每层 1 次 AllGather(前向) + 1 次 ReduceScatter(反向)，通信量增加

  ZeRO-Infinity：进一步将分片卸载到 CPU 内存和 NVMe SSD，突破 GPU 显存限制。

  实践选择：ZeRO-2 是性能和内存的最佳平衡点（无额外通信开销）。ZeRO-3 用于单卡放不下的超大模型。

- **关联论文/模型**："ZeRO: Memory Optimizations Toward Training Trillion Parameter Models" (Rajbhandari et al., 2020, arXiv:1910.02054)；DeepSpeed；FSDP 基于此设计
- **参考分析链接**：
  - https://www.deepspeed.ai/tutorials/zero/
  - https://arxiv.org/abs/1910.02054

## 本篇小结

本篇 8 个并行策略术语沿"切什么维度 + 内存如何分片"两条线归位：

- **DP** 每卡完整模型分不同数据，最简单但要求单卡放下完整模型；反向后 AllReduce 同步梯度。
- **TP** 层内切权重矩阵，各卡算矩阵乘法的不同部分再 AllReduce/AllGather 拼接；NVLink 域内有效，是 MoE 大模型推理标配。
- **PP** 层间切分，数据像流水线依次通过各卡，解决深模型单卡放不下的问题，但有流水线气泡开销；GPipe 和 1F1B 两种调度。
- **EP** MoE 专用，把不同专家分到不同 GPU，token 通过 All-to-All 通信路由到专家所在卡。
- **CP** 把输入序列按长度切分到多卡，各卡处理不同段落并通过环式通信交换 KV 实现完整注意力；用于超长上下文训练和推理。
- **SP** 切序列的激活值，主要用于减少 TP 中 LayerNorm/Dropout 的冗余计算；是 TP 的辅助，与 CP 的区别是 SP 依附于 TP、CP 是独立的注意力并行。
- **FSDP** DP 的进化版，参数/梯度/优化器状态全部分片，计算时按需 AllGather 恢复完整参数；PyTorch 原生支持的大模型训练方案。
- **ZeRO** DeepSpeed 的内存优化三阶段：ZeRO-1 分片优化器状态、ZeRO-2 加分片梯度、ZeRO-3 加分片参数（= FSDP）；ZeRO-2 是性能与内存的最佳平衡点（无额外通信），ZeRO-3 用于单卡放不下的超大模型。

组合关系：大模型训练标配 TP+EP+PP+DP/ZeRO 多维并行（TP 在 NVLink 域内、PP 跨节点、DP/ZeRO 切数据、EP 切专家），推理标配 TP+EP。FSDP 是 ZeRO-3 的工程实现，两者理论基础一致。


本系列共 11 篇，本文是第 6 篇。
