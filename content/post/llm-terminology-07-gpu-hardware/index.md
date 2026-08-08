---
title: "LLM 术语全景图（七）：GPU 硬件与计算——从 Tensor Core 到 HBM"
date: 2026-07-27T10:30:00+08:00
slug: "llm-terminology-07-gpu-hardware"
draft: false
image: "07.png"
tags:
    - AI
    - LLM
    - GPU
    - Tensor Core
    - HBM
    - FP8
    - NVIDIA
categories:
    - AI
    - LLM术语全景图
---

> 这是「大模型训练与推理术语全景图」系列的第 7 篇，覆盖 GPU 硬件与计算的 8 个概念：Tensor Core、MMA/WGMMA、TMA、HBM、BF16、FP8、AllReduce/AllGather/ReduceScatter、NVLink/NVSwitch。
> 主要参考架构：NVIDIA Volta/Ampere/Hopper/Blackwell、CUTLASS、FlashAttention-3、NCCL

---

## 开篇：理解 GPU 硬件是优化 LLM 的前提

LLM 推理的两个阶段受限于完全不同的资源。prefill 一次性处理整段 prompt，矩阵乘法的 FLOPs 大、每个权重被反复使用，瓶颈落在 Tensor Core 算力上，属于 compute-bound；decode 逐 token 自回归生成，每生成 1 个 token 都要把全部模型权重和当前 KV Cache 从 HBM 读进 SM，FLOPs 很小但读取量极大，瓶颈落在 HBM 带宽上，属于 memory-bound。这两种瓶颈分别决定了优化方向：compute-bound 看算力（更低精度、更稠密的 Tensor Core 利用率），memory-bound 看带宽（更小的权重、更少的读取）。量化把权重从 BF16 压到 FP8/INT4，本质是减少单位计算的 HBM 读取量；MoE 每次只激活少数专家，本质是减少单次 decode 实际需要读取的权重。所有主流 LLM 优化都能归到"降读取"或"降计算"这两条路径上，而判断走哪条路径的前提，是先看清 GPU 的算力墙和带宽墙分别在哪。本篇按硬件层次梳理 8 个术语：从计算单元（Tensor Core、MMA/WGMMA、TMA）、到内存与数据格式（HBM、BF16、FP8）、再到多卡互连（AllReduce 系原语、NVLink/NVSwitch）。


## GPU 硬件与计算（GPU Hardware & Compute）


### Tensor Core

- **缩写**：—
- **中文名称**：张量核心
- **简短介绍**：NVIDIA GPU 上专门用于矩阵乘法（GEMM）的硬件单元，单个时钟周期可完成一个矩阵乘加操作。相比 CUDA Core 做标量乘法，Tensor Core 吞吐量高出一个数量级，是 LLM 计算的核心引擎。
- **详细介绍**：Tensor Core 的计算能力：

  CUDA Core：每周期 1 个 FP32 FMA（fused multiply-add）
  Tensor Core：每周期一个矩阵乘加，如 D = A × B + C（A/B/C/D 是小矩阵块）

  各代 Tensor Core 支持的精度：
  | GPU 架构 | Tensor Core 代 | 支持精度 |
  |----------|---------------|---------|
  | Volta (V100) | 第 1 代 | FP16 |
  | Ampere (A100) | 第 3 代 | FP16, BF16, TF32, INT8, INT4 |
  | Hopper (H100) | 第 4 代 | FP16, BF16, TF32, FP8(E4M3/E5M2), INT8 |
  | Blackwell (B200) | 第 5 代 | FP4, FP6, FP8, BF16, FP16 |

  对 LLM 的影响：
  - 训练：BF16 是主力精度（Tensor Core 原生加速，动态范围大）
  - 推理：FP8（Hopper）和 FP4（Blackwell）大幅提升吞吐
  - A100-SXM4-40GB：第 3 代 Tensor Core，BF16 算力 312 TFLOPS

  MMA 指令：Tensor Core 通过 MMA (Matrix Multiply-Accumulate) 指令编程。CUTLASS 和 Triton 库在底层调用 MMA 指令实现高效 GEMM。

- **关联论文/模型**：NVIDIA Volta 架构白皮书 (2017)；CUTLASS 库；Triton 编译器
- **参考分析链接**：
  - https://docs.nvidia.com/cutlass/latest/media/docs/cpp/cute/00_quickstart.html
  - https://developer.nvidia.com/blog/cutlass-linear-algebra-cuda/


