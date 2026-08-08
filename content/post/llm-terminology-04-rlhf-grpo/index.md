---
title: "LLM 术语全景图（四）：训练方法——从 RLHF 到 GRPO 的演进"
date: 2026-07-27T09:45:00+08:00
slug: "llm-terminology-04-rlhf-grpo"
draft: false
image: "04.png"
tags:
    - AI
    - LLM
    - GRPO
    - RLHF
    - DPO
    - DeepSeek R1
categories:
    - AI
    - LLM术语全景图
---

> 这是「大模型训练与推理术语全景图」系列的第 4 篇，覆盖训练方法的 6 个概念：GRPO、RLHF、DPO、SFT、Knowledge Distillation、Muon Optimizer。
> 主要参考模型：DeepSeek R1、Qwen3、Llama 3、Kimi K2、GPT-4

---

## 开篇：后训练范式的三个阶段

后训练（post-training）在 2022-2026 年间走过三个可辨识的阶段。第一阶段是 SFT（Supervised Fine-Tuning）：用人工标注的高质量 (prompt, response) 对微调预训练基座，让模型学会遵循指令、采用对话格式输出，几乎所有对话 LLM 的第一步后训练都是 SFT。第二阶段是 RLHF/DPO 对齐人类偏好：RLHF 先训练 Reward Model 再用 RL 优化，DPO 则跳过 Reward Model 直接从偏好数据优化，目标是让模型输出符合人类对"有用、无害、诚实"的偏好。第三阶段是 GRPO/RLVR 用可验证奖励驱动推理能力涌现：RLVR（RL with Verifiable Rewards）用规则验证奖励（如数学答案对错、代码测试通过）替代学到的 Reward Model，GRPO 去掉 PPO 的 Critic 用组内相对优势做基线。DeepSeek R1-Zero 跳过 SFT 直接做纯 GRPO（RLVR）就涌现出长链推理和"Aha Moment"，是这一阶段的标志性事件——它证明 RL 可独立于 SFT 涌现推理能力，但 R1 正式版仍加入 SFT 阶段以提升最终可用性。本篇按这条主线梳理 6 个训练方法术语。


## 训练方法（Training Methods）


### Group Relative Policy Optimization

- **缩写**：GRPO
- **中文名称**：组相对策略优化
- **简短介绍**：DeepSeek 提出的强化学习算法，去除 PPO 的独立价值网络（critic），改用同一 prompt 的多个采样响应的组内平均作为基线，大幅降低训练内存和计算成本。
- **详细介绍**：GRPO vs PPO：

  PPO 需要训练 4 个模型：Actor、Critic、Reward Model、Reference Model
  GRPO 只需 3 个：Actor、Reward Model、Reference Model（去掉 Critic）

  GRPO 的优势计算：
  1. 对同一 prompt x，采样 G 个响应 {o_1, ..., o_G}
  2. 用 Reward Model 或规则打分得到 {r_1, ..., r_G}
  3. 组内归一化：advantage_i = (r_i - mean(r)) / std(r)
  4. 用 advantage 更新 Actor（带 PPO 的 clip 机制）

  关键优势：
  - 省去 Critic 模型——PPO 需训练 Actor 和 Critic 两个模型，GRPO 只训练 Actor 一个，需训练的模型显存减少约一半
  - 组内相对比较天然适合推理任务（数学/代码有明确对错）
  - DeepSeek R1-Zero 证明纯 RL（无 SFT）也能涌现推理能力（"Aha Moment"）

  DeepSeek R1 训练流程：
  1. R1-Zero：V3 基座 → 纯 GRPO（无 SFT）→ 涌现推理
  2. R1：V3 基座 → 少量 SFT（CoT 数据）→ GRPO → SFT（600K+200K 样本）→ GRPO

  采用 GRPO 的模型：DeepSeek R1、Qwen3（部分阶段）、Kimi K1.5 等。

- **关联论文/模型**："DeepSeekMath" (arXiv:2402.03300, 首次提出 GRPO)；DeepSeek R1 (arXiv:2501.12948)
- **参考分析链接**：
  - https://snorkel.ai/grpo
  - https://huggingface.co/learn/llm-course/en/chapter12/3
  - https://www.youtube.com/watch?v=bAWV_yrqx4w (Yannic Kilcher GRPO 讲解)


### Reinforcement Learning from Human Feedback

