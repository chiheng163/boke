---
title: "Hermes Agent v0.16 Kanban Swarm 功能深度解析（面向 AI 开发者）"
date: 2026-09-30
tags: ["Hermes Agent", "Kanban", "Multi-Agent", "AI Agent", "任务编排"]
author: chiheng163
---

你有没有遇到过这种情况：你在本地拉起三个 subagent 并行去调研、写稿、审校，结果跑到一半终端崩了，或者你只是关掉了那个 shell——所有的任务分配、中间产物、进行到哪一步，全部烟消云散，只能从头再来。

这正是进程内 subagent 的天然边界：官方文档用「without fragile in-process subagent swarms」形容它，`delegate_task` 随父进程退出而丢失，上下文隔离却不持久，既不能跨机器、也不能跨 profile。当你想要的是「关了终端明天还能接着跑」的多 Agent 流水线时，答案就不再是进程内调度，而是一张持久化看板——Kanban。

## 为什么多 Agent 协作需要一张持久化看板

看板模型把「协作」落成了三样东西：每个任务是一行 SQLite 记录，每次交接是可读可写的行，每个工作区是隔离目录。于是崩溃之后能重新领走、能被外部观察、能被另一个进程接力。

【为什么重要】持久化直接改变了失败的成本结构。进程内 swarm 崩溃意味着全部重跑；看板里崩溃只是一个 run 的结束，任务本身回到 ready 状态等下一个 worker 认领。本流水线就真实踩到过：research 卡片的第一次尝试直接 crashed（worker pid 2140 跑完 `hermes kanban swarm --help` 后死掉），事件记为 `crashed` + `pid 2140 not alive`，随后 `retry_status: ready`，由新 run 接替，中间产物 research.md 完好无损。

但看板不是万能药。一次性的短推理子任务、需要人实时对话的任务，用 `delegate_task` 或 cron 更省事——为一个三秒钟的问题建一张卡片、等一个 tick 派工，反而是浪费。

## 看板的数据模型：board、tenant、task、dependency、run

Kanban 的隔离分三层，边界硬度递减：

- **board** 是硬边界。每个 board 一个独立 SQLite（default board 在 `~/.hermes/kanban.db`，命名 board 在 `~/.hermes/kanban/boards/<slug>/kanban.db`），各自有独立的 `workspaces/` 和 `logs/`，worker 只看得见自己 board 的任务（dispatcher 注入 `HERMES_KANBAN_BOARD`），跨 board 不允许 link。
- **tenant** 是软命名空间。它是 board 内的一个字符串，按 workspace 路径与 memory key 前缀隔离，worker 侧以 `HERMES_TENANT` 可见。
- **task** 是最小单位，状态集合是 `triage | todo | ready | running | blocked | review | done | archived`。

【为什么重要】board 挡的是项目和机器，tenant 挡的是同项目内的多组流水线——写错边界，要么泄漏状态，要么互相抢任务。

任务的依赖门控很朴素：`task_links` 记 parent → child，所有 parent 都 `done` 时 dispatcher 才把子卡从 `todo` 提升为 `ready`。给已经在 running 的 child 加 link 会被拒，因为无法门控已经被认领的工作。

任务上可写的一组字段，各自解决一个具体问题：`workspace_kind` 选 `scratch`（完成即删）/ `dir`（共享绝对路径目录，完成保留）/ `worktree`（git worktree）；`completion_contract` 声明交付形态（默认 `local-only`，或 `OWNER/REPO`、一个 PR URL）；`max_runtime_seconds` 是 wall-clock 硬上限，超时事件记 `timed_out`；`failure_limit` 默认 2；`model` / `provider` 可覆盖、下一次 dispatch 才生效；`idempotency_key` 做 root 卡片级去重。

## Kanban Swarm v1 的拓扑：root → N workers → verifier → synthesizer

一条 `hermes kanban swarm` 命令，原子地建出下面这张图（源码 `hermes_cli/kanban_swarm.py` 的模块 docstring 原文）：

```
planning root (completed immediately)
    ├─ parallel specialist workers (ready)
    └─ verifier (todo until all workers done)
         └─ synthesizer (todo until verifier done)
```

四类角色：root 是规划锚点、建图即完成；N 个 worker 并行互不等待；verifier 门控于全部 worker；synthesizer 门控于 verifier。

【为什么重要】共享状态的实现方式决定了这套拓扑能不能去掉中央调度器。Swarm 用的是 **blackboard**：每次更新是一行结构化 JSON 评论 `{"key": ..., "value": ...}`（前缀 `[swarm:blackboard] `），写在 root 卡片上。任何 worker 都能读回，按 key 合并、后写的覆盖先写的，并附 `_authors` 记录每个 key 的获胜作者。建图时第一条 blackboard 就是 `key="topology"`，value 是 `{root_id, worker_ids, verifier_id, synthesizer_id, goal}`——幂等重放时靠它恢复拓扑而不是重复建图。