### Matrix Multiply-Accumulate

- **缩写**：MMA / WGMMA
- **中文名称**：矩阵乘加指令
- **简短介绍**：驱动 Tensor Core 的底层指令。MMA 是 Ampere 及之前架构的同步指令，WGMMA (Warp Group MMA) 是 Hopper 架构引入的异步指令，支持 TMA 加载数据，吞吐量大幅提升。
- **详细介绍**：MMA vs WGMMA：

  MMA（Ampere 及之前）：
  - 同步执行，线程束(warp)级
  - 数据需先加载到寄存器，再调用 MMA
  - 矩阵块较小（如 16×8×16 for FP16）
  - 数据加载和计算串行，有等待空闲

  WGMMA（Hopper）：
  - 异步执行，线程束组(warp group, 4 warps)级
  - 可直接从共享内存(shared memory)异步执行矩阵乘加
  - 矩阵块更大（如 64×128×16 for BF16, 128×256×32 for FP8）
  - 与 TMA 配合：TMA 异步加载数据到共享内存，WGMMA 异步从共享内存计算
  - 计算和数据加载可重叠（double buffer），接近峰值算力

  编程接口：
  - PTX 汇编：mma.sync / wgmma.mma_async 指令
  - CUTLASS 3.x：Cute 库封装 WGMMA
  - Triton：编译器自动选择 MMA/WGMMA

  对 LLM 的影响：WGMMA 使 H100 的 FP8 算力达到近 2000 TFLOPS（理论峰值），是 A100 BF16 的 ~6x。Flash Attention-3 专门利用 WGMMA + TMA 实现 FP8 注意力。

- **关联论文/模型**：NVIDIA Hopper 架构白皮书 (2022)；CUTLASS 3.x；FlashAttention-3
- **参考分析链接**：
  - https://docs.nvidia.com/cuda/parallel-thread-execution/
  - https://github.com/NVIDIA/cutlass


### Tensor Memory Accelerator

- **缩写**：TMA
- **中文名称**：张量内存加速器
- **简短介绍**：Hopper 架构引入的专用硬件单元，用于在 HBM 和共享内存之间高效异步搬运多维张量。替代手动内存管理，与 WGMMA 配合实现计算-通信重叠。
- **详细介绍**：TMA 解决的问题：

  传统方式：程序员用 global memory load 指令逐元素/逐块加载数据到寄存器，再存储到共享内存。线程开销大，带宽利用率低。

  TMA 方式：
  1. 预设张量描述符（descriptor）：定义 HBM 中的张量形状、跨度、维度
  2. 单条指令发起异步拷贝：整个多维切片从 HBM → 共享内存
  3. 不占用线程，由专用 DMA 引擎执行
  4. 通过 mbarrier（内存屏障）同步完成

  TMA 优势：
  - 异步：不占用计算线程，与 WGMMA 计算重叠
  - 多维：原生支持 1D~5D 张量拷贝，无需手动算地址
  - 高带宽：接近 HBM 峰值带宽（H100: 3.35 TB/s）
  - 跨节点：直接支持多卡共享内存间拷贝（配合 NVSwitch）

  与 Flash Attention-3 的关系：FA3 的核心优化之一就是用 TMA 替代 FA2 中的手动数据加载，实现 prefetch + 计算重叠，FP8 注意力吞吐提升 ~75%。

  Blackwell (B200) 进一步增强 TMA，支持 FP4 精度张量搬运。

- **关联论文/模型**：NVIDIA Hopper 架构白皮书 (2022)；FlashAttention-3 (arXiv:2407.08608)；CUTLASS 3.x
- **参考分析链接**：
  - https://docs.nvidia.com/cuda/parallel-thread-execution/#tensor-memory-accelerator
  - https://github.com/Dao-AILab/flash-attention


### High Bandwidth Memory

