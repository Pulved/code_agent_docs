## 环境搭建

### 前置条件

- [uv](https://docs.astral.sh/uv/getting-started/installation/)（Python 包管理器）
- Node.js & npm（前端构建）

### 1. 安装系统依赖

C/C++ 解析需要系统级 `libclang` 库。在运行 `uv sync` 之前，通过系统包管理器安装：

- **Ubuntu/Debian**: `sudo apt install libclang1`
- **macOS**: `brew install libclang`
- **其他发行版**: 在系统包仓库中搜索 `libclang` 并安装

### 2. 创建 Python 环境

在 `code_agent/` 目录下执行：

```bash
uv sync
```

此命令会自动创建 `.venv` 虚拟环境并安装所有锁定的 Python 依赖，无需手动配置 conda 或执行 pip。

项目运行至少需要以下能力（均由 `uv sync` 自动处理）：

- `fastapi`
- `uvicorn`
- `clang` Python bindings
- `aider` 可执行命令在 `PATH` 中

如需 GPU 支持（例如运行基于 vLLM 的离线评估），可手动安装：

```bash
uv pip install vllm
```

调用 OpenRouter / OpenAI 时，可以在界面或 CLI 中传入 API Key，也可以设置环境变量：

```bash
export OPENROUTER_API_KEY=sk-or-v1-...
export OPENAI_API_KEY=sk-...
```

### 3. 安装前端依赖

在 `code_agent/` 目录下执行：

```bash
cd webui
npm install
```