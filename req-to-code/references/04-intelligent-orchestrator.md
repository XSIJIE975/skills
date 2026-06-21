# Intelligent Orchestrator

调度 Agent 执行任务，强制 TDD 流程。工作流的核心执行引擎。

## 输入

- 任务计划（来自 task-decomposer）
- 任务依赖关系和优先级
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
其他？ → unspecified-high 或 unspecified-low
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

每个任务必须遵循：

1. **Red** — 先写失败的测试
   - 根据验收标准编写测试用例
   - 运行测试，确认失败（输出应为红色）
2. **Green** — 最小实现使测试通过
   - 编写刚好让测试通过的代码
   - 运行测试，确认通过
3. **Refactor** — 重构保持测试通过
   - 清理代码结构
   - 提取重复逻辑
   - 运行测试，确认仍通过

### 5. 执行验证

每个任务完成后：
- 运行相关测试，确认全部通过
- 检查代码是否符合项目风格
- 更新状态文件中对应任务的状态

## 约束

- 没有测试的代码不允许标记为完成
- 测试覆盖率不低于 80%（项目有覆盖率工具时）
- 每个任务的执行结果必须可追溯（记录在状态文件中）
- 并行执行时注意文件锁冲突
