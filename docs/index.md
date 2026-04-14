# Code_Agent 说明文档  

一个面向 C/C++ 工程的、基于静态解析增强上下文的代码代理，使用 NaturalCC 理解项目结构，用 Prompt Agent 组织上下文，再借助 Aider 实际落地代码修改。

---

## 概述

`code_agent` 是一个把 **NaturalCC 的静态语义解析能力** 和 **Aider 的代码修改能力** 组合起来的代码代理实现，目标是让大模型在修改 C/C++ 项目代码时，不只是看当前片段，而是先理解项目里的函数、变量、结构体、头文件关系，再生成更贴近真实工程上下文的 prompt，最后交给 Aider 执行修改。

### 代码职责

从代码职责上看，它可以分成两层：

1. `agent_ui.py`、`aider_runner.py`、`completion_prompt_agent.py`
   这部分是“可交互的 agent 主链路”，负责收集用户输入、构造 prompt、调用 Aider。
2. `rag/`
   这部分是“项目解析与提示增强底座”，负责解析 C/C++ 项目、抽取符号关系、拼接检索上下文，以及配套的离线评测脚本。

换句话说，这个目录实现的并不是一个通用聊天 agent，而是一个更偏向 **C/C++ 项目代码补全、函数实现补全、重构辅助** 的上下文增强代理。

---

## 核心特点

和纯粹把代码片段直接发给模型相比，`code_agent` 的特点是：

- 它先理解项目，再让模型动手。
- 它特别针对 C/C++ 项目。
- 它不仅支持补全，还能驱动 Aider 直接修改文件。
- 它同时保留了研究实验脚本，说明这个目录既有“可用工具”的一面，也有“论文/实验代码”的一面。

### 主要文件

当前最像“产品主链路”的文件是：

- `agent_ui.py`
- `aider_runner.py`
- `completion_prompt_agent.py`
- `rag/preprocess.py`
- `rag/cfile_parse.py`
- `rag/node_prompt.py`

---

## 环境依赖

- 需要安装并可执行 `aider`。
- 需要 `libclang`，否则 `rag/cfile_parse.py` 无法工作。
- 需要对应 Python 依赖，如 `gradio`、`clang`、`transformers`、`torch`、`tiktoken`、`yaml`、`attridict`、`requests`、`vllm`、`python-Levenshtein`、`py2neo` 等。


!!! note "Note"
    如果你在安装 `libclang` 时遇到路径问题，请检查系统的环境变量设置。
    如果你遇到了"libclang not found."问题，请手动设置`LIBCLANG_PATH`。

---

## 注意事项

从当前代码看，运行这套 agent 至少要注意下面几点：

- `rag/config.yaml` 和 `rag/utils.py` 中包含大量本地实验路径，开箱即用性一般，需要按本机环境调整。
- `rag/utils.py` 里直接设置了 `CUDA_VISIBLE_DEVICES`，这会影响运行环境。
- `json2neo4j.py` 中 Neo4j 地址、账号和输入文件路径都是硬编码的。
- 这套解析链目前明显是围绕 C/C++ 设计的，不是通用多语言 agent。

---

## 运行

### 启动 UI

```bash
python -m code_agent.agent_ui
```

或直接：

```bash
python code_agent/agent_ui.py
```

### 命令行模式

```bash
python -m code_agent.aider_runner \
  -f src/main.c \
  -i "补全 parse_flags 函数的完整实现" \
  -m openrouter/deepseek/deepseek-chat
```

### 仅预览 prompt

```bash
python -m code_agent.aider_runner \
  -f src/main.c \
  -i "补全 parse_flags 函数的完整实现" \
  --preview
```

---

## 入门

启动后页面会显示：

- 顶部紧凑抬头区，整体横向居中铺开
- 左侧“项目与模型配置”卡片：底部两个按钮切换“常规配置”和“工程概览 / 进阶控制”
- 右侧“任务与执行”卡片：底部三个按钮切换“开发指令”“命令行生成内容”“操作说明 / 状态”
- 右侧三个视图上方不再额外放统一“任务状态”条，状态信息收拢到各自视图内部
- 右侧执行动作触发后，会自动跳转到“命令行生成内容”面板展示结果
- 左右卡片底部按钮固定在卡片外层底部，正文改为独立滚动区，避免内容从按钮下方“透出来”
- 桌面端左右主卡片默认压在首屏内，并按浏览器宽高自适应收缩；超出的内容会在各自卡片主体区滚动，滚动条默认隐藏，交互时再显现
- “命令行生成内容”面板内部采用双栏：左侧优先放日志/输出，右侧放等效 CLI 命令草案
- `agent_ui.py` 负责主布局与核心逻辑，`agent_ui_assets.py` 负责抬头、CSS 和静态说明文案，`agent_ui_bindings.py` 负责事件绑定

---

## 示例

这里可以添加一些演示图片。

---

## 核心执行流程

主链路可以概括为：

1. 用户在 Gradio 页面或命令行里输入需求、目标文件、模型、API Key。
2. `aider_runner.py` 负责规范化项目目录和目标文件路径。
3. `CompletionPromptAgent` 调用 `rag/preprocess.py` 对项目做静态解析。
4. 解析器基于 `libclang` 提取函数、变量、结构体、typedef、include 和符号关系。
5. `CompletionPromptAgent` 根据用户意图推断补全类型：
   - 成员补全
   - 变量补全
   - 函数签名补全
   - 函数体补全
   - 类型补全
6. 生成的项目语义 prompt 会被包进一段更高层的系统指令里。
7. `aider_runner.py` 把最终 prompt 写入临时文件，并调用 `aider --message-file ...`。
8. Aider 根据这个上下文去修改目标文件。

其中一个很关键的实现细节是：

- Aider 可以接收多个目标文件。
- 但当前 prompt 构造时，**第一个目标文件** 会被当作主要解析目标文件，用来推断符号和生成上下文。

---

## 目录结构概览

```text
code_agent/
├── __init__.py
├── agent_ui.py                      # Gradio 页面入口
├── agent_ui_assets.py               # UI 静态 HTML/CSS/说明文案
├── agent_ui_bindings.py             # UI 按钮/输入事件绑定
├── aider_runner.py                  # Aider 调度与命令行入口
├── completion_prompt_agent.py       # 当前主用的 Prompt 编排器
├── test_api.py                      # OpenRouter API 连通性测试
└── rag/
    ├── __init__.py
    ├── cfile_parse.py               # 基于 libclang 的 C/C++ AST 解析
    ├── preprocess.py                # 项目级解析与关系清洗
    ├── node_prompt.py               # 基于符号图的上下文拼接
    ├── generator.py                 # 离线 prompt 生成器
    ├── tokenizer.py                 # 模型 token 长度控制
    ├── tokenizer copy.py            # 旧版 tokenizer 备份
    ├── completion_prompt_agent_origin.py
    ├── main.py                      # 批量生成 prompt 的脚本入口
    ├── eval.py                      # HF 评测
    ├── evaluation.py                # 旧版/平行评测脚本
    ├── eval_vllm.py                 # vLLM 双路评测
    ├── eval_vllm3.py                # vLLM 三路评测
    ├── json2neo4j.py                # 图谱导入 Neo4j 的实验脚本
    ├── config.yaml                  # 模型与 token 配置
    └── utils.py                     # 数据集/结果/默认模型常量
```

---

## 获取帮助

如果需要获取关于Code_Agent的帮助，请使用[GitHub issues](https://github.com/CGCL-codes/naturalcc/issues)