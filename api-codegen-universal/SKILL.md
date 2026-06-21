---
name: api-codegen-universal
description: "OpenAPI 3.0/3.1 和 Apifox 的通用解析适配器，提供类型生成、路径分类、参数提取等能力。作为 api-codegen-runner 的底层依赖，当需要理解代码生成流程、自定义解析行为、或排查生成问题深层原因时触发。"
---

# API Codegen Universal

OpenAPI 3.0/3.1 和 Apifox 的通用解析适配层。提供标准化的类型定义、解析器接口和适配器实现，是 `api-codegen-runner` 的底层引擎。

**GitHub:** [XSIJIE975/api-codegen-universal](https://github.com/XSIJIE975/api-codegen-universal)
**npm:** `api-codegen-universal`, `@api-codegen-universal/core`, `@api-codegen-universal/openapi`, `@api-codegen-universal/apifox`

## When to Use

- 需要理解 API 代码生成的底层架构
- 自定义或扩展 OpenAPI/Apifox 的解析逻辑
- 排查 api-codegen-runner 生成结果的深层问题
- 需要直接使用底层解析器（不经过 api-codegen-runner 的模板层）
- 了解适配器扩展机制以支持新的 API 规范

## Architecture

```
api-codegen-universal (主入口)
├── @api-codegen-universal/core    — 核心类型定义和标准接口
├── @api-codegen-universal/openapi — OpenAPI 3.0/3.1 适配器
└── @api-codegen-universal/apifox  — Apifox 适配器
```

**Core** 定义的标准接口：
- `Adapter` — 解析器接口，所有数据源适配器必须实现
- `Config` — 通用配置类型
- `StandardSchema` — 统一的标准数据模型

## Key Capabilities

### OpenAPI Adapter
- 解析 OpenAPI 3.0/3.1 规范（JSON/YAML）
- 路径分类（按 tag 或自定义规则）
- 参数提取（path, query, body）
- Schema 类型推导（支持泛型、嵌套结构）
- 接口生成（TypeScript 类型定义）
- AST 工具（用于类型转换）

### Apifox Adapter
- 与 Apifox 项目同步
- 兼容性修复（自动处理 Apifox 与标准 OpenAPI 的差异）
- 聚合警告输出（单次汇总，减少日志噪音）
- 支持 `logLevel` 和自定义 `logger`

## Usage as Library

```typescript
import { createAdapter } from 'api-codegen-universal';
// 或直接使用特定适配器
import { OpenAPIAdapter } from '@api-codegen-universal/openapi';
import { ApifoxAdapter } from '@api-codegen-universal/apifox';

const adapter = createAdapter({ type: 'openapi', input: 'https://example.com/openapi.json' });
const schema = await adapter.parse();
// schema 包含标准化的 paths, types, tags...
```

## Logging Configuration

适配器统一日志接口：

```typescript
{
  logLevel: 'error' | 'warn' | 'info' | 'debug', // 默认: 'error'
  logger: { warn: (msg, meta) => void },           // 自定义 logger，默认 console
  logSampleLimit: 10,                               // warnings 中 samples 上限
}
```

Apifox 适配器在 `logLevel >= 'warn'` 时将兼容性修复警告聚合成单次汇总输出。

## Best Practices

1. **使用 api-codegen-runner**：大多数场景应通过上层 `api-codegen-runner` 使用，不需要直接操作本库
2. **适配器扩展**：如需支持新的 API 规范格式，实现 `Adapter` 接口即可
3. **调试**：设置 `logLevel: 'debug'` 获取完整的解析流水线日志
4. **类型安全**：解析后的 `StandardSchema` 已经过类型校验，可以安全直接使用
