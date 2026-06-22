# Deep Review Loop

独立第三方审查 → 分析 → 修复 → 重新审查，循环直到质量达标。

核心原则：**代码不能自己审查自己。**

## 输入

- 代码变更（git diff）
- 原始需求规格
- 测试报告

## 输出

- 审查报告（`review-report.md`）
- 修复后的代码（如有）

## 流程

### 1. 准备审查材料

收集并写入系统临时目录（`req-to-code/deep-review/`）：
- `code-changes.diff` — 完整 diff
- `change-stats.txt` — 变更统计（`git diff --stat`）
- `changed-files.txt` — 变更文件列表
- `requirements.md` — 需求规格（从 OpenSpec 目录或 PR 描述收集）
- `test-report.md` — 测试结果和覆盖率

**diff 拆分策略**（完整 diff 超 500KB 时）：
- 按顶层目录拆分（`src/`、`tests/`、`docs/` 等各生成一个 diff）
- 每个子 diff 独立审查
- 审查报告合并为一份，统一分类 Critical/Major/Minor

### 2. 执行审查

审查者必须与代码作者独立。使用 Codex CLI 执行审查。

审查维度：正确性、安全性、性能、可维护性、边界条件。

**审查方式**（按优先级）：

1. **Codex CLI + OMX** — `codex "$deep-interview ..."` 第一轮，`codex "$prometheus-strict ..."` 第二轮
2. **Codex CLI 无 OMX** — 两次 `codex "审查以下代码变更..."` 从不同角度覆盖所有审查维度
3. **Codex CLI 不可用** — 使用 `task(subagent_type="oracle", ...)` 做独立审查

审查 prompt 中需注入需求规格和变更 diff。

### 3. 分析审查结果

对审查发现分类：

| 严重级别 | 处理方式 |
|---------|---------|
| 严重（Critical） | 必须修复，阻塞归档 |
| 重要（Major） | 应当修复，记录技术债 |
| 建议（Minor） | 可选修复，记录建议 |

### 4. 修复循环

```
审查 → 发现问题 → 分析 → 修复 → 重新审查
         ↑                              |
         └──────────────────────────────┘
```

- 每轮修复后重新运行审查
- 最多循环 3 轮
- 3 轮后仍有严重问题 → 升级给用户决策

### 5. 生成审查报告

写入 `review-report.md`，包含：
- 审查摘要（通过/未通过、问题统计）
- 问题详情（每轮发现的问题和处理结果）
- 最终结论

## 约束

- 审查者必须独立于代码作者
- 严重问题不允许标记为"已知问题"跳过
- 每轮审查结果必须持久化（写入报告文件）
- 审查过程的临时文件在归档后清理
