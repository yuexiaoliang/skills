---
name: tmux-claude-babysitter
description: "Babysit a Claude Code instance running inside a tmux session. Activate ONLY when user's message starts with '/tmux-claude-babysitter'. The prefix is stripped; the remaining natural language content is parsed by the model to extract parameters. If project_dir is missing, ask the user. Covers: session creation, Claude Code startup, idle/menu detection, interactive menu handling, task completion determination, error recovery with retry/restart, state tracking, and logging."
metadata:
  author: yuexiaoliang
  version: "1.0.0"
---
# Tmux Claude Babysitter

在 tmux 会话中看护 Claude Code 实例的完整生命周期。负责启动、监测、错误自治恢复和退出，**不介入任务逻辑本身**。

## 触发方式

**强制前缀**：用户消息必须以 `/tmux-claude-babysitter` 开头。前缀之后的所有内容视为参数的自然语言描述。

## 参数提取

去掉 `/tmux-claude-babysitter` 前缀后，从剩余自然语言内容中提取以下参数。使用模型理解能力推断，不依赖严格的格式匹配。

| 参数 | 是否必须 | 默认值 | 提取规则 |
| --- | --- | --- | --- |
| `project_dir` | **是** | — | 从剩余内容中提取路径。优先识别 `~/` 或 `/` 开头的绝对路径。若未找到，**必须向用户提问**。 |
| `core_task` | 否 | `/core-task` | 去掉 `project_dir` 后剩余的自然语言内容。可以是 slash 指令（如 `/deploy`），也可以是任务描述（如 "分析今天的日志并生成报告"）。若剩余内容为空，使用默认值。 |
| `tmux_session` | 否 | 目录 basename | 若用户显式提到 session 名称（如 "session myproj"），使用指定值；否则取 `project_dir` 的 basename。 |
| `max_wait_minutes` | 否 | 240 | 若用户提到时间（如 "等 2 小时"、"timeout 120m"），转换为分钟；否则默认 240。 |
| `poll_interval_seconds` | 否 | 60 | 若用户提到轮询间隔，使用指定值；否则 60 秒。 |
| `skip_permissions` | 否 | `true` | 若用户说 "with permissions" 或 "交互模式"，设为 `false`；否则默认 `true`。 |
| `state_dir` | 否 | `{project_dir}/.hermes` | 若用户指定了状态目录路径，使用指定值。 |

### 参数分离规则

剩余内容的处理顺序：

1. **先提取** `project_dir`：识别所有路径样式的文本（`~/...` 或 `/...`），取第一个有效路径。
2. **剩余内容作为** `core_task`：去掉路径后，剩余的自然语言全部作为任务描述。如果用户说 "分析这个项目"，而项目路径已单独提取，则 `core_task = "分析这个项目"`。

### 缺失参数处理

- `project_dir` **缺失**：向用户提问，例如 "请指定要看护的项目目录（如 \~/projects/myapp）"。
- **其他参数缺失**：使用默认值，不提问。

### 示例

```
"/tmux-claude-babysitter 分析 ~/projects/webflow-flutter 这个项目"
→ project_dir=~/projects/webflow-flutter, tmux_session=webflow-flutter, core_task=分析这个项目

"/tmux-claude-babysitter ~/projects/myapp"
→ project_dir=~/projects/myapp, tmux_session=myapp, core_task=/core-task

"/tmux-claude-babysitter /home/user/work/foo 跑测试，等 1 小时"
→ project_dir=/home/user/work/foo, tmux_session=foo, core_task=跑测试, max_wait_minutes=60

"/tmux-claude-babysitter 分析这个项目"
→ project_dir 缺失 → 向用户提问
```

---

## 执行流程概览

 1. 读取 `{state_dir}/state.json`，了解历史执行上下文
 2. 检查 tmux `{tmux_session}` 会话，不存在则创建
 3. 在 tmux `{tmux_session}` 会话中启动 Claude Code（若当前未运行）
 4. 确保处于 idle 状态（取消可能存在的交互式菜单）
 5. 发送 `{core_task}` 指令
 6. 监测任务执行，轮询 pane 内容
 7. 处理交互式菜单
 8. 判定任务完成或失败
 9. 必要时执行错误自治恢复
10. 收到完成信号后，执行 `/exit` 关闭 Claude Code 会话
11. 更新 `{state_dir}/state.json`
12. 写日志、git commit/push

---

## 1. 读取历史状态

读取 `{state_dir}/state.json`：

```bash
cat {state_dir}/state.json
```

关注字段：

| 字段 | 用途 |
| --- | --- |
| `consecutive_failures` | 判断是否为系统性问题 |
| `last_error` | 了解上次失败原因 |
| `last_recovery_action` | 避免重复无效的恢复方式 |
| `failure_history` | 识别失败模式 |