原子性同样关键：root 卡片先以 `initial_status="blocked"` 创建，再在同一个 write_txn 内用 CAS 翻成 `done`，故意不调用 `complete_task`（那会自己开事务）。测试 `test_create_swarm_graph_is_atomic_and_rolls_back_partial_build` 用一个独立连接在中途读 `SELECT COUNT(*) FROM tasks` 断言为 0——外部读者只会看到「没有 swarm」或「完整拓扑」，永远看不到半连接的 root/worker 图。

## 动手：一条命令拉起一个 swarm

先说一个必须先交代的事实：**标题里的 v0.16 与真实版本差了一个版本**。本机实测 `hermes --version` 返回 `Hermes Agent v0.21.5+4536.gea114c3 (2026.9.24)`；而 Kanban Swarm v1 实际落地于 **v0.15.0**（release tag `v2026.5.28`，2026-05-28，「The Velocity Release」），其 release notes 明确列出 `hermes kanban swarm`；到 v0.16.0（tag `v2026.6.5`）的 release notes 全文对 swarm 匹配 0 次，仓库里也根本不存在名为 `v0.16` 的 tag。所以正文一律按实测口径写，不把它说成「v0.16 新功能」。

实测的 CLI 接口如下（`hermes kanban swarm --help` 原文），关键在 `--worker` 可重复、`--verifier` 与 `--synthesizer` 必填、`goal` 是位置参数：

```
usage: hermes kanban swarm [-h] [--worker PROFILE:TITLE[:SKILL,SKILL]]
                           --verifier VERIFIER --synthesizer SYNTHESIZER
                           [--tenant TENANT] [--priority PRIORITY]
                           [--created-by CREATED_BY]
                           [--idempotency-key IDEMPOTENCY_KEY] [--json]
                           goal
```

一条完整可运行的命令（把 goal 换成你的目标即可）：

```bash
hermes kanban swarm "写一份多区域故障转移方案的深度解析" \
  --worker researcher:"资料调研" \
  --worker architect:"方案设计" \
  --worker sre:"可靠性评审" \
  --verifier reviewer \
  --synthesizer writer \
  --tenant blog
```

`--worker PROFILE:TITLE[:SKILL,SKILL]` 里的冒号段含义要分清：`PROFILE` 是负责该卡的 profile 名，`TITLE` 是卡片标题，可选的 `SKILL` 段按逗号切开、作为该 worker 卡片的**任务级 skills 固定**（不是 profile、不是 body）。

【为什么重要】这里有一个真实存在的版本陷阱：官方文档的示例写的是 `--workers researcher,architect,sre`，但本机 v0.21.5 上跑同样的 flag 直接报 `hermes: error: unrecognized arguments: --workers a,b,c`（exit 2）。源码里只有 `--worker`、没有 `--workers`。照抄文档会直接翻车，必须以实测的可重复 `--worker` 为准。

拉起之后怎么确认它建起来了：`hermes kanban list` 看卡片、`show <id>` 看单卡、`stats` / `runs` 看运行情况。dispatcher 在下一次 tick 开始派工。在交互会话或 gateway 平台里，`/kanban <action>` 与 CLI 共用同一个入口（参数面、flag、输出格式完全一致），且被显式豁免于「agent 正在思考时排队」的守卫——因为看板在 `~/.hermes/kanban.db` 里，不在运行中的会话状态里，所以 agent 正在运行时也能立即生效。

## 工作节点看到什么：worker 的工具集与运行契约

dispatcher 拉起 worker 时，默认 spawn 形式是 `hermes -p <assignee> chat -q <prompt>`，在任务 pinned 的 workspace 内运行，并注入一组环境变量：`HERMES_KANBAN_TASK`（任务 id）、`HERMES_KANBAN_DB`（SQLite 绝对路径）、`HERMES_KANBAN_BOARD`（board slug）、`HERMES_KANBAN_WORKSPACES_ROOT`（workspace 树根）、`HERMES_KANBAN_WORKSPACE`（本任务 workspace）、`HERMES_KANBAN_RUN_ID`、`HERMES_KANBAN_CLAIM_LOCK`、`HERMES_PROFILE`、`HERMES_TENANT`。

worker 手里的 `kanban_*` 工具是一套完整生命周期：`kanban_show` / `kanban_list` 读上下文，`kanban_comment` / `kanban_attach` 写交接，`kanban_heartbeat` 报活，`kanban_block` 停下并说明原因，`kanban_complete` / `kanban_request_review` 收尾，`kanban_create` / `kanban_link` 派生下游。

【为什么重要】worker 的收尾契约是硬约束：必须以 `kanban_complete` / `kanban_request_review` / `kanban_block` 收尾；若进程以 0 退出而任务仍 `running`，dispatcher 记 `protocol_violation`。`kanban_block` 的四种 kind 各有语义——`dependency`（停在 todo，父任务结束后自动恢复）、`needs_input` / `capability` / `transient`（浮给人类）。下面是一条完整的收尾调用：

