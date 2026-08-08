---
title: "推测解码技术全景对比与 DeepSpec 落地分析"
date: 2026-07-22T18:00:00+08:00
slug: "speculative-decoding-deepspec-analysis"
draft: false
image: "ae72ff10-6529-4be6-8ecb-333f95ccda2c.png"
tags:
    - AI
    - LLM
    - Speculative Decoding
    - DeepSpec
    - vLLM
    - SGLang
categories:
    - AI
---

> 项目：DeepSpec | 撰写日期：2026-07-22
> 目标：系统讲解推测解码技术优缺点，对比 DeepSpec 项目包含的三种方案（DSpark / DFlash / Eagle3）以及 DeepSeek MTP，并梳理各方案在 vLLM 和 SGLang 两大推理引擎中的落地情况（附 PR 链接）。

---

## 一、推测解码技术概述

### 1.1 核心思想

推测解码（Speculative Decoding）借鉴 CPU 投机执行的思想：先用一个「又快又小」的草稿模型（draft model）猜接下来的 γ 个 token，再让「又慢又准」的目标模型（target model）一次性验证这一整块。猜对的直接用，猜错的修正。

加速公式（DSpark 论文式 1）：

```
L = (T_draft + T_verify) / τ
```

- τ：每轮目标模型平均接受的 token 数（接受长度）
- T_draft：草稿模型生成花费
- T_verify：目标模型验证花费

加速只有三条路：**草稿更快**（降低 T_draft）、**草稿更准**（提高 τ）、**验证更聪明**（降低 T_verify / 按需截断）。

### 1.2 无损保证

推测解码最精妙的数学性质：**无论草稿模型多烂，输出分布严格等于目标模型。**

核心机制是拒绝采样（Rejection Sampling）：

- 接受概率 = min(1, p_target(x) / p_draft(x))
- 被拒绝位置从残差分布重采一个修正 token（bonus token）

两路径合并后，最终输出每个 token 的概率严格等于 p_target(x)。草稿模型只影响速度，不影响质量。

### 1.3 三种草稿生成方式

这是理解各方法差异的关键：

**纯串行（自回归派）**：逐 token 生成，每步一次前向
- 优点：每个 token 基于已确定的前文，准确
- 缺点：γ 次前向，T_draft ∝ γ；为控延迟被迫用极浅网络（1 层），容量受限

**纯并行（并行派）**：一次前向同时出 γ 个 token
- 优点：1 次前向出全部 token，T_draft 几乎与 γ 无关；可用更深网络（5 层）
- 缺点：位置间独立，无法建模依赖，后缀衰减严重（多模态碰撞问题）

**半自回归（混合派）**：并行 backbone 出基线 + 串行 head 注入依赖
- 优点：兼顾速度和后端质量
- 缺点：串行 head 有少量额外延迟（极小）

---

## 二、DeepSpec 项目中的三种方案对比

DeepSpec 仓库当前包含三种草稿模型算法，代表了推测解码的三个技术方向。

### 2.1 DSpark（半自回归 + 置信度调度）

**论文**：arXiv 2607.05147（DeepSeek-AI，2026）

**架构**：

```
[并行 backbone: 5层，一次出全部基线 logits]
        ↓
[串行 Markov head: 轻，逐位注入前一个 token 的偏置]
        ↓
[置信度头: 预测每个位置的接受概率，调度器据此截断]
```

**核心技术**：

1. **半自回归生成**：并行 backbone 一次出 γ 个基线 logit，串行 Markov head 逐位加低秩转移偏置 B_k = W2·W1[x_{k-1}]（rank=256），使第 k 位依赖前面已采样的 token。每个 token 概率都是精确 softmax，满足无损保证。

2. **置信度调度验证**：
   - 置信度头：给每个位置打条件接受概率 c_k
   - STS 校准：逐位温度校准，ECE 从 3%-8% 降到约 1%
   - 硬件感知前缀调度器：按生存概率贪心截断，最大化全局吞吐 Θ = τ × SPS(B)，且截断只依赖已处理前缀（因果性/无损）

3. **训练损失**：CE(0.1) + L1/TV(0.9) + 置信度 BCE(1.0)，位置加权 w_k = e^{-(k-1)/γ}

**离线效果**（Table 1，平均接受长度 τ，block_size=7）：

| 目标模型 | vs Eagle3 | vs DFlash |
|---|---|---|
| Qwen3-4B | +30.9% | +16.3% |
| Qwen3-8B | +26.7% | +18.4% |
| Qwen3-14B | +30.0% | +18.3% |
| Gemma4-12B | 全面领先 | 跨模型族泛化 |