- **缩写**：RLHF
- **中文名称**：基于人类反馈的强化学习
- **简短介绍**：通过人类偏好数据训练奖励模型，再用该奖励模型通过 RL 优化 LLM 输出。ChatGPT 的核心训练方法，使模型输出符合人类偏好。
- **详细介绍**：RLHF 的标准三阶段：

  1. SFT（监督微调）：用人类标注的高质量对话微调基座模型
  2. Reward Model 训练：用人类偏好评判数据（A > B 或评分）训练奖励模型
  3. RL 优化：用 PPO/GRPO 等算法，以 Reward Model 的分数为奖励优化 SFT 模型

  关键组件：
  - Preference Data：人类标注 (prompt, response_A, response_B, preference)
  - Reward Model：学习预测人类偏好分数
  - Reference Model：KL 散度约束，防止 RL 优化偏离原始分布

  变体：
  - RLAIF：用 AI（而非人类）生成偏好数据，降低标注成本
  - DPO（Direct Preference Optimization）：跳过 Reward Model，直接从偏好数据优化
  - RLVR（RL with Verifiable Rewards）：规则验证奖励（如数学答案对错），无需 Reward Model

  GPT-4/Claude 的训练使用 RLHF；DeepSeek R1 使用 RLVR + GRPO；Qwen3 使用 DPO + GRPO 组合。

- **关联论文/模型**："InstructGPT" (Ouyang et al., 2022, arXiv:2203.02155)；"Constitutional AI" (Claude, Anthropic)；ChatGPT、GPT-4、Claude
- **参考分析链接**：
  - https://huggingface.co/blog/zh/rlhf


### Direct Preference Optimization

- **缩写**：DPO
- **中文名称**：直接偏好优化
- **简短介绍**：跳过显式 Reward Model 训练，直接从偏好数据对优化 LLM。将 RLHF 的两阶段（训练 RM + RL 优化）简化为单阶段，训练更简单稳定。
- **详细介绍**：DPO 的核心思想：

  传统 RLHF：偏好数据 → 训练 RM → RL 优化 Actor
  DPO：偏好数据 → 直接优化 Actor（用特定损失函数）

  DPO 损失：

  L_DPO = -log σ(β · [log π(y_w|x)/π_ref(y_w|x) - log π(y_l|x)/π_ref(y_l|x)])

  其中 y_w 是偏好响应，y_l 是不偏好响应，π 是当前策略，π_ref 是参考策略。

  优势：无需训练和部署独立的 Reward Model，训练流程更简单，显存更少。
  劣势：对偏好数据质量更敏感，不如 PPO/GRPO 灵活。

  采用 DPO 的模型：Qwen3（部分后训练阶段）、Llama 3 Instruct、Mistral 等。

- **关联论文/模型**："Direct Preference Optimization" (Rafailov et al., 2023, arXiv:2305.18290)；Llama 3、Qwen3
- **参考分析链接**：
  - https://huggingface.co/blog/pref-tuning


### Supervised Fine-Tuning

- **缩写**：SFT
- **中文名称**：监督微调
- **简短介绍**：用人工标注的高质量输入-输出对微调预训练基座模型，使其学会遵循指令和对话格式。几乎所有对话 LLM 的第一步后训练。
- **详细介绍**：SFT 的训练方式：

  输入：高质量 (prompt, response) 对
  损失：标准的 next-token prediction loss（仅在 response 部分计算）

  数据规模：
  - DeepSeek R1：SFT 阶段使用 600K 推理样本 + 200K 通用样本
  - Qwen3：数十万到百万级 SFT 样本
  - Llama 3：超过 10M 对话样本

  SFT 在训练流程中的位置：
  - 预训练（海量无标注文本）→ SFT（标注对话）→ RLHF/DPO（偏好优化）

  DeepSeek R1 的特殊路径：R1-Zero 跳过 SFT 直接做纯 RL，证明 RL 可独立涌现推理能力。但 R1 正式版仍加入了 SFT 阶段以提升可用性。

- **关联论文/模型**："InstructGPT" (Ouyang et al., 2022)；DeepSeek R1 (技术报告)；Qwen3 (技术报告)
- **参考分析链接**：
  - https://cameronrwolfe.substack.com/p/understanding-and-using-supervised


### Knowledge Distillation

