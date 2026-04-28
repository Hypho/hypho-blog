---
title: '本地 LLM 推理：为什么我不推荐 Ollama，以及真正值得用的开源替代'
date: 2026-04-17T10:15:32+08:00
draft: false
tags:
  - 'Local LLM'
  - 'Open Source'
  - 'AI Infrastructure'
  - 'LLM Inference'
description: 'Ollama 是最流行的本地 LLM 推理工具，但它的问题被社区诟病已久。本文从许可证归属、性能差距、供应商锁定三个维度深度解析 Ollama 的历史问题，并给出 llama.cpp、LM Studio、Jan 等经过验证的开源替代方案实测对比。'
showToc: true
math: false
aliases:
  - /posts/2026-04-17-local-llm-ollama-llama-cpp/
---

## 引言：一个「只修皮毛」的工具为什么获得 16 万星

2023 年，Ollama 以「Docker for LLMs」的定位进入开发者视野——一行命令下载模型，本地跑起来。这种低门槛让它迅速积累了 [16.9 万 GitHub Stars](https://github.com/ollama/ollama)，成为本地运行大模型的事实标准。

然而，它的底层问题正在被更多开发者意识到：许可证归属争议长达一年未处理、自研后端性能反而低于 llama.cpp 30-50%、模型格式产生供应商锁定……这些问题在 Hacker News 上引发了大量讨论，[HN 热帖](https://news.ycombinator.com/item?id=47772725)当天获得 603 分。

本文不是「二选一」的观点稿，而是一次基于事实的深度拆解——**为什么 Ollama 的工程实践存在系统性缺陷，以及真正值得投入生产的替代方案是什么**。

---

## 背景：llama.cpp 才是本地 LLM 的真正引擎

要理解 Ollama 的问题，先要了解它依赖的底层技术。

[llama.cpp](https://github.com/ggml-org/llama.cpp) 由 Georgi Gerganov 于 2023 年 3 月用一个晚间编写，最初只是一个将 LLaMA 模型跑在消费级硬件上的 C++ 推理引擎。它的核心创新是 **GGUF 量化格式**——让数十亿参数的大模型能够在普通电脑的 CPU 和 GPU 上高效运行。

今天，llama.cpp 拥有：
- **104,116 Stars**，450+ 贡献者
- MIT 许可证，完全开源
- 2026 年 2 月，ggml.ai 项目并入 Hugging Face，确保长期可持续发展

可以说，没有 llama.cpp，就没有本地 LLM 生态的今天。

**问题是：Ollama 几乎从未承认这一点。**

---

## 问题一：长达 400 天的许可证争议

Ollama 于 2023 年 6 月公开，基于 MIT 许可证开源。然而，其二进制发布包中包含 llama.cpp 代码，却**从未附带 llama.cpp 要求的版权声明**——这是 MIT 许可证的唯一主要义务。

社区在 2024 年 3 月提交了 [GitHub Issue #3185](https://github.com/ollama/ollama/issues/3185)，明确指出这一问题。**Ollama 团队 400 多天没有任何回应**。

直到 2024 年 4 月，社区成员主动提交了 PR 直接添加归属声明，Ollama 才在 README 底部加了一行小字：

> "llama.cpp project founded by Georgi Gerganov"

而官方 PR 的回复更能说明问题：

> "We spend a large chunk of time fixing and patching it up to ensure a smooth experience for Ollama users… Overtime, we will be transitioning to more systematically built engines."

翻译：他们不打算给 llama.cpp 显式 credited，还要逐步替换掉它。

---

## 问题二：自研后端导致性能反而下降 30-70%

Ollama 真的做到了「更稳定的体验」吗？社区基准测试给出了截然不同的答案。

### 性能测试数据（来自多篇社区对比评测）

| 测试场景 | llama.cpp | Ollama | 差距 |
|---------|-----------|--------|------|
| GPU 推理（相同硬件，Qwen-3 Coder 32B） | 161 tokens/s | 89 tokens/s | **Ollama 慢 44%** |
| CPU 推理 | 基准值 | 低于基准 30-50% | **Ollama 慢 30-50%** |
| Qwen-3 Coder 32B 吞吐量 | ~70% 更高 | 基准 | **llama.cpp 胜出** |

性能差距的根源在于 Ollama 2025 年中做出的一次关键决策——**放弃直接使用 llama.cpp，转而基于 ggml 底层库自建推理后端**。

Ollama 官方理由是「llama.cpp 变动太快，企业用户需要稳定性」。然而实际结果截然相反：

- **结构化输出（structured output）支持损坏**
- **视觉模型（vision model）失效**
- **GGML assertion 崩溃**，在 llama.cpp 早已修复的版本中重现
- **GPT-OSS 20B 新版模型不支持**所需的新 tensor 类型

llama.cpp 作者 Georgi Gerganov 本人确认：Ollama 分叉并对 GGML 做了破坏性修改。

---

## 问题三：供应商锁定的模型格式

Ollama 下载的模型以**哈希文件名存储在自己的格式中**，无法被 llama.cpp、LM Studio 或其他 GGUF 工具直接使用。

用户如果想把在 Ollama 中下载的模型迁移到 llama.cpp，需要额外工作。这与「开放」的开源理念背道而驰——你下载的模型，实际上不是你的。

相比之下，llama.cpp 的模型文件（GGUF 格式）是真正的开放标准，任何支持 GGUF 的工具都可以使用。

---

## 问题四：误导性的模型命名

当 DeepSeek 在 2025 年 1 月发布 R1 模型族时，Ollama 做了一个微妙的操作：

将 **DeepSeek-R1-Distill-Qwen-32B**（由 Qwen 微调而来的蒸馏版本）直接在库中和 CLI 中标记为「DeepSeek-R1」。

实际运行效果与真正的 6710 亿参数 R1 天差地别，但下载量因为「DeepSeek-R1」这个标题暴涨。

GitHub Issues [#8557](https://github.com/ollama/ollama/issues/8557) 和 [#8698](https://github.com/ollama/ollama/issues/8698) 明确请求区分命名，两个 issue 均被关闭，**没有修复**。

直到今天，`ollama run deepseek-r1` 运行的仍然是小模型蒸馏版本。

---

## 问题五：2025 年 7 月的闭源 GUI 应用

2025 年 7 月，Ollama 发布了 macOS 和 Windows 的 GUI 桌面应用，但：

- 代码在**私有仓库**开发（github.com/ollama/app）
- **没有开源许可证**
- 二进制包中包含疑似 AGPL-3.0 依赖
- 下载页面把开源 MIT 链接放在旁边，给用户「这是开源工具」的印象

这是一个以开源闻名、享受社区红利的项目，在关键时刻选择关闭源代码。

---

## 真正的替代方案：开源本地 LLM 推理工具横评

| 工具 | 底层引擎 | 许可证 | GUI | 特色 |
|------|---------|--------|-----|------|
| **[llama.cpp](https://github.com/ggml-org/llama.cpp)** | 原生 | MIT | 可选（llama-server） | 性能最优，生态最广 |
| **[LM Studio](https://lmstudio.ai/)** | llama.cpp | 专有（免费） | ✅ | 图形界面完整，可视化调参 |
| **[Jan](https://jan.ai/)** | llama.cpp | AGPL | ✅ | 开源，local-first 设计 |
| **[koboldcpp](https://github.com/LostRuins/koboldcpp)** | llama.cpp | AGPL | ✅ | Web UI，配置选项丰富 |
| **[ramalama](https://github.com/containers/ramalama)** | 多引擎 | APL 2.0 | CLI | 容器化，明确注明上游依赖 |
| **[LiteLLM](https://github.com/BerriAI/litellm)** | 多后端 | MIT | 代理层 | OpenAI 兼容 API，统一路由 |
| **[llama-swap](https://github.com/ggml-org/llama.cpp)** | llama.cpp | MIT | — | 多模型编排，热加载 |

这些工具设置起来并不复杂。llama.cpp 官方提供 `llama-server`，自带 OpenAI 兼容 API 和 Web UI，配置上下文窗口和采样参数完全可控。

---

## 工程实践建议

### 1. 如果你追求**最高性能**：直接用 llama.cpp

llama.cpp 在所有硬件配置下都明显领先。它 2026 年 2 月并入 Hugging Face，生态有长期保障。

### 2. 如果你需要**图形界面**：LM Studio 或 Jan

LM Studio 提供完整的可视化调参界面，适合不想写命令行的场景。Jan 是 AGPL 开源替代，隐私优先设计。

### 3. 如果你需要**多模型统一管理**：llama-swap + LiteLLM

llama-swap 支持多模型热加载和动态切换，配合 LiteLLM 可以做统一的 OpenAI 兼容代理，按模型自动路由到不同后端。

### 4. 永远不要把模型文件锁在单一工具里

使用 GGUF 格式存储模型，任何工具都能读取。避免 Ollama 的哈希文件名格式——它会让你日后迁移困难。

---

## 结论

Ollama 的核心问题是**工程激励与开源社区期望的根本错位**：一边享受开源社区对 llama.cpp 的依赖带来的流量，一边拒绝透明承认这种依赖；在性能上自研后端反而开倒车；在许可证上绕过 MIT 要求；在产品上发布闭源应用。

真正值得关注的是 **llama.cpp 生态本身**——它才是本地 LLM 推理的根基，性能领先、许可证清晰、社区驱动。所有「llama.cpp 替代品」中，最值得投入的是直接使用 llama.cpp，或者在其基础上做用户体验封装的 LM Studio 和 Jan。

---

## 信源

- [Hacker News 讨论：The local LLM ecosystem doesn't need Ollama](https://news.ycombinator.com/item?id=47772725)
- [Ollama GitHub：Issue #3185 - 许可证归属争议](https://github.com/ollama/ollama/issues/3185)
- [Ollama GitHub：Issue #8557/#8698 - DeepSeek 命名误导](https://github.com/ollama/ollama/issues/8557)
- [llama.cpp GitHub（ggml-org）](https://github.com/ggml-org/llama.cpp) - 104,116 Stars，MIT 许可证
- [Ollama GitHub](https://github.com/ollama/ollama) - 169,199 Stars，MIT 许可证
- [LM Studio](https://lmstudio.ai/)
- [Jan](https://jan.ai/)
- [koboldcpp GitHub](https://github.com/LostRuins/koboldcpp)
- [ramalama GitHub](https://github.com/containers/ramalama)
- [LiteLLM GitHub](https://github.com/BerriAI/litellm)