具体（Qwen3-4B, GSM8K）：Eagle3=5.14，DFlash=5.40，DSpark=6.11

**线上效果**（DeepSeek-V4-Flash/Pro 真实流量，对比 MTP-1）：

| 引擎 | 等吞吐下单用户提速 | 中等 SLA 下吞吐提升 |
|---|---|---|
| V4-Flash | +60%-85% | +51% |
| V4-Pro | +57%-78% | +52% |

**优点**：
- 接受长度 τ 最高（~6+），块越长优势越大（γ=7 时 +16%，γ=15 时 +30%）
- 2 层 DSpark 超过 5 层 DFlash——注入一点自回归比堆并行层更划算
- 串行 head 几乎不增延迟（γ 从 4→16，每轮延迟只比 DFlash 多 0.2%-1.3%）
- 置信度调度首次让高并发生产环境安全使用多 token 推测解码
- 负载自适应：低并发扩验证预算榨干算力，高并发自动收缩保护 batch 容量

**缺点**：
- 需要为每个目标模型单独训练草稿（紧耦合：同族、hidden_size 匹配、tokenizer 一致）
- 置信度调度器的线上版本（异步 ZOS 适配、变长路由 kernel）属 DeepSeek 内部框架，未开源
- 对本身接受率极低的复杂查询，草稿侧固定开销无法回收

### 2.2 DFlash（纯并行）

**论文**：arXiv 2602.06036

**架构**：DSpark 去掉 Markov head 和置信度头

DeepSpec 仓库中，DFlash 和 DSpark 共用同一个 Trainer 和模型代码，仅配置不同：
- `markov_rank=0` → Markov head 变零矩阵，退化为纯并行
- `confidence_head_alpha=0` → 不训练置信度头
- `l1_loss_alpha=0`，`ce_loss_alpha=1.0` → 只用 CE loss

**生成方式**：
```
backbone([anchor, mask, mask, ..., mask]) → [l₁, l₂, ..., l₇] → 独立采样
1 次草稿前向，但位置间无依赖
```

**优点**：
- 1 次前向出全部 token，T_draft 几乎与 γ 无关
- 可用深网络（5 层），位置 1 的接受率最高（数学 0.88，优于自回归派的 0.81）
- 实现简单（无串行 head，无置信度调度）
- 训练只需 CE loss，不需置信度标签

**缺点**：
- **后缀衰减**：后面位置因位置间独立，接受率急剧下降（位置 2-7 从 0.72 降到 0.63）
- 多模态碰撞：上下文可能是 "of course" 也可能是 "no problem"，并行 head 同时推 "course" 和 "problem"，串成 "of problem"
- 无法在高并发下安全使用长块（无置信度截断机制，验证浪费 batch 容量）
- 接受长度 τ ~4-5，不如半自回归派

### 2.3 Eagle3（纯串行 + TTT）

**论文**：arXiv 2503.01840

**架构**：1 层自回归草稿头 + Test-Time Training（TTT）

**核心技术**：
- 草稿模型只有 1 层（`draft_num_hidden_layers=1`），逐 token 串行生成
- TTT（Test-Time Training）：推理过程中草稿模型参数动态更新，每 7 步用目标模型反馈微调
- 读目标模型最后层 hidden state，共享 embedding 和 lm_head

**生成方式**：
```
draft(t₁) → draft(t₂) → ... → draft(t₇)
7 次草稿前向，但每步有完整依赖
```

**优点**：
- 每个 token 基于已确定前文，准确度高
- TTT 机制让草稿适应当前上下文，长期对话中越来越准（分布外泛化能力强）
- 1 层网络前向+反向极快，TTT 微调开销小
- 第一个 token 准确率最高（前缀匹配生存的关键）

**缺点**：
- T_draft ∝ γ，块越长草稿越慢，被迫用极浅网络（1 层），容量受限
- 位置 1 的接受率不如并行派（数学 0.81 vs 0.88），因 1 层网络容量小
- TTT 推理时有额外梯度计算开销
- 高并发下多 token 验证开销大（无置信度调度）
- 接受长度 τ ~4-5，块越长劣势越明显

### 2.4 三方案对比总表

