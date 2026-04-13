---
title: 'GuppyLM: 用一个 Colab 笔记本，在 5 分钟内训练出你自己的 LLM'
date: 2026-04-12T12:12:44+08:00
draft: false
tags:
  - 'LLM'
  - 'Open Source'
  - 'Machine Learning'
  - 'Tutorial'
  - 'HuggingFace'
description: '一个 9M 参数的小模型，如何让大模型不再看起来像黑箱？GuppyLM 通过最简单的 vanilla transformer 架构，展示了从零训练 LLM 的完整流程。'
showToc: true
math: false
---

昨天在 HN 上看到一个很有想法的项目：作者在 5 分钟内，用一个 Colab 笔记本，从零训练出了一个 9M 参数的语言模型 GuppyLM。

不是跑 demo，不是微调，是**从数据生成、tokenizer、模型架构、训练循环到推理**全部从零开始。

---

## 真实案例：一条鱼能告诉你 LLM 内部发生了什么

GuppyLM 是一个假装自己是热带鱼 Guppy 的小模型。它说的话听起来很傻：

> You> what is the meaning of life?
> Guppy> food. the answer is always food.

这显然不是 GPT-4。但重点不在这里。**重点是：你能完整看到它是怎么被训练出来的。**

项目地址：https://github.com/arman-bd/guppylm  
在线 Demo（浏览器直接跑，无需服务器）：https://arman-bd.github.io/guppylm/

---

## 框架拆解：GuppyLM 的技术架构

GuppyLM 是一个**极简 vanilla transformer**，没有 GQA、没有 RoPE、没有 SwiGLU——怎么简单怎么来。

**核心参数：**

| | |
|---|---|
| 参数量 | 8.7M |
| 层数 | 6 |
| 隐层维度 | 384 |
| 注意力头数 | 6 |
| FFN 维度 | 768（ReLU）|
| 词表大小 | 4,096（BPE）|
| 最大序列长度 | 128 tokens |
| Norm | LayerNorm |
| 位置编码 | Learned embeddings |

整个架构就是教科书级别的 transformer。**没有花活，这是刻意设计的**——作者想让读者看清每一行代码在做什么。

**训练过程：**

1. 在 Colab 里**一键运行**（T4 GPU，约 5 分钟）
2. 生成 60K 条合成对话，涵盖 60 个话题（ greetings, feelings, food, light, bubbles...）
3. 从零训练 tokenizer 和模型
4. 上传 HuggingFace 或本地下载

合成数据的例子——每条对话都在教模型像一条鱼一样思考：

> You> are you hungry  
> Guppy> yes. always yes. i will swim to the top right now. i promise to eat all of it.

---

## 关键工程洞察

**1. 训练 LLM 不是什么魔法**  
这是作者最想传递的信息。GuppyLM 证明了：不需要 PhD，不需要百卡集群，不需要 thousand-dollar cloud bill。只要一个 notebook 和 5 分钟。

这对 AI 解决方案架构师意味着什么？当你在向团队解释 LLM 的工作原理时，GuppyLM 是一个完美的**可视化教学工具**——不是 PPT，不是论文，是一行行可以运行的代码。

**2. 小模型是理解大模型的最佳窗口**  
GuppyLM 的每个组件都能在笔记本上完整复现。你可以在这个规模上调试 attention 可视化、过拟合行为、tokenizer 效果，然后直观理解这些机制在 70B 规模下会如何表现。

**3. 合成数据 + 小模型 = 快速迭代**  
60K 对话，6 话题，纯合成数据。在真实大模型训练里，这对应的是数据工程 + RLHF + 规模化——但在这个规模，你可以快速实验、破坏、修复，建立直觉。

---

## 信源引用

- GitHub 仓库：https://github.com/arman-bd/guppylm
- HuggingFace 模型：https://huggingface.co/arman-bd/guppylm-9M
- 浏览器在线 Demo：https://arman-bd.github.io/guppylm/
- Colab 训练笔记：https://colab.research.google.com/github/arman-bd/guppylm/blob/main/train_guppylm.ipynb
- Colab 使用笔记：https://colab.research.google.com/github/arman-bd/guppylm/blob/main/use_guppylm.ipynb
- Medium 介绍文章：https://arman-bd.medium.com/build-your-own-llm-in-5-minutes-i-made-mine-talk-like-a-fish-e20c338a3d14
