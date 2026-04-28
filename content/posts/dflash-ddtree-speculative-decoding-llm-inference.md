---
title: '单卡 207 tok/s：DFlash + DDTree 让 Qwen3.5-27B 在 RTX 3090 上跑出推理新纪录'
date: '2026-04-21T10:05:00+08:00'
draft: false
tags: ['LLM Inference', 'Speculative Decoding', 'CUDA', 'Qwen', 'Consumer GPU']
description: 'DFlash 块扩散草稿 + DDTree 树验证，在单张 RTX 3090 上把 Qwen3.5-27B Q4_K_M 推到 207 tok/s，比自回归解码快 5.46 倍。Lucebox 项目开源了首个 GGUF 版本的 DFlash 实现，揭开消费级 GPU 跑大模型推理的新思路。'
showToc: true
math: false
aliases:
  - /posts/2026-04-21-dflash-ddtree-speculative-decoding-llm-inference/
---

一个 27B 参数的大模型，在一张 2021 年买的游戏显卡上能跑多快？

Lucebox 团队给出了一个让很多人没想到的数字：**207.6 token/s**。用的还是 Qwen3.5-27B 官方模型，不是蒸馏，不是 INT8 量化残血版——就是 Q4_K_M 量化版本，目标加草稿模型全部加载在一张 24 GB VRAM 的 RTX 3090 上。

这个成绩靠的不是等英伟达下一代消费级显卡，而是对**解码算法本身**动刀子。

## 为什么自回归解码是瓶颈

大多数人聊 LLM 推理优化，会先想到量化、KV cache 压缩、batch 并行。但对单卡消费级 GPU 来说，这些都已经做到头了——Q4_K_M 量化能压缩到约 16 GB，再压下去效果肉眼可见地降。

问题出在**自回归解码本身**。每生成一个 token，GPU 要完整跑一遍 27B 参数的前向传播。27B 参数在 Q4_K_M 下大约 16 GB，VRAM 带宽是 936 GB/s——每次解码都要把这 16 GB 从显存读一遍，理论带宽利用率撑死不到 20%。这是机械式的物理限制，不是软件优化能绕过去的。

speculative decoding（投机解码）解决的就是这个问题：用一个**小草稿模型**一次生成多个候选 token，再用**大模型**一次验证整串。如果草稿猜得准，大模型只跑一次就能吐出五六个 token，GPU 计算资源用得更充分。

## DFlash：块扩散草稿，比 Chain EAGLE 更容易命中

主流投机解码方案是 EAGLE（及其 chain 版），草稿模型做自回归预测，每步大约能接受 3 个 token。**DFlash（2026）** 换了个思路：用**块扩散（block diffusion）** 做草稿——一个 5 层非因果的去噪网络，同时预测多个位置，而不是逐个生成。

效果如何？看数字：

| 任务 | AR 基线（tok/s） | DFlash+DDTree（tok/s） | 加速比 | 平均接受长度 |
|------|:----------------:|:---------------------:|:------:|:------------:|
| HumanEval | 37.78 | **129.52** | **3.43×** | 8.31 |
| Math500 | 37.71 | **110.51** | **2.93×** | 7.04 |
| GSM8K | 37.65 | **96.15** | **2.55×** | 6.14 |

AR 基线是 Qwen3.5-27B Q4_K_M 在 RTX 3090 上的原生速度，也是 llama.cpp、SGLang AWQ 等框架在同一硬件上的天花板。DFlash 把这个天花板一口气推到 3 倍以上——而且**平均每次验证接受 8 个 token**，比 EAGLE 的 ~3 高了将近两倍。

但这里有个工程问题：官方 DFlash 实现跑在 BF16 下，需要 60+ GB VRAM 的 B200 专业卡才能装下目标+草稿+验证树。消费级 GPU 想用，得从头移植。

## GGUF 移植：Q4_K_M 是 24 GB 的最优解

Lucebox 的贡献就是这个移植。他们选了 **Q4_K_M 量化目标（~16 GB）+ BF16 草稿（~3.46 GB）+ DDTree 验证树（budget=22）**，刚好能在 24 GB VRAM 里塞下整个推理链路。

GGUF 是 llama.cpp 的量化格式，在消费级 GPU 上生态最成熟。选择 ggml 作为运行时而非 libllama，是因为 ggml 有**原生 Gated DeltaNet CUDA kernel**，不需要额外改硬件抽象。

