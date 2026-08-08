---
title: "LLM 术语全景图（九）：参数高效微调——LoRA 与 QLoRA"
date: 2026-07-27T11:00:00+08:00
slug: "llm-terminology-09-peft-lora"
draft: false
image: "09.png"
tags:
    - AI
    - LLM
    - LoRA
    - QLoRA
    - PEFT
    - Fine-tuning
categories:
    - AI
    - LLM术语全景图
---


> 这是「大模型训练与推理术语全景图」系列的第 9 篇，覆盖参数高效微调（Parameter-Efficient Fine-Tuning）的 3 个概念：LoRA、QLoRA、PEFT。
> 主要参考模型/框架：HuggingFace PEFT、bitsandbytes、vLLM

---

## 开篇：PEFT 把全量微调的显存墙拆掉

全量微调大模型的显存成本几乎不可承受：65B 模型全量微调需约 780GB 显存，对应 40+ 张 A100， optimizer state 占了大头。参数高效微调（PEFT）只训练少量额外参数，绕开这堵墙，其中 LoRA 的低秩适配是核心思想——在冻结的预训练权重旁加一对小矩阵 A、B，只训练它们。QLoRA 进一步把篇 8 的量化技术用到微调场景：QLoRA = NF4 量化 + LoRA，先把基座量化到 4-bit 冻结，再在上面训 LoRA，把 65B 微调显存从 ~780GB 压到 ~33GB，单张 A100 80GB 即可。本篇沿"LoRA 原理 → QLoRA 量化叠加 → PEFT 方法谱系"三个概念展开，点出与篇 8 量化方法的衔接。


## 参数高效微调（Parameter-Efficient Fine-Tuning）

> 全量微调大模型成本高昂。参数高效微调（PEFT）只训练少量额外参数，达到接近全量微调的效果。


### Low-Rank Adaptation

- **缩写**：LoRA
- **中文名称**：低秩适配
- **简短介绍**：在冻结的预训练权重旁添加低秩分解的适配矩阵（A×B），只训练 A 和 B（参数量 << 原始权重），达到接近全量微调的效果。最流行的 PEFT 方法。
- **详细介绍**：LoRA 的数学形式：

  原始权重 W (d×d)，LoRA 添加：ΔW = B × A
  - A: r×d (降维), B: d×r (升维), r << d (通常 r=8/16/64)
  - 前向：y = W·x + B·A·x = (W + ΔW)·x
  - 训练时 W 冻结，只训练 A 和 B

  参数量对比（d=4096, r=8）：
  - 全量微调：4096 × 4096 = 16.7M 参数/层
  - LoRA：4096 × 8 + 8 × 4096 = 65K 参数/层（减少 256x）

  初始化技巧：A 用随机高斯初始化，B 用零初始化 → 训练初期 ΔW=0，不改变原模型行为。

  推理时合并：W_merged = W + B×A，合并后无额外推理开销（与全量微调相同速度）。

  LoRA 的应用位置：通常加在 Attention 的 Q/V 投影上（最有效），也可加在 K/V 和 FFN。

  Alpha 缩放：ΔW = (α/r) × B×A，α 控制 LoRA 更新的强度，通常 α=2r。

  多 LoRA 服务：一个基座模型 + 多个 LoRA 适配器，可在推理时热切换（vLLM 支持多 LoRA 服务）。

- **关联论文/模型**："LoRA: Low-Rank Adaptation of Large Language Models" (Hu et al., 2021, arXiv:2106.09685)；PEFT 库 (HuggingFace)；vLLM 多 LoRA
- **参考分析链接**：
  - https://arxiv.org/abs/2106.09685
  - https://huggingface.co/docs/peft/en/package_reference/lora


### Quantized LoRA

