## Feature Plugin 系统

**Advanced** 面板现在由插件架构驱动。每个功能都是 `plugins/` 下的一个 `FeaturePlugin`。前端根据每个插件的 `config_schema` 动态渲染表单，因此添加新功能**不需要**修改任何前端代码。

### 架构

- `plugins/base.py` — `FeaturePlugin` 抽象基类、`ExecutionMode`（`aider`/`direct`/`hybrid`）、`ConfigField` 表单定义、`ExecutionContext`。
- `plugins/registry.py` — `@register_plugin` 类装饰器；插件在导入时自动注册。
- `plugins/dispatcher.py` — 将执行路由到 AIDER、DIRECT 或 HYBRID 模式。
- `plugins/code_completion.py` — 原有的 `symbol`/`completion_type`/`prefix` 逻辑，已迁移为插件。

### 执行模式

| 模式 | 行为 | 示例 |
|------|------|------|
| `aider` | 生成 prompt → 调用 Aider → 修改代码文件 | 代码补全 |
| `direct` | 直接调用外部 API → 返回报告 / 写入文件 | 图片生成 HTML |
| `hybrid` | 通过 API 分析 → 生成修复 prompt → Aider 修复 | 漏洞检测 |

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