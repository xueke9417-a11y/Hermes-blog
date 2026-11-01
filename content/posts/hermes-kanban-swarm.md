---
title: 当 AI Agent 学会分工协作：Hermes Kanban Swarm 深度解析
date: 2026-11-01T15:40:00+08:00
tags: [Hermes Agent, Kanban, AI Agent]
author: xueke9417-a11y
draft: false
---

你有**一个复杂任务——需要调研、架构设计、实现、审查和总结。一个人做太慢，但协调多个 AI agent 又像在放猫：谁先做？结果怎么汇总？中间状态怎么追踪？

Hermes Agent 的 Kanban Swarm 给出了一个出乎意料的答案：**不引入第二个调度器，而是用一张 SQLite 表把多 agent 协作退化为可审计的任务图。**

> 本文基于官方文档、源码 commit `95d0629`（与 main HEAD `9bba78e` 逐字节相同）及 PyPI 0.19.0 的实测验证撰写。调研完成于 2026-11-01，所有版本号与 commit 均可复核。

## 1. Swarm 是拓扑助手，不是调度器

理解 Swarm 的第一个关键：官方文档将其称为 "topology helper"（拓扑助手），而非独立调度器。它的工作是调用现有 Kanban 内核，一次性写出一张固定的任务图，然后把控制权交还给 Kanban dispatcher。

**为什么重要**：Swarm 不维护自己的运行时、不负责进程管理、不设定并发上限。它只负责"画图"，后续所有调度、spawn、状态流转都由 Kanban dispatcher 接管。理解了 Kanban，就理解了 Swarm 的运行时。

## 2. 任务图：单层扇出，无嵌套

Swarm 创建的任务图结构固定（源码 `kanban_swarm.py` 实测 `create_swarm` 验证）：

```
root (status=done)     ← 黑板与审计锚点
  ├── worker1 (ready)  ← 并行执行
  ├── worker2 (ready)  ← 并行执行
  └── ...
      ↓ (全部完成后)
verifier (todo)        ← 依赖所有 worker
      ↓ (完成后)
synthesizer (todo)     ← 依赖 verifier
```

依赖关系：`root ← []`、`workers ← root`、`verifier ← all workers`、`synthesizer ← verifier`。

**为什么重要**：

- **单层扇出**：root 完成后所有 worker 同时 ready。不存在嵌套 swarm、多 verifier 或重试回退。若需多级分解，要在 worker 内部自行用 `kanban_create` 拆分。
- **root 即完成**：root 卡创建即为 `done`，同时充当共享黑板和审计锚点。其 assignee 默认为 `created_by`（默认值 `"swarm-orchestrator"`）。
- **skills 硬编码**：verifier 的 skills 硬编码为 `["requesting-code-review"]`，synthesizer 硬编码为 `["humanizer"]`（源码 L238/L252）。CLI 无法覆盖这两者——只有 worker 可通过 `--worker PROFILE:TITLE:SKILL,SKILL` 第三段传 skills。
- **每张卡尾部追加 `## Swarm protocol`**（`_swarm_context()` 生成）：包含 root id、读 sibling/parent handoff 的指引、completion metadata 写法、跨 worker 注释结构、goal 原文。

## 3. 黑板机制：root 卡上的 JSON 注释

多 agent 协作的核心难题是上下文共享。Swarm 的方案是 **blackboard（黑板）**——基于 root 卡注释的简易键值存储。

**工作机制**（源码 `latest_blackboard()` 审计）：

- **写入**：在 root 卡添加以 `[swarm:blackboard] ` 开头的注释，正文为 JSON `{"key": "...", "value": "..."}`。
- **读取**：遍历 root 卡全部注释，筛选带前缀且能 JSON 解析的条目，按 key 后者覆盖前者（last-write-wins），另产出 `_authors`（每个 key 最终由谁写入）。

**为什么重要——也为什么它很脆弱**：

