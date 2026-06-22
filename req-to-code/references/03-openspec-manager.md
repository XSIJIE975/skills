# OpenSpec Manager

管理 OpenSpec 工作流，遵循 OpenSpec 标准工作流：初始化项目 → 需求转 specs → propose 变更 → 门禁确认 → 验证完整性。

**核心原则**：全程使用 OpenSpec 工作流工具（`openspec` CLI + 对应的 AI slash command 模式），Agent 不直接写入 spec 文件内容。

## 输入

- 需求描述（markdown 格式，来自 document-converter）
- 项目名称（可选）
- 变更类型：feature / bugfix / refactor

## 输出

- OpenSpec 规格目录（`openspec/specs/`）
- OpenSpec 变更目录（`openspec/changes/<change-name>/`）
- 需求规格、技术设计、任务清单
- 已确认的门禁状态（写入状态文件）

## 流程

### 1. 初始化项目

检测 `openspec/` 目录是否已存在：

- 已存在 → 跳过
- 不存在 → 运行 `openspec init`：

```bash
openspec init
```

`openspec init` 会自动创建项目结构，并根据选择的 AI 工具生成对应的 slash command 技能文件。
项目结构如下：

```
openspec/
├── specs/              # 空的规格目录
├── changes/            # 空的变更目录
└── config.yaml         # 项目配置
```

> 如果已有 `openspec/` 目录，用 `openspec update` 刷新技能文件。

### 2. 需求文档转 specs

如果有需求文档（来自阶段 2 的 markdown），将其转换为 OpenSpec 规格格式。
使用 OpenSpec 的 **explore 工作流**：Agent 读取需求文档，按领域拆分，写入 `openspec/specs/` 目录。

**操作**：读取需求 markdown，将其转换为 OpenSpec specs 格式：

```
openspec/specs/<domain>/spec.md

## Requirements

### Requirement: <功能名称>
The system SHALL/MUST/SHOULD <行为描述>。

#### Scenario: <场景名称>
- GIVEN <前置条件>
- WHEN <操作>
- THEN <预期结果>
```

**规则**：
- 每个需求按领域拆分为独立文件（`auth/spec.md`、`payments/spec.md`、`ui/spec.md` 等）
- 每个需求用 SHALL/MUST/SHOULD 描述行为
- 每个需求配 1-N 个 Given/When/Then 场景
- spec 只写行为，不写实现细节
- 如果没有需求文档或用户跳过此步，直接进入第 3 步

### 3. 创建变更（Propose）

使用 OpenSpec 的 **propose 工作流**：一次性生成提案、delta specs、技术设计、任务清单。

**操作**：

```bash
openspec new change <change-name> \
  --description "Type: <feature|bugfix|refactor> - <short description>" \
  --json
```

生成目录结构：
```
openspec/changes/<change-name>/
├── .openspec.yaml     # 变更元数据
├── proposal.md        # 变更提案
├── specs/             # Delta 规格（只包含本次变更的需求）
│   └── <domain>/
│       └── spec.md
├── design.md          # 技术设计
└── tasks.md           # 任务清单
```

**生成 artifacts**：
Artifact 文件通过 OpenSpec CLI 的 `openspec instructions` 获取模板指令，Agent 基于模板填充内容。不直接写 markdown。

```bash
# 获取各 artifact 的模板指令
openspec instructions proposal --change <change-name> --json
openspec instructions specs --change <change-name> --json
openspec instructions design --change <change-name> --json
openspec instructions tasks --change <change-name> --json
```

`openspec instructions` 返回：
- 当前 artifact 的模板（标题结构、章节要求）
- 依赖的 artifact 内容（用于上下文关联）
- 项目配置中的自定义规则（如 tech stack、API 风格等）
- 必填字段和可选字段标记

Agent 根据模板指令生成文件，确保符合 OpenSpec schema 规范。

### 4. 门禁 1：提案 + 需求规格（Human-in-the-Loop）

生成 proposal.md 和 specs/ 后，向用户展示确认。

**🔒 门禁展示**：用 3 个结构化问题确认：

```
已生成提案和需求规格，请确认：

1. 【范围】目标和范围是否正确？有没有遗漏的需求或不该包含的内容？
   → 正确 / 需要修改（具体说明）

2. 【验收】验收标准是否能衡量成功？是否可测试？
   → 可以 / 需要调整

3. 【约束】有没有隐含的约束没写进去？（技术栈限制、性能要求、合规要求等）
   → 没有 / 有（说明）
```

