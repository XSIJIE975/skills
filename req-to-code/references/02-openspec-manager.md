# OpenSpec Manager

管理 OpenSpec 工作流：初始化项目、创建变更提案、生成规格文档、验证完整性。

## 输入

- 需求描述（markdown 格式，来自 document-converter）
- 项目名称（可选）
- 变更类型：feature / bugfix / refactor

## 输出

- OpenSpec 变更目录（`openspec/changes/<change-name>/`）
- 需求规格（`specs/requirements.md`）
- 技术设计（`design.md`）
- 任务清单（`tasks.md`）

## 流程

### 1. 初始化项目

- 检查 `openspec/` 目录是否已存在，已存在则跳过
- 运行 `openspec init` 创建项目结构
- 配置 `openspec/openspec.json`（项目名、版本、设置）

### 2. 创建变更提案

变更命名格式：`<type>-<YYYYMMDD-HHMMSS>-<short-desc>`

执行 `openspec propose` 生成变更目录，包含：
```
openspec/changes/<change-name>/
├── proposal.md        # 变更提案
├── specs/             # 需求规格目录
├── design.md          # 技术设计
└── tasks.md           # 任务清单
```

### 3. 填充规格文档

基于转换后的需求 markdown，生成三份文档：

**需求规格**（`specs/requirements.md`）：
- 功能需求（按模块组织）
- 非功能需求（性能、安全、可用性）
- 约束条件
- 验收标准

**技术设计**（`design.md`）：
- 架构方案
- 数据模型
- API 设计
- 技术选型和理由

**任务清单**（`tasks.md`）：
- 按依赖顺序排列的任务列表
- 每个任务包含：描述、验收标准、预估时间

### 4. 验证完整性

检查清单：
- [ ] 需求规格覆盖所有输入需求
- [ ] 技术设计包含架构图或组件关系
- [ ] 任务清单无遗漏（与需求规格交叉检查）
- [ ] 每个任务有明确的验收标准
- [ ] 无 "TBD"、"待定" 等模糊标记

发现缺失时：记录问题，补充后重新验证。

## 错误处理

- OpenSpec 未安装 → 报告并提供安装命令
- `openspec init` 失败 → 检查权限和磁盘空间
- 规格文档生成不完整 → 列出缺失项，要求补充