整个移植约 2000 行 C++/CUDA 代码，fork 了 llama.cpp 增加了三个 tree-mode 操作：

```cpp
ggml_ssm_conv_tree      // SSM 卷积树结构
ggml_gated_delta_net_tree        // Gated DeltaNet 树验证
ggml_gated_delta_net_tree_persist // 持久化状态
```

硬编码了 Qwen3.5-27B + Qwen3.5-27B-DFlash 这一个模型组合，没有通用性——但消费级单卡跑 27B 规模，这个组合恰好是最有价值的一个。

## 128K context 怎么塞进去

原生 DFlash 在 128K 上下文时会爆显存，因为 KV cache 太大。Lucebox 的解法是**Q4_0 KV cache + 滑动 target_feat ring**（4096 槽位）：

- Q4_0 比 BF16 省 8 倍显存
- Sliding window 限制历史 KV 长度
- target_feat 做 ring buffer，超长 prompt 自动驱逐旧状态

代价是 Acceptance Length 约下降 3%，但换来了**128K context 稳定在 24 GB 内**，128K 模式下仍能跑出 ~135 tok/s。这对需要超长上下文的应用（代码库分析、长文档对话）是实打实的突破。

## 关键洞察：消费级 GPU 的 LLM 推理还没到头

看完 Lucebox 的实现，有几个工程结论值得关注：

**1. 量化不是终点，算法才是**

Q4_K_M 已经把 27B 压到 16 GB，这个路径的边际收益越来越小。但在 Q4_K_M 基础上加 DFlash，能在不降低模型质量的前提下再快 3 倍——这说明解码算法的优化空间远没有到头。下一个突破点很可能是**草稿模型和目标模型的协同训练**，而不是继续压缩精度。

**2. GGUF 生态正在变厚**

Llama.cpp 的 GGUF 量化格式原本是"省显存"的妥协方案，但随着 ggml 支持更多的 CUDA kernel（SSM、Gated DeltaNet），它在某些场景下已经不只是"能用"，而是"最优解"。这对想在消费级硬件上做推理优化的开发者是好消息——不需要自己写 CUDA kernel，现成的生态已经能跑起来。

**3. 单卡 27B 的实用边界在拓宽**

207 tok/s 意味着什么？实际体验大概是：每秒能吐出大约 150-200 个中文词，或者 300-400 个英文词。这个速度已经能支持**实时交互式 AI 助手**——不是等几秒才回一个字，而是几乎感知不到延迟的流式响应。加上 128K 上下文，单卡 27B 能做的事比去年这个时候多了很多。

## 快速上手

项目在 GitHub，clone 下来大概需要知道这几步：

```bash
# 1. 克隆（含 submodule）
git clone --recurse-submodules https://github.com/Luce-Org/lucebox-hub
cd lucebox-hub/dflash

# 2. 编译（CUDA 12+, RTX 3090/sm_86）
cmake -B build -S . -DCMAKE_CUDA_ARCHITECTURES=86 -DCMAKE_BUILD_TYPE=Release
cmake --build build --target test_dflash -j

# 3. 下载模型（约 20 GB）
huggingface-cli download unsloth/Qwen3.5-27B-GGUF Qwen3.5-27B-Q4_K_M.gguf --local-dir models/
huggingface-cli download z-lab/Qwen3.5-27B-DFlash model.safetensors --local-dir models/draft/

# 4. 运行
python3 scripts/run.py --prompt "def fibonacci(n):"
```

完整的 benchmark 结果在 [RESULTS.md](https://github.com/Luce-Org/lucebox-hub/tree/main/dflash/RESULTS.md)，包括不同模型、不同 context 长度下的吞吐和内存数据。

## 信源

- Lucebox DFlash 实现：https://github.com/Luce-Org/lucebox-hub
- DFlash 论文：https://arxiv.org/abs/2602.06036
- DDTree 论文：https://arxiv.org/abs/2604.12989
- Lucebox 官方博客：https://lucebox.com/blog/dflash27b
- Qwen3.5-27B GGUF 量化模型：https://huggingface.co/unsloth/Qwen3.5-27B-GGUF
- Qwen3.5-27B-DFlash 草稿模型：https://huggingface.co/z-lab/Qwen3.5-27B-DFlash