| 维度 | DSpark | DFlash | Eagle3 |
|---|---|---|---|
| 生成方式 | 半自回归 | 纯并行 | 纯串行 |
| 草稿层数 | 5 层 | 5 层 | 1 层 |
| 串行依赖 | Markov head（低秩 rank=256） | 无 | 完整自回归 |
| 置信度调度 | 有（STS 校准 + 贪心调度器） | 无 | 无 |
| T_draft vs γ | 几乎无关 + 极小串行开销 | 几乎无关 | ∝ γ（线性增长） |
| 位置 1 接受率 | 高（深网络 + 串行修正） | 最高（深网络） | 中（1 层容量受限） |
| 后缀衰减 | 轻微（Markov head 注入依赖） | 严重（位置间独立） | 无（逐 token 依赖） |
| TTT 推理时训练 | 无 | 无 | 有（每 7 步微调） |
| 训练损失 | CE + L1/TV + 置信度 BCE | 仅 CE | CE + step loss |
| 接受长度 τ | ~6+（最高） | ~4-5 | ~4-5 |
| 高并发安全 | 是（置信度截断） | 否 | 否 |
| 分布外适应 | 依赖训练数据覆盖 | 依赖训练数据覆盖 | TTT 动态适应 |
| 代码位置 | `modeling/dspark/` | `modeling/dspark/`（配置不同） | `modeling/eagle3/` |

---

## 三、DeepSeek MTP 与推测解码的关系

### 3.1 MTP 是什么

MTP（Multi-Token Prediction，多 Token 预测）是 DeepSeek-V3 技术报告提出的**训练阶段**技术。预训练时不只预测下一个 token，还同时预测未来第 2、3...个 token，迫使模型学更好的表征。

### 3.2 MTP 架构

DeepSeek-V3 的 MTP 是串行链式的，每个模块的输入依赖前一个模块的输出（保持完整因果链）：

```
主模型 → hidden h₀ → MTP模块1 → 预测 token₂ → hidden h₁ → MTP模块2 → 预测 token₃ → ...
```

每个 MTP 模块：输入 = concat(RMSNorm(h_{k-1}), RMSNorm(Emb(token_k))) → 投影 → Transformer → 预测 token_{k+1}

### 3.3 MTP vs 推测解码

| 维度 | MTP | 推测解码 |
|---|---|---|
| 本质 | 训练阶段技术 | 推理阶段技术 |
| 主要目的 | 改善训练表征质量 | 加速推理 |
| 草稿来源 | MTP 模块（预训练时联合训练） | 独立小模型 / 挂载头 / n-gram |
| 训练方式 | 与主模型联合预训练 | 草稿模型单独训练 |
| 推理时 | 默认丢弃 MTP 模块 | 必须有草稿模型参与 |
| 结合点 | MTP 模块可当推测解码的草稿模型 | DeepSeek 生产环境就是如此 |

### 3.4 DeepSeek 生产环境的实际做法

- 预训练时：主模型 + MTP 模块联合训练，MTP loss 权重 0.1
- 推理时：保留 MTP-1（只保留第一个 MTP 模块，预测 2 个 token），因为高并发下多 token 验证开销过大
- MTP-1 第二 token 接受率约 85%-90%，约 1.8x 推理加速
- DSpark 论文的核心贡献之一：**首次让高并发生产环境安全使用多 token 推测解码**，此前只能用 MTP-1

### 3.5 MTP vs EAGLE/DSpark 的技术亲缘

共同点：都读目标模型 hidden state，都有轻量 Transformer 层 + 共享 embedding/unembedding。

关键区别：
- MTP 模块：**预训练时与主模型联合训练**，权重与主模型耦合优化
- EAGLE/DSpark 草稿：**后训练阶段单独训练**，主模型冻结

MTP 当草稿用时接受率不如专门训练的方案（MTP-1 τ~2 vs DSpark τ~6+），但 MTP 的额外好处是**训练阶段就提升了主模型质量**，这是推测解码草稿不具备的。

---

## 四、各方案在 vLLM 中的落地情况

vLLM 是当前最活跃的开源 LLM 推理引擎，对推测解码的支持最为全面。

### 4.1 总览

vLLM 支持的推测解码方法（截至 2026-07）：

| 方法 | 支持状态 | 配置 method |
|---|---|---|
| EAGLE / EAGLE-3 | 已合并，生产可用 | `eagle3` |
| DFlash | 已合并，生产可用 | `dflash` |
| DSpark | 已合并，积极完善中 | `dspark` |
| DeepSeek MTP | 已合并，成熟 | `deepseek_mtp` |
| P-EAGLE（并行 EAGLE） | 已合并 | `eagle3` + `parallel_drafting: true` |
| N-gram | 已合并 | `ngram` |
| 独立草稿模型 | 已合并 | `draft` |

官方文档：https://docs.vllm.ai/en/latest/features/speculative_decoding

### 4.2 Eagle3 在 vLLM

Eagle3 于 2026 年 4 月开始在 vLLM 中落地，Red Hat 团队主导集成。

关键 PR：