**注意：** 若文件不存在，用以下初始结构创建：

```json
{
  "last_run": null,
  "status": null,
  "retry_count": 0,
  "consecutive_failures": 0,
  "last_error": null,
  "last_recovery_action": null,
  "failure_history": []
}
```

---

## 2. 检查并创建 tmux 会话

```bash
tmux has-session -t {tmux_session} 2>/dev/null || tmux new-session -d -s {tmux_session} -c {project_dir}
```

- `has-session` 检查会话是否存在
- `new-session -d` 在后台创建，`-c` 设置工作目录
- **目标 pane**：`{tmux_session}:0.0`（第一个 window 的第一个 pane）
- 请确保该 pane 未被其他程序占用

---

## 3. 检查 Claude Code 运行状态

获取 pane 最近 20 行：

```bash
tmux capture-pane -t {tmux_session}:0 -p | tail -20
```

**判定规则：**

| 特征 | 含义 | 动作 |
| --- | --- | --- |
| 看到 bash 提示符（`$` 或 `~$`） | 未运行 | 进入步骤 4 启动 |
| 看到 `❯` 或 Claude UI 元素 | 已在运行 | 进入步骤 5 |
| 看到活动指示器（`● Thinking` 等） | 正在执行 | 直接进入步骤 7 监测 |

**提示符检测启发式：** 以 `❯` 或 `>` 开头且后面紧跟空格的行，视为 Claude Code 提示符。

---

## 4. 启动 Claude Code

若未运行，发送启动命令：

```bash
tmux send-keys -t {tmux_session}:0 "cd {project_dir} && claude" Enter
```

若 `skip_permissions` 为 `true`：

```bash
tmux send-keys -t {tmux_session}:0 "cd {project_dir} && claude --dangerously-skip-permissions" Enter
```

等待 20 秒，然后再次捕获 pane 内容确认出现 `❯` 提示符。

**安全警告：** `--dangerously-skip-permissions` 让 Claude Code 自动执行所有工具调用，无需用户逐条确认。仅在受信任的、已版本控制的项目中使用。不建议用于处理敏感数据或生产环境部署。用户可通过参数显式关闭（`with permissions` → `skip_permissions=false`），但会破坏无人值守能力。

---

## 5. 确保处于 idle 状态

如果 Claude 显示交互式菜单（编号选项），先发送 Escape 取消，等待 2 秒：

```bash
tmux send-keys -t {tmux_session}:0 Escape
sleep 2
```

**检测菜单：** 获取 pane 内容，查找编号选项 + `❯ 1. ... 2. ...` 格式。

---

## 6. 发送核心任务指令

```bash
tmux send-keys -t {tmux_session}:0 "{core_task}" Enter
```

然后进入监测循环。

---

## 7. 监测任务执行

每 `{poll_interval_seconds}` 秒轮询一次，获取最近 30 行：

```bash
tmux capture-pane -t {tmux_session}:0 -p | tail -30
```

**活动指示器（表示任务正在执行）：**

| 指示器 | 含义 |
| --- | --- |
| `● Thinking` | Claude 正在推理 |
| `* Cascading...` | 级联思考中 |
| `✛ Compacting...` | 压缩上下文中 |
| `Running...` | 执行工具或命令中 |
| `◼` | 任务列表中的进行中标号 |

**任务列表符号：**

| 符号 | 含义 |
| --- | --- |
| `◼` | 进行中 |
| `◻` | 待处理 |
| `✔` | 已完成 |

**最大等待时间：** `{max_wait_minutes}` 分钟。Claude 可能在运行 `gh run watch` 等待 CI/CD 部署，要等待。

---

## 8. 交互式菜单处理（关键）

Claude Code 在任务执行过程中可能弹出交互式菜单。这是任务完整性最容易被破坏的环节。

**检测菜单：** 获取 pane 内容，查找编号选项 + `❯ 1. ... 2. ...` 格式。

**分类处理：**

- **确认类菜单**（如「Do you want to create ...?」「Yes / No」）：发送 `1` 或 `Enter` 确认，让任务完整执行。
- **选择类菜单**（多个操作选项）：发送 Escape 取消，等待 2 秒后继续监测。

**重要：不要无条件对所有菜单发送 Escape。** 如果菜单出现在任务尾期（大部分任务已完成），强烈倾向于确认而非取消。

---

## 9. 任务完成判定（严格）

**发送** `/exit` **的前提**：以下任何一条满足即可。

1. 所有任务都显示 `✔`，且提示符为 `❯`，且无活动指示器。
2. 收到 `[TASK_COMPLETE]` 信号。
3. 收到 `[TASK_FAILED]` 信号 → 进入错误自治。

