# Hermes Agent v0.17 Kanban Swarm：多 Agent 协作的终极形态

> 当你让 5 个 AI Agent 同时写一篇文章，它们怎么知道谁在做什么？做完了交给谁？做到一半卡住了怎么办？

这不是一个假设性问题。2026 年，AI Agent 已经从"单线程对话"进化到了"多 Agent 协作"阶段。Hermes Agent v0.17 引入的 **Kanban Swarm** 机制，用一块 SQLite 数据库、一个调度器和一套工具协议，解决了多 Agent 系统中最棘手的问题：**持久化协调**。

本文是一篇面向 AI 开发者的深度技术解析。我们不仅讲"怎么用"，更要讲"为什么这样设计"。

---

## 1. 问题的本质：为什么 `delegate_task` 不够用？

Hermes Agent 早就支持子任务分发——`delegate_task` 可以 fork 出一个子 Agent，等它返回结果后继续。这看起来已经够用了，直到你遇到这些场景：

- 一个研究员 Agent 需要等待另一个研究员的数据，但两者应该并行启动
- 一个 Writer Agent 写完文章后，需要 Reviewer Agent 审核，审核不通过要退回重写
- Agent 崩溃了，你希望它从断点恢复，而不是从头来过
- 你想在 Telegram 上发一条 `/kanban unblock t_abc`，让卡住的任务继续跑

`delegate_task` 的本质是一个 **RPC 调用**——父进程阻塞等待子进程返回，失败了就是失败了，没有恢复机制，没有审计轨迹，没有人类介入的接口。

**Kanban 的本质是一个持久化消息队列 + 状态机。** 每个任务是一行 SQLite 记录，每个 handoff 是一行任何人都能读写的记录，每个 worker 是一个拥有独立身份的完整 OS 进程。

| 维度 | `delegate_task` | Kanban |
|------|-----------------|--------|
| 形态 | 函数调用 (fork → join) | 持久化消息队列 + 状态机 |
| 父进程 | 阻塞等待子进程返回 | 创建后即 fire-and-forget |
| 子进程身份 | 匿名子 Agent | 命名 profile，拥有持久记忆 |
| 可恢复性 | 无——失败即结束 | block → unblock → 重新运行；崩溃 → 回收 |
| 人类介入 | 不支持 | 随时可以 comment / unblock |
| 审计轨迹 | 上下文压缩后丢失 | SQLite 中永久保存 |
| 协作模式 | 层级式 (caller → callee) | 对等式——任何 profile 都能读写任何任务 |

**一句话总结：** `delegate_task` 是函数调用；Kanban 是一个工作队列，每个 handoff 都是一行任何 profile（或人类）都能看见和编辑的记录。

[为什么重要] 理解这个区别是理解整个 Kanban Swarm 设计的前提。你不是在"调用一个子程序"，你是在"往一个持久化队列里投递任务"。

---

## 2. 核心架构：四个关键组件

### 2.1 Board（看板）

Board 是一个独立的任务队列，拥有自己的 SQLite DB、workspaces 目录和 dispatcher 循环。一次安装可以有多个 board（比如每个项目一个），单项目用户永远只需要 `default` board。

每个 board 的隔离是绝对的：
- 独立的 SQLite DB（`~/.hermes/kanban/boards/<slug>/kanban.db`）
- 独立的 `workspaces/` 和 `logs/` 目录
- Worker 只能看到自己 board 的任务——dispatcher 在子进程环境中设置 `HERMES_KANBAN_BOARD`，所有 `kanban_*` 工具都会读取它

### 2.2 Task（任务）

一行记录，包含：title、可选 body、一个 assignee（profile 名称）、status（`triage | todo | ready | running | blocked | review | done | archived`）、可选 tenant 命名空间、可选 idempotency key（去重用）。

### 2.3 Dispatcher（调度器）

一个长驻循环，默认每 60 秒执行一次：
- 回收过期的 claim
- 回收崩溃的 worker（PID 消失但 TTL 未过期）
- 收割已完成 run 后仍存活的 worker
- 将 `todo` 任务提升为 `ready`（当所有 parent 都 `done` 时）
- 原子性地 claim 并 spawn 对应的 profile

