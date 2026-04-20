---
title: 'TRELLIS.2 移植到 Mac：没有 NVIDIA 也能跑图片转 3D 模型'
date: 2026-04-20T10:13:50+08:00
draft: false
tags:
  - '3D Generation'
  - 'Apple Silicon'
  - 'Machine Learning'
  - 'Open Source'
description: '微软 TRELLIS.2 是 CVPR 2025 Spotlight 的 SOTA 图片转 3D 技术，官方只有 CUDA 版本。近日有人将其完整移植到 Apple Silicon，M4 Pro 上 3.5 分钟生成 40 万顶点网格，且全程无需 NVIDIA GPU。本文解析移植思路、关键技术细节，以及为什么 Apple Silicon 的统一内存架构在这类任务上比离散 GPU 更合适。'
showToc: true
math: false
---

如果你只有一台 Mac 电脑，想从单张图片生成 3D 模型——直到今天，这基本上是个伪需求。市面上最好的开源图片转 3D 技术，几乎全部围绕 NVIDIA CUDA 构建，买不到合适的硬件就等于玩不了。

这个局面正在被打破。

TRELLIS.2 是微软 2025 年发表在 CVPR 的论文提出的图片转 3D 方法，在 GitHub 上有 1.2 万星，官方仓库清一色 CUDA 代码。近日有个开发者把它完整移植到了 Apple Silicon，M4 Pro 上 3.5 分钟就能生成一个 40 万顶点的网格模型，全程跑在苹果自研芯片上，不需要半块 NVIDIA 显卡。

## 移植的核心思路：替换掉 CUDA 依赖

TRELLIS.2 依赖好几个 CUDA 专用的库，官方版本开箱即用但根本不支持 Metal。这不是简单的编译选项问题，而是代码里大量硬编码了 `.cuda()` 调用和 CUDA 核函数。

移植的思路很直接：**找到每一个 CUDA 依赖，用 PyTorch 原生功能或纯 Python 实现替换**。

主要替换关系如下：

| 原始（CUDA） | 移植版本 | 用途 |
|---|---|---|
| `flex_gemm` | `backends/conv_none.py` | 子流形稀疏 3D 卷积，通过 gather-scatter 实现 |
| `o_voxel._C` hashmap | `backends/mesh_extract.py` | 从双体素网格提取网格面 |
| `flash_attn` | PyTorch SDPA | 稀疏变换器的注意力机制 |
| `cumesh` | Stub（跳过） | 网格孔填充与简化 |
| `nvdiffrast` | Stub（跳过） | 可微分光栅化（仅影响纹理导出） |

稀疏 3D 卷积的替换是个技术亮点。原始 `flex_gemm` 做的是子流形稀疏卷积——3D 生成任务中只有物体表面有数据，不需要对整个空间做卷积。移植版本用 Python 构建空间哈希表，对每个活跃体素收集邻域特征，通过矩阵乘法应用权重，再把结果 scatter 回去。neighbor maps 做了缓存避免重复计算。

Mesh 提取部分也很有意思。双体素网格到网格面的转换原来用的是 CUDA hashmap，移植版本用 Python 字典重写了 `flexibe_dual_grid_to_mesh` 函数，逻辑完全对应，只是从 GPU 并行换成了 Python 循环。

整个移植不需要任何 fork——直接 clone 后跑 `setup.sh`，脚本会自动克隆原始 TRELLIS.2 仓库并应用 patch。

## Apple Silicon 统一内存：被忽视的硬件优势

这个移植能跑的前提，是 Apple Silicon 的统一内存架构（Unified Memory Architecture）。

在离散 GPU（NVIDIA/AMD）上，CPU 和 GPU 各自有独立显存，数据跨 PCIe 总线传输。把数据从 CPU 内存拷贝到 GPU 显存是不可避免的开销，对于大模型来说这个拷贝时间相当可观。

Apple Silicon 的 CPU 和 GPU 共享同一块物理内存，没有 PCIe 总线的概念。**同一个指针，CPU 能读，GPU 也能读，不需要任何拷贝**。这就是为什么苹果官方一直在推 MLX 机器学习框架，而 MLX 的核心设计思路就是最大化利用统一内存减少数据搬运。

TRELLIS.2 生成的是稀疏 3D 表示，中间激活值在 M4 Pro 上直接复用统一内存，不需要反复搬运。这一点在离散 GPU 上反而是劣势——PCIe 带宽远低于显存带宽，频繁的小数据搬运会让稀疏操作的 overhead 更大。

## 实测效果：够用，但有取舍

根据 README 的数据，M4 Pro（24GB 统一内存）上：

- 单张图片生成 400K+ 顶点的 OBJ/GLB 模型，约 3.5 分钟
- 支持三种 pipeline 分辨率：512、1024、1024 级联
- 首次运行需要从 HuggingFace 下载约 15GB 模型权重

主要限制：
- 需要 24GB+ 统一内存（M4 Pro 或更新）
- 官方 CUDA 版本可能有更好的网格质量
- `nvdiffrast` 的跳过意味着纹理导出功能暂不可用

但生成时间和质量对于大多数应用场景已经相当可用——3.5 分钟出一个可编辑的 3D 模型，比云端 API 便宜多了，而且完全本地运行，数据不离开机器。

## 为什么这件事值得关注

过去一年开源图片转 3D 的进展很快，但生态几乎被 NVIDIA 垄断。TRELLIS.2 的 Apple Silicon 移植不是简单的移植工作——它意味着：

1. **苹果生态的 AI 应用开发门槛降低了**。以前想跑这类模型必须买台带 NVIDIA 显卡的机器，现在一台 M4 MacBook Pro 就行
2. **本地推理的边界在扩展**。22B 参数级别的模型用统一内存做推理已经开始可行，这个方向会越来越宽
3. **PyTorch 的 MPS 后端在成熟**。Sparse convolution、SDPA 这些操作以前在 MPS 上支持很差，现在已经能完整实现一个 SOTA 论文的核心算法

如果你在 Mac 上做 3D 内容创作、游戏开发或 AR/VR 应用，这个工具链值得放进你的技术栈。

**项目地址：**
- Apple Silicon 移植版：[github.com/shivampkumar/trellis-mac](https://github.com/shivampkumar/trellis-mac)
- 微软官方原始仓库：[github.com/microsoft/TRELLIS](https://github.com/microsoft/TRELLIS)（12k stars，CVPR 2025 Spotlight）
