---
name: tmux-claude-babysitter
description: 'Babysit a Claude Code instance running inside a tmux session. Activate ONLY when user''s message starts with ''/tmux-claude-babysitter''. The prefix is stripped; the remaining natural language content is parsed by the model to extract parameters. If project_dir is missing, ask the user. Covers: session creation, Claude Code startup, idle/menu detection, interactive menu handling, task completion determination, error recovery with retry/restart, state tracking, and logging.'
metadata: '{"author":"yuexiaoliang","version":"1.0.1"}'
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
 4. 进入干净 idle 状态：取消可能存在的菜单；若 Claude 是上次遗留运行的，发送 `/clear` 清理历史上下文
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
| 看到 `❯` 或 Claude UI 元素 | 已在运行（上次遗留） | 进入步骤 5；**步骤 5.2 会发送 `/clear` 清理历史上下文** |
| 看到活动指示器（`● Thinking` 等） | 正在执行 | 直接进入步骤 7 监测 |

**提示符检测启发式：** 以 `❯` 或 `>` 开头且后面紧跟空格的行，视为 Claude Code 提示符。

**记录状态**：本步骤需要记录"本次 babysitter 是否启动了 Claude"（in-memory 标志 `claude_started_by_self`），步骤 5.2 据此决定是否清理上下文。

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

## 5. 进入干净 idle 状态

### 5.1 取消可能存在的菜单

如果 Claude 显示交互式菜单（编号选项），先发送 Escape 取消，等待 2 秒：

```bash
tmux send-keys -t {tmux_session}:0 Escape
sleep 2
```

**检测菜单：** 获取 pane 内容，查找编号选项 + `❯ 1. ... 2. ...` 格式。

### 5.2 清理历史上下文（仅当 Claude 是上次遗留运行时）

**触发条件**：步骤 3 中的 `claude_started_by_self` 标志为 `false`，即第 3 节检测结果为「已在运行」、本次 babysitter 没有执行步骤 4 启动 Claude。

**理由**：上一轮任务的对话历史会污染本次任务 —— 浪费 context、可能让 Claude 误判"已经做过 X"、可能触发不必要的 compaction。本次 babysitter 自己刚启动的 Claude 进程上下文本来就是干净的，无需此步。

**操作**：

```bash
tmux send-keys -t {tmux_session}:0 "/clear" Enter
sleep 3
```

然后捕获 pane 内容确认 `❯` 提示符仍在，且 pane 不再展示历史对话痕迹（通常 `/clear` 会清空可视区域，仅保留欢迎信息或空白）。

**容错**：

- 若 `/clear` 后看到确认菜单，按 5.1 处理（多数版本不需要）。
- 若 3 秒后 `❯` 未恢复或 Claude 出现异常状态，回退到一次"重启会话"流程（参考 10.3）：发送 `/exit`、等待、重新 `claude --dangerously-skip-permissions` 启动。
- 若 `claude_started_by_self` 为 `true`（本次 babysitter 自己启动了 Claude），**跳过本节**。

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

### 瞬态错误识别

监测时若 pane 内容（最近 30 行）中出现**瞬态错误**特征，进入接续执行恢复（见 10.1）。

**判定原则（语义而非关键词匹配）：**

瞬态错误 = 失败原因发生在 Claude 自身工作之外、且重试有合理成功概率的错误。典型来源：

- 模型 API / SDK 远端调用失败（5xx 服务端、429 限流、529 过载、网络超时、连接重置、流式中断）
- 网络层故障（DNS、TCP、TLS、代理）
- 临时性资源不可用（gateway timeout、service unavailable）

**代表性示例**（仅用于辅助理解，**不是闭集**；遇到语义类似的错误同样视为瞬态）：

- `API Error: 529 overloaded` / `OverloadedError`
- `Request timed out` / `APITimeoutError` / `Gateway timeout`
- `Connection reset` / `fetch failed` / `ECONNRESET` / `APIConnectionError`
- `Streaming error` / `stream interrupted`
- `Rate limit exceeded` / `Too many requests`

**反例（不是瞬态错误，跳过接续，直接走 10.2 重试核心任务）：**

- `[TASK_FAILED]` 信号 —— 业务侧主动汇报失败，应整体重试
- 权限被拒（`Permission denied`、`401 Unauthorized`、`403 Forbidden`）—— 重试无意义
- 语法/逻辑错误（如 Claude 自身写出错误代码导致 lint/test/build 失败）—— 属任务内问题，由 Claude 自己修正
- 任务列表 `◼` 卡住但 pane 无任何错误信息 —— 属意外中断，按任务被中断路径处理
- 资源不存在 / 配置缺失（`No such file`、`Repository not found`）—— 重试不解决根因

**判定指引：**

不确定时倾向判定为"瞬态错误"。代价分析：

- 误判为瞬态：最多浪费 62 分钟退避（2+5+10+15+30）后自动回退到 10.2，代价可控。
- 误判为非瞬态：立即重发 `{core_task}`，**丢弃已完成进度**，代价高。

故判定阈值偏向接续。

---

## 8. 交互式菜单处理（关键）

Claude Code 在任务执行过程中可能弹出交互式菜单。这是任务完整性最容易被破坏的环节。

