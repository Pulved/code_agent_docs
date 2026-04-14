## 1. `rag/` 子目录解读

`rag/` 是这个 agent 的“项目理解引擎”。如果不看这里，就只能把它理解成一个 Aider UI；但真正体现 NaturalCC 特征的内容，大都在这一层。

### 1.1 `preprocess.py`

定义了 `CProjectParser`，负责：

- 遍历项目目录中的 `.c/.cpp/.h/.hpp` 文件。
- 调用 `CParser` 逐个解析文件。
- 汇总成项目级 `parse_res`。
- 初始化 `CProjectSearcher`。
- 清洗符号关系，把一些无效的项目内引用剔除掉。

这个文件相当于项目级入口，负责把“很多源码文件”变成“可检索的语义图”。

### 1.2 `cfile_parse.py`

这是底层 AST 解析器，依赖 `clang.cindex` 与 `libclang`。它做的是最基础但最关键的工作：

- 自动尝试寻找 `libclang` 动态库。
- 解析 C 文件 AST。
- 提取并存储：
  - include
  - 全局变量
  - 局部变量
  - 函数定义/声明
  - 函数体
  - 结构体 / 联合体 / 枚举
  - typedef
  - docstring / 注释
  - 类型关系 `Typeof`
  - 赋值关系 `Assign`

最终输出是一个按符号名组织的结构化字典，供上层搜索和 prompt 生成使用。

### 1.3 `node_prompt.py`

这是项目级语义搜索器 `CProjectSearcher`，主要负责：

- 规范化 `parse_res` 的文件结构。
- 判断某个 include 是否是项目内头文件。
- 从某个符号出发，沿 `include` 和 `rels` 做 DFS。
- 把跨文件相关符号按近似拓扑顺序组织起来。
- 最终生成适合喂给模型的代码上下文片段。

可以把它理解为：

- `cfile_parse.py` 负责“抽取知识”
- `node_prompt.py` 负责“组织知识”

### 1.4 `generator.py`

这个模块更偏向离线 RAG prompt 构造，主要服务 `rag/main.py` 和评测流程。它会：

- 读取某个项目的已解析图文件。
- 从源码中抽取用户 include 的头文件。
- 找到对应头文件里的函数定义信息。
- 把这些上下文和原始代码拼接起来。
- 使用 `tokenizer.py` 控制长度，避免超过模型输入上限。

### 1.5 `tokenizer.py`

这是模型输入裁剪器，适配了多种代码模型与 GPT 系列，包括：

- CodeGen / CodeGen2.5
- SantaCoder
- StarCoder / StarCoder2
- CodeLlama
- DeepSeek-Coder
- Qwen2.5-Coder / Qwen3
- Llama3
- GPT-3.5 / GPT-4

核心职责是：

- 根据模型配置加载 tokenizer。
- 计算 token 长度。
- 估算 prompt 可用长度。
- 以“prompt + suffix + program”的形式做截断和拼接。

`tokenizer copy.py` 基本上是旧版本备份，保留了较早的实现，不属于当前主链路。

### 1.6 `main.py`

这是一个离线批处理入口，不是 UI agent 的主入口。它面向评测数据集，负责：

- 读取数据集 JSONL。
- 为每个样本调用 `CGenerator.retrieve_prompt()`。
- 支持超时控制、断点续跑、分批写出。

更像是“批量生成 prompt 数据”的脚本。

### 1.7 评测脚本

`rag/` 里有多份评测脚本，明显能看出这是一个持续试验中的研究型目录：

- `eval.py`
  使用 Hugging Face 模型做评测，对比 raw input 与 prompt-enhanced input。
- `evaluation.py`
  也是 HF 评测脚本，和 `eval.py` 高度相似，像是较早版本或保留版本。
- `eval_vllm.py`
  使用 vLLM 做双路评测。
- `eval_vllm3.py`
  三路评测：`raw`、`model prompt`、`langchain prompt`。

这些文件都不是在线 agent 的必要依赖，而是用于验证 prompt 增强是否真的提升代码补全效果。

### 1.8 其他辅助/实验文件

- `completion_prompt_agent_origin.py`
  旧版 prompt agent，保留了更早的补全逻辑。
- `json2neo4j.py`
  把解析结果导入 Neo4j，属于实验型脚本，而且路径和连接信息是硬编码的。
- `config.yaml`
  模型路径和 token 上限配置文件，里面包含大量本地路径，说明当前实现偏研究/本地实验环境。
- `utils.py`
  统一放了一批常量，例如数据集目录、结果输出目录、默认模型名、CUDA 可见卡配置等。
