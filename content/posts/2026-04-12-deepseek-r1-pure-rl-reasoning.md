---
title: 'DeepSeek-R1: 如何用纯强化学习点燃大模型的推理能力'
date: 2026-04-12T11:06:07+08:00
draft: false
tags:
  - 'LLM'
  - '强化学习'
  - '推理能力'
  - 'DeepSeek'
  - 'AI论文复现'
description: '深入解析 DeepSeek-R1 系列如何通过纯强化学习（不依赖人类标注的推理轨迹）激发大语言模型的推理能力，包含 GRPO 算法原理、开源模型实测与工程化建议。'
showToc: true
math: false
---

## 真实事件引入：一场无需"标准答案"的推理革命

2025 年 1 月，DeepSeek 团队发表了一篇震惊业界的论文 [DeepSeek-R1](https://github.com/deepseek-ai/DeepSeek-R1)：他们成功训练出了一个推理能力与 OpenAI o1-1217 持平的大模型，而整个训练过程**没有任何人类标注的推理轨迹**——没有人工写好的思维链示例，没有 SFT（监督微调）作为前置步骤，只有强化学习（RL）。

这意味着什么？过去我们认为，想要让模型学会"认真思考"，必须先有人类专家手把手教它一步步推理。DeepSeek-R1 证明了这不是必要的——模型可以在完全没有"老师示范"的情况下，自己进化出自我验证、反思和动态策略调整的能力。

这不仅是一个性能指标的突破，更是对 LLM 训练范式的一次根本性重新思考。

---

## 背景与问题定义

传统推理模型（如 GPT-4、Claude）的推理能力主要来自两种手段：

1. **CoT（Chain-of-Thought） prompting**：在 prompt 里诱导模型生成中间推理步骤。效果有限，依赖模型自身的隐含知识。
2. **SFT（监督微调）**：让人类标注员手写大量"问题 → 推理过程 → 答案"的数据，然后用这些数据微调模型。这需要大量人力，且推理质量受标注员水平限制。

核心问题在于：**人类标注的推理轨迹是瓶颈**——成本高、规模受限、质量参差不齐。更重要的是，人类推理轨迹是静态的，无法覆盖模型可能遇到的所有复杂问题。

DeepSeek-R1 提出的核心研究问题是：

> **能否完全摆脱人类标注的推理轨迹，通过纯强化学习让 LLM 自发涌现出高级推理能力？**

---

## 框架核心拆解：GRPO 算法与两阶段训练 pipeline

### 1. DeepSeek-R1-Zero：纯 RL 的诞生

DeepSeek-R1-Zero 是整个系列的起点，也是最能说明问题的实验：**直接对 Base Model（DeepSeek-V3-Base）应用大规模 RL，不经过任何 SFT**。

结果令人意外——模型在训练过程中**自发涌现**出了以下行为：

- **自我验证（Self-Verification）**：模型在推理过程中会回头检查自己的中间步骤是否正确
- **反思（Reflection）**：当发现错误时，模型会主动推翻重来
- **长思维链（Long CoT）**：模型生成超长的推理序列，将复杂问题分解为多步子问题

这些行为没有任何人教过，完全是 RL 信号驱动下自发产生的。

### 2. GRPO：Group Relative Policy Optimization

DeepSeek 没有使用标准的 PPO（Proximal Policy Optimization）算法，而是自研了 **GRPO**——一种 PPO 的变体，专门为推理任务优化。

标准 PPO 需要一个独立的 Critic 模型来评估每个 action 的价值，而 GRPO 通过**组内相对排序**的方式淘汰了对 Critic 的依赖：

```python
# GRPO 核心思想（伪代码）
def grpo_update(policy, prompts, reward_fn):
    # 对每个 prompt，采样 G 个候选 response
    responses = [policy.sample(prompt) for _ in range(group_size)]
    # 用 reward_fn 打分
    rewards = [reward_fn(r) for r in responses]
    # 组内相对排序：只保留排名前 1/K 的样本作为正例
    advantage = rank_relative(rewards, group)
    # 用 PPO-style 策略更新
    policy.update(advantage)
```

**核心改进**：不需要训练一个单独的 Critic 模型，大幅降低了训练成本和内存开销。

### 3. DeepSeek-R1：加入冷启动数据

DeepSeek-R1-Zero 虽然推理能力强，但存在**可读性问题**——输出中经常出现语言混杂、格式混乱。为此，DeepSeek-R1 在 RL 之前引入了**少量冷启动 SFT 数据**（由 DeepSeek-V3 生成），来解决可读性和格式问题。

完整的 4 阶段 pipeline 如下：

```
┌─────────────────────────────────────────────────────────────┐
│  Stage 1: SFT（冷启动）                                      │
│  用 DeepSeek-V3 生成的少量高质量 CoT 数据微调 Base Model     │
├─────────────────────────────────────────────────────────────┤
│  Stage 2: RL（推理能力强化）                                  │
│  基于 GRPO，用可验证任务（数学、代码、逻辑）进行强化学习        │
├─────────────────────────────────────────────────────────────┤
│  Stage 3: SFT（拒绝采样）                                    │
│  收集 RL 模型生成的输出，筛选高质量样本，再次 SFT             │
├─────────────────────────────────────────────────────────────┤
│  Stage 4: RL（偏好对齐）                                     │
│  用 reward model + PPO 进行人类偏好对齐                       │
└─────────────────────────────────────────────────────────────┘
```

### 4. Distillation：小模型也能"涌现"推理

DeepSeek 团队还做了一个关键实验：用 DeepSeek-R1 生成的推理数据，对小模型进行 SFT 微调。结果非常反直觉——

| 模型 | AIME 2024 pass@1 | MATH-500 pass@1 | CodeForces Rating |
|------|-----------------|-----------------|-------------------|
| GPT-4o | 9.3 | 74.6 | 759 |
| o1-mini | 63.6 | 90.0 | 1820 |
| **DeepSeek-R1-Distill-Qwen-32B** | **72.6** | **94.3** | **1691** |

32B 的蒸馏模型在数学基准上超越了 o1-mini，这说明**大模型发现的推理模式可以通过数据蒸馏有效传递**，而不需要小模型从头做 RL。

---

## 关键洞察：工程化建议与选型结论

### 1. 纯 RL 训练 LLM 是可行的，但需要正确的任务设计

DeepSeek-R1-Zero 成功的关键在于**可验证奖励（verifiable rewards）**——数学题有标准答案，代码可以编译运行，RL 可以得到无歧义的反馈。对于没有明确验证函数的任务（如开放域推理），纯 RL 效果会大打折扣。**工程建议**：如果你的任务有可验证的答案/测试用例，纯 RL 值得尝试；否则考虑结合人类反馈（RLHF）或过程奖励模型。

### 2. 温度设置对推理模型至关重要

DeepSeek 官方明确建议使用 **temperature 0.5-0.7**（推荐 0.6），过低会导致重复行为，过高会导致推理质量不稳定。更重要的是：**避免 system prompt**，所有指令都应放在用户 prompt 中。这与常规 Instruct 模型的使用方式截然不同。**工程建议**：在集成 DeepSeek-R1 系列模型时，严格遵循官方推荐配置。

### 3. 强制模型输出思考过程可以显著提升推理质量

DeepSeek 团队发现模型有时会跳过思考过程直接给答案，导致推理能力退化。官方推荐的解决方案是**强制模型以 `\<think\>
` 开头**，确保每次输出都经过显式的推理步骤。这是一个低成本的推理质量提升技巧，适用于所有支持自定义输出的推理模型。

### 4. Distillation 是复现大模型推理能力的最高效路径

DeepSeek-R1-Distill 系列证明，从大模型蒸馏推理数据比在小模型上从头做 RL 效率高得多——DeepSeek-R1-Distill-Qwen-32B 的训练成本远低于 o1-mini，但数学推理能力相当甚至更好。**工程建议**：如果你的场景需要部署较小规模的推理模型，优先考虑用大模型蒸馏数据微调，而不是从头训练 RL。

---

## 信源引用

- **DeepSeek-R1 论文 & GitHub 仓库**：[https://github.com/deepseek-ai/DeepSeek-R1](https://github.com/deepseek-ai/DeepSeek-R1)（92k stars，36 commits，MIT 许可证）
- **arXiv 原文**：[https://arxiv.org/abs/2501.12948](https://arxiv.org/abs/2501.12948)（v2 版本，2026年1月更新）
- **DeepSeek-V3 架构**：[https://github.com/deepseek-ai/DeepSeek-V3](https://github.com/deepseek-ai/DeepSeek-V3)
- **HuggingFace 模型权重**：DeepSeek-R1-Zero 和 DeepSeek-R1-Distill 系列均已开源于 [HuggingFace](https://huggingface.co/deepseek-ai)
- **官方使用建议**（来自 GitHub README）：temperature 0.5-0.7，避免 system prompt，强制 `\<think\>` 开头

---

## 总结

DeepSeek-R1 系列最重要的贡献不是某个基准测试的分数，而是一个**范式验证**：大语言模型的推理能力不需要人类手把手教，通过合理的 RL 算法和可验证的奖励设计，模型可以自己学会反思、验证和策略调整。

对于 AI 工程师和研究者而言，这意味着：
- **训练范式**上，纯 RL 是解锁推理能力的可行路径，尤其适合有明确验证标准的任务
- **推理部署**上，严格遵循温度和 prompt 配置是保证推理质量的前提
- **模型选型**上，蒸馏路径为中小规模部署提供了高性价比选择

开源社区已经可以自由获取 DeepSeek-R1-Zero 及全部 Distill 模型的权重，有条件的读者不妨用 vLLM 或 SGLang 本地部署实测，感受一下"机器自发思考"是什么体验。