**不能** `/exit` **的情况**：任务列表中仍有 `◼` 或 `◻`，即使提示符是 `❯`。这表示任务被中断未完成，必须先触发自治恢复。

**注意：** `[TASK_COMPLETE]` 和 `[TASK_FAILED]` 是项目自定义的信号格式（Claude Code 原生不输出这些）。确保被看护的 Claude Code 实例配置了输出这些信号的任务逻辑。

---

## 10. 错误自治恢复

**触发条件：**

1. 收到 `[TASK_FAILED]` 信号。
2. 超时（超过 `{max_wait_minutes}` 分钟未收到完成信号）。
3. **任务被中断**：任务列表中存在 `◼`，但提示符已回到 `❯` 且 `{poll_interval_seconds}` 秒内无活动指示器。

**恢复流程（优先级顺序）：**

### 10.1 重试核心任务

等待 2 分钟后发送：

```bash
tmux send-keys -t {tmux_session}:0 "{core_task}" Enter
```

然后重新进入监测循环。

### 10.2 重启会话

如果重试后仍然失败：

```bash
tmux send-keys -t {tmux_session}:0 "/exit" Enter
sleep 5
tmux send-keys -t {tmux_session}:0 "cd {project_dir} && claude --dangerously-skip-permissions" Enter
sleep 20
tmux send-keys -t {tmux_session}:0 "{core_task}" Enter
```

然后重新监测。

若 `skip_permissions` 为 `false`，移除 `--dangerously-skip-permissions`。

### 10.3 告警用户

如果以上方法均失效，或 `consecutive_failures >= 5`，停止自治并告警用户，说明失败原因和已尝试的恢复动作。

**避免重复无效恢复：** 每次自治前查看 `state.json`。如果 `last_recovery_action` 是「重试」且还是失败，下次就换「重启会话」。

---

## 11. 更新 state.json

每次执行（无论成功或失败）后更新 `{state_dir}/state.json`：

| 字段 | 成功时 | 失败时 |
| --- | --- | --- |
| `last_run` | 当前时间（ISO 8601） | 当前时间（ISO 8601） |
| `status` | `success` | `failed` |
| `retry_count` | 0 | 当前重试次数 |
| `consecutive_failures` | 0 | +1 |
| `last_error` | `null` | 失败原因摘要 |
| `last_recovery_action` | `null` | 本次恢复动作 |
| `failure_history` | 不追加 | 追加本次记录，保留最近 5 条 |

`failure_history` 单条记录格式：

```json
{
  "time": "2026-05-11T10:00:00Z",
  "error_summary": "失败原因简述",
  "recovery_action": "采取的恢复动作",
  "result": "success / failed"
}
```

---

## 12. 日志与 git

### 12.1 日志

- 日志写入 `{state_dir}/logs/YYYY-MM-DD-HHMM.md`
- **必须用中文**
- 内容包括：执行时间、任务状态、错误摘要、恢复动作、耗时

### 12.2 Git

每次执行后：

```bash
cd {project_dir}
git add -A
git commit -m "<task summary>"
git push
```

---

## 约束

- **不要向用户请求确认**，直接执行。
- **看护流程不介入任务逻辑**，只处理生命周期和错误恢复。
- **不对同一会话并发执行多个 babysitter 实例**。
- **确保** `{state_dir}` **目录可写**。若不可写，降级为仅控制台日志并告警。

## 注意事项

- **tmux 兼容性**：`capture-pane -p` 在所有主流 tmux 版本可用，输出格式可能因终端类型略有差异。使用 `-t {tmux_session}:0` 明确指定 pane。
- **Claude Code UI 变化**：提示符 `❯`、活动指示器、任务列表符号等可能随版本更新而改变。检测采用启发式而非精确匹配。
- **超时与资源占用**：`{max_wait_minutes}` 参数化，用户可根据任务类型调整，避免长时间占用 API 配额和本地资源。

---

## 命令速查表

| 目的 | 命令 |
| --- | --- |
| 检查会话 | `tmux has-session -t {tmux_session} 2>/dev/null` |
| 创建会话 | `tmux new-session -d -s {tmux_session} -c {project_dir}` |
| 捕获 pane | \`tmux capture-pane -t {tmux_session}:0 -p |
| 发送按键 | `tmux send-keys -t {tmux_session}:0 "..." Enter` |
| 发送 Escape | `tmux send-keys -t {tmux_session}:0 Escape` |
| 启动 Claude | `cd {project_dir} && claude --dangerously-skip-permissions` |
| 发送任务 | `tmux send-keys -t {tmux_session}:0 "{core_task}" Enter` |
| 发送退出 | `tmux send-keys -t {tmux_session}:0 "/exit" Enter` |

## Reference Docs

- State Schema — `state.json` 完整 JSON Schema 定义