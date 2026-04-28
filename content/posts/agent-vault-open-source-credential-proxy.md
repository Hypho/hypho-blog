---
title: 'Agent Vault：用代理模式堵住 AI Agent 的凭证泄露风险'
date: '2026-04-24T10:05:53+08:00'
draft: false
tags: ['AI Agent', 'Security', 'Open Source', 'DevOps']
description: 'AI Agent 是非确定性系统，传统的"把密钥直接给程序"模式在它们面前完全失效。Infisical 开源的 Agent Vault 用 HTTP 代理思路重新定义凭证管理——密钥从不离开金库，Agent 只管调用 API。本文深度解析其设计原理、架构细节和实际效果。'
showToc: true
math: false
aliases:
  - /posts/2026-04-24-agent-vault-open-source-credential-proxy/
---

如果你在生产环境跑过 AI Agent，大概率遇到过一个头疼的问题：**Agent 怎么安全地访问那些需要 API Key 的服务？**

传统方案很简单：把密钥配置在环境变量里，Agent 启动时读取。但这套逻辑是给"确定性程序"设计的——程序行为可预测，不会被外部指令诱导去做你没想过的事。

AI Agent 不一样。它们是非确定性的，能被 prompt injection 诱导，能被恶意网页操纵，能在 RAG 流程里接收有害指令。**密钥一旦进了 Agent 的上下文，就等于随时可能被抽走。**

这是一个真实存在的威胁，不是理论推演。Infisical 最近的博客详细描述了攻击路径：攻击者通过文档注入、恶意网页或工具调用让 Agent "主动"把环境里的密钥发到攻击者控制的端点。哪怕你上了多层 guardrails，也没有办法保证 Agent 绝对不泄露。

## 传统解法为什么不够用

业界的应对思路大概分三类：

**① 短命凭证（Short-lived Tokens）**

OAuth2 的 access/refresh token 模式，API 返回临时凭证，过期自动失效。配合自动化密钥轮换，攻击者拿到的那串字符很快变成废纸。

听起来合理，但本质上只是**降低窗口期**，没有解决根本问题——凭证依然会泄露，攻击者只要在失效前用完就赚了。

**② 防火墙和网络隔离**

只允许 Agent 访问特定 IP 段，不允许出站直连。攻击者通过 Agent 发起请求，同样会经过那些被允许的端点，该泄露还是泄露。

**③ 自行实现凭证代理**

Anthropic 的 Managed Agents 架构、Vercel 的 credential brokering、Cloudflare 的 outbound workers，都走了同一条路：Agent 的请求经过一个代理层，由代理负责在请求发出前把凭证注入，Agent 自己从不直接接触密钥。

这条路是对的，但每家公司都得自己造轮子。

## Agent Vault 的思路

[Infisical 新开源的 Agent Vault](https://github.com/Infisical/agent-vault) 把这条路做成了通用产品。它的核心设计原则只有一条：**Agent 永远拿不到金库里的密钥，只能通过代理间接使用。**

实现方式很巧妙——它本质是一个本地 HTTPS 透明代理。Agent 把请求发向目标 API，流量经过 Agent Vault 代理时，代理在网络层注入正确凭证，然后转发出去。整个过程 Agent 感知不到凭证的存在，它只是正常调用 `fetch("https://api.github.com/...")` 而已。

用他们自己的话说：**Brokered access, not retrieval**。

## 核心架构

Agent Vault 跑起来之后会暴露两个端口：

- `14321`：HTTP API，用于管理金库、创建会话、配置凭证
- `14322`：TLS 加密的透明 HTTPS 代理，Agent 所有的出站请求都经过这里

工作流程是这样的：

```
Agent 调用 API（如 GitHub API）
    ↓
请求发往目标域名（如 api.github.com）
    ↓
流量经过 localhost:14322（Agent Vault 透明代理）
    ↓
代理根据会话中配置的凭证，在网络层注入 Authorization header
    ↓
代理将请求转发到真实目标
    ↓
目标服务收到带凭证的请求，返回数据
    ↓
代理将响应透传给 Agent
```

密钥**从未出现在应用层**，Agent 进程的内存里从来没有那串 secrets。

## 实际怎么用

对于本地 Agent（Claude Code、Cursor、Codex、OpenCode 等），用 CLI 启动就行：

```bash
agent-vault run -- claude
```

`agent-vault run` 会创建一个 scoped session，设置 `HTTPS_PROXY` 和 CA 证书环境变量，然后启动 Agent 进程。之后 Agent 所有 HTTPS 流量都经过代理，凭证注入全自动。

如果 Agent 是跑在容器里（Docker、Daytona、E2B 等沙箱环境），Agent Vault 提供了 TypeScript SDK：

```typescript
import { AgentVault, buildProxyEnv } from "@infisical/agent-vault-sdk";

const av = new AgentVault({
  token: "YOUR_TOKEN",
  address: "http://localhost:14321",
});

const session = await av.vault("default").sessions.create({
  vaultRole: "proxy"
});

// 获取代理配置和环境变量，传入沙箱
const env = buildProxyEnv(session.containerConfig!, certPath);
const caCert = session.containerConfig!.caCertificate;

// 在沙箱内设置好环境变量，Agent 正常调用 API
// fetch("https://api.github.com/...") — 凭证自动注入，Agent 不可见
```

这意味着无论 Agent 跑在哪里，只要能设置环境变量，就能接入 Agent Vault。

## 安全细节

Agent Vault 在存储层也做了加固：凭证用 AES-256-GCM 加密存储，数据加密密钥（DEK）由 master password 通过 Argon2id 派生。**轮换 master password 不需要重新加密所有凭证**，因为 DEK 本身被密码保护，密码变了只影响 DEK 的 wrapping。

不想用密码也行，适合 PaaS 环境的 passwordless 模式了解一下。

代理层还保留了完整的请求日志（method、host、path、status、latency、涉及的凭证 key），方便审计。**请求体、header、query string 不记录**，避免日志本身成为新的敏感数据源。

## 选型建议

坦白说，Agent Vault 不是银弹。它的设计针对的是**需要调用外部 API 的 AI Agent**这个具体场景——如果你在跑的 Agent 根本不访问外部服务，这个方案就用不上。

但如果你在生产环境部署了 AI coding agent（Claude Code、Cursor 等），或者在用 RAG pipeline 让 Agent 访问各种 SaaS API，**Agent Vault 基本上是目前开源世界里最完整的解法**。

它比自行维护一个凭证代理服务省事得多，Infisical 本身处理着数十亿次密钥调用的线上流量，方案经过了实际生产的检验。378 个 GitHub stars、22 个 fork、昨天刚有 commit，活跃度也在线。

对于还在用"把 API Key 写进 .env 文件然后塞给 Agent"这种方案的团队，这是一个值得评估的升级路径。

---

**信源：**

- [Agent Vault GitHub 仓库](https://github.com/Infisical/agent-vault)（MIT 协议，Infisical 开源）
- [Agent Vault 官方文档](https://docs.agent-vault.dev)
- [Agent Vault 介绍博客](https://infisical.com/blog/agent-vault-the-open-source-credential-proxy-and-vault-for-agents)（详细阐述了 credential exfiltration 威胁模型和解决方案设计思路）