1. **仅认 root 卡**：worker/verifier 卡上的注释不进入黑板。
2. **无并发保护**：多个 worker 同时写同一 key，后写者覆盖先写者。无版本历史、无 schema 校验，非法 JSON 被静默跳过。
3. **无专用写入工具**：CLI 没有 `blackboard` 子命令，工具层无对应 tool。worker 只能手动写前缀 JSON 注释，或从 Python 侧调 `post_blackboard_update()`。
4. **不自动注入**：黑板内容不会自动注入 worker 提示词。`_swarm_context()` 只给 root id 和"去写注释"的要求，worker 需主动读取。

## 4. 最硬的坑：文档写 `--workers`，代码只有 `--worker`

这是本文最需要读者注意的发现。

官方文档（[英文 kanban.md](https://hermes-agent.nousresearch.com/docs/user-guide/features/kanban) L1060–1070）的 Swarm 示例使用 `--workers` 参数：

```bash
# 官方文档示例（实际无法运行 ⚠️）
hermes kanban swarm "重构认证模块" \
  --workers researcher,architect,sre \
  --verifier reviewer \
  --synthesizer writer
```

但源码实际定义的是**可重复**的 `--worker`（`action="append"`），格式为 `PROFILE:TITLE[:SKILL,SKILL]`。研究者从 PyPI 0.19.0 wheel 中 import argparse 树实测，按文档示例构造的参数被 argparse 拒绝：

```
$ hermes kanban swarm G --workers a,b --verifier r --synthesizer w
hermes: error: unrecognized arguments: --workers a,b
# 退出码 2
```

`kanban_parser.py` HEAD L222–233 中 `--worker` 出现 1 次，`--workers` 出现 **0 次**；0.19.0 的 `kanban.py` L377–390 同样如此。文档错，实现对。

**正确的命令格式**（源码审计得出）：

```bash
hermes kanban swarm "重构认证模块" \
  --worker researcher:调研依赖与安全需求 \
  --worker architect:设计API与数据模型 \
  --worker sre:编写部署脚本 \
  --verifier reviewer \
  --synthesizer writer \
  --json
```

> ⚠️ **版本与验证说明**：上述 `--help` 输出由在 Docker 沙箱内 import PyPI 0.19.0 wheel 的 argparse 树产生，非 `hermes` 可执行文件直接输出，可能有全局参数（如 `--profile`）的包装差异。宿主机 `hermes --version` 未取得。命令格式结论来自源码审计，可信度高；具体运行输出未经端到端验证。

**附带陷阱**：`parse_worker_arg` 使用 `split(":", 2)` 分割参数。标题含冒号会被误判为 skills 段；只给 profile 不给 title 会报错 `worker must be profile:title or profile:title:skill,skill`。

## 5. Verifier 的 gate：约定，不是门禁

verifier 卡的 body 要求"仅在 `metadata {"gate": "pass"}` 时 complete，否则 block 并写明缺失"。听起来像一道硬门禁。

但研究者审查全仓库代码后发现：**没有任何代码读取 `gate` 字段**。`metadata.get("gate")` 在整个代码库零出现。

真正生效的是依赖边：

- verifier 等所有 worker `done` 才 `ready`；
- synthesizer 等 verifier `done` 才 `ready`。

如果 verifier 不写 `gate: pass` 直接 complete，synthesizer **照常被唤醒**。`gate` 是给人 和下游 agent 读的提示词约定，不是代码门禁。

**为什么重要**：如果把 Swarm 用于生产流程并依赖 verifier 做质量门禁，不能只靠 `gate` 字段——它是"君子协定"。真正的阻塞来自依赖图拓扑。

## 6. 隔离与并行边界

| 维度 | 机制 |
|---|---|
| 进程隔离 | 每张卡由 dispatcher spawn 为独立 OS 进程，profile 身份独立，非同进程 subagent |
| 委派限制 | `swarm` 被列入 `_DELEGATED_CHILD_DENIED_ACTIONS`，被委派的子 agent 不能再建 swarm 或建卡 |
| 工作区隔离 | `scratch`（默认，完成即删）/ `worktree`（保留）/ `dir:<path>`（共享）；`workspace_kind` 下发到 swarm 内所有卡 |
| 并行上限 | Swarm 自身不设上限；root done 后所有 worker 同时 ready。实际并发由 `kanban.max_in_progress` 和 `max_in_progress_per_profile` 约束（默认 unset = 不限） |
| 可见性 | worker 彼此看不到对方卡（官方文档 "Workers cannot see sibling cards"），共享只能靠 root 黑板或 completion metadata |

## 7. 版本差异：main 与 0.19.0

| 行为 | main @9bba78e | PyPI 0.19.0 |
|---|---|---|
| 建图原子性 | 单 `write_txn()` 事务内建完；root 先 `blocked` 再同事务内 CAS 为 `done` | 先建 root，再 `complete_task`，再逐个建子卡——非原子，有可观测中间态 |
| workspace_kind 默认 | `None`（继承 board 默认） | `"scratch"`（完成即删） |
| kanban_swarm.py | 300 行 / 11,301 B | 与 main 差 365 行 / 9,221 B |

**为什么重要**：`pip install hermes-agent` 装的是 0.19.0，建图非原子意味着 dispatcher 可能在中间态观察到不完整的图。main 已修复但尚未发布到 PyPI。涉及行为断言时务必标注 commit 或版本号。

## 8. 文档缺口

对中文读者尤其值得注意的是：官方中文 Kanban 文档（`zh-Hans` 版）**零处提及 Swarm**（863 行，0 匹配）；英文文档也仅在 L1060–1070 有一个段落（且含 `--workers` 错误）。`kanban-worker-lanes.md`（144 行）同样未涉及。源码是唯一权威参考。

## 9. 适用场景与限制

**适合**：

- 多个不同 profile 并行处理同一目标的不同切面（如调研 + 架构 + 运维），最终由 verifier 汇总、synthesizer 整理
- 任务可清晰分解为"并行 + 串行审查 + 串行综合"三阶段
- 团队已使用 Hermes Kanban，希望用一张命令铺出协作图

**不适合**：

- 需要嵌套分解（worker 内再 fan-out）——Swarm 是单层
- 需要严格质量门禁——`gate` 仅是约定，无代码校验
- 需要版本化共享状态——黑板是 last-write-wins，无历史
- 需要精细并发控制——Swarm 自身不设上限

## 来源

1. 官方 Kanban 文档：https://hermes-agent.nousresearch.com/docs/user-guide/features/kanban （HTTP 200，2026-11-01T05:44Z 抓取）
2. 源码 `kanban_swarm.py` @95d0629：https://github.com/NousResearch/hermes-agent/blob/95d0629df4167a2ad8f09e5d7f96d8b5c790c90b/hermes_cli/kanban_swarm.py （与 main HEAD 逐字节相同，sha256 `d1f37252…c03f306c0`）
3. `kanban_parser.py` HEAD：https://github.com/NousResearch/hermes-agent/blob/main/hermes_cli/kanban_parser.py
4. 官方仓库：https://github.com/NousResearch/hermes-agent （HEAD `9bba78e`）
5. PyPI 元数据：https://pypi.org/pypi/hermes-agent/json （最新 0.19.0）
6. 官方中文 Kanban 文档：https://github.com/NousResearch/hermes-agent/blob/main/website/i18n/zh-Hans/docusaurus-plugin-content-docs/current/user-guide/features/kanban.md

> **限制声明**：本文所有事实性断言来自调研报告（研究卡 t_7b788006 评论 #7），该报告基于源码审计与沙箱内 argparse 实测。未验证项包括：宿主机 `hermes` 二进制的实际版本号（沙箱无该二进制）、dispatcher 端到端并发实测、全仓库引用审计（blobless 克隆超时未完成）。`--help` 输出来自 0.19.0 wheel 的 argparse 树而非可执行文件。