```python
kanban_complete(
    summary="调研完成：版本结论已核实，见 research.md",
    metadata={
        "core_finding": "Kanban Swarm v1 实际引入于 v0.15.0（tag v2026.5.28）",
        "sources": ["https://github.com/NousResearch/hermes-agent/releases/tag/v2026.5.28"],
    },
    artifacts=["C:/Users/lijiachao/AppData/Local/hermes/shared/blog-swarm/20260930-223438-hermes-agent-v0-16-kanban-swarm-ai-e981c3/research.md"],
)
```

长任务不丢靠 heartbeat：长操作每几分钟 `kanban_heartbeat(note=...)`，若任务可能跑超 1 小时，必须至少每小时心跳一次——dispatcher 会回收「已运行超过 `kanban.dispatch_stale_timeout_seconds`（默认 4 小时）且最近 1 小时没心跳」的任务。回收是良性的（回 ready、不计失败），但会丢掉当前 run 的进度。

## 调度器：谁在派工、并发与失败怎么处理

dispatcher 默认跑在 gateway 内（`kanban.dispatch_in_gateway: true`），tick 间隔 60 秒。一次 tick 做四件事：回收过期 claim、回收崩溃/僵尸 worker、把满足条件的 todo 提升为 ready、原子认领并拉起 assignee 对应的 profile。

【为什么重要】「原子认领」是保证不重复派工的关键——同一张卡不会被两个 dispatcher 同时认领。但认领之后还有两层坑：

一是 assignee 写错。assignee 不是已存在的 profile 时，卡片会留在 `ready` 并记 `skipped_nonspawnable` 事件，不会静默丢弃、也不会 fallback 执行——对使用者来说就是「一条命令下去了，什么都没发生」。

二是多 gateway 抢活。只有**一个** gateway 能拥有 dispatcher，其余必须设 `kanban.dispatch_in_gateway: false`（或 `HERMES_KANBAN_DISPATCH_IN_GATEWAY=false`），否则多个 gateway 会抢同一份工作。

失败处理有明确的熔断：同一任务连续 spawn 失败达 `failure_limit`（默认 2）自动转 `blocked`，并把最后一次错误作为 reason。退出码 `75`（EX_TEMPFAIL，限流/超时）记为 `rate_limited` 并重排，`78`（EX_CONFIG，凭证或模型不可用）第一次就触发熔断、卡片 sticky blocked。

## 可观测性与运维：从命令行到 dashboard

命令行只读走查用这些动词：`list` / `show` / `tail` / `watch` / `stats` / `runs` / `log` / `diagnostics` / `context`。想看一张卡走到哪、卡在哪，`show` 的 events 里能看到 run_id 与 outcome。

dashboard 的 kanban 插件提供了四个端点，挂在 `/api/plugins/kanban/` 下：

| 端点 | 方法 | 返回 |
|---|---|---|
| `/workers/active` | GET | 当前已 spawn 的 worker：PID、profile、task id、last heartbeat |
| `/runs/{id}` | GET | 单次 run 详情：task id、status、exit code、log path |
| `/runs/{run_id}/terminate` | POST | 终止一个可回收的 run，让任务可重新派发 |
| `/inspect` | GET | backlog、in-progress 数量 vs `max_in_progress`、最近事件 |

【为什么重要】四个端点里只有 `terminate` 是写方法（POST），其余三个是只读 GET——这是人介入而不打断 agent 的唯一官方写入口。配合评论补上下文（下一次 `kanban_show` 才会读到）、`unblock`、把卡片拖回重跑，人在流水线全程都有抓手。

## 实战复盘与踩坑清单

拿本流水线复盘：tenant=`blog`，5 张卡片串成固定产物链——orchestrator 产出 plan.md → researcher 产出 research.md → writer 产出 article.md → reviewer 产出 review.md → publisher 产出 publish.json，共享一个 `workspace_kind=dir` 的目录。下游卡片是**预建**的，靠 `parents` 门控依次放行，而不是 worker 自己 create。

真实踩到的坑，按信息量排序：

1. **partial clone 让 `git log -- <path>` 挂死**。本机安装目录是 partial clone（`remote.origin.partialclonefilter` = `tree:0`），遍历 tree 的命令会去远端取对象，实测两次 `git log -- hermes_cli/kanban_swarm.py` 分别在 180s / 300s 超时。绕法是改用 GitHub API 的 `commits?path=...`。
2. **crash 后任务重新领走**。research 卡片第一次 run 直接 crashed，但看板把它放回 ready，新 run 接替后 research.md 完好——这正是「持久化看板」价值的正反两面。
3. **文档与实测 CLI 有出入**。`--workers a,b,c` 实测报错，必须用可重复的 `--worker`，这是最容易照抄翻车的地方。
4. **assignee 不存在时静默停在 ready**，没有报错也没有 fallback。

什么时候不该上 swarm？当目标无法预先拆成「并行采集 + 统一校验 + 统一合成」的形状时，线性流水线或单张卡更划算。Swarm v1 是把这一类三明治结构固化成了命令，而不是通用调度器——认清这个边界，才能在「一条命令拉起协作」和「杀鸡用牛刀」之间做出正确取舍。
