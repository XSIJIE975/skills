---
name: req-to-code
description: "从需求文档（docx/pdf/md）自动生成代码的全流程工作流。6 个阶段覆盖文档转换→需求管理→任务分解→智能编排→深度审查→归档交付。强制 TDD，支持多 Agent 编排。用户说'启动工作流'或提供需求文档时触发。"
compatibility: "需要安装：OpenCode (latest)、Codex CLI (latest)、OpenSpec (latest)、markitdown-mcp (latest)。推荐使用 OMO 和 OMX 插件增强功能。"
---

# Req-to-Code - 多 Agent 工作流系统

## 概述

从需求文档到代码交付的 6 阶段自动化工作流。核心理念：需求驱动，TDD 强制，质量门禁。

## 触发条件

- 用户提供需求文档（docx/pdf/md）
- 用户说"启动企业工作流"或类似指令

## 工作流阶段

```
文档转换 → 需求管理 → 任务分解 → 智能编排 → 深度审查 → 归档交付
```

| 阶段 | 模块 | 输入 | 输出 |
|------|------|------|------|
| 0 | 初始化 | 用户指令 | 状态文件 |
| 1 | document-converter | docx/pdf/txt/html | markdown |
| 2 | openspec-manager | markdown 需求 | OpenSpec 提案 + 规格 |
| 3 | task-decomposer | OpenSpec 任务清单 | 细化任务计划 + 依赖图 |
| 4 | intelligent-orchestrator | 任务计划 | 代码 + 测试（TDD） |
| 5 | deep-review-loop | 代码变更 + 需求规格 | 审查报告 |
| 6 | archive-manager | 所有产出物 | 交付报告 + 归档 |

## 阶段 0：初始化

创建 `.req-to-code-state.json` 跟踪全流程状态：

```json
{
  "workflow_id": "ew-YYYYMMDD-<8位hex>",
  "started_at": "ISO-8601",
  "current_phase": "initialization",
  "status": "running",
  "input": { "document_path": "...", "project_name": "..." },
  "env": { "os": "linux|macos|windows", "tmp_dir": "...", "path_sep": "/|\\" },
  "phases": {
    "document_conversion": {"status": "pending|running|completed|failed|skipped"},
    "openspec_management": {"status": "pending|running|completed|failed|skipped"},
    "task_decomposition": {"status": "pending|running|completed|failed|skipped"},
    "intelligent_orchestration": {"status": "pending|running|completed|failed|skipped"},
    "deep_review": {"status": "pending|running|completed|failed|skipped"},
    "archive": {"status": "pending|running|completed|failed|skipped"}
  },
  "recovery_checkpoint": null,
  "errors": []
}
```

**每次阶段转换时更新状态文件。失败时记录错误并执行恢复策略（见 references/07）。**

## 阶段 1：文档转换

将 docx/pdf/txt/html 转换为统一 markdown 格式。详见 [references/01](references/01-document-converter.md)。

核心约束：
- 使用 markitdown MCP 的 `convert_to_markdown(uri)` 工具
- `.md` 文件跳过转换
- 转换后清理多余空行和格式噪音

## 阶段 2：需求管理

创建 OpenSpec 提案和规格文档。详见 [references/02](references/02-openspec-manager.md)。

核心约束：
- 需要 OpenSpec CLI（`npm install -g @fission-ai/openspec@latest`）
- 生成三份文档：需求规格、技术设计、任务清单
- 验证规格完整性后才进入下一阶段

## 阶段 3：任务分解

细化任务，分析依赖，分配优先级。详见 [references/03](references/03-task-decomposer.md)。

核心约束：
- 使用 brainstorming skill 澄清模糊需求
- 使用 writing-plans skill 编写详细计划
- 输出：每个任务的详细步骤、验收标准、依赖关系、预估时间
- 识别可并行执行的任务组

## 阶段 4：智能编排

