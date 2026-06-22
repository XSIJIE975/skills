# 前置依赖与环境

## 依赖检查

工作流启动时验证以下工具可用：

| 工具 | 必要性 | 验证方式 |
|------|--------|---------|
| OpenSpec CLI (v1.0+) | 必需 | `openspec --version` |
| Codex CLI | 可选 | `codex --version`（无则降级为 Oracle 审查） |
| markitdown MCP | 必需 | MCP 服务器可用性检查 |
| Git | 必需 | `git --version` |
| Node.js (20.19.0+) | 必需（OpenSpec 依赖） | `node --version` |

缺失依赖时：报告错误并提供安装命令，不自动安装。

## 安装

```bash
# OpenSpec CLI
npm install -g @fission-ai/openspec@latest

# markitdown-mcp（通过 uv）
# Linux/macOS: curl -LsSf https://astral.sh/uv/install.sh | sh
# macOS (brew): brew install uv
# Windows:      powershell -ExecutionPolicy ByPass -c "irm https://astral.sh/uv/install.ps1 | iex"
uv pip install markitdown-mcp
```

## 配置优先级

所有配置遵循：OMO > OMX > 默认值。

| 优先级 | 配置源 | 说明 |
|--------|--------|------|
| 1 (最高) | OMO 配置 | 提供 agents/categories 模型映射、Team Mode |
| 2 | OMX 配置 | 提供工作流命令、模型配置 |
| 3 (最低) | 内置默认 | 各阶段的默认行为 |

具体配置文件路径因 OS 而异（Agent 自动检测 OS 并适配路径）。

## 快速启动

1. 安装依赖（见上方）
2. 将需求文档（.docx/.pdf/.md）放在项目根目录
3. 在 OpenCode 中说："启动工作流" 或 "请处理 ./requirements.md"

首次运行会自动检测环境。如需自定义配置，修改 OMO 或 OMX 配置文件。
