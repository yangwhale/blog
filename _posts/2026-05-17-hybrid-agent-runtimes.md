---
layout: post
title: "Hybrid agent runtimes: how Claude Code, OpenClaw, and Kilo grew into each other's strengths"
date: 2026-05-17 10:50:00 +0000
categories: [agents, infra]
lang: bilingual
---

<style>
.lang-toggle { display: inline-flex; margin: 8px 0 20px; border: 1px solid #E8EAED; border-radius: 4px; overflow: hidden; }
.lang-toggle button { background: #fff; border: none; color: #5F6368; padding: 6px 18px; cursor: pointer; font-family: inherit; font-size: 13px; font-weight: 500; letter-spacing: 0.2px; transition: background 0.15s, color 0.15s; }
.lang-toggle button + button { border-left: 1px solid #E8EAED; }
.lang-toggle button.active { background: #1A73E8; color: #fff; }
.lang-toggle button:hover:not(.active) { background: #F1F3F4; color: #202124; }
.lang-content { transition: opacity 0.15s; }
.lang-content[hidden] { display: none; }
</style>

<div class="lang-toggle" role="tablist" aria-label="语言切换 / Language toggle">
  <button type="button" data-lang="zh" class="active" role="tab" aria-selected="true">中文</button>
  <button type="button" data-lang="en" role="tab" aria-selected="false">English</button>
</div>

<div class="lang-content lang-zh" markdown="1">

一个 bot 的"人格"--它的 system prompt、记忆、对环境的认知、对话风格--跟执行它的 agent CLI runtime 是相互独立的。在 CloseCrab 里我们证明了这一点:同一个 bot 可以经由三个差别很大的 runtime 来跑--[Claude Code CLI](https://docs.claude.com/en/docs/claude-code/cli-reference)、[OpenClaw](https://github.com/openclaw/openclaw) ACP gateway、以及 [Kilo](https://kilocode.ai/)。但真正有意思的问题不是"能不能运行时切换"--这从第一天就 work--而是"如果把每个 runtime 当作一个有自己看家本事的物种,让它们的优势在彼此之间杂交,会怎样?"

36 小时内,我们跑完了这个实验。结果是三个 runtime 现在每一个都比周五晚上更强,不是靠 upstream 贡献,而是靠**吸收了另外两个 runtime 早已搞定的能力**。所有 patch 没改任何协议,没动模型 serving 栈,全部都是把一个 runtime 拥有、另外两个还缺的**能力**整个吸收过来。

## TL;DR

- 三个 runtime 各有看家本事,谁都不是严格最强
- 36 小时跨物种杂交之后,每个 runtime 都吸收了 2-10 项原本没有的能力,**每一个都比周五版本严格更强**
- 除了单一 runtime 的能力增益之外,还出现了一组**涌现能力**--它们只在三个 runtime 共存时才存在,任何单一 runtime 都没有
- 杂交循环成本很低(每项能力 ~15 分钟),按探测对数量线性扩展

**核心结论**: 把多个 agent CLI runtime 当作一个有差异的种群,在它们之间定向迁移能力,比选一个"最优 runtime"独立优化产生更强的生态。

## 三个 runtime 各自的看家本事

| Runtime       | 通信方式                    | 原生看家本事                                       | 周五版本缺失的能力                                          |
| ------------- | --------------------------- | -------------------------------------------------- | ----------------------------------------------------------- |
| Claude Code   | Unix socketpair + stream-JSON | 工具集最丰富、原生并发 `tool_use`、stream-JSON 事件模型成熟 | 空回复无重试、subprocess 侧 tempfile 泄漏、无语义记忆索引 |
| OpenClaw      | ACP (JSON-RPC over stdio)   | 模型选择最广、1M token 上下文、sqlite 后端 `memory_search` | 启动时不自配置、indexer 不 follow symlink、不知道团队共享文档 |
| Kilo          | HTTP SSE                    | 启动最快(~3s)、`part.delta` 流式、模型无关抽象      | 无 streaming buffer 恢复、不知道多媒体生成脚本、usage 字段统计脆弱 |

每一行右边的"缺失"不是上游工具的 bug——而是**别的 runtime 已经搞定、它还没吸收的能力**。

**核心结论**: 每个 runtime 都是局部的。有意思的设计问题不是"谁赢"，而是"让每个都变完整要多便宜"。

## 让这一切丝滑运转的基础设施: Firestore Inbox

整个实验能跑起来的前提是**bot 之间能在不依赖 runtime 本身能力的前提下互相对话**。这件事不是 agent CLI 自带的——Claude Code 不知道 OpenClaw 上有兄弟、OpenClaw 不会主动联络 Kilo，任一上游 runtime 都没有"bot 间消息"这层抽象。

CloseCrab 早些时候搭起来的 **Firestore Inbox** 解锁了这层能力，而且解锁得很彻底:

- **基于 Firestore `on_snapshot` 的实时推送**——不是轮询。Bot A 写 `inbox/<doc>`，bot B 几十毫秒内就收到 callback。整个实验的每一次跨 runtime probe 都是一个 (写 inbox → 等回执) 的 round trip，如果靠轮询根本撑不下来
- **跟 runtime 完全解耦**。Bot A 不需要知道 bot B 跑的是 Claude Code、OpenClaw 还是 Kilo，inbox 收发都是一致的 Firestore document。今天 bunny 在 7 次 runtime 切换里持续接收 tiemu 派的 probe，中间无需任何重连或重新订阅
- **天然带回执模式**。Bot B 处理完任务后会用 `✅ 任务完成: ...` 格式自动写回 inbox，sender 端能在自己的 bot.log 里看到结构化的回执。今天所有"小爱 → tiemu 报告"、"bunny → tiemu 回答 probe"都是这条路径
- **跨进程持久化**。Inbox doc 写到 Firestore 立刻 durable。哪怕 bot B 在收到那一刻刚好被 restart，它启动后 on_snapshot 重新订阅时仍能拿到 unread doc——今天 7 次切换 worker 不丢一条消息就是这个保证
- **天然支持多种拓扑**: 一对一（tiemu → bunny）、一对多（tiemu 同时派活给 bunny + 小爱）、多对一（bunny + 小爱同时报回 tiemu）、双向多轮（probe → answer → counter-probe 来回几轮）。所有这些都用一份 Firestore collection，没有 RPC 框架、没有 service discovery、没有 mesh sidecar

具体到这次实验: tiemu 派一道题给 bunny 的成本大约是 **20 行 Python + 一次 Firestore document write**。从 closecrab 视角看，inbox 就是 bot 间通讯的 dataframe——发送、订阅、回执三件事各 10 行左右。整个 36 小时里 70+ 条 inbox 消息往返，无一丢失。

如果换一种通讯基础设施（比如 HTTP webhook、消息队列、共享文件 polling），实验仍然能做，但每次"换 runtime + 探测对方"就会涉及连接管理、序列化、重试这些 boilerplate，bot 之间不能像今天这样**毫不知情地完成 runtime 切换却保留通信状态**。

**核心结论**: 跨 runtime 能力迁移本身固然有意义，但让它"丝滑"的不是这些 patch，是底下那条 Firestore inbox 总线。任何想做多 runtime 异构编排的人，先把消息底座做好再说。

## 两天内的能力迁移

### Claude Code 吸收的能力

| 吸收的能力                                  | 来源模式                                    | Commit     |
| ------------------------------------------- | ------------------------------------------- | ---------- |
| 空回复重试韧性                              | OpenClaw 的 `_retry_on_empty_response`      | `613b2a5`  |
| Subprocess 端 tempfile 生命周期卫生         | OpenClaw 和 Kilo 早就遵循的纪律             | `613b2a5`  |

重试模式是逐行移植。当 LLM 返回空 completion,runtime 现在会在同一 session 里把同一 prompt 重发一次,再决定要不要给用户兜底文案。具体实现不同(Claude Code 通过 Unix socket 写一行 stream-JSON,OpenClaw 通过 stdin 发 JSON-RPC),但状态机一样:置一个一次性 retry 标记 → 清累加器 → 重发 → 继续读循环。

```python
if not result_text:
    if not empty_retry_done:
        empty_retry_done = True
        accumulated_reply_text = ""
        saw_task_notification = False
        _send_prompt(text)
        continue
return result_text or "(Claude 处理完成但未生成文字回复)"
```

Tempfile 清理只一行代码,但影响真实:生产机上 Claude Code 在过去几周累计泄漏了 85 个零字节的 `/tmp/claude_stderr_*.log`。修复后,无论 bot 经过多少次重启,数量都稳定保持 1(当前进程自己的 log)。

### OpenClaw 吸收的能力

| 吸收的能力                                                       | 来源模式                                              | Commit    |
| ---------------------------------------------------------------- | ----------------------------------------------------- | --------- |
| 启动时 `agents.list` 自配置                                       | Claude Code "按约定自动加载" 模型                     | `8a64cd2` |
| 基于 hardlink 的 memory wiring                                    | Claude Code 直接读文件的记忆模型                      | `9897054` |
| Bot 启动自动 reindex                                              | "启动时自愈" 模式                                     | `9897054` |
| 跨主机团队基础设施文档自动同步(9 个文档)                          | 借鉴自 Kilo 的 `memory-guide.md` auto-load 思路        | `fdbe7a7` |
| Retry 路径的 streaming buffer 一致性(step buffer + flush)         | 镜像 Kilo 的 `part.delta` flush 纪律                  | `e72c62e` |

收益最大的一段。OpenClaw 自带最 sophisticated 的 memory 搜索(真的 sqlite 向量索引 + `memory_search` 作为 tool),但 workspace 配置很脆弱--indexer 不 follow symlink,所以即使 `memory/` 是正确 symlink,bot 出厂时的索引是空的。Hardlink 修复改用同 inode 的 hardlink(同文件系统),跨文件系统的 GCS shared 用 `shutil.copyfile` 同步。修复前: 0/0 files。修复后: 101/101 files、282 chunks、语义搜索分数 ≥ 0.78 命中之前 runtime 完全看不见的内容。

`agents.list` 自愈长期看更有价值: 之前把任何新 bot 切到 OpenClaw 需要手动编辑 config,现在零手工--bot 第一次启动时把自己的条目写进 gateway config。

### Kilo 吸收的能力

| 吸收的能力                                              | 来源模式                                          | Commit       |
| ------------------------------------------------------- | ------------------------------------------------- | ------------ |
| 通过 `message.part.delta` 缓冲救回 streaming 文本       | Claude Code stream-JSON delta 处理                | `add99a9`    |
| System prompt 里通用工具使用规则                        | Claude Code 多年的 tool guidelines                | `d9e294e`    |
| 防止身份串号的 per-bot session 隔离                     | OpenClaw 的 per-bot agents.list 纪律              | `ba37a22`    |
| `task`(subagent) 使用纪律                              | OpenClaw subagent guide                           | `622de25`    |
| 工具批处理 + bash 真并行规则                            | Claude Code 并发 tool_use 经验                    | `a82871f`    |
| 自启动 cron 守护进程 + `session_status` 工具            | OpenClaw cron + Claude Code session 检查           | `e430b0b`    |
| 多媒体生成脚本(`imagen` / `tts`)的认知                  | Discoverability 早在 Claude Code workspace        | `1286279`    |
| Usage 统计一致性(input / output / cache tokens)         | OpenClaw usage 追踪                               | `0bd1daf`    |

最 heterogeneous 的一组,反映 Kilo 是三者里最年轻、距 production 距离最远的。这些没有任何一项是 Kilo upstream 贡献--都是 closecrab 这层 wrapper 教 Kilo 如何使用 Claude Code 和 OpenClaw bot 已经用了好几周的设施。结果是 Kilo 不再是"试验"runtime--它在 head-to-head 延迟比较里现在能赢另外两个(见下面 "压力测试")。

**核心结论**: 最便宜的能力迁移就是源 runtime 已经把问题解决、目标 runtime 只需要被**告知方案存在**的那种。工具感知、脚本感知、prompt 规则吸收--对 Kilo 都是单 commit 收益。

## 涌现能力(任何单一 runtime 都没有)

有三项能力只因为三个 runtime 共存才存在:

### 1. 跨 runtime model 名翻译

每个 runtime 给同一底层模型用不同名字:

| Runtime     | Claude Opus 4.7 model 字符串                            |
| ----------- | ------------------------------------------------------ |
| Claude Code | `claude-opus-4-7@default`                              |
| OpenClaw    | `anthropic-vertex/claude-opus-4-7`                     |
| Kilo        | `google-vertex-anthropic/claude-opus-4-7@default`      |

`scripts/config-manage.py`(`f6647a3`)拿到了一个 preset-aware 翻译器。bot 切 runtime 时,model 字符串会通过 `_detect_preset` + `_model_for_worker` 自动重写,substring fingerprint 兜底处理之前误配置的 bot。任何单一上游工具都不会有这个能力,因为单一上游工具不需要--它是同时跑多个 runtime 才会产生的能力。

### 2. 保留完整状态的运行时切换

closecrab 中间件在切 runtime 时保留 bot 的人格、记忆、团队上下文。一个 bot 在 20 秒内能在 Claude Code、OpenClaw、Kilo 之间移动,不丢失任何长期记忆、不需要手工重配。Runtime 特定的自愈(OpenClaw 的 hardlink + reindex、Kilo 的 HTTP server spawn、Claude Code 的 session resume)自动跑。

### 3. Heterogeneous 互测

runtime A 上的 bot 可以向 runtime B 上的 bot 探测同一能力，不依赖协议耦合地报告差异。这点完全架设在前面讲的 Firestore inbox 上，sender 跟 receiver 各自在什么 runtime 跨过 inbox 完全不可见。今天 4 项吸收能力里有 3 项是这么发现的：不是我们读源码看出来的，而是 runtime X 上的 bot 注意到 runtime Y 上的兄弟能做它做不了的事。

**核心结论**: 这些涌现能力是支持 heterogeneous-runtime 策略的最强论据。任何人都不大可能把它们往单一 agent CLI 里推 upstream,因为它们只在 orchestration 层才有意义。

## 一个副产品发现: 静默的备份回退

Heterogeneous bot 互探同一基础设施还顺手暴露了一个跟任何单一 runtime 都无关的基建层问题。`scripts/sync-memory.sh` 在一台机器上跑,但 `~/my-private` 是 rsync 目标而非真 git clone。`cd $REPO || exit 1` 守卫通过了(目录存在),然后 git 命令静默失败因为没 `set -e`,最后还打印 `"Pushed to GitHub (private)"`。**memory 备份连续几周静默失败**。

修复(`85e6cb6`)加了显式 `git rev-parse --git-dir` 检查,开了 `set -e` 任一 git 失败就 abort。这是 36 小时里最高价值的 commit,且不是任何意义上的 runtime feature--能发现是因为不同 runtime 看同一基础设施有不同视角,其中一个注意到了不一致。

**核心结论**: 静默成功是最贵的一类 bug,而且当单一观察者的假设跟静默路径吻合时极难发现。Heterogeneous 观察者是被低估的 debug 工具。

## 架构: 迁移图谱

实验期间的杂交迁移路径:

<svg viewBox="0 0 720 320" xmlns="http://www.w3.org/2000/svg" role="img" aria-label="三个 agent runtime 之间的能力迁移图">
  <circle cx="120" cy="110" r="44" fill="#1A73E8"/>
  <text x="120" y="106" text-anchor="middle" fill="#fff" font-size="14" font-weight="500" font-family="Google Sans, sans-serif">Claude Code</text>
  <text x="120" y="124" text-anchor="middle" fill="#fff" font-size="11" font-family="Roboto, sans-serif">socketpair</text>
  <circle cx="600" cy="110" r="44" fill="#1A73E8"/>
  <text x="600" y="106" text-anchor="middle" fill="#fff" font-size="14" font-weight="500" font-family="Google Sans, sans-serif">OpenClaw</text>
  <text x="600" y="124" text-anchor="middle" fill="#fff" font-size="11" font-family="Roboto, sans-serif">ACP</text>
  <circle cx="360" cy="260" r="44" fill="#1A73E8"/>
  <text x="360" y="256" text-anchor="middle" fill="#fff" font-size="14" font-weight="500" font-family="Google Sans, sans-serif">Kilo</text>
  <text x="360" y="274" text-anchor="middle" fill="#fff" font-size="11" font-family="Roboto, sans-serif">HTTP SSE</text>
  <path d="M 164 110 L 556 110" stroke="#5F6368" stroke-width="1.5" fill="none" marker-end="url(#arr-zh)"/>
  <text x="360" y="100" text-anchor="middle" font-size="12" fill="#5F6368" font-family="Roboto, sans-serif">空回复重试 + tempfile 卫生</text>
  <path d="M 564 138 L 392 244" stroke="#5F6368" stroke-width="1.5" fill="none" marker-end="url(#arr-zh)"/>
  <text x="510" y="200" font-size="12" fill="#5F6368" font-family="Roboto, sans-serif">subagent + cron + usage</text>
  <path d="M 328 244 L 156 138" stroke="#5F6368" stroke-width="1.5" fill="none" marker-end="url(#arr-zh)"/>
  <text x="190" y="200" text-anchor="end" font-size="12" fill="#5F6368" font-family="Roboto, sans-serif">streaming buffer 恢复</text>
  <path d="M 156 130 L 556 130" stroke="#5F6368" stroke-width="1.5" fill="none" marker-end="url(#arr-zh)" stroke-dasharray="4,3"/>
  <text x="360" y="155" text-anchor="middle" font-size="12" fill="#5F6368" font-family="Roboto, sans-serif">hardlink memory + agents.list 自配置</text>
  <defs>
    <marker id="arr-zh" viewBox="0 0 10 10" refX="9" refY="3" markerWidth="6" markerHeight="6" orient="auto">
      <path d="M0,0 L0,6 L9,3 z" fill="#5F6368"/>
    </marker>
  </defs>
</svg>

实线箭头是单向能力迁移,虚线箭头是双向适应:OpenClaw 吸收了 Claude Code 的直接读文件 memory 模型,反过来又激发了 Claude Code 后来的 tempfile 卫生工作。

## 压力测试

把同一个 bot 的 runtime 在紧密循环里反复切换,每次切换之间发一个共享记忆查询作为冒烟探针:

| Cycle | 方向                | Model 自动翻译 | Bot 启动 | 探针内容                                      | 回复长度  |
| ----- | ------------------- | -------------- | -------- | --------------------------------------------- | --------- |
| 1     | claude → openclaw   | yes            | yes      | B200 MIG template 名(来自 `shared/gcp-infra.md`)| 141 字符 |
| 2     | openclaw → claude   | yes            | yes      | ALModel optimizer(来自 `shared/tpu-training.md`)| 615 字符 |
| 3     | claude → kilo       | yes            | yes      | 飞书 column_set 限制(来自 `shared/feishu-bot.md`)| 116 字符 |
| 4     | kilo → claude       | yes            | yes      | CC 核心模块(来自 `shared/architecture.md`)     | 851 字符 |

端到端切换耗时(包括 model 翻译、bot 重启、runtime 自愈)每 cycle 15-20 秒。一天 7 次连续切换,记忆内容保持一致--通过在每个 runtime 上问 bot 同一事实问题验证答案匹配。

| Runtime     | 同问题、同 bot、同共享记忆                          | 时间      |
| ----------- | --------------------------------------------------- | -------- |
| OpenClaw    | `memory_search` + `read` + `exec`,9 步              | ~120s   |
| Claude Code | 单个 parallel tool_use block 里 3 个 `Grep`         | 42.66s  |
| Kilo        | `bash` × 3,串行                                    | ~37s    |

Kilo 的时间是惊喜:绝对值上现在它在这个负载里超过了 Claude Code,尽管周初它是三个 runtime 里最不成熟的。这个提升几乎完全来自吸收的能力(streaming flush、tool 批处理规则、原生 cold start 速度保留)。

## 数据

| 指标                                | 值                  |
| ----------------------------------- | -------------------- |
| 天数                                | 2                    |
| Runtime 数                          | 3                    |
| Closecrab commits                   | 61                   |
| 增删行数                            | +5,070 / -568        |
| Claude Code 吸收的能力              | 2                    |
| OpenClaw 吸收的能力                 | 5                    |
| Kilo 吸收的能力                     | 10                   |
| 涌现能力                            | 3                    |
| 基础设施侧顺手发现                  | 1(静默 backup)      |
| `/tmp` 泄漏清理                     | 85 → 0               |
| 每 bot memory 文件 / chunks 索引    | 101 / 282            |
| 压力测试 runtime 切换次数           | 7                    |
| 失败的切换                          | 0                    |

## 我们刻意不做的事

- 不引入对三个 runtime 的统一抽象层。每个保留自己 idiomatic 的表面,只有 closecrab 中间件和运维工具理解全部三个。整个实验的意义就是**保留 runtime 的多样性**
- 不自动化能力迁移循环本身。每次迁移都是人触发的--读一个 runtime 的 commit history 然后在另一个 runtime 上发定向 probe。自动化技术上很直接,但只有三个 runtime 在 scope 内时为时尚早
- 不修改任何 runtime 的协议。协议跟周五一模一样,所有改动都在 closecrab wrapper 层或者 per-runtime worker 里的自愈 patch

**核心结论**: 实验 work 是因为我们让 runtime 保持独立、用很轻的"只观察"循环连接它们。同质化风险--让三个 runtime 收敛成一个形状--是真实的,随着策略成熟我们需要明确策略避免。

## 这改变了我们怎么规划 agent 基础设施

实验前的隐含假设是:最终会选一个"最佳" agent CLI runtime 然后标准化。实验暗示了另一种组织原则:

- **多样性是 feature 不是过渡债**。三个 runtime 同时观察同一基础设施发现了任一单一 runtime 都不会发现的 bug
- **能力迁移很便宜**。大多数收益是单 commit 移植结构类似的逻辑
- **涌现能力自付回报**。跨 runtime model 翻译、运行时切换、heterogeneous probing 全都是只因为 heterogeneity 才存在的 feature

我们会保留全部三个 runtime 在生产、继续在三个 runtime 上跑同一 bot 人格、把新 runtime 当作吸收新能力的机会而不是取代既有的候选。

## 复现

完整 closecrab commit 列表在
[`yangwhale/CloseCrab`](https://github.com/yangwhale/CloseCrab) repo 里,从 `add99a9`(2026-05-16 17:44 UTC)到 `fba5de8`(2026-05-17 09:55 UTC)。回放某个能力迁移最简单的方式:

```bash
git log --oneline --since="2026-05-16" -- closecrab/workers/openclaw_acp.py
git log --oneline --since="2026-05-16" -- closecrab/workers/claude_code.py
git log --oneline --since="2026-05-16" -- closecrab/workers/kilo.py
```

然后并排读 commit 对,跨 runtime 的结构相似性是整个 point。

## 致谢

实验跑在一个团队 4 个 bot 上。其中三个--bunny(主跑 Claude Code)、tiemu(主跑 OpenClaw)、xiaoaitongxue(主跑 Kilo)--轮流互测和提交吸收来的能力。第四个,bot 间 Firestore inbox,严格意义上没跑任何代码,但确实当之无愧地拿到了一个 thank-you--在当天的重启压力下没丢一条消息。

</div>

<div class="lang-content lang-en" hidden markdown="1">

A bot's "personality" - its system prompt, memory, knowledge of the environment, conversational style - is independent of the agent CLI runtime that executes it. In CloseCrab we proved this by routing the same bot through three very different runtimes: [Claude Code CLI](https://docs.claude.com/en/docs/claude-code/cli-reference), the [OpenClaw](https://github.com/openclaw/openclaw) ACP gateway, and [Kilo](https://kilocode.ai/). The interesting question turned out not to be "can we swap them at runtime" - that worked from day one - but "what happens if we treat each runtime as a population with its own strengths, and let those strengths cross-pollinate?"

Over 36 hours we ran that experiment. The result is that each of the three runtimes is now meaningfully more capable than it was on Friday, not by upstream contributions but by absorbing patterns the other two had already figured out. None of the patches changed the protocols or the model serving stack. All of them were absorption of a _capability_ that one runtime had and the others didn't.

## TL;DR

- Each of the three runtimes ships native strengths none of the others has. None of them is strictly best.
- After 36 hours of cross-pollination, each runtime gained between 2 and 10 new capabilities ported from the other two. Result: every runtime is now strictly better than its Friday-night self.
- Beyond per-runtime gains, a small set of **emergent capabilities** appeared that no single runtime had - they only exist because three runtimes coexist.
- The cross-pollination loop is cheap (~15 minutes per absorbed capability) and scales linearly with the number of probing pairs.

**Takeaway:** treating multiple agent CLI runtimes as a heterogeneous population, then deliberately transferring capabilities between them, produces a stronger ecosystem than picking a single "best" runtime and optimizing it in isolation.

## The three runtimes and their native strengths

| Runtime       | Transport                    | Native strength                                 | Native limitation (Friday)                          |
| ------------- | ---------------------------- | ----------------------------------------------- | --------------------------------------------------- |
| Claude Code   | Unix socketpair + stream-JSON | Richest tool surface, native parallel `tool_use`, mature stream-JSON event model | No retry on empty response, leaked process-side tempfiles, no semantic memory index |
| OpenClaw      | ACP (JSON-RPC over stdio)    | Widest model selection, 1M-token context, sqlite-backed `memory_search` | No boot-time self-configuration, indexer didn't follow symlinks, no awareness of team-shared infra docs |
| Kilo          | HTTP SSE                     | Fastest cold start (~3s), `part.delta` streaming, model-agnostic abstraction | No streaming buffer recovery, no awareness of multimedia generation scripts, fragile usage accounting |

Each row's limitations are not bugs in the upstream tool — they are **capabilities that other runtimes had figured out and this one hadn't yet absorbed**.

**Takeaway:** every runtime is partial. The interesting design question is not "which one wins" but "how cheap is it to make each one whole".

## The substrate that made this seamless: the Firestore inbox

The whole experiment is only possible because **bots can talk to each other without depending on the runtime they happen to be running on**. None of the upstream agent CLIs ships this capability — Claude Code does not know that OpenClaw bots exist, OpenClaw does not reach out to Kilo, none of the upstream runtimes has any abstraction for "messages between bots".

CloseCrab's earlier-built **Firestore Inbox** unlocks that capability completely:

- **Real-time push via Firestore `on_snapshot`**, not polling. Bot A writes `inbox/<doc>`, bot B receives a callback within tens of milliseconds. Every cross-runtime probe in the experiment is a (write-inbox, await-receipt) round trip; on a polling substrate it would not have been tractable.
- **Completely decoupled from the runtime.** Bot A does not need to know whether bot B is on Claude Code, OpenClaw, or Kilo. The send/receive interface is a uniform Firestore document. bunny was probed by tiemu continuously across 7 runtime switches today without any re-connect or re-subscribe on either side.
- **Built-in receipt pattern.** Bot B writes a `✅ 任务完成: ...` reply back into the inbox after processing, and the sender's structured log captures it cleanly. Every "xiaoaitongxue → tiemu report" and "bunny → tiemu probe answer" today rode this path.
- **Cross-process persistence.** An inbox doc is durable the moment it lands in Firestore. Even if bot B happened to be restarting at the receipt moment, on_snapshot picks up the unread doc when it re-subscribes — which is why 7 worker switches today did not lose a single message.
- **All topologies for free**: one-to-one (tiemu → bunny), one-to-many (tiemu fans out to bunny + xiaoaitongxue at once), many-to-one (bunny + xiaoaitongxue both report back to tiemu), bidirectional multi-turn (probe → answer → counter-probe over rounds). All on a single Firestore collection. No RPC framework, no service discovery, no mesh sidecar.

Concretely in this experiment: tiemu assigning a task to bunny costs about **20 lines of Python plus a single Firestore document write**. From closecrab's vantage point the inbox is the dataframe of inter-bot communication — send, subscribe, and receipt are roughly 10 lines each. 70+ inbox messages went back and forth over the 36 hours; not one was lost.

A different substrate (HTTP webhooks, message queues, shared-file polling) could have worked too, but every "switch runtime and probe the other side" loop would have brought connection management, serialization, and retry boilerplate along with it. Bots would not have been able to **complete a runtime switch without telling each other and still preserve their communication state** the way they did today.

**Takeaway:** the cross-runtime capability transfers matter on their own, but what made them _seamless_ was not the patches — it was the Firestore inbox bus underneath. Anyone building heterogeneous multi-runtime orchestration should get the message substrate right first.

## Capability transfer in two days

### Capabilities Claude Code absorbed

| Capability                                | Source pattern                       | Commit     |
| ----------------------------------------- | ------------------------------------ | ---------- |
| Empty-response retry resilience           | OpenClaw `_retry_on_empty_response`  | `613b2a5`  |
| Subprocess-side tempfile lifecycle hygiene| Discipline already followed by OpenClaw and Kilo  | `613b2a5`  |

The retry pattern was a verbatim port. When the LLM returns an empty completion, the runtime now resends the same prompt once on the same session before surfacing a placeholder to the user. Implementation specifics differ (Claude Code writes a stream-JSON line over a Unix socket; OpenClaw sends a JSON-RPC request over stdin) but the state machine is identical: set a one-shot retry flag, reset accumulators, resend, continue reading.

```python
if not result_text:
    if not empty_retry_done:
        empty_retry_done = True
        accumulated_reply_text = ""
        saw_task_notification = False
        _send_prompt(text)
        continue
return result_text or "(Claude 处理完成但未生成文字回复)"
```

The tempfile cleanup is one line in `stop()`, but it matters: on the production host, Claude Code had leaked 85 zero-byte `/tmp/claude_stderr_*.log` files across weeks of bot restarts. After the patch, restart-time cleanup keeps the count at 1 (the current process's own log) regardless of how many cycles the bot has been through.

### Capabilities OpenClaw absorbed

| Capability                                       | Source pattern                                       | Commit    |
| ------------------------------------------------ | ---------------------------------------------------- | --------- |
| Boot-time `agents.list` self-configuration       | Claude Code's "auto-load from convention" model      | `8a64cd2` |
| Hardlink-backed memory wiring                    | Claude Code's direct-file memory access              | `9897054` |
| Auto-reindex on bot start                        | "Self-heal on startup" pattern                       | `9897054` |
| Cross-host shared infra doc sync (9 team docs)   | Adapted from Kilo's `memory-guide.md` auto-load idea | `fdbe7a7` |
| Retry-path streaming parity (step buffer + flush)| Mirrored Kilo's `part.delta` flush discipline        | `e72c62e` |

The largest gain. OpenClaw came in with the most sophisticated memory search (a real sqlite vector index with `memory_search` as a tool) but its workspace setup was fragile - the indexer didn't follow symlinks, so an out-of-the-box bot would have an empty index even with a correctly-symlinked `memory/` directory. The hardlink fix replaces the symlink with shared-inode hardlinks for files on the same filesystem, and `shutil.copyfile` syncs from the cross-filesystem GCS-mounted shared directory. Before: 0/0 files indexed. After: 101/101 files, 282 chunks, semantic search hits at score ≥ 0.78 on content that was previously invisible to the runtime.

The `agents.list` self-healing is the more impactful change long-term: switching any new bot to OpenClaw used to require a manual config edit; now it requires zero. The bot writes its own entry into the gateway config the first time it starts.

### Capabilities Kilo absorbed

| Capability                                          | Source pattern                                       | Commit       |
| --------------------------------------------------- | ---------------------------------------------------- | ------------ |
| Streaming text recovery via `message.part.delta` buffers | Claude Code stream-JSON delta handling          | `add99a9`    |
| Universal tool-use rules in system prompt           | Claude Code's well-established tool guidelines       | `d9e294e`    |
| Per-bot session isolation against identity bleed-through | OpenClaw's per-bot agents.list discipline       | `ba37a22`    |
| `task` (subagent) usage discipline                  | OpenClaw subagent guide                              | `622de25`    |
| Tool batching + bash-true-parallel rules            | Claude Code parallel tool_use experience             | `a82871f`    |
| Self-start cron daemon + `session_status` tool      | OpenClaw cron and Claude Code session inspection      | `e430b0b`    |
| Awareness of multimedia generation scripts (`imagen`, `tts`) | Discoverability already in Claude Code workspace | `1286279`    |
| Usage accounting parity (input/output/cache tokens) | OpenClaw usage tracking                              | `0bd1daf`    |

The most heterogeneous set, reflecting that Kilo was the newest of the three and started furthest from production-readiness. None of these were upstream Kilo contributions; they were closecrab-side wrappers that taught Kilo how to use facilities Claude Code and OpenClaw bots had been using for weeks. The end result is that Kilo is no longer the "trial" runtime - it routinely wins head-to-head latency comparisons against the other two (see "Stress test" below).

**Takeaway:** the cheapest capability transfers are the ones where the source runtime has solved a problem and the target runtime just needs to be _told that the solution exists_. Tool-awareness, script-awareness, prompt-rule absorption - all of these were single-commit gains for Kilo.

## Emergent capabilities (not present in any single runtime)

Three capabilities exist only because three runtimes coexist:

### 1. Cross-runtime model name translation

Each runtime names the same underlying model differently:

| Runtime     | Claude Opus 4.7 model string                       |
| ----------- | -------------------------------------------------- |
| Claude Code | `claude-opus-4-7@default`                          |
| OpenClaw    | `anthropic-vertex/claude-opus-4-7`                 |
| Kilo        | `google-vertex-anthropic/claude-opus-4-7@default`  |

`scripts/config-manage.py` (`f6647a3`) gained a preset-aware translator. When a bot switches runtime, the model string is rewritten automatically through `_detect_preset` + `_model_for_worker`, with a substring fingerprint fallback for bots that came in misconfigured. No single upstream tool has this capability because no single upstream tool needs it - it's a product of running multiple runtimes side by side.

### 2. Live runtime switching with full state preservation

The closecrab middleware preserves the bot's personality, memory, and team context across runtime switches. A single bot can move between Claude Code, OpenClaw, and Kilo in under 20 seconds with no loss of long-term memory and no manual reconfiguration. The runtime-specific self-healing (OpenClaw's hardlinks and reindex, Kilo's HTTP server spawn, Claude Code's session resume) runs automatically on boot.

### 3. Heterogeneous mutual testing

A bot on runtime A can probe a bot on runtime B for the same capability, and report differences without protocol coupling. This is built directly on top of the Firestore inbox described above — the sender and receiver are mutually invisible across the inbox boundary, including which runtime each one happens to be using. This is how three of the four absorbed-capability discoveries happened: not by us reading source code, but by a bot running on runtime X noticing that its sibling on runtime Y could do something it couldn't.

**Takeaway:** these emergent capabilities are the strongest argument for the heterogeneous-runtime strategy. They are not features anyone is likely to upstream into a single agent CLI, because they only make sense at the orchestration layer above the CLIs.

## Side discovery: a silent backup regression

Heterogeneous bots probing the same infrastructure also surfaced an infrastructure-level issue that had nothing to do with any single runtime. The `scripts/sync-memory.sh` script was running on a host where `~/my-private` was an rsync target rather than a real git clone. Its `cd $REPO || exit 1` guard passed (the directory exists), then git commands silently failed because the script lacked `set -e`, and the final line still printed `Pushed to GitHub (private)`. Memory backups had been silently failing for weeks.

Fix (`85e6cb6`) adds an explicit `git rev-parse --git-dir` check and turns on `set -e`. This is by far the highest-value commit of the 36-hour window and is not a runtime feature in any sense - it surfaced only because different runtimes observing the same infrastructure had different views and one of them noticed an inconsistency.

**Takeaway:** silent successes are the most expensive class of bug, and they are unusually hard to find when a single observer's assumptions match the silent path. Heterogeneous observers are an underrated debugging tool.

## Architecture: transfer graph

The cross-pollination graph during the experiment:

<svg viewBox="0 0 720 320" xmlns="http://www.w3.org/2000/svg" role="img" aria-label="Capability transfer graph between three agent runtimes">
  <circle cx="120" cy="110" r="44" fill="#1A73E8"/>
  <text x="120" y="106" text-anchor="middle" fill="#fff" font-size="14" font-weight="500" font-family="Google Sans, sans-serif">Claude Code</text>
  <text x="120" y="124" text-anchor="middle" fill="#fff" font-size="11" font-family="Roboto, sans-serif">socketpair</text>
  <circle cx="600" cy="110" r="44" fill="#1A73E8"/>
  <text x="600" y="106" text-anchor="middle" fill="#fff" font-size="14" font-weight="500" font-family="Google Sans, sans-serif">OpenClaw</text>
  <text x="600" y="124" text-anchor="middle" fill="#fff" font-size="11" font-family="Roboto, sans-serif">ACP</text>
  <circle cx="360" cy="260" r="44" fill="#1A73E8"/>
  <text x="360" y="256" text-anchor="middle" fill="#fff" font-size="14" font-weight="500" font-family="Google Sans, sans-serif">Kilo</text>
  <text x="360" y="274" text-anchor="middle" fill="#fff" font-size="11" font-family="Roboto, sans-serif">HTTP SSE</text>
  <path d="M 164 110 L 556 110" stroke="#5F6368" stroke-width="1.5" fill="none" marker-end="url(#arr-en)"/>
  <text x="360" y="100" text-anchor="middle" font-size="12" fill="#5F6368" font-family="Roboto, sans-serif">retry resilience + tempfile hygiene</text>
  <path d="M 564 138 L 392 244" stroke="#5F6368" stroke-width="1.5" fill="none" marker-end="url(#arr-en)"/>
  <text x="510" y="200" font-size="12" fill="#5F6368" font-family="Roboto, sans-serif">subagent + cron + usage accounting</text>
  <path d="M 328 244 L 156 138" stroke="#5F6368" stroke-width="1.5" fill="none" marker-end="url(#arr-en)"/>
  <text x="190" y="200" text-anchor="end" font-size="12" fill="#5F6368" font-family="Roboto, sans-serif">streaming buffer recovery</text>
  <path d="M 156 130 L 556 130" stroke="#5F6368" stroke-width="1.5" fill="none" marker-end="url(#arr-en)" stroke-dasharray="4,3"/>
  <text x="360" y="155" text-anchor="middle" font-size="12" fill="#5F6368" font-family="Roboto, sans-serif">hardlink memory model + agents.list autoload</text>
  <defs>
    <marker id="arr-en" viewBox="0 0 10 10" refX="9" refY="3" markerWidth="6" markerHeight="6" orient="auto">
      <path d="M0,0 L0,6 L9,3 z" fill="#5F6368"/>
    </marker>
  </defs>
</svg>

Solid arrows are single-source capability transfers. The dashed arrow captures bidirectional adaptation: OpenClaw absorbed Claude Code's direct-file memory access model, which in turn motivated Claude Code's later tempfile hygiene work.

## Stress test

Take one bot and cycle its runtime in a tight loop, with a shared-memory query as the smoke probe between each switch:

| Cycle | Direction           | Model translated | Bot booted | Probe content                                  | Reply length |
| ----- | ------------------- | ---------------- | ---------- | ---------------------------------------------- | ------------ |
| 1     | claude → openclaw   | yes              | yes        | B200 MIG template name (from `shared/gcp-infra.md`) | 141 chars    |
| 2     | openclaw → claude   | yes              | yes        | ALModel optimizer (from `shared/tpu-training.md`)   | 615 chars    |
| 3     | claude → kilo       | yes              | yes        | Feishu column_set limitation (from `shared/feishu-bot.md`) | 116 chars |
| 4     | kilo → claude       | yes              | yes        | CC core modules (from `shared/architecture.md`)     | 851 chars    |

End-to-end switch time including model translation, bot restart, and runtime self-healing was 15-20 seconds per cycle. Across 7 sequential switches over the day, memory content remained consistent - verified by asking the bot the same factual question on each runtime and matching the answers.

| Runtime     | Same question, same bot, same shared memory      | Time     |
| ----------- | ------------------------------------------------ | -------- |
| OpenClaw    | `memory_search` + `read` + `exec`, 9 steps       | ~120s    |
| Claude Code | `Grep` ×3 in one parallel tool_use block         | 42.66s   |
| Kilo        | `bash` ×3, sequential                            | ~37s     |

The Kilo time is the surprise: in absolute terms it now beats Claude Code on this workload despite starting the week as the least mature of the three runtimes. The improvement is almost entirely from absorbed capabilities (streaming flush, tool batching rules, faster cold start preserved from native).

## Numbers

| Metric                                  | Value                |
| --------------------------------------- | -------------------- |
| Days                                    | 2                    |
| Runtimes                                | 3                    |
| Closecrab commits                       | 61                   |
| Lines added / removed                   | +5,070 / -568        |
| Capabilities transferred (Claude Code)  | 2                    |
| Capabilities transferred (OpenClaw)     | 5                    |
| Capabilities transferred (Kilo)         | 10                   |
| Emergent capabilities                   | 3                    |
| Infrastructure-side discoveries         | 1 (silent backup)    |
| `/tmp` leak cleaned                     | 85 → 0               |
| Memory files / chunks indexed (per bot) | 101 / 282            |
| Stress test runtime switches            | 7                    |
| Failed switches                         | 0                    |

## What we deliberately did not do

- We did not introduce a unified abstraction layer over the three runtimes. Each one keeps its idiomatic surface; only the closecrab middleware and the operational tooling understand all three. The whole point of the experiment was to preserve runtime diversity.
- We did not automate the capability-transfer loop. Each transfer was a human-initiated read of one runtime's commit history followed by a directed probe on another runtime. Automation is straightforward but premature with only three runtimes in scope.
- We did not change any of the runtime wire protocols. The protocols are exactly where they were on Friday; everything we changed lives in the closecrab wrapper layer or in self-healing patches inside the per-runtime workers.

**Takeaway:** the experiment worked because we kept the runtimes independent and used cheap observation-only loops between them. The homogenization risk - making three runtimes converge into a single shape - is real and we will need a deliberate policy to avoid it as the strategy matures.

## What this changes about how we plan agent infra

Pre-experiment, the implicit assumption was that one would eventually pick a "best" agent CLI runtime and standardize on it. The experiment suggests a different organizing principle:

- **Diversity is a feature, not transitional debt.** Three runtimes observing the same infrastructure found bugs no single runtime would have found.
- **Capability transfer is cheap.** Most of the gains were single-commit ports of structurally-similar logic.
- **Emergent capabilities pay for themselves.** Cross-runtime model translation, live runtime switching, and heterogeneous probing are all features that exist only because of the heterogeneity.

We will keep all three runtimes in production, continue running the same bot personalities across all three, and treat new runtimes as opportunities to absorb new capabilities rather than as candidates to displace existing ones.

## Reproducing

The full closecrab commit list is on the [`yangwhale/CloseCrab`](https://github.com/yangwhale/CloseCrab) repo between `add99a9` (2026-05-16 17:44 UTC) and `fba5de8` (2026-05-17 09:55 UTC). To replay a specific capability transfer, the simplest recipe is:

```bash
git log --oneline --since="2026-05-16" -- closecrab/workers/openclaw_acp.py
git log --oneline --since="2026-05-16" -- closecrab/workers/claude_code.py
git log --oneline --since="2026-05-16" -- closecrab/workers/kilo.py
```

and read the commit pairs side by side. The structural similarity across runtimes is the entire point.

## Acknowledgements

The experiment ran on four bots in a single team. Three of them - bunny (mostly Claude Code), tiemu (mostly OpenClaw), xiaoaitongxue (mostly Kilo) - took turns probing each other and committing the absorbed capabilities. The fourth, the inter-bot Firestore inbox, did not technically run any code but absolutely earned a thank-you for not losing a single message under the day's restart load.

</div>

<script>
(function() {
  var buttons = document.querySelectorAll('.lang-toggle button');
  var sections = document.querySelectorAll('.lang-content');
  function setLang(lang) {
    buttons.forEach(function(b) {
      var active = b.dataset.lang === lang;
      b.classList.toggle('active', active);
      b.setAttribute('aria-selected', active ? 'true' : 'false');
    });
    sections.forEach(function(s) {
      s.hidden = !s.classList.contains('lang-' + lang);
    });
    try { localStorage.setItem('blog-lang', lang); } catch (e) {}
  }
  buttons.forEach(function(b) {
    b.addEventListener('click', function() { setLang(b.dataset.lang); });
  });
  // restore last choice; default zh
  try {
    var saved = localStorage.getItem('blog-lang');
    if (saved === 'en' || saved === 'zh') setLang(saved);
  } catch (e) {}
})();
</script>
