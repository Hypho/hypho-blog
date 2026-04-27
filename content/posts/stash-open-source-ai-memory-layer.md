---
title: '每个 AI Agent 都在重复昨天的自己：一个开源记忆层想要改变这个'
date: 2026-04-27T10:11:06+08:00
draft: false
tags: ['AI Agent', 'LLM', 'Open Source', 'Memory', 'MCP']
description: '每个 LLM 对话都是从零开始——你反复解释你是谁、项目背景是什么、之前踩过什么坑。下一次对话，AI 还是同样犯错。Stash 是一个开源的记忆基础设施，通过 8 阶段认知管道把 AI 的每一次对话经验转化为结构化知识，形成知识图谱和自我模型，让 Agent 在多轮对话中真正"记住"并学习。'
showToc: true
math: false
---

你有没有这种感觉：每天早上醒来，前一天学的东西大部分都忘了？

LLM 就是这样工作的。

每个对话 session，模型都是从零开始。它不记得你是谁，不记得你上次做了什么决定，更不记得那个方案三个月前就试过并且失败了。你花 20 分钟解释背景，下一个 session 又得重来一遍。

这不是 AI 的 bug——这是架构限制。大多数 Agent 的"记忆"，就是把整段对话历史塞进 prompt，靠上下文窗口撑着。贵、慢，而且换一个新 session 照样失忆。

**Stash** 想要解决这个问题。它的 slogan 很直接：**Your AI has amnesia. We fixed it.**

## 这个项目是做什么的

[Stash](https://github.com/alash3al/stash) 是一个开源的持久化记忆层，专门给 AI Agent 用。它不是一个聊天机器人，而是一个基础设施——在 Agent 和外部世界之间加了一层认知处理管道。

核心思路：**Episodes become facts. Facts become patterns. Patterns become wisdom.**

AI 的每一次对话、每一个决定、每一次成功和失败，都被记录下来，经过一个 8 阶段的管道，转化成结构化的知识。事实与事实之间建立关联，关联形成模式，模式沉淀为真正的理解。

```
原始对话
    ↓
Episode 记录（原始事件）
    ↓
Fact 提取（去掉了时间戳和情绪的事实）
    ↓
Relationship 建立（事实之间的连接）
    ↓
Pattern 检测（反复出现的模式）
    ↓
Goal Tracking（目标状态）
    ↓
Failure Pattern（失败教训）
    ↓
Hypothesis & Confidence（假设与置信度衰减）
    ↓
Wisdom（长期知识）
```

这个管道是**增量**的——每次运行只处理新数据，不会重复劳动。

## 它跟 RAG 不一样

你可能听说过 RAG（Retrieval Augmented Generation）。Stash 官方文档里有一段话说得很清楚：

> RAG 是一个聪明的搜索算法，但它不是记忆。它不记得你的对话，不学习，不了解你。每次问答都是从零开始——只是一个更高级的文件搜索引擎。

Stash 学的是你 Agent 经历过的一切：对话、决定、成败。它不需要你写任何东西，它自己从经验里推断出来。

本质上，RAG 是**搜索过去的文档**，Stash 是**记住过去的经历**。一个是图书馆，一个是经验。

## MCP 原生支持

Stash 通过 [MCP（Model Context Protocol）](https://modelcontextprotocol.github.io/introduction) 提供服务，任何支持 MCP 的 Agent 都可以直接接入。

```bash
# Docker 一键启动
git clone https://github.com/alash3al/stash.git
cd stash
cp .env.example .env   # 填入你的 API key 和模型
docker compose up
```

支持的 Agent 包括：Claude Desktop、Cursor、Windsurf、Cline、Continue、OpenAI Agents、Ollama、OpenRouter——只要支持 MCP 就能用。

它提供 **28 个工具**，覆盖从最基础的 `remember`（记住）和 `recall`（回忆）到高级的因果链推理、矛盾检测、假设管理。

## Namespace 层级记忆

最有意思的设计是 **Namespace 层次结构**。

每个 Agent 可以有多个命名空间，比如 `/self`（自我认知）、`/projects/stash`（某个项目的上下文）、`/projects/cartona`。读取 `/projects` 会自动包含下面所有子路径的记忆。

配合 `init` 命令，Stash 会自动创建 `/self` 命名空间，Agent 用自己的记忆层来构建自身能力、局限和偏好的模型——**Agent 知道自己知道什么，也知道自己不知道什么**。

## 实际效果

根据项目在 [LoCoMo-10](https://github.com/snap-research/locomo) 基准上的测试（1534 个 QA 对，10 个多轮对话），Stash 实现了 **59% 的 Recall@5**，比 Zep Cloud 的 28% 高出一倍多。

当然，这个数字只是一个基准。真正有价值的是：你的 Agent 不会再在同一个地方摔倒两次。

## 选型建议

如果你在搭建需要多轮协作的 Agent 系统，比如：

- 需要跨 session 保持上下文的技术助手
- 研究 Agent（需要积累文献阅读记忆）
- 代码生成 Agent（需要记住项目规范和历史决策）

Stash 值得一试。它的核心优势是：**不需要改动 Agent 本身的代码，只需要加一层 MCP 集成**。

对于需要完全私有化的场景，它支持 Ollama 本地模型 + PostgreSQL + pgvector，完全离线可用。

但需要注意：Stash 目前还很新（2026-04-24 创建，287 stars），8 阶段管道的实际效果需要你在真实项目中验证。如果你的 Agent 场景比较简单，可能不需要这么重的记忆基础设施。

---

**信源：**

- Stash GitHub: https://github.com/alash3al/stash
- Stash 官网: https://alash3al.github.io/stash/
- HN 讨论: https://news.ycombinator.com/item?id=44133706
- LoCoMo-10 基准: https://github.com/snap-research/locomo
