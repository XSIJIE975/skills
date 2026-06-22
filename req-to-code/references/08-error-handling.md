# Error Handling

工作流的错误处理和恢复机制。

## 错误分类

| 类型 | 示例 | 处理方式 |
|------|------|---------|
| 瞬态（transient） | 网络超时、MCP 暂时不可用、文件锁冲突 | 指数退避重试，最多 3 次 |
| 可跳过（skippable） | 非关键阶段的警告级错误 | 标记跳过，继续下一阶段 |
| 需回滚（rollback） | 数据损坏、状态不一致、关键依赖缺失 | 回滚到上一个检查点 |
| 需升级（escalation） | 权限不足、磁盘耗尽、需用户手动操作 | 状态设为 blocked，报告用户 |

## 重试策略

瞬态错误使用指数退避：

| 重试次数 | 等待时间 |
|---------|---------|
| 第 1 次 | 2 秒 |
| 第 2 次 | 5 秒 |
| 第 3 次 | 15 秒 |

3 次失败后标记阶段为 failed，进入升级流程。

## 回滚机制

回滚步骤：
1. 读取状态文件中的 `recovery_checkpoint`
2. 将当前阶段状态设为 `failed`
3. 清理失败阶段产生的中间产物
4. 将 `current_phase` 回滚到检查点
5. 从检查点重新执行

检查点设置时机：
- 每个阶段开始前，将 `recovery_checkpoint` 设为上一个已完成阶段
- 阶段内关键步骤完成后，更新检查点

## 门禁状态持久化

阶段 3（OpenSpec Manager）引入了用户确认门禁。门禁状态需要持久化到状态文件中，
以便会话中断后恢复时无需用户重新确认已通过的门禁。

### 门禁状态结构

每个门禁在状态文件中记录：

```json
{
  "sub_phase": "propose_requirements",
  "gate_status": "approved",
  "version": 2,
  "confirmed_at": "2026-06-22T10:30:00",
  "user_feedback": ["补充了性能约束"]
}
```

字段说明：
- `sub_phase`：门禁阶段名（`propose_requirements` / `design_tasks`）
- `gate_status`：`awaiting_decision` | `approved` | `rejected` | `needs_revision`
- `version`：该文档版本的序号（每次修订 +1）
- `confirmed_at`：确认时间戳
- `user_feedback`：用户反馈记录（用于追溯）

### 恢复策略

当会话在门禁阶段中断后恢复：

1. 读取 `recovery_checkpoint` 确定最后一个已确认的门禁
2. **不重新展示已确认的门禁**，直接从下一个未完成的门禁或阶段继续
3. 如果中断发生在门禁展示中（`gate_status: awaiting_decision`），恢复时重新展示该门禁
4. 恢复展示时只展示 diff（基于上次版本），不展示完整文档

### 会话中断场景与恢复行为

| 中断位置 | gate_status | 恢复行为 |
|---------|-------------|---------|
| 自动化生成中 | 无 | 从阶段开头重新执行生成 |
| 门禁展示中，用户未回应 | `awaiting_decision` | 重新展示该门禁 |
| 用户已确认门禁 1，门禁 2 展示中 | 门禁 1: `approved` / 门禁 2: `awaiting_decision` | 跳过门禁 1，重新展示门禁 2 |
| 两个门禁均已确认 | 均为 `approved` | 跳过整个阶段 |
| 用户拒绝（rejected） | `rejected` | 保留用户反馈，重新展示修订版 |

## 并发安全（并行执行时状态文件保护）

阶段 5 使用 Team Mode 或 Background Agents 并行执行任务时，
多个 Agent 可能同时读写状态文件，导致数据损坏。

### 方案：独立任务状态文件

每个并行任务写入独立的状态文件，由 orchestrator 统一合并：

```
# 主状态文件（orchestrator 独占写入）
.req-to-code-state.json

# 各任务独立状态文件（Agent 写入独立的文件，无冲突）
.req-to-code-state/task-<task-id>.json
.req-to-code-state/task-<task-id>.json
...
```