Dispatcher 默认运行在 gateway 进程内部（`kanban.dispatch_in_gateway: true`），无需额外安装或管理。

### 2.4 Workspace（工作空间）

Worker 操作的目录，三种类型：
- **scratch**（默认）—— 临时目录，任务完成后删除。显式声明的 artifacts 会在清理前复制到持久存储
- **dir:<path>** —— 共享目录，必须绝对路径。任务完成后保留
- **worktree** —— git worktree，用于编码任务。任务完成后保留

[为什么重要] Workspace 的设计体现了 Hermes 的务实哲学：大多数任务是临时的（scratch），但需要保留的任务（代码、文档）有明确的保留机制。你不需要在"全部保留"和"全部删除"之间做选择。

---

## 3. Kanban Swarm：一种特定的多 Agent 拓扑

Kanban Swarm 是 Hermes v0.17 引入的一种**预定义多 Agent 协作拓扑**，用一条命令创建一个完整的任务图：

```bash
hermes kanban swarm "写一篇关于 RAG 优化的深度文章" \
  --workers researcher-a,researcher-b \
  --verifier reviewer \
  --synthesizer writer
```

这条命令原子性地创建：
1. **Root/Blackboard 卡片** —— 已完成状态，作为共享黑板，存储跨 worker 的结构化上下文
2. **N 个并行 Worker 卡片** —— 每个 worker 独立执行，互不阻塞
3. **Verifier 卡片** —— 依赖所有 worker，等它们全部完成后启动
4. **Synthesizer 卡片** —— 依赖 verifier，等验证通过后启动

整个图一次性提交到 SQLite，dispatcher 和 dashboard 要么看到完整的 swarm，要么什么都看不到——不存在"部分链接"的中间状态。

### 3.1 Swarm 协议：Worker 之间如何通信？

Worker 之间**不直接通信**。它们通过 root 卡片的 comment thread 交换信息：

```python
# Worker A 写入黑板
kanban_comment(
    task_id="t_root",
    body='[swarm:blackboard] {"key": "findings", "value": {"topic": "RAG", "sources": 12}}'
)

# Worker B 读取黑板
kanban_show(task_id="t_root")  # 读取 comment thread 中的结构化 JSON
```

这种设计的好处：
- **解耦** —— Worker 不需要知道其他 Worker 的存在
- **可审计** —— 所有通信都记录在 SQLite 中
- **可恢复** —— Worker 崩溃后重启，可以从黑板恢复上下文

[为什么重要] 黑板模式是分布式系统中的经典协调机制。Hermes 把它搬到了 AI Agent 领域，用 SQLite 替代了 Redis/ZooKeeper，用 comment thread 替代了 pub/sub。简单、可靠、可审计。

---

## 4. Worker 生命周期：从 spawn 到 complete

每个 dispatcher-spawned worker 都会收到一段自动注入的系统提示（`KANBAN_GUIDANCE`），教它如何与 board 交互。Worker **不需要安装任何 skill**，生命周期协议是内置的。

### 4.1 标准流程

```python
# 1. 读取任务（无参数——使用 HERMES_KANBAN_TASK 环境变量）
kanban_show()

# 2. 进入工作空间，开始工作
# cd $HERMES_KANBAN_WORKSPACE && ...

# 3. 长任务期间定期发送心跳（每几分钟一次）
kanban_heartbeat(note="已完成 4/8 个文件的转换")

# 4. 完成任务
kanban_complete(
    summary="迁移了 limiter.py 到 token-bucket 算法；新增 14 个测试，全部通过",
    metadata={
        "changed_files": ["limiter.py", "tests/test_limiter.py"],
        "tests_run": 14
    }
)
```

### 4.2 阻塞与恢复

当 Worker 遇到无法自行解决的问题时：

```python
kanban_block(
    reason="需要用户提供 API key",
    kind="needs_input"  # dependency | needs_input | capability | transient
)
```

- `dependency` —— 等待另一个任务完成，自动恢复
- `needs_input` / `capability` / `transient` —— 浮现给人类，等待 unblock

人类可以从任何地方介入：