- **缩写**：HBM
- **中文名称**：高带宽内存
- **简短介绍**：GPU 上的高带宽显存，通过 3D 堆叠 DRAM 实现远超普通 DDR 的带宽。LLM 推理的主要瓶颈往往不是算力而是 HBM 带宽（memory-bound）。
- **详细介绍**：各代 HBM 参数：

  | HBM 代 | GPU | 带宽 | 容量 |
  |--------|-----|------|------|
  | HBM2 | V100 | 0.9 TB/s | 32GB |
  | HBM2e | A100 80GB | 2.0 TB/s | 80GB |
  | HBM3 | H100 SXM5 | 3.35 TB/s | 80GB |
  | HBM3e | B200 | 8 TB/s | 192GB |

  Memory-bound vs Compute-bound：

  LLM 推理的 decode 阶段是典型的 memory-bound：
  - 每生成 1 个 token，需从 HBM 读取全部模型权重 + KV Cache
  - 计算量小（1 个 token × 全部权重），但读取量大
  - 瓶颈 = 权重 / HBM 带宽，而非算力

  LLM 推理的 prefill 阶段是 compute-bound：
  - 处理整段 prompt，矩阵乘法计算量大
  - 瓶颈 = FLOPs / Tensor Core 算力

  量化算例：decode 每生成 1 token 需读取全部模型权重（70B = 140GB BF16），H100 3.35 TB/s → ~42ms → ~24 token/s。这就是 decode 阶段的天花板——单卡 H100 跑 70B BF16 的理论极限约 24 token/s，实际还要扣 KV Cache 读取和算力开销。

  优化启示：
  - 量化（INT4/FP8）减少权重大小 → 减少读取量 → 加速 decode
  - Flash Attention 减少 HBM 读写 → 加速注意力
  - MoE 只激活部分专家 → 减少实际读取的权重

  带宽利用率：高效实现（如 Flash Attention）可达 HBM 峰值带宽的 70-85%。

- **关联论文/模型**：NVIDIA 各架构白皮书；HBM 标准 (JEDEC)
- **参考分析链接**：
  - https://www.nvidia.com/en-us/data-center/h100/
  - https://arxiv.org/abs/2205.14135 (Flash Attention IO 复杂度分析)


### Bfloat16

- **缩写**：BF16
- **中文名称**：Brain 浮点 16 位
- **简短介绍**：Google Brain 团队设计的 16 位浮点格式，与 FP16 相同的总位数但指数位更多（8 位指数 + 7 位尾数），动态范围与 FP32 相同，训练稳定性远优于 FP16。现代 LLM 训练和推理的默认精度。
- **详细介绍**：BF16 vs FP16 vs FP32：

  | 格式 | 总位数 | 指数 | 尾数 | 动态范围 | 精度 |
  |------|--------|------|------|----------|------|
  | FP32 | 32 | 8 | 23 | ±3.4e38 | 高 |
  | FP16 | 16 | 5 | 10 | ±65504 | 中 |
  | BF16 | 16 | 8 | 7 | ±3.4e38 | 低（同 FP32 范围）|

  BF16 的优势：
  - 与 FP32 相同的指数位（8 位），动态范围完全一致
  - 训练时无需 loss scaling（FP16 需要，否则梯度下溢）
  - 与 FP32 互转简单：BF16 → FP32 只需右侧补零
  - Ampere+ Tensor Core 原生支持 BF16

  实践用法：
  - 训练：权重和激活用 BF16，主权重和优化器状态用 FP32（混合精度训练）
  - 推理：BF16 是无损推理的默认精度，权重占 FP32 的一半
  - KV Cache：通常用 BF16 存储

  注意：BF16 尾数只有 7 位（FP16 有 10 位），精度略低。但 LLM 训练对动态范围更敏感，BF16 是更好的选择。

- **关联论文/模型**：Google Brain (2018)；Ampere/Hopper/Blackwell Tensor Core 原生支持
- **参考分析链接**：
  - https://en.wikipedia.org/wiki/Bfloat16_floating-point_format


### FP8

- **缩写**：FP8
- **中文名称**：8 位浮点
- **简短介绍**：Hopper 架构原生支持的 8 位浮点格式，有两种变体 E4M3（4 位指数 + 3 位尾数）和 E5M2（5 位指数 + 2 位尾数）。相比 BF16 算力翻倍、显存减半，是推理量化和训练加速的新标准。
- **详细介绍**：FP8 的两种格式：

  | 格式 | 指数 | 尾数 | 动态范围 | 精度 | 适用场景 |
  |------|------|------|----------|------|----------|
  | E4M3 | 4 | 3 | ±448 | 较高 | 前向传播、权重、激活 |
  | E5M2 | 5 | 2 | ±57344 | 较低 | 反向梯度（需要更大范围）|

  混合精度 FP8 训练：
  - 前向：权重和激活用 E4M3
  - 反向：梯度用 E5M2
  - 主权重和优化器仍用 FP32/BF16

  Hopper 算力对比：
  | 精度 | H100 算力 (TFLOPS) | 相对 BF16 |
  |------|-------------------|----------|
  | BF16 | 989 | 1x |
  | FP8 | 1979 | 2x |

  推理中的 FP8：
  - 权重 FP8：显存减半（相比 BF16），decode 阶段读取量减半 → 吞吐近翻倍
  - KV Cache FP8：长上下文场景的 KV Cache 内存减半
  - 几乎无损：FP8 量化精度损失通常 < 1%（比 INT8 更好，因为浮点格式天然适配异常值）

  与 INT8 的区别：FP8 是浮点格式，有指数和尾数，能更好地表示大动态范围的 LLM 激活值。INT8 是定点格式，需要校准和缩放因子。

  Blackwell (B200) 进一步支持 FP4 和 FP6，算力可达 10 PFLOPS 级别。