**检测菜单：** 获取 pane 内容，判断 Claude 是否正在等待用户做选择或确认。典型迹象包括编号选项、Yes/No 确认、Allow/Deny 权限请求、或其他需要用户回应的交互提示。结合内容语义自行判断，不要只依赖固定格式匹配。

**如何处理**

不要依赖固定规则。每次遇到菜单时，结合**当前任务的目标**和**菜单请求的具体内容**自行判断：

- 如果该操作是推进当前任务所必需的（例如创建日志文件、写入项目内文件、执行构建脚本），则确认。
- 如果该操作明显超出当前任务范围，或涉及高风险行为（删除数据、修改系统配置、向外部发送敏感数据），则拒绝。
- 如果不确定，倾向确认 —— `skip_permissions=true` 模式下卡住比误点一个低风险操作后果更严重。

菜单出现在任务尾期（大部分任务已完成）时，强烈倾向确认，避免取消导致前功尽弃。

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
4. **瞬态错误**：pane 中观察到第 7 节「瞬态错误识别」描述的特征（无需等到完成信号）。

**恢复流程（优先级顺序）：**

### 10.1 接续执行（瞬态错误优先）

**适用**：符合第 7 节「瞬态错误识别」的判定原则时。

**动作**：根据当前接续次数（in-memory 计数 `continue_attempts`，本次 babysitter 运行内累计）做退避，然后发送接续指令：

| `continue_attempts` | 等待时间 |
| --- | --- |
| 0（首次） | 2 分钟 |
| 1 | 5 分钟 |
| 2 | 10 分钟 |
| 3 | 15 分钟 |
| 4 | 30 分钟 |

```bash
sleep <等待秒数>
tmux send-keys -t {tmux_session}:0 "请继续" Enter
```

发送后 `continue_attempts += 1`，重新进入第 7 节监测循环。

**重要约束：**

- **接续不占用** `consecutive_failures` **和** `retry_count`：仅在 `failure_history` 中追加一条 `recovery_action: "接续执行"` 记录，便于事后审计；但不递增 `consecutive_failures`。理由：连续 5 次告警阈值是为策略性失败设计的，不应被网络抖动消耗。
- **接续上限**：当 `continue_attempts >= 5` 且最新一次接续后仍判定为瞬态错误，**放弃接续**，回退到 10.2。
- **回退后清零**：进入 10.2 时将 `continue_attempts` 清零。
- **接续成功判定**：发送"请继续"后，若监测到活动指示器（`● Thinking`、`* Cascading...`、`Running...` 等）或任务列表 `◻ → ◼ → ✔` 推进，视为接续成功，重置 `continue_attempts = 0`，继续正常监测。

### 10.2 重试核心任务

**适用**：非瞬态错误，或 10.1 接续 5 次仍失败。

等待 2 分钟后发送：

```bash
tmux send-keys -t {tmux_session}:0 "{core_task}" Enter
```

然后重新进入监测循环。本次记入 `consecutive_failures += 1`，`last_recovery_action = "重试核心任务"`。

### 10.3 重启会话

如果 10.2 重试后仍然失败：

```bash
tmux send-keys -t {tmux_session}:0 "/exit" Enter
sleep 5
tmux send-keys -t {tmux_session}:0 "cd {project_dir} && claude --dangerously-skip-permissions" Enter
sleep 20
tmux send-keys -t {tmux_session}:0 "{core_task}" Enter
```

然后重新监测。若 `skip_permissions` 为 `false`，移除 `--dangerously-skip-permissions`。记入 `consecutive_failures += 1`，`last_recovery_action = "重启会话"`。

### 10.4 告警用户

如果以上方法均失效，或 `consecutive_failures >= 5`，停止自治并告警用户，说明失败原因、本次接续次数（若有）和已尝试的恢复动作。

**避免重复无效恢复：** 每次自治前查看 `state.json`。如果 `last_recovery_action` 是「重试核心任务」且本次仍失败，下次就换「重启会话」。

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

**接续执行的特殊规则：**

- 接续尝试**不**递增 `consecutive_failures` 和 `retry_count`。
- 但仍向 `failure_history` 追加一条记录，`recovery_action: "接续执行"`，`error_summary` 写入观察到的瞬态错误文本摘要（截断到合理长度），便于事后追查。
- `last_recovery_action` 在接续期间记为 `"接续执行"`；接续成功后正常运行不立即清零，待整体任务完成时随 `status: success` 一并清零。
- `continue_attempts` 计数仅存在于内存（本次 babysitter 运行内），不持久化到 `state.json`。

---

## 12. 日志与 git

### 12.1 日志

- 日志写入 `{state_dir}/logs/YYYY-MM-DD-HHMM.md`
- **必须用中文**
- 内容包括：执行时间、任务状态、错误摘要、恢复动作、瞬态错误的接续次数（若发生）、耗时

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
| 清理上下文 | `tmux send-keys -t {tmux_session}:0 "/clear" Enter` |
| 发送任务 | `tmux send-keys -t {tmux_session}:0 "{core_task}" Enter` |
| 发送接续 | `tmux send-keys -t {tmux_session}:0 "请继续" Enter` |
| 发送退出 | `tmux send-keys -t {tmux_session}:0 "/exit" Enter` |

## Reference Docs

- State Schema — `state.json` 完整 JSON Schema 定义