---
title: "DeepSeek Harness 使用指南：从跑通到可复现"
date: 2026-11-10T00:28:32+08:00
tags: ["DeepSeek", "AI Agent", "LLM 工程实践"]
author: "aatianya"
---

DeepSeek Harness（命令 `dsh`）是 DeepSeek AI 官方开源的 agent harness，MIT 协议。它的定位只有一句话：Agent = Model + Harness。模型负责推理，harness 负责让 agent 理解环境、调用工具、在真实环境里持续工作。如果你已经在用 DeepSeek 模型 API，想把它接进 agent 循环，这篇文章讲的是怎么跑通，以及哪几个配置真正决定效果。

## 它补的是模型之外的那层

官方把模型、工具、skills、会话、沙箱、存储、循环、调度、UI 全部做成可替换重组的插件。

它与「直接调 OpenAI 兼容接口」不是二选一：harness 本身就能把自定义 OpenAI 兼容端点接进来（填 Provider ID、基础 URL、协议三选一），所以是「包一层」而非「替代」；官方没有给出两者的量化对比。

harness 层自带三样东西：受守卫的工具流水线、按 step 追加已接纳事实的会话日志、协作式取消。

## 最小可用：装、配、跑通第一条

两条官方路径：

- Node 路线：装 Node.js（npm 包声明 `node >= 24.9.0`），只需一条命令 `npx @deepseek-ai/dsh web`，默认在 `http://127.0.0.1:3080` 起 Web UI，加 `--no-open` 可只起服务，不自动打开浏览器。
- Python 路线：前置 Python 3.10+ 与 Git，`python -m pip install deepseek-harness-sdk`，自带原生运行时，不需要系统 Node.js。

鉴权：在「设置 → 模型」里填 DeepSeek API Key 并保存，路由立即生效，无需重启；密钥只写不可读，存在 `$DSH_HOME/.credentials.yaml`。

一个易漏前置：必须先选中工作区，否则会话输入框不可用。

官方 quickstart 的第一个任务就是一句 `Summarize this repository and identify its main packages.` agent 可读写工作区文件、跑命令、委派子任务、维护计划，需审批的操作会先问。

## 决定效果的三个开关

配置集中在 `cordis.patch.yml` 的 profile patch，改完下一次请求生效，不需要重启。

- 上下文与输出上限：`defaultContextWindow` 默认 1,000,000；`defaultMaxTokens` 默认 128,000。
- 重试：`retryPolicy` 省略时是 normal 模式，最多重试 5 次，可改 `{mode: normal, maxRetries: 3}`。
- 并发：`maxParallelToolCalls` 默认 10，设为 1 即串行；它是 volatile 字段，热改后从下一批工具组生效。

另有两件事值得单独记：

- 上下文裁剪默认不启用，要挂载 `@deepseek-ai/dsh-compaction-basic` 与 `@deepseek-ai/dsh-command-compact` 才有自动压缩和 `/compact`；压缩只把较早历史换成摘要，被遮蔽内容仍留在会话日志里，所以回放能确定性地重现。
- 没有内置轮次预算。官方明说「工具调用或 steering 会让当前轮次继续」，要限流，就得自己在 `agent/turn-stopping` 等扩展点上取消。

## 取舍与上手清单

三个已知坑：

- 无稳定版：npm 上最新版本是 `0.2.2-rc.1`，41 个已发布版本全部是 prerelease，README 直接警告「developer preview and iterating rapidly. THERE WILL BE COMPATIBILITY-BREAKING CHANGES.」
- 长会话成本（用户报告，非官方数据）：有用户解析 100 个会话日志，累计上下文 28.3 亿 token，原因是每个 step 重发整个上下文，累计消耗随步数近似二次增长。
- 安全：SAFETY.md 明确「未经过安全审计，不得视为安全或生产可用」，沙箱与审批只能降低风险，不保证隔离，建议在一次性 VM 或容器里跑。

上手三步：

1. 先 `npx @deepseek-ai/dsh web` 跑通一条：选中工作区、填好 API Key，发出官方 quickstart 里那个 summarize 任务。
2. 再进 `cordis.patch.yml` 调 `retryPolicy` 和 `maxParallelToolCalls`，观察对成本和行为的影响。
3. 每次跑完记录模型名 + 参数 + dsh 版本号，否则两次结果之间不可比。
