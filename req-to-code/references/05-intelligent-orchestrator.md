# Intelligent Orchestrator

调度 Agent 执行任务，强制 TDD 流程。工作流的核心执行引擎。

## 输入

- 执行计划：`openspec/changes/<change-name>/tasks.md`（来自 task-decomposer 的细化任务清单）
  - 包含每个任务的实现步骤、验收标准、依赖关系、测试要求
  - 不读取外部 plan 文件（如 `docs/superpowers/plans/`）
- 任务依赖关系和优先级（内联在 tasks.md 中）
- OMO/OMX 配置（自动检测）

## 输出

- 实现代码
- 测试代码（先于实现编写）
- 执行报告（每个任务的状态、测试结果、耗时）

## 流程

### 1. 配置检测

按优先级检测配置源：

| 优先级 | 配置源 | 检测路径 | 提供能力 |
|--------|--------|---------|---------|
| 1 | OMO | `~/.config/opencode/oh-my-openagent.jsonc` | agents/categories 模型映射、Team Mode |
| 2 | OMX | `~/.codex/config.toml` 或 `~/.codex/omx.json` | 工作流命令、模型配置 |
| 3 | 默认 | 无配置文件 | 内置默认映射 |

提取关键配置：
- `agents` — 每个 agent 的 model 映射
- `categories` — 每个 category 的 model 映射
- `team_mode.enabled` — 是否使用 Team Mode
- `team_mode.max_parallel_members` — 最大并行度

### 2. 任务分类

对每个任务按决策树分类：

```
涉及 UI/CSS/样式？ → visual-engineering
涉及 UI 设计/交互创意？ → artistry
涉及复杂算法/性能优化？ → ultrabrain
涉及架构/系统设计？ → deep
简单修改/配置变更？ → quick
其他？
  ├─ 涉及核心业务逻辑/数据处理/多个文件 → unspecified-high
  └─ 简单文本/配置/文档单文件修改 → unspecified-low
```

### 3. 执行策略

**Team Mode（OMO 启用时）**：
- 创建 team，分配任务给 team members
- 独立任务并行执行
- 依赖任务按序执行

**Background Agents（降级方案）**：
- 使用 `task(run_in_background=true)` 调度
- 独立任务同时启动
- 等待完成通知后收集结果

### 4. TDD 流程（强制）

每个任务的 prompt 中**必须显式注入 TDD 指令**。调度 Agent 在构建 task prompt 时，自动拼接以下模板：

```
你正在实现的任务来自 req-to-code 工作流。

【强制 TDD 要求】
你必须严格遵循 TDD 流程，顺序不可颠倒：
1. 【Red】先编写失败的测试用例（根据验收标准）
2. 运行测试，确认失败
3. 【Green】编写最小实现代码使测试通过
4. 运行测试，确认通过
5. 【Refactor】重构代码，保持测试通过
6. 再次运行测试，确认仍通过

不遵循 TDD 的代码将被拒绝。
```

**TDD 验证**：
- 任务完成后，检查测试文件的 git commit 时间是否早于实现文件（或对比创建时间）
- 如果测试和实现在同一个 commit 中，检查文件内代码顺序（测试代码是否在实现代码之前）
- 如果测试在实现之后编写，标记为 "TDD 违规"
- **建议**：Agent 在 Red 阶段后做一次 commit（`git add tests/ && git commit -m "test: add failing tests"`），Green 阶段后再做第二次 commit，使 TDD 流程可追溯
- 违规处理：打回重新按 TDD 顺序执行

**Background Agents 模式**：
如果使用 `task(run_in_background=true)` 调度，TDD 指令直接内联在 prompt 中，不依赖 Agent 的预配置知识。

**Team Mode**：
如果使用 Team Mode，TDD 指令写入 team member 的 prompt 中。

**tasks.md 中的 TDD 字段**（由阶段 4 生成）：
每个任务在细化后应包含 TDD 要求的显式说明（格式见阶段 4 的 tasks.md 模板）。

### 5. 执行验证

每个任务完成后：
- 运行相关测试，确认全部通过
- 检查代码是否符合项目风格
- 更新状态文件中对应任务的状态

## 约束

- 没有测试的代码不允许标记为完成
- 新增代码的测试覆盖率不低于项目原有水平（项目有覆盖率工具时）；无覆盖率工具时，确保每个验收标准对应至少一个测试用例
- 每个任务的执行结果必须可追溯（记录在状态文件中）
- 并行执行时注意文件锁冲突
- 执行计划的唯一来源是 `openspec/changes/<change-name>/tasks.md`
  - 不查找也不使用 `docs/superpowers/plans/` 目录下的文件
  - tasks.md 中已包含所有执行所需信息（步骤、测试、依赖）