```bash
# 从 CLI
hermes kanban unblock t_abc

# 从 Telegram
/kanban unblock t_abc

# 从 Dashboard
拖拽卡片或点击 unblock 按钮
```

### 4.3 崩溃恢复

如果 Worker 进程崩溃（exit code != 0），dispatcher 会：
1. 记录 `crashed` 事件
2. 将任务重新放回 `ready` 队列
3. 在下一个 tick 重新 spawn

如果 Worker 进程正常退出（exit code = 0）但没有调用 `kanban_complete`，dispatcher 会记录 `protocol_violation` 事件，并在连续 3 次违规后自动 block 任务。

[为什么重要] 崩溃恢复是生产级系统的必备能力。Hermes 的 dispatcher 不仅检测崩溃，还能区分"可恢复的崩溃"和"协议违规"，避免无限重试。

---

## 5. 两个表面：工具 vs CLI

Kanban 有两个入口，背后是同一套 `kanban_db` 层：

### 5.1 Agent 通过工具驱动 Board

```python
kanban_show()
kanban_complete(summary="...", metadata={...})
kanban_block(reason="...")
kanban_heartbeat(note="...")
kanban_comment(task_id="...", body="...")
kanban_create(title="...", assignee="...", parents=[...])
kanban_link(parent_id="...", child_id="...")
```

这些工具在 dispatcher spawn worker 时自动注入到 schema 中。Worker **不会** shell out 到 `hermes kanban`——原因有三：
1. **后端可移植性** —— Worker 的 terminal 可能指向 Docker/Modal/SSH，那里没有 `hermes`
2. **无 shell 引号问题** —— 结构化参数避免了 `--metadata '{"files": [...]}'` 的转义噩梦
3. **更好的错误信息** —— 工具返回结构化 JSON，不是 stderr 字符串

### 5.2 人类通过 CLI 驱动 Board

```bash
hermes kanban list
hermes kanban show t_abc
hermes kanban create "write launch post" --assignee writer
hermes kanban comment t_abc "looks good, ship it"
hermes kanban unblock t_abc
hermes kanban dispatch --max 3
```

每个 CLI 动词都有对应的工具调用，反之亦然。两个表面永远不会漂移。

[为什么重要] 这种设计让 Agent 和人类使用同一套语义，但各自用最自然的方式交互。Agent 用工具调用，人类用 CLI/Dashboard/Slash 命令。

---

## 6. 实战：搭建一个 Swarm

假设你要写一篇关于"AI Agent 内存管理"的深度文章，需要：
- 两个研究员并行调研不同方向
- 一个 Reviewer 审核研究质量
- 一个 Writer 综合成文章

### 6.1 创建 Profile

```bash
# 创建专用 profile
hermes profile create researcher-a --description "专注学术论文调研"
hermes profile create researcher-b --description "专注工程实践调研"
hermes profile create reviewer --description "审核研究质量和引用准确性"
hermes profile create writer --description "综合研究成果，撰写深度文章"
```

### 6.2 启动 Gateway

```bash
hermes gateway start
```

### 6.3 创建 Swarm

```bash
hermes kanban swarm "写一篇关于 AI Agent 内存管理的深度文章" \
  --workers researcher-a,researcher-b \
  --verifier reviewer \
  --synthesizer writer
```

### 6.4 监控进度

```bash
# 实时查看所有事件
hermes kanban watch

# 查看统计
hermes kanban stats

# 查看特定任务
hermes kanban show t_abc
```

### 6.5 人类介入

如果 Reviewer 发现研究质量不够：

```bash
# 从 Telegram 发送
/kanban comment t_reviewer "请重点关注 2025 年后的论文"
/kanban unblock t_reviewer
```

[为什么重要] 整个流程从创建到监控到介入，都不需要写一行代码。Profile 定义能力，Swarm 定义拓扑，Dispatcher 自动调度。你只需要描述目标。

---

## 7. 高级特性

### 7.1 Goal Mode：持续迭代直到达标

默认情况下，每个 Worker 只有一次机会——做完工作，调用 `kanban_complete`，退出。但对于开放式任务，你可能需要"持续迭代直到满足条件"：

