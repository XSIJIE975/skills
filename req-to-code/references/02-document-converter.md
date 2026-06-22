# Document Converter

将 docx/pdf/txt/html 转换为标准 markdown。使用 markitdown MCP 工具。

## 输入

- 文件路径（docx/pdf/txt/html）
- 可选：输出目录（默认同目录）

## 输出

- 转换后的 `.md` 文件

## 转换流程

### 1. 输入验证

检查文件是否存在、格式是否支持：

| 扩展名 | 支持 | 说明 |
|--------|------|------|
| `.docx` | ✅ | 通过 markitdown MCP 转换 |
| `.pdf` | ✅ | 通过 markitdown MCP 转换 |
| `.txt` | ✅ | 直接读取，包装为 markdown |
| `.html`/`.htm` | ✅ | 通过 markitdown MCP 转换 |
| `.md` | ℹ️ | 跳过转换 |
| 其他 | ❌ | 报告不支持 |

### 2. 执行转换

调用 markitdown MCP 的 `convert_to_markdown(uri)` 工具，URI 格式：

**普通路径**（Linux/macOS/Git Bash）：
- 本地文件：`file:///absolute/path/to/document.docx`
- 远程 URL：`https://example.com/doc.pdf`

**Windows 路径**：
- 本地文件：`file:///C:/Users/name/document.docx`（Agent 自动处理转义）



### 3. 输出清理

转换后执行：
- 移除连续空行（保留最多 1 个）
- 移除行尾空白
- 确保文件以单个换行结尾
- 如果转换失败或输出为空，报告错误并保留原始文件

### 4. 结果验证

- 确认输出文件存在且非空
- 输出转换统计：原始大小 → 转换后大小、行数
- 如果 `.md` 文件是源文件（无需转换），记录跳过原因

## 错误处理

- 文件不存在 → 报告错误，请求正确路径
- 格式不支持 → 报告不支持的格式
- markitdown MCP 不可用 → 报告工具缺失，建议安装
- 转换输出为空 → 保留原始文件，报告转换失败
