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