- **缩写**：QLoRA
- **中文名称**：量化低秩适配
- **简短介绍**：LoRA + 量化的组合。将基座模型量化到 4-bit (NF4) 冻结，只在量化模型上训练 LoRA 适配器。使在单张消费级 GPU 上微调 70B 模型成为可能。
- **详细介绍**：QLoRA 的关键创新：

  1. NF4 (NormalFloat 4-bit) 量化：
     - 针对 LLM 权重正态分布设计的 4-bit 量化格式
     - 量化分位数与正态分布的分位数对齐，信息损失最小
     - 比 INT4 精度更高

  2. 双重量化（Double Quantization）：
     - 量化常量（scale/zero-point）本身也量化（8-bit）
     - 进一步节省显存（每参数省 ~0.4 bit）

  3. 分页优化器（Paged Optimizer）：
     - 用 PagedAttention 同款技术管理优化器状态
     - 显存峰值时自动将优化器状态卸载到 CPU

  效果：
  - 65B 模型全量微调需 ~780GB 显存（40+ 张 A100）
  - QLoRA 微调 65B 仅需 ~33GB（单张 A100 80GB 足够）
  - 精度接近全量 BF16 微调（差距 < 1%）

  训练流程：
  1. 加载 4-bit NF4 量化模型（冻结）
  2. 添加 LoRA 适配器（FP16 训练）
  3. 前向：4-bit 反量化 → 计算 → LoRA 适配
  4. 反向：梯度只流过 LoRA 参数

  实践工具：HuggingFace PEFT + bitsandbytes

- **关联论文/模型**："QLoRA: Efficient Finetuning of Quantized LLMs" (Dettmers et al., 2023, arXiv:2305.14314)；bitsandbytes；HuggingFace PEFT
- **参考分析链接**：
  - https://arxiv.org/abs/2305.14314
  - https://huggingface.co/blog/4bit-transformers-bitsandbytes


### Parameter-Efficient Fine-Tuning

- **缩写**：PEFT
- **中文名称**：参数高效微调
- **简短介绍**：一类只训练模型少量参数（而非全量参数）的微调方法总称。包括 LoRA、Adapter、Prefix Tuning、Prompt Tuning 等。HuggingFace PEFT 库统一了这些方法的接口。
- **详细介绍**：PEFT 方法谱系：

  | 方法 | 额外参数位置 | 参数量占比 | 推理开销 |
  |------|-------------|-----------|---------|
  | LoRA | 权重低秩分解 | 0.1~1% | 可合并，无额外开销 |
  | Adapter | 层后插入小网络 | 1~5% | 增加一层计算 |
  | Prefix Tuning | KV 前加可学习前缀 | 0.1% | 增加 prefix 长度 |
  | Prompt Tuning | 输入嵌入加可学习向量 | <0.01% | 增加输入长度 |
  | P-Tuning v2 | 每层加 prefix | 0.1~1% | 同 prefix tuning |

  LoRA 是当前最主流的选择（可合并、无推理开销、效果好）。

  PEFT 的核心价值：
  - 单卡可微调大模型（QLoRA: 单卡 70B）
  - 多任务：一个基座 + 多个 LoRA 适配器
  - 存储高效：LoRA 适配器仅几十 MB
  - 可组合：多个 LoRA 可叠加/混合

- **关联论文/模型**："PEFT: Parameter-Efficient Fine-Tuning" (HuggingFace)；LoRA、Adapter、Prefix Tuning 系列论文
- **参考分析链接**：
  - https://huggingface.co/docs/peft/
  - https://github.com/huggingface/peft


## 本篇小结

本篇 3 个 PEFT 术语沿"低秩适配 → 量化叠加 → 方法谱系"归位：

- **LoRA** 用低秩分解 ΔW = B×A（A: r×d, B: d×r, r << d）把可训参数压到全量的 0.1~1%，d=4096/r=8 时 65K vs 16.7M（减少 256x）；初始化技巧（A 随机高斯、B 零初始化）保证训练初期 ΔW=0；推理时合并 W_merged = W + B×A 后无额外开销，这是相对 Adapter 的关键优势；α=2r 缩放控制更新强度；vLLM 支持多 LoRA 热切换。
- **QLoRA** 是篇 8 量化方法在微调场景的落地：NF4（针对权重正态分布设计的 4-bit）+ 双重量化 + 分页优化器三个创新，把 65B 全量微调 ~780GB 压到 ~33GB 单卡 A100 80GB，精度差距 < 1%；流程是 4-bit NF4 冻结 + LoRA FP16 训练，工具链 HuggingFace PEFT + bitsandbytes。
- **PEFT** 是方法总称，谱系表中 LoRA 凭"可合并、无推理开销、效果好"成为当前最主流选择，其余 Adapter/Prefix/Prompt/P-Tuning v2 各有参数位置与推理开销取舍。

QLoRA 正是篇 8 量化与篇 9 PEFT 的结合点：把 NF4 量化技术用在微调而非推理，让单卡微调大模型从奢望变为例行。

本系列共 11 篇，本文是第 9 篇。系列导读见：[LLM 术语全景图：系列导读](/p/llm-terminology-series-guide/)
