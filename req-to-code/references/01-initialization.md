# Initialization

工作流初始化阶段：创建状态文件、检测环境、配置 `.gitignore`。

## 状态文件 Schema

工作流创建 `.req-to-code-state.json` 跟踪全流程状态：

```json
{
  "workflow_id": "ew-YYYYMMDD-<8位hex>",
  "started_at": "ISO-8601",
  "current_phase": "initialization",
  "status": "running",
  "input": { "document_path": "...", "project_name": "..." },
  "env": { "os": "linux|macos|windows", "shell": "bash|powershell", "tmp_dir": "..." },
  "phases": {
    "document_conversion": {"status": "pending|running|completed|failed|skipped"},
    "openspec_management": {"status": "pending|running|completed|failed|skipped"},
    "task_decomposition": {"status": "pending|running|completed|failed|skipped"},
    "intelligent_orchestration": {"status": "pending|running|completed|failed|skipped"},
    "deep_review": {"status": "pending|running|completed|failed|skipped"},
    "archive": {"status": "pending|running|completed|failed|skipped"}
  },
  "recovery_checkpoint": null,
  "document_history": {},
  "errors": []
}
```

字段说明：
- `workflow_id`：全局唯一工作流 ID，格式 `ew-YYYYMMDD-<8位hex>`
- `status`：`running` | `completed` | `failed` | `cancelled` | `user_interrupted`
- `phases`：各阶段状态，每次阶段转换时更新
- `recovery_checkpoint`：记录最近完成的阶段，用于失败恢复
- `document_history`：文档历史版本，用于门禁 diff 展示（见 03-openspec-manager.md）
- `errors`：错误日志数组

## 门禁状态扩展（阶段 3 OpenSpec Manager）

阶段 3 完成后，`phases.openspec_management` 扩展为：

```json
{
  "phases": {
    "openspec_management": {
      "status": "completed",
      "sub_phase": "verification",
      "gate_status": "approved",
      "gates": {
        "propose_requirements": {
          "status": "approved",
          "version": 2,
          "confirmed_at": "2026-06-22T10:30:00",
          "user_feedback": ["补充了性能约束"]
        },
        "design_tasks": {
          "status": "approved",
          "version": 1,
          "confirmed_at": "2026-06-22T10:35:00",
          "user_feedback": []
        }
      },
      "recovery_checkpoint": "propose_requirements"
    }
  }
}
```

## 任务状态文件（阶段 5 并行执行）

并行执行时，每个任务写入独立的状态文件：

```
.req-to-code-state/task-<task-id>.json
```

```json
{
  "task_id": "task-01",
  "status": "completed",
  "test_passed": true,
  "coverage": "85%",
  "errors": [],
  "started_at": "ISO-8601",
  "completed_at": "ISO-8601"
}
```

## 环境检测

初始化时检测以下信息，写入状态文件的 `env` 字段：

| 检测项 | 方式 |
|--------|------|
| OS 类型 | `uname -s`（POSIX）或 `$env:OS`（PowerShell） |
| Shell 类型 | 根据 OS 推断 `bash` 或 `powershell` |
| 临时目录 | POSIX 用 `/tmp`，Windows 用 `$env:TEMP` |

## .gitignore

状态文件（`.req-to-code-state.json`）和任务状态目录（`.req-to-code-state/`）包含运行时信息，不应提交到 git。
初始化阶段自动写入 `.gitignore`：

```
# req-to-code 运行时状态
.req-to-code-state.json
.req-to-code-state/
```
