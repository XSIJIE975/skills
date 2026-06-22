---
name: req-to-code
description: "从需求文档（docx/pdf/md）自动生成代码的全流程工作流。7 个阶段覆盖初始化→文档转换→需求管理→任务分解→智能编排→深度审查→归档交付。强制 TDD，支持多 Agent 编排。用户说'启动工作流'或提供需求文档时触发。"
compatibility: "需要安装：OpenCode (latest)、Codex CLI (latest)、OpenSpec (latest)、markitdown-mcp (latest)。推荐使用 OMO 和 OMX 插件增强功能。"
---

# Req-to-Code - 多 Agent 工作流系统

## 概述

从需求文档到代码交付的 7 阶段自动化工作流。核心理念：需求驱动，TDD 强制，质量门禁。

## 触发条件

- 用户提供需求文档（docx/pdf/md）
- 用户说"启动企业工作流"或类似指令

## 工作流阶段

```
初始化 → 文档转换 → 需求管理 → 任务分解 → 智能编排 → 深度审查 → 归档交付
```

| 阶段 | 模块 | 输入 | 输出 |
|------|------|------|------|
| 1 | 初始化 | 用户指令 | 状态文件 |
| 2 | document-converter | docx/pdf/txt/html | markdown |
| 3 | openspec-manager | markdown 需求 | OpenSpec 提案 + 规格（已确认） |
| 4 | task-decomposer | OpenSpec 任务清单 | 更新后的 tasks.md（含依赖分析） |
| 5 | intelligent-orchestrator | 任务计划 | 代码 + 测试（TDD） |
| 6 | deep-review-loop | 代码变更 + 需求规格 | 审查报告 |
| 7 | archive-manager | 所有产出物 | 交付报告 + 归档 |

## 阶段 1：初始化

创建状态文件，检测运行环境。详见 [references/01-initialization.md](references/01-initialization.md)。

**约束**：
- 每次阶段转换时必须更新状态文件
- 失败时记录错误并执行恢复策略
- 状态文件和任务状态目录不应提交到 git（自动加入 .gitignore）

## 阶段 2：文档转换

将 docx/pdf/txt/html 转换为统一 markdown 格式。详见 [references/02-document-converter.md](references/02-document-converter.md)。

**约束**：
- 输出格式：统一 markdown
- 幂等性：已为 .md 的文件不重复转换
- 输出质量：去除格式噪音，保留语义结构
- 转换失败时保留原始文件

## 阶段 3：需求管理

创建 OpenSpec 提案和规格文档，通过结构化门禁与用户确认。
详见 [references/03-openspec-manager.md](references/03-openspec-manager.md)。

**约束**：
- 需要 OpenSpec CLI
- 门禁确认方式：结构化问题（非开放式对话）
- 确认后验证完整性才进入下一阶段
- 文档修订后通过 diff 展示变更，不重新展示全文

## 阶段 4：任务分解

细化任务，分析依赖，分配优先级。
详见 [references/04-task-decomposer.md](references/04-task-decomposer.md)。

**约束**：
- 模糊任务分级处理：简单缺失用选择题确认，真正模糊需深入探讨
- 深入探讨后必须将结论更新到 OpenSpec 文档中
- 细化结果写入 tasks.md，不创建独立 plan 文件
- 细化完成后做 1 次汇总确认（结构化问题）
- 识别可并行执行的任务组
- 每个任务的复杂度不超过"高"（达到则拆分）

## 阶段 5：智能编排

调度 Agent 执行任务，强制 TDD 流程。详见 [references/05-intelligent-orchestrator.md](references/05-intelligent-orchestrator.md)。

**约束**：
- 强制 TDD：Red（测试）→ Green（实现）→ Refactor（重构）
- 没有测试的代码不允许标记为完成
- 新增代码的测试覆盖率不低于项目原有水平
- 每个任务的执行结果必须可追溯
- 并行执行时注意状态文件冲突

## 阶段 6：深度审查

独立第三方审查，循环修复。详见 [references/06-deep-review-loop.md](references/06-deep-review-loop.md)。

**约束**：
- 审查者必须独立于代码作者
- 审查维度：正确性、安全性、性能、可维护性
- 严重问题必须修复才能通过
- 最多循环 3 轮，超过则升级给用户决策

## 阶段 7：归档交付

归档变更，生成交付报告。详见 [references/07-archive-manager.md](references/07-archive-manager.md)。

**约束**：
- 归档前验证：所有任务完成 + 审查通过 + 测试通过
- 临时文件必须清理
- 状态文件在归档成功后删除

## 错误处理和恢复

详见 [references/08-error-handling.md](references/08-error-handling.md)。

恢复策略（按优先级）：
1. **自动重试** — 网络超时、临时不可用
2. **跳过** — 非关键阶段的警告级错误
3. **回滚** — 数据损坏、状态不一致
4. **辅助模式** — 自动化失败后 Agent 提供步骤指导，用户手动执行
5. **升级** — 权限不足、需用户手动操作

## 配置和执行

详见 [references/09-prerequisites.md](references/09-prerequisites.md)。

- 配置优先级：OMO > OMX > 默认

