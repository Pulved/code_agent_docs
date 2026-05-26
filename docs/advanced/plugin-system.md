## Feature Plugin 系统

**Advanced** 面板现在由插件架构驱动。每个功能都是 `plugins/` 下的一个 `FeaturePlugin`。前端根据每个插件的 `config_schema` 动态渲染表单，因此添加新功能**不需要**修改任何前端代码。

### 架构

- `plugins/base.py` — `FeaturePlugin` 抽象基类、`ExecutionMode`（`aider`/`direct`/`hybrid`）、`ConfigField` 表单定义、`ExecutionContext`。
- `plugins/registry.py` — `@register_plugin` 类装饰器；插件在导入时自动注册。
- `plugins/dispatcher.py` — 将执行路由到 AIDER、DIRECT 或 HYBRID 模式。
- `plugins/code_completion.py` — 原有的 `symbol`/`completion_type`/`prefix` 逻辑，已迁移为插件。
- `plugins/code_summary.py` — NaturalCC + Aider dry-run 代码总结。
- `plugins/code_repair.py` — AIDER 模式的代码修复提示词，用于 bug、编译错误和测试失败。
- `plugins/vulnerability_detection.py` — 漏洞分析插件，支持可选的 Aider 自动修复。

### 执行模式

| 模式 | 行为 | 示例 |
|------|------|------|
| `aider` | 生成 prompt → 调用 Aider → 修改代码文件或输出 dry-run 报告 | 代码补全、代码修复、代码总结 |
| `direct` | 直接分析 → 返回报告 / 写入文件 | 静态报告 |
| `hybrid` | 通过 API 分析 → 生成修复 prompt → Aider 修复 | 漏洞检测 |

### 内置代码总结功能

功能名：`code_summary`（AIDER 模式）

执行方式：
- 对选中的目标文件，或项目下的源码文件，构造正常的 NaturalCC 语义 prompt。
- 使用 `--dry-run` 调用 Aider，因此总结过程不会修改文件。
- 使用所选模型生成更深入的代码理解报告。

主要配置项：
- `summary_scope`：`targets`（仅目标文件）或 `project`（全项目源码）
- `detail_level`：`brief` / `standard` / `detailed`
- `include_symbols`：要求 Aider 包含关键符号和数据流
- `max_files`：发送给 NaturalCC 和 Aider 的文件数量上限

### NaturalCC / libclang 版本对齐

NaturalCC 要求 Python `clang` bindings 与系统安装的 `libclang` 版本匹配。本项目固定 `clang==18.1.8`，对应 Ubuntu LLVM 18 / `libclang1-18` 系列。如果系统使用其他 LLVM 主版本，需要把 `clang` 依赖和锁文件调整为与 `libclang.so` 相同的主版本。

### 内置代码修复功能

功能名：`code_repair`（AIDER 模式）

执行方式：
- 根据用户指令、修复类型、可选错误日志和可选额外上下文生成聚焦修复提示词。
- 复用现有 NaturalCC 语义 prompt 路径，然后交给 Aider 修改目标文件。
- 默认偏向最小修复，并尽量保持现有接口不变。

主要配置项：
- `repair_type`：`bug_fix` / `compile_error` / `test_failure` / `safe_refactor`
- `failure_log`：编译错误、测试失败、堆栈或运行时报错
- `extra_context`：约束、期望行为或复现说明
- `allow_refactor`：必要时允许小范围辅助重构

### 内置漏洞检测功能

功能名：`vulnerability_detection`（HYBRID 模式）

执行方式：
- 阶段 1：进行基于规则的静态漏洞扫描并生成报告。
- 阶段 2（可选）：当 `auto_fix=true` 时，生成修复指令并调用 Aider 对目标文件进行修复。

主要配置项：
- `scan_scope`：`targets`（仅目标文件）或 `project`（全项目）
- `severity_threshold`：`low` / `medium` / `high` / `critical`
- `rule_profile`：`default` / `c_cpp` / `web`
- `auto_fix`：是否执行自动修复阶段
- `max_findings`：报告中最大告警条数
- `extra_instruction`：额外修复约束

使用建议：
- 如果要开启 `auto_fix`，先选择好目标文件。
- 建议先用 `auto_fix=false` 查看扫描结果，再决定是否自动修复。

### 如何添加新功能插件

1. 在 `plugins/` 下创建新文件，例如 `plugins/my_feature.py`。
2. 继承 `FeaturePlugin`，实现 `metadata`、`config_schema` 和 `execute`。
3. 用 `@register_plugin` 装饰类。
4. 重启后端。前端会自动显示新功能并渲染其表单。

示例：

```python
# plugins/my_feature.py
from typing import Any, Dict, Generator, List, Optional
from code_agent.plugins.base import (
    FeaturePlugin, FeatureMetadata, ExecutionMode,
    ConfigField, ConfigFieldType, ExecutionContext, PluginResult,
)
from code_agent.plugins.registry import register_plugin


@register_plugin
class MyFeaturePlugin(FeaturePlugin):

    @property
    def metadata(self) -> FeatureMetadata:
        return FeatureMetadata(
            name="my_feature",           # 唯一标识
            label="My Feature",          # 显示名称
            description="功能描述",
            execution_mode=ExecutionMode.DIRECT,  # 或 AIDER / HYBRID
        )

    @property
    def config_schema(self) -> List[ConfigField]:
        return [
            ConfigField(
                name="my_param",
                label="My Parameter",
                type=ConfigFieldType.TEXT,   # text / textarea / select / switch / file
                required=True,
                default="",
                placeholder="输入值",
                help_text="显示在字段下方的帮助文本",
            ),
        ]

    def execute(self, context: ExecutionContext) -> Generator[str, None, None]:
        # yield 字符串作为日志输出
        yield "开始执行...\n"
        # ... 你的业务逻辑 ...
        # 完成后 yield PluginResult（用于 DIRECT / HYBRID 模式）
        yield PluginResult(success=True, message="完成！")
```

### 配置字段类型

| 类型 | 渲染为 | 额外属性 |
|------|--------|---------|
| `text` | `<input type="text">` | `placeholder`, `default` |
| `textarea` | `<textarea>` | `placeholder`, `default` |
| `select` | `<select>` | `options: [{value, label}]`, `default` |
| `switch` | `<input type="checkbox">` | `default` (bool) |
| `file` | `<input type="file">` | `accept`, `multiple` |

### 文件上传

如果插件配置包含 `file` 类型字段，前端会自动以 `multipart/form-data` 发送请求。上传的文件在 `context.uploaded_files` 中以 `{field_name: UploadFile}` 的形式提供。

### 插件相关的 API 变更

`/api/bootstrap` 现在返回：

```json
{
  "features": [{"name": "...", "label": "...", "execution_mode": "..."}],
  "schemas": {"feature_name": [{"name": "...", "type": "...", ...}]},
  "default_feature": "code_completion"
}
```

`/api/run` 同时接受 JSON（向后兼容）和 `multipart/form-data`（用于文件上传）。请求体应包含：

```json
{
  "feature": "my_feature",
  "feature_config": {"my_param": "value"}
}
```