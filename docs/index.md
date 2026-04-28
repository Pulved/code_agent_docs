# NaturalCC Code Agent

[English README](readme-en.md)

`code_agent` 是一个本地代码编辑代理，把静态项目理解能力和 Aider 的文件修改能力组合在一起。

它会先解析目标项目，收集函数、变量、类型、成员、include 和符号关系；再根据用户任务生成语义增强 prompt；最后把 prompt 交给 Aider，由 Aider 修改选中的目标文件。

项目提供两种使用方式：

- 图形界面：FastAPI 后端 + React/Vite 前端。
- 命令行：`aider_runner.py`。

注意：第一个目标文件始终是 NaturalCC 构造 prompt 时使用的主解析文件。Aider 仍然可以接收并修改多个目标文件。

## 项目是什么

这不是通用聊天机器人，而是面向项目上下文的代码补全和代码修改代理。

典型任务：

- 补全函数体
- 补全函数签名
- 补全变量、成员或类型
- 按项目现有风格做小范围代码修改
- 在执行 Aider 前预览最终语义 prompt

主要文件：

- `agent_web_api.py`：FastAPI 后端，同时可服务打包后的前端。
- `webui/`：React + Vite 图形界面。
- `aider_runner.py`：CLI 入口和 Aider 调度逻辑。
- `completion_prompt_agent.py`：语义 prompt 构造逻辑。
- `rag/c/`：C/C++ 解析和上下文检索。
- `rag/java/`：Java 解析和 prompt 路径。
- `test_api.py`：OpenRouter key / 连通性检查脚本。

## 注意事项和限制

- C/C++ 解析依赖 `libclang`。
- 部分解析路径仍使用偏 C 语言的 libclang 参数，C++ 语法覆盖可能不完整。
- `rag/` 中包含离线研究和评测脚本，其中部分脚本带有本地路径假设。
- 当前项目主要依赖 smoke check，还没有正式自动化测试体系。
- `test_api.py` 只用于 API 连通性检查，不是 parser 或 UI 测试。