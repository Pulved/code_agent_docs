## 使用 CLI

在 `code_agent/` 目录下执行命令。

### 仅预览 Prompt

```bash
uv run python aider_runner.py \
  -dir /path/to/project \
  -f src/foo.c include/foo.h \
  -i "补全 foo 函数实现" \
  --preview
```

### 执行 Aider 修改文件

```bash
uv run python aider_runner.py \
  -dir /path/to/project \
  -f src/foo.c include/foo.h \
  -i "根据现有风格完善 foo 函数实现" \
  -m openrouter/deepseek/deepseek-chat
```

### 常用 CLI 参数

```bash
-dir /path/to/project
-f src/foo.c include/foo.h
-i "你的修改或补全需求"
-m openrouter/deepseek/deepseek-chat
-key sk-...
-s parse_flags
-t function_body
--prefix parse_
--preview
```

`-t` 支持：

```text
member
variable
function
function_body
type
```

## API 接口

`agent_web_api.py` 提供：

- `GET /api/health`
- `GET /api/bootstrap`
- `GET /api/models`
- `GET|POST /api/workspace/scan`
- `GET /api/browse`
- `POST /api/command-preview`
- `POST /api/prompt/preview`
- `POST /api/run`

`/api/run` 会返回按行分隔的 JSON 事件，前端用它实时显示 Aider 日志。