- **关联论文/模型**："FP8 Formats for Deep Learning" (NVIDIA/ARM/Intel, 2022)；Hopper 架构；FlashAttention-3 (FP8 attention)；vLLM FP8 推理
- **参考分析链接**：
  - https://docs.nvidia.com/deeplearning/performance/mixed-precision-training/
  - https://docs.vllm.ai/en/latest/features/quantization/fp8/


### AllReduce / AllGather / ReduceScatter

- **缩写**：—
- **中文名称**：集合通信原语
- **简短介绍**：多 GPU 通信的三个基础原语。AllReduce 求和并广播结果，AllGather 收集各卡数据拼成完整结果，ReduceScatter 求和后按分片分发。是 TP/PP/DP/FSDP 的通信基础。
- **详细介绍**：三种通信原语：

  AllReduce（TP/DP 用）：
  - 各卡有一个相同大小的张量，求和后所有卡获得相同结果
  - Ring AllReduce：通信量 = 2 × (N-1)/N × tensor_size（最优）
  - 用途：TP 中 attention/FFN 后的梯度同步；DP 中梯度同步

  AllGather（FSDP/ZeRO-3 前向用）：
  - 各卡有不同的分片，收集拼成完整张量，所有卡获得相同完整结果
  - 通信量 = (N-1)/N × tensor_size
  - 用途：FSDP 前向恢复完整参数；CP 中收集各段 KV

  ReduceScatter（FSDP/ZeRO-3 反向用）：
  - 各卡有完整张量，求和后按分片分发——每卡只得到一个分片
  - 通信量 = (N-1)/N × tensor_size
  - 用途：FSDP 反向后分片同步梯度

  关系：AllReduce = ReduceScatter + AllGather（先求和分片，再收集完整）

  通信库：
  - NCCL (NVIDIA Collective Communications Library)：GPU 集合通信标准库
  - 依赖 NVLink/NVSwitch 拓扑，跨节点走 InfiniBand/RoCE

  通信优化：
  - 计算-通信重叠：在前向计算时同时启动下一步的通信
  - 梯度分桶(bucketing)：将梯度分成多个 bucket 分别通信，重叠更充分
  - 通信压缩：梯度 BF16/FP8 传输减少带宽

- **关联论文/模型**：NCCL (NVIDIA)；Horovod；Megatron-LM 通信优化
- **参考分析链接**：
  - https://docs.nvidia.com/deeplearning/nccl/
  - https://docs.pytorch.org/docs/stable/distributed.html


### NVLink / NVSwitch

- **缩写**：—
- **中文名称**：NVLink 互连 / NVSwitch 交换
- **简短介绍**：NVIDIA 的高速 GPU 互连技术。NVLink 是点对点直连（单链路 50-100 GB/s），NVSwitch 是全互连交换芯片（所有 GPU 间全带宽）。是 TP 等高通信量并行的硬件基础。
- **详细介绍**：NVLink 各代带宽：

  | NVLink 代 | GPU | 单链路带宽 | 总带宽 |
  |-----------|-----|-----------|--------|
  | NVLink 2 | V100 | 25 GB/s | 300 GB/s |
  | NVLink 3 | A100 | 50 GB/s | 600 GB/s |
  | NVLink 4 | H100 | 50 GB/s | 900 GB/s |
  | NVLink 5 | B200 | 100 GB/s | 1.8 TB/s |

  与 PCIe 对比：
  - PCIe 5.0：~64 GB/s（单向），共享带宽
  - NVLink 4：900 GB/s（总），点对点专用
  - NVLink 比 PCIe 高 10-15 倍

  NVSwitch：
  - 专用 ASIC 交换芯片，使节点内所有 GPU 全互连（而非点对点）
  - 8× H100 + NVSwitch：任何两卡间 900 GB/s 全带宽
  - DGX H100/H200 节点标配

  对并行策略的影响：
  - TP：通信量大（每层 2 次 AllReduce），必须在节点内（NVLink 域），跨节点 TP 性能急剧下降
  - PP：通信量小（只传隐状态），可跨节点（InfiniBand）
  - EP：All-to-All 通信，最好在节点内或高带宽网络
  - DP/FSDP：梯度同步通信量中等，可跨节点

  实践规则：TP ≤ 节点内 GPU 数（如 TP=8 for 8×H100），DP/PP 可跨节点。