| PR | 标题 | 合并日期 |
|---|---|---|
| [#37512](https://github.com/vllm-project/vllm/pull/37512) | MiniMax-M2: add Eagle3 speculative decoding support | 2026-04-06 |
| [#39450](https://github.com/vllm-project/vllm/pull/39450) | Add Gemma4 Eagle3 support | 2026-04-10 |
| [#43132](https://github.com/vllm-project/vllm/pull/43132) | [Spec Decode] Add Qwen3 architecture support for EAGLE3 | 2026-06-21 |
| [#42764](https://github.com/vllm-project/vllm/pull/42764) | [Model] Support post-norm architecture for EAGLE-3 speculators | 2026-05-19 |
| [#41826](https://github.com/vllm-project/vllm/pull/41826) | Added peagle speculators support | 2026-05-12 |

Red Hat 博客报道 Eagle3 在 vLLM 中最高可达 2.5x 加速：
https://developers.redhat.com/articles/2025/07/01/fly-eagle3-fly-faster-inference-vllm-speculative-decoding

P-EAGLE（并行版 Eagle3）在 vLLM 中通过 `parallel_drafting: true` 启用，B200 上比 Eagle3 快 1.05x-1.69x：
https://vllm.ai/blog/2026-03-13-p-eagle

### 4.3 DFlash 在 vLLM

DFlash 于 2026 年 4 月开始在 vLLM 中落地，同样由 Red Hat 团队主导。

关键 PR：

| PR | 标题 | 合并日期 |
|---|---|---|
| [#38300](https://github.com/vllm-project/vllm/pull/38300) | [Speculative Decoding] Add DFlash speculators config parsing | 2026-04-15 |
| [#43445](https://github.com/vllm-project/vllm/pull/43445) | [Spec Decode] Allow causal DFlash | 2026-05-28 |
| [#44586](https://github.com/vllm-project/vllm/pull/44586) | [MRV2][Spec Decode] DFlash | 2026-06-10 |
| [#45181](https://github.com/vllm-project/vllm/pull/45181) | [Spec Decode] Support mixed KV page sizes for DFlash | 2026-06-21 |
| [#45319](https://github.com/vllm-project/vllm/pull/45319) | [Model][Dflash] Enable Dflash for Qwen3NextForCausalLM | 2026-06-12 |
| [#46770](https://github.com/vllm-project/vllm/pull/46770) | [Model Runner V2][DFlash] Enable dflash attention backend selection | 2026-06-26 |
| [#47914](https://github.com/vllm-project/vllm/pull/47914) | [Spec Decode] Support hybrid (SWA + full attention) DFlash drafters | 2026-07-08 |

vllm-project/speculators 项目提供 DFlash 草稿模型训练，训练后可直接部署到 vLLM：
https://github.com/vllm-project/speculators

### 4.4 DSpark 在 vLLM

DSpark 于 2026 年 7 月 1 日合并到 vLLM，目前处于积极完善阶段，有大量 follow-up PR 在修复边缘情况和扩展支持。

关键 PR：

| PR/Issue | 标题 | 状态 | 日期 |
|---|---|---|---|
| [#46910](https://github.com/vllm-project/vllm/issues/46910) | [Feature]: Support DSpark for DeepSeek V4（Feature Issue） | Closed (COMPLETED) | 2026-06，2026-07-02 关闭 |
| [#46995](https://github.com/vllm-project/vllm/pull/46995) | [Spec Decode] DSpark（主 PR） | Merged | 2026-07-01 |
| [#47093](https://github.com/vllm-project/vllm/pull/47093) | [Spec Decode] DSpark speculators checkpoint support | Merged | 2026-07-02 |
| [#47216](https://github.com/vllm-project/vllm/pull/47216) | [Spec Decode][DSpark] Add Gemma4-12B DSpark draft model | Merged | 2026-07-16 |
| [#47419](https://github.com/vllm-project/vllm/pull/47419) | [ROCm] Enable DeepSeek-V4 DSpark on AMD (MI350X/MI355X) | Merged | 2026-07-10 |
| [#47677](https://github.com/vllm-project/vllm/pull/47677) | [XPU] Add DSpark speculative decoding support for DeepSeek-V4 | Merged | 2026-07-16 |
| [#47377](https://github.com/vllm-project/vllm/pull/47377) | [Spec Decode] Add DSpark support for Qwen3.5 target models | Open | 2026-07 |
| [#48692](https://github.com/vllm-project/vllm/pull/48692) | [MRV2][Spec Decode] Adaptive Speculative Decoding - Initial Support | Open | 2026-07 |
| [#47584](https://github.com/vllm-project/vllm/pull/47584) | [Spec Decode][Perf] Rowwise-fp8 draft lm_head for DSpark (opt-in) | Open | 2026-07 |

主 PR #46995 的设计说明：
- 支持加载 DeepSeek-V4 DSpark 模型和 DeepSeek 训练的 Qwen3-DSpark 模型
- DSpark 使用非因果滑动窗口注意力，复用现有 SparseMLA 后端（扩展 topk），无需重新实现 MLA attention
- 支持通过 `--speculative-config` 配置

vLLM Ascend 也有对应 RFC：
- [vllm-ascend#11126](https://github.com/vllm-project/vllm-ascend/issues/11126)：[RFC]: Add DSpark speculative decoding support for DeepSeek-V4

### 4.5 DeepSeek MTP 在 vLLM

DeepSeek MTP 是 vLLM 中最早支持的推测解码方法之一，2025 年 2 月即合并。

关键 PR：

| PR | 标题 | 合并日期 |
|---|---|---|
| [#12755](https://github.com/vllm-project/vllm/pull/12755) | [Model][Speculative Decoding] DeepSeek MTP spec decode | 2025-02-19 |
| [#44821](https://github.com/vllm-project/vllm/pull/44821) | fix: prefix DeepSeek V4 MTP projections | 2026-06-10 |
| [#44420](https://github.com/vllm-project/vllm/pull/44420) | [feature] add index share feature for DSA MTP | 2026-06-07 |
| [#46994](https://github.com/vllm-project/vllm/pull/46994) | [Spec][V2] Support MTP speculative decoding under pipeline parallelism | Open |

配置示例：
```json
{"method": "deepseek_mtp", "num_speculative_tokens": 1}
```

MTP 已支持 PP > 1（pipeline parallelism），是当前最成熟的推测解码选项。

### 4.6 vLLM 推测解码生态：vllm-project/speculators

vLLM 团队维护了专门的草稿模型训练库：
https://github.com/vllm-project/speculators

- 统一训练框架，训练后直接部署到 vLLM
- 支持 P-EAGLE、DFlash、EAGLE-3 训练
- 已发布多个模型：Qwen3-8B DFlash（τ~3.74）、Gemma4-31B DFlash/Eagle3 等
- DFlash 模型通过 PR #38300 实现无缝部署

---

## 五、各方案在 SGLang 中的落地情况

SGLang 是 LMSYS 团队维护的高性能推理引擎，对推测解码的支持同样全面。

### 5.1 总览

SGLang 支持的推测解码方法：

| 方法 | 支持状态 | 配置 |
|---|---|---|
| EAGLE-2 | 已合并，成熟 | `--speculative-algorithm EAGLE` |
| EAGLE-3 | 已合并，成熟 | `--speculative-algorithm EAGLE3` |
| DFlash | 已合并，成熟 | `--speculative-algorithm DFLASH` |
| DSpark | 已合并，积极完善中 | `--speculative-algorithm DSPARK` |
| MTP | 已合并，成熟 | DeepSeek 模型原生支持 |
| N-gram | 已合并 | `--speculative-algorithm NGRAM` |
| 独立草稿模型 | 已合并 | `--speculative-algorithm STANDALONE` |

官方文档：https://docs.sglang.ai/advanced_features/speculative_decoding.html

### 5.2 Eagle3 在 SGLang

Eagle3 通过 SpecForge 框架训练，部署到 SGLang：

- SpecForge（sgl-project/SpecForge）：Eagle3 训练框架
  - 仓库：https://github.com/sgl-project/SpecForge
  - 博客：https://www.lmsys.org/blog/2025-07-25-spec-forge
- SGLang Eagle3 已支持多种模型（Llama、Qwen3、Kimi-K2.6 等）

关键 PR（SGLang 中的 Eagle3 相关）：

| PR | 标题 | 合并日期 |
|---|---|---|
| [#26506](https://github.com/sgl-project/sglang/pull/26506) | [spec decoding] support kimi-k2.6-eagle3.1-mla draft | 2026-05-28 |
| [#29464](https://github.com/sgl-project/sglang/pull/29464) | Fix EAGLE draft hidden dim extraction and centralize spec helpers | 2026-06-28 |
| [#26726](https://github.com/sgl-project/sglang/pull/26726) | fix(spec-dec): treat num_nextn_predict_layers=0 the same as absent for EAGLE3 | 2026-06-05 |
| [#30947](https://github.com/sgl-project/sglang/pull/30947) | [EAGLE] perf: Fuse topk=1 draft postprocess | 2026-07-16 |
| [#31380](https://github.com/sgl-project/sglang/pull/31380) | [Spec] Consolidate the verify step into eagle_worker_common | 2026-07-16 |

### 5.3 DFlash 在 SGLang

DFlash 在 SGLang 中已成熟，支持多种后端和优化。

关键 PR：

| PR | 标题 | 合并日期 |
|---|---|---|
| [#29338](https://github.com/sgl-project/sglang/pull/29338) | [Spec] Add DFLASH basic sanity CI test | 2026-06-29 |
| [#29228](https://github.com/sgl-project/sglang/pull/29228) | [Spec] Merge dflash triton kernels into a single dflash.py | 2026-06-25 |
| [#29541](https://github.com/sgl-project/sglang/pull/29541) | [Spec] Publish DFLASH verify read-done event for fine-grained WAR barrier | 2026-06-29 |
| [#29446](https://github.com/sgl-project/sglang/pull/29446) | Add Laguna XS.2.1 DFlash support to SGLang | 2026-07-02 |
| [#29218](https://github.com/sgl-project/sglang/pull/29218) | [Spec] DFlash: support pure-MLA targets with an fp8 KV cache (Kimi-K2.x-NVFP4) | 2026-07-08 |
| [#29995](https://github.com/sgl-project/sglang/pull/29995) | [Spec] Remove the ServerArgs clone hack from DFlashWorkerV2 | 2026-07-03 |
| [#31468](https://github.com/sgl-project/sglang/pull/31468) | [Spec] DFlash: remove per-step host syncs (spec-v2 overlap) | 2026-07-18 |
| [#31677](https://github.com/sgl-project/sglang/pull/31677) | [Spec] Extract DFlash compact draft-cache rebuild helpers | 2026-07-20 |
| [#29884](https://github.com/sgl-project/sglang/pull/29884) | [Doc] Cookbook: Laguna-XS-2.1 (DFlash low-latency + high-throughput) | 2026-07-02 |

### 5.4 DSpark 在 SGLang

DSpark 于 2026 年 7 月 12 日合并到 SGLang，LMSYS 团队发布了详细的工程博客。

主 PR 及关键 follow-up：

| PR | 标题 | 状态 | 日期 |
|---|---|---|---|
| [#29488](https://github.com/sgl-project/sglang/issues/29488) | [Feature] Support DSpark Speculative Decoding for DeepSeek V4（Feature Issue） | Open | 2026-06-27 |
| [#30261](https://github.com/sgl-project/sglang/pull/30261) | [Spec] Add DSpark: confidence-scheduled speculative decoding（主 PR） | Merged | 2026-07-12 |
| [#31434](https://github.com/sgl-project/sglang/pull/31434) | [Perf] Cache uniform ragged-verify layout for DSpark verify-all compact | Merged | 2026-07-16 |
| [#31985](https://github.com/sgl-project/sglang/pull/31985) | [Perf] Fold dspark dense draft embedding into the draft graph | Merged | 2026-07-22 |
| [#31986](https://github.com/sgl-project/sglang/pull/31986) | [Perf] Stack dspark dense draft per-layer ctx KV projection into one GEMM | Merged | 2026-07-22 |
| [#30720](https://github.com/sgl-project/sglang/pull/30720) | [WIP] Enable GLM-5.2 DSpark compact verify functionality | Open | 2026-07 |
| [#31026](https://github.com/sgl-project/sglang/pull/31026) | [ROCm] Add DeepSeek-V4 DSpark for AMD GPU (MI350X/MI355X) | Open | 2026-07 |
| [#30964](https://github.com/sgl-project/sglang/pull/30964) | [AMD] Support DeepSeek V4 DSpark on AMD HIP platform | Open | 2026-07 |
| [#31139](https://github.com/sgl-project/sglang/pull/31139) | [PP&Spec] enable speculative decoding (MTP&DSpark) under PP | Open | 2026-07 |
| [#31466](https://github.com/sgl-project/sglang/pull/31466) | [Spec] DSpark support prefill/decode disaggregation | Open | 2026-07 |
| [#31513](https://github.com/sgl-project/sglang/pull/31513) | [Spec] Support DSPARK in disaggregation decode mode | Open | 2026-07 |

SGLang DSpark 工程博客（强烈推荐阅读）：
https://www.lmsys.org/blog/2026-07-06-dspark-sglang

博客核心内容：
- SGLang 实现了 DSpark 的置信度调度器、逐请求变长验证（ragged verify）、全 CUDA graph 覆盖
- 三种验证模式：`static`（全块验证）、`compact`（按调度器截断，生产路径）、`cap-accept`（全块验证但只提交截断窗口，用于观测上限）
- Ragged verify under full CUDA graphs：变长请求前打包到一个紧凑缓冲区，按总量对齐到捕获的 graph tier，截断后重放真正更小的 graph
- SPS cost table：离线 profiling 步时间模型，在线让调度器决定每请求验证预算
- 零开销调度（ZOS）：集成到 SGLang 的 overlap scheduler

启动命令示例（DeepSeek-V4-Flash）：
```bash
python3 -m sglang.launch_server \
  --model-path deepseek-ai/DeepSeek-V4-Flash-DSpark \
  --speculative-algorithm DSPARK \
  --tp 4 --dp-size 4 --enable-dp-attention --enable-dp-lm-head \
  --moe-a2a-backend none --moe-runner-backend flashinfer_mxfp4 \
  --swa-full-tokens-ratio 0.1 --chunked-prefill-size 1024 \
  --mem-fraction-static 0.8 --cuda-graph-max-bs 192 \
  --max-running-requests 1024 --disable-radix-cache \
  --trust-remote-code --host 0.0.0.0 --port 30000
```

### 5.5 MTP 在 SGLang

SGLang 从 2025 年 1 月起就提供 DeepSeek 模型的 day-0 支持，MTP 支持较早成熟。

关键 PR：

| PR | 标题 | 合并日期 |
|---|---|---|
| [#26471](https://github.com/sgl-project/sglang/pull/26471) | DeepSeek-V4 Online Compress support MTP | 2026-06-16 |
| [#24436](https://github.com/sgl-project/sglang/pull/24436) | [Gemma 4] Adding MTP support | 2026-05-07 |
| [#30333](https://github.com/sgl-project/sglang/pull/30333) | [AMD] Fix DeepSeek V4 MTP accuracy issue | 2026-07-07 |
| [#30238](https://github.com/sgl-project/sglang/pull/30238) | [AMD] Support two batch overlap with MTP on DeepSeekV4 | 2026-07-16 |
| [#28982](https://github.com/sgl-project/sglang/pull/28982) | fix(mtp): avoid mtp perf regression in deepseek when enable eplb | 2026-07-09 |

SGLang MTP 教程：
https://company.hpc-ai.com/blog/sglang-speculative-decoding-tutorial

---

## 六、DeepSpec 落地情况总结

### 6.1 落地成熟度对比

| 方案 | vLLM | SGLang | 备注 |
|---|---|---|---|
| Eagle3 | 生产可用（2026-04 起合并，多模型支持） | 生产可用（SpecForge 训练 + SGLang 部署） | 两个引擎都成熟 |
| DFlash | 生产可用（2026-04 起合并，多后端支持） | 生产可用（多 kernel 优化，多模型支持） | 两个引擎都成熟 |
| DSpark | 已合并（2026-07-01），积极完善中 | 已合并（2026-07-12），积极完善中 | 大量 follow-up PR 修复边缘情况 |
| DeepSeek MTP | 成熟（2025-02 起合并） | 成熟（day-0 支持） | 两个引擎最早支持的方法 |

### 6.2 DSpark 落地特点

DSpark 在两个引擎中的落地都处于「核心功能已合并、积极扩展」阶段：

**vLLM 侧**：
- 主 PR #46995 支持加载 DeepSeek-V4 DSpark 和 Qwen3-DSpark 检查点
- 已扩展到 Gemma4-12B（#47216）、ROCm（#47419）、XPU（#47677）
- 正在添加 Qwen3.5 支持（#47377）、自适应推测解码（#48692）
- Red Hat 团队发布了 GLM-5.2 DSpark speculator preview：
  https://huggingface.co/RedHatAI/GLM-5.2-speculator.dspark-preview

**SGLang 侧**：
- 主 PR #30261 实现了完整的置信度调度 + ragged verify + CUDA graph
- LMSYS 发布了详细工程博客，展示了 DSpark 对 MTP 和 non-spec 的吞吐/延迟优势
- 正在扩展到 GLM-5.2（#30720, #31047）、ROCm（#30964, #31026）、PD 分离（#31466, #31513）
- 性能优化 PR 持续合并（#31985, #31986 融合 embedding 和 KV projection）

### 6.3 从研究到生产的完整路径

```
1. 训练（DeepSpec / SpecForge / vllm-project/speculators）
   准备数据 → 用目标模型重新生成回答 → 预计算 target cache → 训练草稿模型

2. 评测（DeepSpec eval.py）
   accept_len / accept_rate@k 指标

3. 部署（vLLM / SGLang）
   --speculative-config '{"method": "dspark", "model": "path/to/draft"}'
   --speculative-algorithm DSPARK --speculative-draft-model-path ...
```

---

## 七、方案选型建议

### 7.1 按场景选

| 场景 | 推荐方案 | 理由 |
|---|---|---|
| 高并发生产服务 | DSpark | 置信度调度负载自适应，不拖垮吞吐 |
| 低并发交互式（batch=1） | Eagle3 / DSpark | 接受率高的方案在低并发下收益最大 |
| 目标模型有原生 MTP | DeepSeek MTP | 零额外训练，直接用 |
| 不想训练草稿模型 | N-gram / Lookahead | 零训练成本，但加速有限 |
| 跨模型族通用 | 独立草稿模型 | 即插即用，但接受率最低 |
| 追求最高接受长度 | DSpark | τ~6+，块越长优势越大 |
| 追求分布外泛化 | Eagle3（TTT） | 推理时动态适应新上下文 |
| 快速验证 / 不确定效果 | DFlash | 实现简单，训练只需 CE loss |

### 7.2 按引擎选

| 需求 | vLLM | SGLang |
|---|---|---|
| 最全面的 spec decode 支持 | 方法最多 | 方法最多 |
| DSpark 置信度调度 | 基础支持 | 完整实现（compact ragged verify） |
| Eagle3 训练框架 | vllm-project/speculators | SpecForge |
| DeepSeek 模型优化 | 支持 | day-0 深度优化 |
| AMD GPU 支持 | 支持（ROCm） | 支持（ROCm） |
| 文档/教程 | 完善 | 完善 + cookbook |

---

## 八、参考文献

- DSpark 论文：arXiv 2607.05147
- DFlash 论文：arXiv 2602.06036
- Eagle3 论文：arXiv 2503.01840
- EAGLE 论文：arXiv 2401.15077
- Medusa 论文：arXiv 2401.10774
- Leviathan 2023（经典推测解码）：arXiv 2211.17192
- DeepSeek-V3 技术报告（MTP 原始来源）
- Better & Faster LLMs via Multi-token prediction：arXiv 2404.19737
- 推测解码综述：arXiv 2411.13157、arXiv 2502.19732
- vLLM 推测解码文档：https://docs.vllm.ai/en/latest/features/speculative_decoding
- SGLang 推测解码文档：https://docs.sglang.ai/advanced_features/speculative_decoding.html
- DSpark in SGLang 博客：https://www.lmsys.org/blog/2026-07-06-dspark-sglang
- SpecForge：https://github.com/sgl-project/SpecForge
- vllm-project/speculators：https://github.com/vllm-project/speculators
- Red Hat Eagle3 博客：https://developers.redhat.com/articles/2025/07/01/fly-eagle3-fly-faster-inference-vllm-speculative-decoding
- P-EAGLE 博客：https://vllm.ai/blog/2026-03-13-p-eagle

---

## 附录：关键 PR 速查索引

### vLLM

| 方案 | 关键 PR | 说明 |
|---|---|---|
| DeepSeek MTP | [#12755](https://github.com/vllm-project/vllm/pull/12755) | 初始支持（2025-02） |
| Eagle3 | [#37512](https://github.com/vllm-project/vllm/pull/37512), [#39450](https://github.com/vllm-project/vllm/pull/39450), [#43132](https://github.com/vllm-project/vllm/pull/43132) | MiniMax-M2 / Gemma4 / Qwen3 支持 |
| DFlash | [#38300](https://github.com/vllm-project/vllm/pull/38300), [#44586](https://github.com/vllm-project/vllm/pull/44586), [#43445](https://github.com/vllm-project/vllm/pull/43445) | 配置解析 / MRV2 / causal DFlash |
| DSpark | [#46995](https://github.com/vllm-project/vllm/pull/46995), [#47093](https://github.com/vllm-project/vllm/pull/47093), [#47216](https://github.com/vllm-project/vllm/pull/47216) | 主 PR / speculators checkpoint / Gemma4 |
| P-EAGLE | [#41826](https://github.com/vllm-project/vllm/pull/41826) | peagle speculators 支持 |

### SGLang

| 方案 | 关键 PR | 说明 |
|---|---|---|
| Eagle3 | [#26506](https://github.com/sgl-project/sglang/pull/26506), [#29464](https://github.com/sgl-project/sglang/pull/29464) | Kimi-K2.6 / 通用 spec helper |
| DFlash | [#29338](https://github.com/sgl-project/sglang/pull/29338), [#29228](https://github.com/sgl-project/sglang/pull/29228), [#29446](https://github.com/sgl-project/sglang/pull/29446) | CI / triton kernel / Laguna XS |
| DSpark | [#30261](https://github.com/sgl-project/sglang/pull/30261), [#31434](https://github.com/sgl-project/sglang/pull/31434), [#31985](https://github.com/sgl-project/sglang/pull/31985) | 主 PR / ragged-verify layout / perf 优化 |
| MTP | [#26471](https://github.com/sgl-project/sglang/pull/26471), [#24436](https://github.com/sgl-project/sglang/pull/24436) | DeepSeek-V4 / Gemma4 MTP |