调度 Agent 执行任务，强制 TDD。详见 [references/04](references/04-intelligent-orchestrator.md)。

核心约束：
- 配置优先级：OMO > OMX > 默认
- 强制 TDD：先写测试（Red）→ 实现（Green）→ 重构
- 每个任务完成后运行测试验证
- 使用 Team Mode 并行执行独立任务

## 阶段 5：深度审查

独立第三方审查，循环修复。详见 [references/05](references/05-deep-review-loop.md)。

核心约束：
- 审查者不能是代码作者（用 Codex 做独立审查）
- 审查维度：正确性、安全性、性能、可维护性
- 严重问题必须修复才能通过
- 最多循环 3 轮，超过则升级给用户

## 阶段 6：归档交付

归档变更，生成交付报告。详见 [references/06](references/06-archive-manager.md)。

核心约束：
- 归档前验证：所有任务完成 + 审查通过 + 测试通过
- 生成交付报告到 `archive/` 目录
- 清理临时文件和状态文件

## 错误处理和恢复

详见 [references/07](references/07-error-handling.md)。

恢复策略（按优先级）：
1. **自动重试** — 网络超时、临时不可用（指数退避，最多 3 次）
2. **跳过** — 非关键阶段的警告级错误
3. **回滚** — 数据损坏、状态不一致（回滚到上一个检查点）
4. **升级** — 权限不足、需要用户手动操作（状态设为 blocked）

## 配置优先级

所有配置遵循：OMO > OMX > 默认值

| 配置源 | 路径 | 说明 |
|--------|------|------|
| OMO (Linux) | `~/.config/opencode/oh-my-openagent.jsonc` | 最高优先级 |
| OMO (macOS) | `~/Library/Application Support/opencode/oh-my-openagent.jsonc` | 最高优先级 |
| OMX | `~/.codex/config.toml` 或 `~/.codex/omx.json` | 次优先级 |
| 默认 | 见各阶段 reference | 最低优先级 |

## 前置依赖检查

工作流启动时验证：
- `openspec --version` — 需要 OpenSpec CLI
- `codex --version` — 需要 Codex CLI（可选，降级为内置审查）
- markitdown MCP — 需要已配置的 MCP 服务器

缺失依赖时：报告错误并提供安装命令，不自动安装。

## 外部工具

| 工具 | 用途 | 仓库 |
|------|------|------|
| [OpenCode](https://github.com/anomalyco/opencode) | 智能编排执行 | CLI 主体 |
| [Codex CLI](https://github.com/openai/codex) | 深度审查 | 独立审查 Agent |
| [OpenSpec](https://github.com/Fission-AI/OpenSpec) | 需求管理 | 规格文档生成 |
| [markitdown-mcp](https://github.com/microsoft/markitdown/tree/main/packages/markitdown-mcp) | 文档转换 | MCP 服务器 |
| [OMO](https://github.com/code-yeongyu/oh-my-openagent) | OpenCode 增强 | Team Mode、多模型编排 |
| [OMX](https://github.com/Yeachan-Heo/oh-my-codex) | Codex 增强 | $deep-interview、$prometheus-strict |

OMO 和 OMX 为可选增强，不安装时工作流以降级模式运行。

## Quick Start

```bash
# 1. 安装依赖
npm install -g @fission-ai/openspec@latest
# markitdown-mcp（选一种方式安装 uv）：
# Linux/macOS: curl -LsSf https://astral.sh/uv/install.sh | sh
# macOS (brew): brew install uv
# Windows:      powershell -ExecutionPolicy ByPass -c "irm https://astral.sh/uv/install.ps1 | iex"
uv pip install markitdown-mcp

# 2. 准备需求文档
# 将 .docx/.pdf/.md 放在项目根目录

# 3. 启动工作流
# 在 OpenCode 中说："启动工作流" 或 "请处理 ./requirements.md"
```

首次运行会自动检测环境。如需自定义配置，修改 OMO 或 OMX 配置文件。