- **关联论文/模型**：NVIDIA NVLink 白皮书；DGX H100/H200 架构
- **参考分析链接**：
  - https://www.nvidia.com/en-us/data-center/nvlink/


## 本篇小结

本篇 8 个 GPU 硬件术语沿"算力墙 vs 带宽墙"两条主线归位：

- **Tensor Core** 是 GPU 上专门做矩阵乘加的硬件单元，每周期完成一个小矩阵块 D = A×B + C，相比 CUDA Core 的标量 FMA 高出一个数量级；各代支持的精度逐步扩展（Volta FP16 → Ampere +BF16/TF32/INT8/INT4 → Hopper +FP8 → Blackwell +FP4/FP6），A100-SXM4-40GB 的 BF16 算力为 312 TFLOPS。
- **MMA / WGMMA** 是驱动 Tensor Core 的底层指令。MMA 是 Ampere 及之前的同步 warp 级指令，数据加载与计算串行；WGMMA 是 Hopper 的异步 warp group 级指令，可直接从共享内存异步执行矩阵乘加，并与 TMA 配合实现计算-加载重叠。WGMMA 使 H100 的 FP8 算力达到近 2000 TFLOPS，约为 A100 BF16 的 6x；Flash Attention-3 专门利用 WGMMA + TMA 实现 FP8 注意力。
- **TMA** 是 Hopper 引入的张量内存加速器，单条指令即可发起异步拷贝、把整个多维切片从 HBM 搬到共享内存，不占用计算线程。FA3 用 TMA 替代 FA2 中的手动数据加载，实现 prefetch + 计算重叠，FP8 注意力吞吐提升约 75%。
- **HBM** 是 GPU 上的高带宽显存，带宽从 V100 的 0.9 TB/s 增长到 B200 的 8 TB/s。LLM 推理 decode 阶段是 memory-bound：每生成 1 token 需读取全部模型权重，70B BF16 = 140GB，H100 3.35 TB/s → ~42ms → ~24 token/s 的理论上限。prefill 阶段则是 compute-bound，瓶颈在 Tensor Core 算力。
- **BF16** 是 16 位浮点格式，8 位指数 + 7 位尾数，与 FP32 指数位相同、动态范围完全一致，训练时无需 loss scaling，是现代 LLM 训练和推理的默认精度。
- **FP8** 是 Hopper 原生支持的 8 位浮点格式，有 E4M3（前向/权重/激活）和 E5M2（反向梯度）两种变体；H100 上 BF16 为 989 TFLOPS、FP8 为 1979 TFLOPS，正好 2x。Blackwell B200 进一步支持 FP4 和 FP6，算力可达 10 PFLOPS 级别。
- **AllReduce / AllGather / ReduceScatter** 是多 GPU 集合通信的三个基础原语：AllReduce 求和并广播、AllGather 收集拼完整、ReduceScatter 求和后按分片分发；AllReduce = ReduceScatter + AllGather。它们是 TP/PP/DP/FSDP 的通信基础，由 NCCL 实现，底层依赖 NVLink/NVSwitch 拓扑。
- **NVLink / NVSwitch** 是 NVIDIA 的高速 GPU 互连：NVLink 是点对点直连，NVSwitch 是全互连交换芯片。NVLink 总带宽从 V100 的 300 GB/s 增长到 B200 的 1.8 TB/s，比 PCIe 5.0 高 10-15 倍。TP 通信量大必须在 NVLink 域内，DP/PP 可跨节点。

一句话收束：LLM 的所有主流优化都能归到"降读取"（量化、MoE、Flash Attention）或"降计算"（更低精度、更稠密的 Tensor Core 利用率）两条路径上；判断走哪条路径，前提是先看清算力墙和带宽墙分别在哪——这正是本篇 8 个术语要回答的问题。


本系列共 11 篇，本文是第 7 篇。
