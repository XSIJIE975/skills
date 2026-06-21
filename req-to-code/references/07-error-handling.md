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
4. **手动介入** — 适用于需要用户操作的场景

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
- 错误日志不可删除（用于事后分析）
