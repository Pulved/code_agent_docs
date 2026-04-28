## 环境搭建

### 1. 激活 Python 环境

```bash
conda activate naturalcc
```

### 2. 安装 Python 依赖

项目运行至少需要：

- `fastapi`
- `uvicorn`
- `clang` Python bindings
- `libclang`
- `aider` 可执行命令在 `PATH` 中

如果当前环境缺少依赖，可按你的本地环境策略安装。常见方式：

```bash
pip install fastapi uvicorn clang aider-chat
conda install -c conda-forge libclang
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