# OpenSpec Manager

管理 OpenSpec 工作流：初始化项目、创建变更提案、生成规格文档、门禁确认、验证完整性。

与用户交互方式：**结构化确认**（Agent 生成 → 展示 → 有限个具体问题 → 用户确认）。
本阶段不使用开放式对话（开放式探讨在阶段 4 的深入探讨对话中处理）。

## 输入

- 需求描述（markdown 格式，来自 document-converter）
- 项目名称（可选）
- 变更类型：feature / bugfix / refactor

## 输出

- OpenSpec 变更目录（`openspec/changes/<change-name>/`）
- 需求规格（`specs/requirements.md`）
- 技术设计（`design.md`）
- 任务清单（`tasks.md`）
- 已确认的门禁状态（写入状态文件）

## 流程

### 1. 初始化项目

- 检查 `openspec/` 目录是否已存在，已存在则跳过
- 运行 `openspec init` 创建项目结构
- 配置 `openspec/openspec.json`（项目名、版本、设置）

### 2. 创建变更目录（自动化，无交互）

OpenSpec CLI 只提供目录创建和基础元数据写入。**实际的 spec 文件（proposal.md、design.md 等）由 Agent 直接写 markdown 文件生成**，而非 CLI 命令。

变更命名格式：`<type>-<YYYYMMDD-HHMMSS>-<short-desc>`

执行 `openspec new change` 创建变更目录（非交互模式）：

```bash
openspec new change <change-name> \
  --description "Type: <feature|bugfix|refactor> - <short description>" \
  --json
```

生成目录结构：
```
openspec/changes/<change-name>/
├── .openspec.yaml     # 变更元数据
├── proposal.md        # 变更提案（由 Agent 生成）
├── specs/             # 需求规格目录（由 Agent 生成）
│   └── requirements.md
├── design.md          # 技术设计（由 Agent 生成）
└── tasks.md           # 任务清单（由 Agent 生成）
```

> **注意**：`openspec new change` 只创建目录和 `.openspec.yaml`。后续所有 `.md` 文件由 Agent 在该步骤中直接写入，不需要 OpenSpec 提供文件模板命令。OpenSpec 本身不提供交互式提出/审查循环，审阅机制由本阶段的门禁系统实现。

### 3. 门禁 1：提案 + 需求规格（Human-in-the-Loop）

基于转换后的需求 markdown，生成 proposal.md 和 specs/requirements.md，然后向用户展示。

**生成阶段**（Agent 自动完成）：

**proposal.md** 包含：
- 变更背景和动机
- 目标（明确的可量化目标）
- 非目标（明确什么不做）
- 变更范围

**specs/requirements.md** 包含：
- 功能需求（按模块组织）
- 非功能需求（性能、安全、可用性）
- 约束条件
- 验收标准（每个需求对应可测试的标准）

**🔒 门禁展示**：向用户展示已生成的内容，用 3 个结构化问题确认：

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
- 用户提出修改 → 修订对应文档，版本号 +1，重新展示（只展示 diff）
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

### 4. 门禁 2：技术设计 + 任务清单（Human-in-the-Loop）

基于已确认的 requirements.md，生成 design.md 和 tasks.md。

**生成阶段**（Agent 自动完成）：

**design.md** 包含：
- 架构方案和组件关系
- 数据模型（实体、关系）
- API 设计（端点、请求/响应格式）
- 技术选型和理由（使用现有技术栈偏好）

**tasks.md** 包含：
- 按依赖顺序排列的任务列表
- 每个任务包含：描述、验收标准、复杂度（低/中/高）
- 标记可并行执行的任务组

**🔒 门禁展示**：向用户展示，用 3 个结构化问题确认：

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

### 5. 完整性验证（自动化，无交互）

检查清单（语义层面 + 语法层面）：

- [ ] 需求规格覆盖所有输入需求（逐条 trace）
- [ ] 技术设计中的每个决策有明确理由
- [ ] 任务清单与需求规格交叉可追溯
- [ ] 每个任务有明确的验收标准
- [ ] 无 "TBD"、"待定" 等模糊标记
- [ ] design.md 中的选型不违反 requirements.md 中的约束

发现缺失时：记录问题，自动补充后重新验证。验证通过后进入阶段 4（Task Decomposer）。

### 6. 快速通道（紧急变更/小变更）

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
