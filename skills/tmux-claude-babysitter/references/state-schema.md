# State Schema

`state.json` 的完整 JSON Schema 定义。

## 文件路径

默认位于 `{project_dir}/.hermes/state.json`，可通过 `state_dir` 参数覆盖。

## Schema

```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "type": "object",
  "required": ["last_run", "status", "retry_count", "consecutive_failures", "last_error", "last_recovery_action", "failure_history"],
  "properties": {
    "last_run": {
      "type": ["string", "null"],
      "format": "date-time",
      "description": "上次执行时间，ISO 8601 格式。首次执行为 null。"
    },
    "status": {
      "type": ["string", "null"],
      "enum": ["success", "failed", null],
      "description": "上次执行状态。首次执行为 null。"
    },
    "retry_count": {
      "type": "integer",
      "minimum": 0,
      "description": "当前重试次数。成功时重置为 0。"
    },
    "consecutive_failures": {
      "type": "integer",
      "minimum": 0,
      "description": "连续失败次数。成功时重置为 0，失败时 +1。"
    },
    "last_error": {
      "type": ["string", "null"],
      "description": "上次失败原因摘要。成功时为 null。"
    },
    "last_recovery_action": {
      "type": ["string", "null"],
      "description": "上次采取的恢复动作。成功时为 null。"
    },
    "failure_history": {
      "type": "array",
      "maxItems": 5,
      "description": "最近 5 条失败记录。",
      "items": {
        "type": "object",
        "required": ["time", "error_summary", "recovery_action", "result"],
        "properties": {
          "time": {
            "type": "string",
            "format": "date-time",
            "description": "失败发生时间，ISO 8601 格式。"
          },
          "error_summary": {
            "type": "string",
            "description": "失败原因简述。"
          },
          "recovery_action": {
            "type": "string",
            "description": "采取的恢复动作。"
          },
          "result": {
            "type": "string",
            "enum": ["success", "failed"],
            "description": "恢复动作的结果。"
          }
        }
      }
    }
  }
}
```

## 初始值

首次执行前，若文件不存在，使用以下初始值创建：

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

## 更新规则

| 字段 | 成功时 | 失败时 |
| --- | --- | --- |
| `last_run` | 当前时间（ISO 8601） | 当前时间（ISO 8601） |
| `status` | `success` | `failed` |
| `retry_count` | 0 | 当前重试次数 |
| `consecutive_failures` | 0 | +1 |
| `last_error` | `null` | 失败原因摘要 |
| `last_recovery_action` | `null` | 本次恢复动作 |
| `failure_history` | 不追加 | 追加本次记录，保留最近 5 条 |

## 使用场景

- **自治恢复决策**：通过 `consecutive_failures` 和 `last_recovery_action` 避免重复无效的恢复方式。
- **失败模式识别**：通过 `failure_history` 识别重复出现的失败模式。
- **告警阈值**：当 `consecutive_failures >= 5` 时，停止自治并告警用户。

## 接续执行的特殊语义

`recovery_action` 是自由字符串字段，约定的取值包括 `"接续执行"`、`"重试核心任务"`、`"重启会话"`。其中 `"接续执行"` 针对瞬态错误（API 错误 / 超时 / 网络抖动 / 限流 / 过载等），与其他恢复动作有以下差异：

| 维度 | 接续执行 | 重试核心任务 / 重启会话 |
| --- | --- | --- |
| 触发 | 符合第 7 节瞬态错误判定原则 | 信号失败 / 超时 / 任务中断 / 接续 5 次失败 |
| 指令 | 发送 `请继续` | 发送 `{core_task}`（或先 `/exit` 再启动 + 发送 `{core_task}`） |
| `consecutive_failures` | **不递增** | +1 |
| `retry_count` | **不递增** | 按重试逻辑递增 |
| `failure_history` | 追加一条（`recovery_action: "接续执行"`，`error_summary` 记录观察到的瞬态错误文本摘要） | 追加一条 |
| 告警阈值 | 内存计数 `continue_attempts`；连续 5 次仍判定为瞬态错误后回退到重试核心任务 | `consecutive_failures >= 5` 触发用户告警 |

`continue_attempts` 仅存在于内存（本次 babysitter 运行内），不持久化到 `state.json`，因为它表达的是"本次执行内的接续退避计数"，跨 babysitter 运行不应携带。