**写入规则**：
1. Orchestrator 调度时：创建任务状态文件占位
2. Agent 执行时：只写入自己的 `.req-to-code-state/task-<id>.json`
3. Agent 完成时：报告 orchestrator，orchestrator 读取任务状态文件，合并到主状态文件
4. 合并后删除任务状态文件

任务状态文件 schema 见 `01-initialization.md` 的「任务状态文件」节。

## 用户主动中断处理

用户可能在工作流执行过程中主动要求中断（如发现方向错误）。

### 中断类型

| 中断方式 | 处理 |
|---------|------|
| 用户说"停止"或"中断" | 暂停当前操作，保存状态 |
| 用户说"回退到第 N 步" | 标记当前阶段状态为 `user_interrupted`，回滚到指定 checkpoint |
| 用户说"取消本次工作流" | 标记状态为 `cancelled`，保留中间产物（用于分析），不归档 |

### 中断后选项

中断后向用户提供：

```
工作流已暂停。你可以选择：
1. 【继续】从中断处继续执行
2. 【回退】回退到上一个阶段（<phase_name>）
3. 【终止】终止本次工作流（保留中间产物，不归档）
4. 【辅助模式】Agent 提供指导，你手动执行下一步
```

### 状态文件更新

```json
{
  "status": "user_interrupted",
  "interrupted_at": "ISO-8601",
  "interrupt_reason": "用户要求重新评估设计",
  "interrupt_options": ["continue", "rollback", "cancel", "assist"]
}
```

## 辅助模式

当自动化工作流在某个步骤失败，或用户对自动生成的结果不满意时，
提供"辅助模式"作为额外的恢复选项：

> **辅助模式**：Agent 提供详细的步骤指导、代码建议、命令示例，用户手动执行。适用于自动化失败但任务本身可手动完成的场景。

辅助模式行为：
- Agent 不再自动执行命令或写文件
- Agent 提供"建议操作"列表（每条包含：做什么、为什么、命令/代码示例）
- 用户手动执行后，告诉 Agent 结果
- Agent 根据结果决定下一步（继续辅助、重试自动化、或升级）

使用场景：
- 权限不足（Agent 无权写某个目录）
- 需手动审核的敏感变更
- 自动化连续 2 次失败后的降级路径

## 错误日志

所有错误写入状态文件的 `errors` 数组：

```json
{
  "errors": [
    {
      "timestamp": "ISO-8601",
      "phase": "document_conversion",
      "error_type": "transient|skippable|rollback|escalation",
      "message": "错误描述",
      "detail": "详细信息（可选）",
      "retry_count": 0,
      "resolved": false
    }
  ]
}
```

## 恢复选项

当阶段失败时，提供以下选项（按优先级）：

1. **从失败阶段重试** — 适用于瞬态错误已解决
2. **跳过失败阶段** — 适用于非关键阶段
3. **回滚到上一个成功阶段** — 适用于状态不一致
4. **辅助模式** — 自动化连续 2 次失败后的降级路径：Agent 提供步骤指导，用户手动执行
5. **手动介入** — 适用于需要用户手动操作的场景（权限不足、环境配置等）

## 状态转换

```
running → completed（成功）
running → failed（失败）
failed  → running（重试）
failed  → skipped（跳过）
failed  → running（回滚后重试）
任何状态 → blocked（需升级）
blocked → running（用户解决后）
```

## 约束

- 每次错误处理后必须更新状态文件
- 回滚必须清理中间产物，避免脏状态
- 升级给用户时必须提供清晰的错误描述和所需操作
- 错误日志不可删除（用于事后分析），但需要管理大小：
  - 状态文件中的 `errors` 数组最多保留 **100 条**
  - 超出时，将最早的 errors 移至 `openspec/changes/archive/<change-name>/errors.json`（归档时清理）
  - 工作流完成后，所有 errors 随归档移至 `openspec/changes/archive/<change-name>/errors.json`