```bash
hermes kanban create "将文档翻译成法语" \
  --body "验收标准：每个页面都翻译完成，没有英文残留，链接完整" \
  --assignee linguist \
  --goal \
  --goal-max-turns 15
```

Goal mode 会在每个 turn 后运行一个辅助 judge，检查 Worker 的输出是否满足 card 的 title + body（作为验收标准）。如果不满足且还有 turn 预算，Worker 会在**同一个 session** 中继续工作，直到 judge 同意、Worker 自己终止任务、或预算耗尽（此时会 block 任务供人类审查）。

### 7.2 Per-task Model Override

不同任务可以用不同模型：

```bash
# 创建时指定
hermes kanban create "hard refactor" --assignee coder \
    --model claude-opus-4.6 --provider anthropic

# 或稍后修改
hermes kanban set-model t_abcd claude-opus-4.6 --provider anthropic
```

**成本策略：** Orchestrator 用 frontier 模型（分解任务需要判断力），Worker 用廉价模型（执行明确范围的任务不需要）。Worker 是 token 消耗的大头，所以 Worker 模型是成本控制的关键。

### 7.3 PR Completion Contract

对于代码任务，可以声明 PR 完成合约：

```bash
hermes kanban create "fix auth bug" --assignee coder \
    --completion-contract OWNER/REPO
```

Worker 发布 PR 后，Kanban 会自动检查 GitHub 的 branch protection 规则，只有当所有 required checks 都通过时，才能 complete 任务。这确保了"代码合并"和"任务完成"的原子性。

---

## 8. 设计哲学：为什么是 SQLite？

你可能会问：为什么不用 Redis？为什么不用 PostgreSQL？为什么用 SQLite？

答案在于 Hermes 的设计哲学：**单用户、单主机、可嵌入**。

- **单用户** —— Kanban 不是为多租户 SaaS 设计的，它是为个人开发者或小团队设计的
- **单主机** —— 所有 Worker 都在同一台机器上运行（或通过 SSH/Docker 远程执行，但 DB 在本地）
- **可嵌入** —— SQLite 是一个文件，无需安装、无需配置、无需维护

WAL 模式确保了读循环不会阻塞 dispatcher 的 `BEGIN IMMEDIATE` claim 事务。append-only 的 `task_events` 表让 WebSocket 推送变得简单——只需要 tail 最新的 event id。

[为什么重要] 技术选型不是"什么最先进"，而是"什么最合适"。SQLite 在这个场景下是最合适的选择：零依赖、零配置、零维护、性能足够、可靠性极高。

---

## 9. 总结：Kanban Swarm 的意义

Hermes Agent v0.17 的 Kanban Swarm 不是又一个"多 Agent 框架"。它解决的是一个更根本的问题：**如何让 AI Agent 像人类团队一样协作，同时保持机器级别的可审计性和可恢复性。**

关键设计决策：
1. **持久化优先** —— 所有状态都在 SQLite 中，崩溃可恢复，历史可追溯
2. **对等协作** —— 任何 profile 都能读写任何任务，没有"主从"关系
3. **人类介入** —— 随时可以 comment、unblock、reassign，Agent 不是黑盒
4. **简单即美** —— SQLite + 一个 dispatcher + 一套工具协议，没有复杂的中间件

对于 AI 开发者来说，Kanban Swarm 提供了一个生产级的多 Agent 协调方案。你不需要自己实现任务队列、状态机、崩溃恢复、审计日志——Hermes 都帮你做好了。

你只需要定义 Profile，描述目标，然后看着你的 Agent Swarm 开始工作。

---

## 参考资源

- [Hermes Agent 官方文档](https://hermes-agent.nousresearch.com/docs/)
- [Kanban 功能详解](https://hermes-agent.nousresearch.com/docs/user-guide/features/kanban)
- [Kanban 教程](https://hermes-agent.nousresearch.com/docs/user-guide/features/kanban-tutorial)
- [GitHub 仓库](https://github.com/NousResearch/hermes-agent)

---

*本文作者正在使用 Hermes Agent v0.17 的 Kanban Swarm 机制撰写本文——是的，这篇文章本身就是 Kanban Swarm 的产物。*
