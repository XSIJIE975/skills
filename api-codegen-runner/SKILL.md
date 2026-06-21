---
name: api-codegen-runner
description: "Generate TypeScript API code from OpenAPI 3.0/3.1 or Apifox specs using EJS templates. 用户需要从 Swagger/OpenAPI/Apifox 生成接口代码、初始化配置、自定义模板，或使用 Vite 插件自动生成时触发。"
---

# API Codegen Runner

基于 EJS 模板引擎的 TypeScript API 代码生成工具。支持 OpenAPI (Swagger) 3.0/3.1 和 Apifox 数据源。

**GitHub:** [XSIJIE975/api-codegen-runner](https://github.com/XSIJIE975/api-codegen-runner)
**npm:** `api-codegen-runner`

## When to Use

- 项目需要从 OpenAPI/Swagger 文档生成 TypeScript 类型定义和请求函数
- 需要对接 Apifox 项目同步生成接口代码
- 需要自定义代码生成模板（EJS）
- 需要在 Vite dev server 启动时自动生成代码
- 需要 Watch 模式监听配置和模板变更自动重新生成

## Quick Start

```bash
# 安装
npm install api-codegen-runner -D

# 1. 初始化配置
npx api-codegen-runner init

# 2. 修改 codegen.config.ts 配置数据源和输出路径

# 3. 生成代码
npx api-codegen-runner generate

# 4. Watch 模式（监听变更自动重新生成）
npx api-codegen-runner generate --watch
```

推荐在 `package.json` 中添加脚本：

```json
{
  "scripts": {
    "api:gen": "api-codegen-runner generate",
    "api:watch": "api-codegen-runner generate --watch"
  }
}
```

## CLI Commands

| Command | Description | Options |
|---------|-------------|---------|
| `init` | 初始化 `codegen.config.ts` | 交互式选择是否 eject 默认模板 |
| `generate` | 执行代码生成（默认命令） | `-c, --config <path>` 指定配置<br>`-w, --watch` 监听模式 |
| `update` | 更新工具版本或模板 | — |

## Config Reference

配置示例：

```typescript
import { defineConfig } from 'api-codegen-runner';

export default defineConfig({
  // 数据源：OpenAPI URL 或本地 JSON 文件路径
  input: 'https://petstore3.swagger.io/api/v3/openapi.json',

  // 或 Apifox 项目配置
  input: {
    projectId: '123456',
    token: 'YOUR_APIFOX_ACCESS_TOKEN',
  },

  output: {
    apiDir: 'src/api',       // API 函数输出目录
    typeDir: 'src/types',    // 类型定义输出目录
    separateTypes: true,     // 是否拆分类型文件
  },

  methodNameCase: 'PascalCase', // 方法名格式
  clean: true, // 生成前清理输出目录

  globalContext: {
    importRequestStr: "import request from '@/utils/request';",
  },
});
```

完整配置选项见 [GitHub 仓库](https://github.com/XSIJIE975/api-codegen-runner)。

## Template Customization

工具内置 EJS 模板引擎，支持完全自定义生成逻辑：

```ejs
<% functions.forEach(function(func) { %>
/**
 * <%= func.description %>
 */
export const <%- func.name %> = (<%- func.paramsSignature %>) => {
  return request.<%= func.method %>('<%- func.url %>'<%- func.hasBody ? ', data' : '' %><%- func.hasQueryParams ? ', params' : '' %>);
};
<% }) %>
```

可用变量：
- `functions` — API 函数列表（name, method, url, paramsSignature...）
- `imports` — 类型导入信息
- `config` — 全局配置上下文
- `utils` — 格式化工具（camelCase, pascalCase, kebabCase...）

## Vite Plugin

```typescript
import { defineConfig } from 'vite';
import { ApiCodegenPlugin } from 'api-codegen-runner';

export default defineConfig({
  plugins: [ApiCodegenPlugin()],
});
```

启动 dev server 时自动生成代码，并监听 `codegen.config.ts` 和 `.ejs` 模板文件变更。

## Best Practices

1. **模板分离**：自定义模板放在独立目录（如 `templates/`），不要放在 `node_modules` 内修改
2. **Apifox 鉴权**：`token` 使用环境变量注入，不要硬编码
3. **增量生成**：使用 `watch` 模式开发，仅在 `generate` 命令中启用 `clean` 以减少全量生成
4. **类型拆分**：大型项目启用 `separateTypes: true`，避免单文件过大导致 LSP 性能问题
5. **生命周期钩子**：使用 `hooks.onComplete` 执行生成后处理（如格式化、lint 修复）