**反馈处理**：
- 用户选择"正确/可以/没有" → 标记为已确认，进入下一阶段
- 用户提出修改 → 修订对应文档（通过 openspec 工具更新，不直接写文件），版本号 +1，重新展示 diff
- 用户有补充约束 → 更新 requirements.md，重新展示 diff

**状态记录**：
```json
{
  "sub_phase": "propose_requirements",
  "gate_status": "approved",
  "documents": {
    "proposal": { "version": 1, "confirmed": true },
    "requirements": { "version": 2, "confirmed": true }
  },
  "user_feedback": [
    { "phase": "propose_requirements", "round": 1, "changes": ["补充了性能约束：API 响应 < 200ms"] }
  ]
}
```

### 5. 门禁 2：技术设计 + 任务清单（Human-in-the-Loop）

基于已确认的 requirements，生成 design.md 和 tasks.md 后向用户展示。

**🔒 门禁展示**：用 3 个结构化问题确认：

```
已生成技术设计和任务清单，请确认：

1. 【技术选型】技术方案是否符合现有系统架构？有没有不能接受的选型？
   → 符合 / 需要调整（具体说明）

2. 【任务边界】任务分解是否清楚？每个任务的边界是否明确？
   → 清楚 / 某些任务需要调整

3. 【优先级】任务优先级和并行分组是否合理？
   → 合理 / 需要调整
```

**反馈处理**（同上：修订 → diff 展示 → 确认）

**状态记录**：
```json
{
  "sub_phase": "design_tasks",
  "gate_status": "approved",
  "documents": {
    "design": { "version": 1, "confirmed": true },
    "tasks": { "version": 1, "confirmed": true }
  }
}
```

### 6. 完整性验证

用 OpenSpec CLI 验证 artifacts 完整性，外加语义检查：

```bash
openspec status --change <change-name> --json
openspec validate --changes --json
```

检查清单：
- [ ] 所有 artifact 状态为 done（`openspec status` 确认）
- [ ] 需求规格覆盖所有输入需求（逐条 trace）
- [ ] 技术设计中的每个决策有明确理由
- [ ] 任务清单与需求规格交叉可追溯
- [ ] 每个任务有明确的验收标准
- [ ] 无 "TBD"、"待定" 等模糊标记
- [ ] design.md 中的选型不违反 requirements.md 中的约束

发现缺失时：补充缺失 artifact 后重新验证。验证通过后进入阶段 4（Task Decomposer）。

### 7. 快速通道（紧急变更/小变更）

如果满足以下**任一**条件，自动启用快速通道：
- 变更类型为 `bugfix`
- 任务数 ≤ 3 **且** 所有任务复杂度均为"低"
- 用户显式指定 "快速模式"

**不适用快速通道的场景**（即使任务数少但复杂性高）：
- 涉及架构变更
- 涉及数据迁移
- 涉及第三方集成
- 涉及安全/权限改造

快速通道行为：
- 合并门禁 1 和门禁 2 为 **1 次确认**
- 一次性展示 proposal + requirements + design + tasks
- 用户一次性确认或修改
- 跳过完整性验证（简化版）

## 文档历史版本（支持 diff 展示）

每次门禁修订时，保存历史版本到状态文件中，以便生成 diff：

```json
{
  "document_history": {
    "proposal": {
      "versions": [
        { "version": 1, "content": "第一版完整内容...", "saved_at": "2026-06-22T10:25:00" },
        { "version": 2, "content": "根据反馈修订后的内容...", "saved_at": "2026-06-22T10:30:00" }
      ]
    },
    "requirements": { "versions": [...] },
    "design": { "versions": [...] },
    "tasks": { "versions": [...] }
  }
}
```

修订后向用户展示时，由 Agent 对比最新两个版本的内容差异并输出变更点摘要（直接文本对照，跨平台一致）。

> **注意**：只保存最近 3 个版本（超过则丢弃最旧的），避免状态文件过大。

## 状态文件门禁记录

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

`recovery_checkpoint` 记录最近一次**已确认**的门禁位置。恢复时从该 checkpoint 继续，用户无需重新确认已通过的门禁。

## 错误处理

- OpenSpec 未安装 → 报告并提供安装命令
- `openspec init` 失败 → 检查权限和磁盘空间
- 规格文档生成不完整 → 列出缺失项，要求补充
- 用户拒绝门禁内容 → 记录反馈，修订后重新展示（最多 3 轮，超过升级给用户决策）
- 门禁确认后会话中断 → 从 `recovery_checkpoint` 恢复，无需重新确认
