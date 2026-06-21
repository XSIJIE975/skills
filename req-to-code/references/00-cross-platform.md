# 跨平台环境适配

所有阶段共用的跨平台检测逻辑。各阶段引用此文件，不重复实现。

## OS 检测

执行 `uname -s` 判断平台：

| 输出 | 平台 | 路径分隔符 | 临时目录 |
|------|------|-----------|---------|
| `Linux` | Linux | `/` | `/tmp` |
| `Darwin` | macOS | `/` | `$TMPDIR` 或 `/tmp` |
| `MINGW*`/`MSYS*`/`CYGWIN*` | Windows (Git Bash) | `/` | `$TEMP` |

## 平台适配命令

| 操作 | POSIX | Windows (Git Bash) |
|------|-------|-------------------|
| 检查文件存在 | `test -f <path>` | 同 POSIX |
| 获取文件大小 | `wc -c < <file> \| tr -d ' '` | 同 POSIX |
| 创建目录 | `mkdir -p <dir>` | 同 POSIX |
| 复制文件 | `cp -r <src> <dst>` | 同 POSIX |
| 删除目录 | `rm -rf <dir>` | 同 POSIX |
| 当前时间 ISO | `date -u +"%Y-%m-%dT%H:%M:%SZ"` | 同 POSIX |
| 时间戳文件名 | `date +%Y%m%d-%H%M%S` | 同 POSIX |

## 状态文件路径

| 文件 | Linux / macOS | Windows |
|------|--------------|---------|
| 工作流状态 | `./.req-to-code-state.json` | 同左 |
| 临时目录 | `/tmp/req-to-code/` | `$TEMP/req-to-code/` |
| 日志 | `~/.local/share/req-to-code/logs/` | `%LOCALAPPDATA%/req-to-code/logs/` |

## 环境变量写入

初始化阶段将检测结果写入状态文件的 `env` 字段，后续阶段从状态文件读取，不硬编码路径：

```json
{
  "env": {
    "os": "linux",
    "tmp_dir": "/tmp/req-to-code",
    "path_sep": "/"
  }
}
```
