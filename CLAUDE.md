# CLAUDE.md

本文档为 Claude Code (claude.ai/code) 在本仓库中工作时提供指导。

## 核心原则：现代极简主义

- **内容优先**：所有设计变更必须以减少干扰为目标
- **视觉风格**：保持简洁、高留空、精致的视觉效果。避免花哨的动画或厚重的 UI 组件
- **字体排版**：优先使用系统原生高质量无衬线字体。保持行高 1.8 和字间距 0.015em

## 技术栈与标准

- **框架**：Hugo (Extended 版本) + PaperMod 主题
- **数学公式**：KaTeX 全局启用
  - 配置位于 `layouts/partials/extend_head.html`
  - 行内公式：`$ ... $`，块级公式：`$$ ... $$`
- **CSS 架构**：自定义样式添加到 `assets/css/extended.css`，**禁止修改主题核心文件**
- **部署**：通过 GitHub Actions 自动部署 (`.github/workflows/hugo.yml`)
  - **关键**：`hugo.yaml` 和工作流中的 `baseURL` 必须使用 **HTTPS**

## SEO 与 GEO

- **AI 友好**：维护 `static/llms.txt` 包含最新的站点结构
- **元数据**：每篇文章必须在 Front Matter 中包含 `description` 和 `tags`
- **社交图片**：默认封面图策略在 `hugo.yaml` 的 `params.images` 中管理

## 常用命令

### 开发
- **构建站点**：在项目根目录运行 `.\bin\hugo.exe` 生成静态站点
- **本地预览**：`.\bin\hugo.exe server`（加 `-D` 包含草稿）
- **创建新文章**：`.\bin\hugo.exe new posts/<文章名称>.md`

### 部署
- 推送到 `master` 分支会触发 GitHub Actions 自动部署

## 项目结构

- `content/posts/` - 博客文章（Markdown + Front Matter）
- `content/pages/` - 静态页面
- `inbox/` - 草稿箱，用于临时写作素材（文章定稿确认后，提议删除对应的 inbox 内容）
- `static/` - 静态资源（图标、图片、llms.txt）
- `themes/PaperMod` - 主题子模块

## 配置与定制

- **主配置**：`hugo.yaml` - 包含菜单、社交链接、数学支持等所有站点配置
- **数学与图表**：通过 `layouts/partials/extend_head.html` 加载 KaTeX 和 Mermaid
- **自定义 CSS**：`assets/css/extended.css`
- **头部扩展**：`layouts/partials/extend_head.html`

## Front Matter 格式

```yaml
---
title: '文章标题'
date: 2026-03-08T19:20:00+08:00
draft: false
tags: ['标签1', '标签2']
description: '用于 SEO 和社交分享的描述'
showToc: true
math: true  # 为本文启用数学公式渲染
---
```

## 重要文件

- `hugo.yaml` - 主配置文件
- `layouts/partials/extend_head.html` - KaTeX 和 Mermaid 配置
- `assets/css/extended.css` - 自定义样式
- `.github/workflows/hugo.yml` - CI/CD 流程
- `static/llms.txt` - 供 LLM 消费的 AI 友好站点结构

## 工作流程规范

- **人工确认**：未经用户明确确认，禁止删除文件（如草稿、资源文件）。删除前务必验证
- **语言**：中文博客，所有博客文章和主要内容必须使用中文
- **无冗余**：添加新功能前，检查是否违背"极简主义"目标
- **图标**：不要删除 `static/` 中的占位图标，除非用永久品牌替换
- **构建**：推送到 GitHub 前，始终运行 `.\bin\hugo.exe` 本地验证
- **提交信息**：使用简洁有意义的消息（如 "feat: add katex support"、"fix: adjust line spacing"）
- **分支**：此个人项目可直接推送到 `master`
