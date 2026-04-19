---
name: jules
description: 使用 Google Jules 异步编码 agent 执行多文件编码任务。通过 jules CLI 创建会话、后台等待、拉取 patch、本地 apply 合并。
argument-hint: <任务描述>
allowed-tools:
  - Bash
  - Read
  - Grep
  - Glob
---

# 使用 Jules 进行开发

Jules 是 Google 的异步编码 agent（基于 Gemini），在云端 VM 里执行任务、跑测试，产出 git patch。配套 CLI 是 [`@xbghc/jules-cli`](https://www.npmjs.com/package/@xbghc/jules-cli)（源码 [xbghc/jules-cli](https://github.com/xbghc/jules-cli)），专为 agent 驱动设计：所有命令 stdout 输出 JSON，错误写 stderr + 非零退出码。

**设计取向**：默认**不让 Jules 开 PR、不留远程分支**。Jules 只当云端沙箱产出 unified diff，分支/PR 完全由本地 agent 自己管——失败的会话零远程痕迹，不会污染 PR 队列。

## 何时委托给 Jules

| 适合 Jules | 本地执行更好 |
|------------|-------------|
| 多文件修改 | 单文件创建 |
| 功能实现、重构 | 配置/小编辑 (<50 行) |
| 复杂逻辑、可能需要调试 | 简单、可预测 |

**经验法则**：Jules 一次往返的成本是"创建 → 等待 → 拉 patch → 本地 apply → 开 PR"，只在预计改动 ≥3 文件或 ≥50 行时才值得。能一次本地改完就本地改。

---

# 第一部分：CLI 使用

## 安装

```bash
# 全局（推荐）
npm i -g @xbghc/jules-cli

# 或按需
npx -y @xbghc/jules-cli <cmd>
```

## 环境变量

- `GOOGLE_JULES_API_KEY`（必需）— 从 https://jules.google/settings 获取。未设置时所有请求命令会 exit 1。

## 命令参考

全部命令的最终结果以 JSON 格式输出到 stdout。错误信息写 stderr，退出码 ≠ 0。

### `jules create` — 创建会话

```
jules create --prompt <p> --title <t> [--source <s>] [--branch <b>] [--auto-create-pr]
```

| 选项 | 说明 |
|------|------|
| `--prompt` | 任务描述（必填）。长文本用 `$(cat prompt.md)` |
| `--title` | 会话标题（必填）。用于 `list` 显示和 stderr 日志定位 |
| `--source` | 仓库标识。默认从 `git remote get-url origin` 推断 |
| `--branch` | 起始分支。默认当前本地分支 |
| `--auto-create-pr` | 让 Jules 自动开 PR（默认不开）。只在你想走 PR 审查流时加，agent 默认用 `jules patch` 拿 diff 更干净 |

**输出**：
```json
{
  "sessionId": "sessions/xxx",
  "name": "sessions/xxx",
  "url": "https://jules.google/task/...",
  "message": "Session created successfully"
}
```

### `jules wait` — 阻塞等待到终止态（关键命令）

```
jules wait <sessionId> [--timeout-minutes <n>]
```

- 每 5 分钟向 **stderr** 写一行 `[ISO时间] state=X url=Y`（便于后台进程日志观察）
- 到达终止态（`COMPLETED` / `FAILED` / `AWAITING_USER_FEEDBACK` / `AWAITING_PLAN_APPROVAL`）时，向 stdout 写完整 JSON 并退出 0
- 不传 `--timeout-minutes` 时**永不超时**

**必须配合 `Bash(run_in_background: true)` 使用**——见下方"执行流程"。

### `jules patch` — 拉取 session 产出的 patch（新核心命令）

```
jules patch <sessionId>
```

聚合 session 所有 activities 里的 `gitPatch`，输出 JSON：

```json
{
  "sessionId": "sessions/xxx",
  "patches": [
    {
      "activityId": "...",
      "createTime": "2026-04-19T...",
      "source": "sources/github/owner/repo",
      "baseCommitId": "a1b2c3d4e5f6",
      "unidiffPatch": "diff --git a/src/foo.ts b/src/foo.ts\n...",
      "suggestedCommitMessage": "Add foo module"
    }
  ]
}
```

**典型用法**：
```bash
jules patch <sessionId> > /tmp/jules-<id>.json
jq -r '.patches[-1].unidiffPatch' /tmp/jules-<id>.json > /tmp/jules-<id>.patch
git apply /tmp/jules-<id>.patch
```

Jules 多阶段迭代时可能产出多个 patch（每个对应一个 activity）。最新的是最后一个，一般只需要 `.patches[-1]`。全部返回是让 agent 自己判断是否需要看中间步骤。

### `jules get` — 查询会话

```
jules get <sessionId>
```

输出：
```json
{
  "id": "sessions/xxx",
  "title": "...",
  "state": "IN_PROGRESS" | "COMPLETED" | ...,
  "url": "https://jules.google/task/...",
  "prUrl": null
}
```

注：`prUrl` 仅在 `--auto-create-pr` 启用时非 null。默认流程下永远 null，用 `jules patch` 拿改动内容。

### `jules list` — 列出会话

```
jules list [--page-size <n>]
```

默认 `--page-size 10`。输出 `[{ id, title, state, url }, ...]`。

### `jules send` — 向会话发消息

```
jules send <sessionId> <message>
```

位置参数：`sessionId` 后面全部 join 成 message。用于 `AWAITING_USER_FEEDBACK` 状态的回复，或给运行中的会话追加反馈。

### `jules approve` — 批准计划

```
jules approve <sessionId>
```

CLI 的 `create` 默认不走计划审批，一般用不到。如果 `state=AWAITING_PLAN_APPROVAL` 说明服务端触发了人工审核，用这个命令放行。

### `jules activities` — 获取会话时间线（诊断用）

```
jules activities <sessionId> [--page-size <n>] [--page-token <t>]
```

返回会话中所有 activity 事件的完整数据。每个 activity 有一个"事件类型"字段：

| 字段 | 内容 |
|------|------|
| `planGenerated.plan` | Jules 生成的执行计划 |
| `planApproved.planId` | 被批准的计划 ID |
| `userMessaged.userMessage` | 用户发的消息（你用 `send` 发的） |
| `agentMessaged.agentMessage` | Jules 回复的消息（`AWAITING_USER_FEEDBACK` 时的提问） |
| `progressUpdated.{title,description}` | 中间进度更新 |
| `sessionCompleted` | 无额外字段 |
| `sessionFailed.reason` | **失败原因**（FAILED 诊断的关键） |
| `artifacts[].changeSet.gitPatch` | 改动补丁（`jules patch` 只提取这个） |

**FAILED 诊断**：`jq '.activities[] | select(.sessionFailed) | .sessionFailed.reason' <jules-activities-output>` 直接拿失败原因，不用去 Web UI。

**Jules 提问时拿内容**：`jq -r '[.activities[] | select(.agentMessaged) | .agentMessaged.agentMessage] | last'` 拿到最后一条 agent 消息。

### `jules sources` — 列出 / 查看已连接仓库

```
jules sources                                   # list
jules sources <sourceId>                        # get single
jules sources [--page-size <n>] [--page-token <t>] [--filter <f>]
```

Source 代表 Jules 已授权访问的 GitHub 仓库。`create` 时的 `--source` 值必须是这里列出的 source 的 `name`（格式 `sources/{id}`）。

**用途**：
- 首次使用 / 跨仓库任务：`jules sources` 先看授权清单
- `create` 报错 "source not found" 时，`jules sources` 核对仓库是否已连接

### `jules delete` — 删除会话

```
jules delete <sessionId>
```

prompt 方向错了，直接删掉重建。因为默认不开 PR、不留分支，删除会话就清空所有远程痕迹。

---

# 第二部分：Jules 工作流

## 前置条件

1. `GOOGLE_JULES_API_KEY` 已设置。
2. **本地改动已 push 到远程**。Jules 只读远程分支。前置接口/类型/依赖没 push，Jules 就拿不到。
3. CLI 默认用 `git remote get-url origin` 和当前分支。要改仓库/分支用 `--source` / `--branch`。

## Prompt 模板

```
## Context
项目背景、技术栈、相关现有文件路径

## Task
具体要实现什么

## Constraints
- Only modify: <可修改的文件/目录>
- Do not modify: <不要碰的文件>

## Criteria
- [ ] 验收标准
- [ ] 测试通过
```

`Constraints` 写明可修改范围，是避免 Jules 跨模块越权修改的最有效手段。

**简短示例**：

```
## Context
Vue 3 + TS 项目，类型在 src/types/user.ts

## Task
在 src/modules/user/ 实现 UserList/UserForm/UserDetail + api.ts + Pinia store

## Constraints
- Only modify: src/modules/user/**
- 复用 src/types/user.ts 的类型

## Criteria
- [ ] CRUD 正常，无 any
- [ ] npm run lint 通过
```

## 执行流程（patch 驱动，推荐）

`create → wait（后台）→ patch → 本地 apply → 本地开 PR → 合格则 merge`。

```bash
# 1. 同步远程
git status && git push

# 2. 创建会话（不加 --auto-create-pr；Jules 默认不会开 PR）
jules create --prompt "$(cat prompt.md)" --title "实现用户管理模块"
# stdout: { "sessionId": "sessions/xxx", ... }

# 3. 后台等待（Bash 调用带 run_in_background: true）
jules wait <sessionId> --timeout-minutes 120 > /tmp/jules-<sessionId>.json

# 4. 进程退出后读取 /tmp/jules-<sessionId>.json 按 state 分支处理
#    COMPLETED 时拉 patch
jules patch <sessionId> > /tmp/jules-<sessionId>.patches.json

# 5. 审查 patch（agent 可以直接读 unidiffPatch 判断改动是否合格）
jq -r '.patches[-1].unidiffPatch' /tmp/jules-<sessionId>.patches.json > /tmp/jules-<sessionId>.patch
# 或交给 agent 直接 Read 上面的 JSON 文件分析

# 6. 合格：本地 apply → 新分支 → push → 开 PR → merge
git checkout -b jules/<短描述> origin/<startingBranch>
git apply /tmp/jules-<sessionId>.patch
git add -A
git commit -m "$(jq -r '.patches[-1].suggestedCommitMessage' /tmp/jules-<sessionId>.patches.json)"
git push -u origin HEAD
gh pr create --fill && gh pr merge --merge
git checkout <startingBranch> && git pull

# 6'. 不合格：丢弃会话（零远程痕迹）
jules delete <sessionId>
rm /tmp/jules-<sessionId>.*
```

**关于后台等待**：`jules wait` 后台运行时不占用上下文，也不阻塞 Claude 其他工具调用。进程到终止态退出时 Claude 收到通知，用 `BashOutput` 按 shell_id 读取，或直接 `Read` 重定向的文件。

**关于 apply 冲突**：`baseCommitId` 字段告诉你 Jules 基于哪个 commit 产出的 patch。如果 `startingBranch` 在 session 跑的过程中推进了，可能冲突。用 `git apply --3way` 或 `git checkout <baseCommitId> -b tmp && git apply` 再 rebase 上最新分支处理。

## 终止态处理

| 状态 | 操作 |
|------|------|
| `COMPLETED` | `jules patch <sessionId>` 拉 diff。agent 读 `unidiffPatch` 判断是否在 `Constraints` 范围内。合格则本地 apply + 开 PR；不合格 `jules delete` 丢弃 |
| `FAILED` | `jules activities <sessionId>` 拿 `sessionFailed.reason` 看失败原因。错误局限在单文件/函数 → `jules send <sessionId> "<修正>"` 追加反馈；prompt 方向错了 → `jules delete` 后重写新建 |
| `AWAITING_USER_FEEDBACK` | Jules 在主动提问，`jules activities <sessionId>` 拿最后一条 `agentMessaged.agentMessage` 看问题；`jules send <sessionId> "<回复>"` 回答 |
| `AWAITING_PLAN_APPROVAL` | 正常不会出现；若出现说明服务端强制人工审核，`jules approve <sessionId>` 放行 |

## 并行会话

两种用途：**多个独立定向任务**（不同模块同时推进）和 **发散任务的并发候选收集**（同一 Mission 起 N 份候选）。

核心模式：**每个 session 单独一次 `Bash(run_in_background: true)`** 跑 `jules wait <id> > /tmp/jules-<id>.json`，文件名即索引，任一完成都会单独通知 Claude。`--title` 取区分度高的描述便于 stderr 日志定位（并发同一 Mission 时建议加后缀 `#1`、`#2`、`#3`）。

patch 的 `baseCommitId` 互相独立，并行 session 的改动可以分别在各自的新分支上 apply，不会冲突。

### 示例：发散任务并发候选（N=3 Bug 狩猎）

发散任务天然是"多候选、取其一"——Jules 非确定性，3 份候选里通常能拿到 1 份可合并的，比串行重试期望耗时短。

```bash
# 1. 连续起 3 个 session（Claude 做 3 次独立的 Bash tool call，或 shell 循环）
jules create --prompt "$(cat mission.md)" --title "Bug Hunt #1"   # → sessionId_a
jules create --prompt "$(cat mission.md)" --title "Bug Hunt #2"   # → sessionId_b
jules create --prompt "$(cat mission.md)" --title "Bug Hunt #3"   # → sessionId_c

# 2. 3 次 Bash(run_in_background: true)，互不阻塞
jules wait <sessionId_a> --timeout-minutes 90 > /tmp/jules-a.json
jules wait <sessionId_b> --timeout-minutes 90 > /tmp/jules-b.json
jules wait <sessionId_c> --timeout-minutes 90 > /tmp/jules-c.json
# Claude 会在每个进程退出时分别收到通知

# 3. 全部退出后，对每个 COMPLETED 的 session 拉 patch
jules patch <sessionId_a> > /tmp/jules-a.patches.json
jules patch <sessionId_b> > /tmp/jules-b.patches.json
jules patch <sessionId_c> > /tmp/jules-c.patches.json

# 4. agent 用 Read 读 3 份 JSON，比较 unidiffPatch 的质量：
#    - 问题定位是否准确
#    - 修复是否最小化（改动范围 vs. 问题大小）
#    - 有没有顺便 scope creep 改别的东西
#    - 提交消息质量

# 5. 选最合格的 1 份：本地 apply + 开 PR
git checkout -b jules/fix-<short> origin/<startingBranch>
git apply /tmp/jules-<best>.patch
git add -A && git commit -m "..." && git push -u origin HEAD
gh pr create --fill && gh pr merge --merge

# 6. 其余全部 delete（零远程痕迹）
jules delete <sessionId_runner_up_1>
jules delete <sessionId_runner_up_2>
```

**推荐 N 值**：发散任务 3–5；定向任务不需要并发（规格明确，一次跑对就够）。超过 5 路并发 API 配额可能打满。

**注意**：FAILED 的 session 也要 `jules delete` 清掉——不 delete 会留在 `jules list` 里一直占视觉噪音。

## 何时用 `--auto-create-pr`

默认流程（不开 PR）适合大多数情况——agent 拿 patch 自己判断后再决定要不要开 PR。

在这些场景下可以考虑加 `--auto-create-pr`：
- 你信任该任务的成功率足够高（>80%），希望省掉"本地 apply + 手动 push"步骤
- 需要 PR 自带 CI/review bot 触发（有些工具只绑定到 PR 事件）
- 团队流程要求 Jules 产出的代码走 PR review

启用后工作流回到传统形式：`create --auto-create-pr → wait → gh pr diff prUrl → gh pr merge`。

## 常见陷阱

- **本地改动没 push**：Jules 看到旧代码，上下文/类型对不上。创建会话前永远先 `git status`。
- **Prompt 没写 Constraints**：Jules 可能动到无关文件。apply patch 前一定审 `unidiffPatch` 的文件列表。
- **没等 `wait` 退出就拉 patch**：`IN_PROGRESS` 状态下 activities 可能还没产出最终 patch。以 `jules wait` 进程退出、`state=COMPLETED` 为准。
- **轮询 `get` 代替 `wait`**：每次 `get` 调用消耗一次工具调用和上下文。`wait` 后台阻塞不消耗上下文，是 agent 友好的唯一方式。
- **直接用 `patches[0]` 而非 `patches[-1]`**：多阶段迭代时第一个是初稿，最后一个才是定稿。一般取最后一个。