- **缩写**：KD / 蒸馏
- **中文名称**：知识蒸馏
- **简短介绍**：用大模型（教师）的输出训练小模型（学生），使小模型获得接近大模型的能力。DeepSeek R1 用 R1 蒸馏出多个小模型。
- **详细介绍**：LLM 蒸馏的常见方式：

  1. 输出蒸馏（Output Distillation / 黑盒蒸馏）：
     - 用教师模型生成高质量响应（含 CoT 推理链）
     - 将这些响应作为 SFT 数据训练学生模型
     - DeepSeek R1 的做法：用 R1 生成 800K 推理数据，蒸馏到 Qwen/Llama 1.5B~70B（官方写法为 Llama-3.1-8B-Base 和 Llama-3.3-70B-Instruct）

  2. Logits 蒸馏（白盒蒸馏）：
     - 学生模型学习教师模型的 logits 分布（软标签）
     - 损失：KL(p_teacher || p_student)
     - 保留更多信息（比硬标签丰富）

  DeepSeek R1 蒸馏模型表现（技术报告表格）：
  - R1-Distill-Qwen-1.5B 在 AIME 28.9%，超过 GPT-4o
  - R1-Distill-Qwen-7B 在 AIME 55.5%
  - R1-Distill-Qwen-32B 接近 o1-mini

  注意：蒸馏不能创造新能力，只能传递教师已有的能力。学生模型的上限是教师模型。

- **关联论文/模型**：DeepSeek R1 (arXiv:2501.12948, 蒸馏部分)；"Distilling Reasoning" 等
- **参考分析链接**：
  - https://pureai.com/articles/2025/02/03/understanding-llm-distillation-enabling-revolutionary-deepseek-r1-model.aspx


### Muon Optimizer

- **缩写**：Muon
- **中文名称**：Muon 优化器
- **简短介绍**：基于矩阵正交化的新型优化器，对 2D 参数矩阵做 SVD 正交化后再更新梯度，收敛速度快于 AdamW。DeepSeek V4 等新一代模型采用。
- **详细介绍**：Muon 的核心思想：

  标准 Adam/AdamW：对每个参数的梯度做一阶/二阶动量估计
  Muon：对 2D 权重矩阵的梯度做正交化（Newton-Schulz 迭代近似 SVD），然后更新

  特点：
  - 仅适用于 2D 矩阵参数（Linear 层权重），1D 参数（bias、norm）仍用 AdamW
  - 正交化使更新方向更稳定，减少训练震荡
  - 收敛更快：在同等性能下可减少训练步数

  采用 Muon 的模型：DeepSeek V4（Muon-based optimization）；Kimi K2（MuonClip，Muon 的改进版本）

  注意：Muon 是 2025 年出现的新优化器，生态尚在发展，但已被多个大型模型验证有效。

- **关联论文/模型**："Muon" (J. Jordan, 2024-2025)；DeepSeek V4 (技术报告)；Kimi K2 技术报告 (MuonClip)
- **参考分析链接**：
  - https://pytorch.org/blog/using-muon-optimizer-with-deepspeed/


## 本篇小结

本篇 6 个训练方法术语沿"后训练三阶段"归位：

- **SFT** 是几乎所有对话 LLM 的第一步，用标注 (prompt, response) 对教会模型对话格式；DeepSeek R1-Zero 跳过 SFT 直接 RL 是例外而非通则，R1 正式版仍回归 SFT。
- **RLHF** 用学到的 Reward Model 优化人类偏好，标准三阶段（SFT → RM → RL）；**DPO** 跳过 RM 直接从偏好数据优化，单阶段更简单但对数据质量更敏感。
- **GRPO** 去掉 PPO 的 Critic，用组内相对优势做基线，需训练的模型从 2 个减到 1 个（Actor），显存减少约一半（按"需训练的模型"计）。GRPO 配合 RLVR 的可验证奖励，是 R1-Zero 涌现推理的算法载体。
- **Knowledge Distillation** 不创造新能力，只传递教师已有能力；R1 蒸馏 800K 推理数据到 Qwen 1.5B~32B 和 Llama 8B/70B，其中 R1-Distill-Qwen-1.5B 在 AIME 28.9% 已超过 GPT-4o。
- **Muon Optimizer** 是 2025 年新优化器，对 2D 矩阵参数做正交化更新，已被 DeepSeek V4 和 Kimi K2（MuonClip）采用，生态发展中。


本系列共 11 篇，本文是第 4 篇。系列导航见：LLM 术语全景图系列导读。
