---
title: 'PyTorch Lightning 供应链攻击复盘：AI 训练依赖为什么不能只靠 pip install'
date: 2026-05-01T10:03:51+08:00
draft: false
tags:
  - 'MLOps'
  - 'Supply Chain Security'
  - 'PyPI'
  - 'Machine Learning'
  - 'DevSecOps'
description: 'PyTorch Lightning 的 lightning 包被曝出现供应链攻击，恶意版本可在导入时窃取凭证、云端密钥和 GitHub token。本文从 AI 训练依赖、PyPI 发布链路、锁文件、哈希校验、CI 隔离和应急轮换角度，分析机器学习团队如何把 Python 依赖安全做成工程流程，而不是事后补救。'
showToc: true
math: false
aliases:
  - /posts/2026-05-01-pytorch-lightning-supply-chain-security/
---

如果你在训练脚本里写过 `pip install lightning`，这次事件就不只是安全圈的新闻。它提醒的是一个更难听的事实：很多 AI 团队的“训练基础设施”，其实建立在一条几乎没人认真审计的依赖链上。模型代码、数据集、实验追踪、云端凭证都在同一个环境里跑，任何一个热门 Python 包被污染，攻击面都会比普通 Web 服务更肥。

HN 上这条 [Semgrep 披露](https://news.ycombinator.com/item?id=47964617) 很快冲到前排，原因也不复杂：被点名的是 Lightning 生态里的 `lightning` 包。Semgrep 的研究文章称，PyPI 上 `lightning` 2.6.2 和 2.6.3 在 2026 年 4 月 30 日被发布为恶意版本，导入时会执行隐藏在 `_runtime` 目录里的混淆 JavaScript payload，尝试窃取凭证、认证 token、环境变量和云端 secret，并带有 Shai-Hulud 风格的仓库投毒行为。原文细节见 [Semgrep: Shai-Hulud Themed Malware Found in the PyTorch Lightning AI Training Library](https://semgrep.dev/blog/2026/malicious-dependency-in-pytorch-lightning-used-for-ai-training/)。

我不想把这篇写成“某某包又中招了”的快讯。快讯今天看完，明天就忘。更值得拆的是：为什么 AI/ML 工程里的同类事故破坏力特别大？以及，一个现实团队到底应该改哪几件事。

先把边界说清楚。PyTorch Lightning 本身是一个真实、活跃且规模很大的开源项目，GitHub 仓库 [Lightning-AI/pytorch-lightning](https://github.com/Lightning-AI/pytorch-lightning) 有 3 万级 stars，近期仍有提交；PyPI 上当前可见的 [lightning 项目页](https://pypi.org/project/lightning/) 也显示稳定版本和维护者信息。这类事件的重点通常不是“项目没价值”，而是“发布链路或账号链路被污染后，价值越大的包越适合被当成入口”。

说白了，热门依赖就是最好的投递渠道。

AI 训练环境为什么更危险？第一，它天然带 secret。为了拉数据、写 S3、连实验平台、访问模型 API、推送镜像，训练机里经常有 `AWS_ACCESS_KEY_ID`、Hugging Face token、Weights & Biases key、GitHub token、数据库只读账号，甚至还有企业内部对象存储凭证。普通后端服务至少还会被平台团队逼着走 secret manager；很多研究环境则是 `.env`、notebook、shell history 混着来。

第二，训练脚本的“导入即执行”风险被低估了。Python 包安装阶段能执行构建脚本，导入阶段也能跑任意初始化逻辑。Semgrep 披露里特别刺眼的一点是：恶意代码不需要你调用某个奇怪 API，只要环境里安装了受影响版本，常规导入路径就可能触发。人话翻译：这不是“你点了钓鱼链接才中招”，而是“你正常跑训练就可能把钥匙交出去”。

第三，AI 项目的依赖树太宽。一个看起来简单的 fine-tuning 项目，下面可能挂着 PyTorch、Lightning、Transformers、tokenizers、datasets、加速库、日志库、可视化库、云 SDK。很多团队用 `pip install -U` 或者裸 `requirements.txt` 管开发环境，版本边界非常松。今天 CI 过了，不代表明天新机器重建环境拿到的是同一批 wheel。

这也是我一直不太喜欢把“可复现实验”只理解成随机种子和数据切分的原因。依赖不可复现，实验就不可复现；依赖不可验证，训练环境也谈不上安全。

工程上第一步不是买更贵的扫描器，而是停止裸装生产训练依赖。至少做到三件小事：锁版本、锁哈希、锁来源。

锁版本很好理解：不要写 `lightning>=2.0` 这种在生产训练镜像里自动漂移的约束。用 `pip-tools`、Poetry、uv 或内部构建系统生成 lockfile，把直接依赖和传递依赖都固定下来。更进一步，pip 支持 `--require-hashes`，可以要求每个包文件匹配预期 hash。这个方案麻烦，尤其在依赖多、平台多的时候会增加维护成本，但它解决的是最核心的问题：同一个版本号对应的文件，不能悄悄换。

PyPI 自己也在往这个方向补基础设施。官方的 [PyPI digital attestations 文档](https://docs.pypi.org/attestations/) 说明，PEP 740 相关机制允许发布者或第三方对包分发文件做数字证明，把具体 wheel/sdist 的内容摘要与发布身份绑定。别误会，attestation 不是银弹；它不能保证代码“没有恶意”，只能让你更清楚“这个文件是谁、通过什么流程发布的”。但在供应链排障里，这已经比肉眼看版本号强很多。

第二步是把训练环境和凭证环境拆开。最糟糕的模式是：CI 拉代码、安装依赖、跑测试、构建镜像、访问云资源，全在一个长寿命 token 下完成。更好的做法是分层：依赖解析和镜像构建发生在无生产 secret 的环境；真正需要访问数据的训练任务只拿最小权限、短期凭证；notebook 和交互式实验环境不要默认继承发布 token。

这听起来像安全八股，但对 AI 团队很实际。因为一旦训练环境泄漏，攻击者拿到的往往不是一台机器，而是一整套数据入口、模型制品和内部代码仓库入口。此前我写 [Agent Vault：用代理模式堵住 AI Agent 的凭证泄露风险](https://blog.hypho.cn/posts/agent-vault-open-source-credential-proxy/) 时也提过类似思路：不要把长期凭证直接塞进会执行不可信上下文的进程里。训练脚本不是 Agent，但风险结构非常像——都是“强能力进程 + 大量外部输入 + 容易携带 secret”。

第三步是给 CI 增加“依赖变更审查”这个关卡。很多团队代码 review 很严格，依赖 review 却很随意：`requirements.txt` 多了几十行，没人看；lockfile 变了几千行，直接点 approve。我的建议是把依赖升级当成小型变更发布：自动标出新增包、维护者变化、下载源变化、是否启用 Trusted Publishing、是否有 yanked/replaced 记录；高风险包升级单独跑沙箱测试，不要和业务代码混在一个 PR 里。

这里有个反直觉点：安全扫描只能发现“已知坏东西”，不能替你判断“这个升级是否应该发生”。比如一个热门 ML 包突然多了 postinstall 行为、突然引入 Node 执行链、突然换了上传方式，这些都未必立刻命中 CVE，却非常值得人工看一眼。

第四步是准备真正能用的应急动作。不是写在 Confluence 里没人看的事故流程，而是可以直接复制执行的那种：冻结依赖源；禁用受影响版本；轮换训练环境里出现过的 cloud key、GitHub token、模型平台 token；清理 CI cache 和 notebook 持久卷；检查近期由自动化账号推送的异常仓库、异常 release、异常 workflow；把镜像重新从干净 lockfile 构建一遍。

如果你用的是 GitHub Actions 或类似 CI，还要特别留意仓库写权限 token。Semgrep 文中提到的仓库投毒行为，真正可怕的不是创建一个奇怪名字的公开仓库，而是攻击者可能利用自动化 token 横向移动。对开源项目这会污染用户信任；对企业内部项目，这可能变成供应链二次传播。

那是不是以后别用 PyPI、别用 Lightning、所有东西都 vendor 进仓库？我觉得这不是现实答案。AI 工程的效率高度依赖开源生态，完全自建会把团队拖死。更合理的判断是：高频实验环境可以保持灵活，但生产训练、评测流水线、模型发布流水线必须像后端服务一样管理依赖。研究阶段追新没问题，交付阶段不能追新。

这里可以借用我在 [本地 LLM 推理引擎之争](https://blog.hypho.cn/posts/local-llm-ollama-llama-cpp/) 里反复强调的一个原则：越靠近生产边界，越要减少隐式行为。推理引擎如此，训练依赖也是如此。你不能一边要求模型产物可审计，一边允许构建环境每天从互联网随机拿一组新包。

最后给一个我认为比较务实的最小清单：

- 生产训练镜像禁止裸 `pip install -U`，必须从 lockfile 构建。
- 对关键依赖启用 hash 校验或内部包镜像，至少保证重建环境拿到同一份文件。
- CI 构建阶段不注入生产 secret；训练运行阶段使用短期、最小权限凭证。
- 依赖升级 PR 单独审查，新增传递依赖和发布元数据变化要可见。
- 对 PyPI、GitHub、云账号 token 建立轮换剧本，事故发生时先轮换再争论。
- 对 notebook、实验机、共享 GPU 服务器做凭证清点，别让历史 `.env` 变成攻击者的奖池。

这次 Lightning 事件未必是最后一次，也大概率不会是最严重的一次。AI 基础设施正在把越来越多高价值资产放进 Python 运行时：数据、模型、私有代码、云资源、评测结果。攻击者不需要理解你的模型结构，只要理解你的安装流程。

有点讽刺，但也很真实：很多团队花几周调一个 1% 的指标提升，却不愿花一天把依赖安装固定下来。等供应链攻击真的打进训练环境，再补这一天就太晚了。
