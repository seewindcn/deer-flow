# DeerFlow Models 模块技术文档

## 目录

1. [模块概述](#模块概述)
2. [目录结构](#目录结构)
3. [核心函数详解](#核心函数详解)
4. [支持的模型提供商](#支持的模型提供商)
5. [思考模式配置](#思考模式配置)
6. [DeepSeek 补丁](#deepseek-补丁)
7. [LangSmith 追踪](#langsmith-追踪)
8. [配置格式示例](#配置格式示例)

---

## 模块概述

DeerFlow Models 模块提供了一个统一的模型工厂系统，用于创建和管理各种大语言模型（LLM）实例。该模块基于 LangChain 的 `BaseChatModel` 抽象，支持多种主流模型提供商，并提供了以下核心功能：

- **统一接口**：通过 `create_chat_model` 函数提供一致的模型创建接口
- **动态解析**：支持运行时动态加载模型类，无需硬编码导入
- **思考模式**：原生支持模型的思考/推理能力配置
- **DeepSeek 优化**：提供补丁版本修复多轮对话中的推理内容保留问题
- **LangSmith 集成**：自动配置 LangSmith 追踪，便于调试和监控
- **灵活配置**：通过 YAML 配置文件管理模型参数，支持环境变量替换

### 架构图

```
┌─────────────────────────────────────────────────────────────────┐
│                        config.yaml                               │
│  ┌────────────────────────────────────────────────────────────┐  │
│  │ models:                                                     │  │
│  │   - name: gpt-4                                             │  │
│  │     use: langchain_openai:ChatOpenAI                        │  │
│  │     ...                                                     │  │
│  └────────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                    AppConfig (src/config)                       │
│  - 加载 YAML 配置                                                │
│  - 环境变量替换                                                   │
│  - 模型配置验证                                                   │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│              create_chat_model (src/models/factory.py)          │
│  - 解析模型类路径                                                 │
│  - 处理思考模式配置                                               │
│  - 应用 DeepSeek 补丁                                            │
│  - 附加 LangSmith 追踪                                           │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                    BaseChatModel 实例                            │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐           │
│  │ ChatOpenAI   │  │ChatAnthropic │  │PatchedDeepSeek│  ...      │
│  └──────────────┘  └──────────────┘  └──────────────┘           │
└─────────────────────────────────────────────────────────────────┘
```

---

## 目录结构

```
backend/src/models/
├── __init__.py              # 模块导出，对外暴露 create_chat_model
├── factory.py               # 模型工厂核心逻辑
└── patched_deepseek.py      # DeepSeek 补丁实现
```

### 文件说明

| 文件 | 职责 |
|------|------|
| `__init__.py` | 模块入口，导出 `create_chat_model` 函数 |
| `factory.py` | 模型工厂核心逻辑，负责创建和配置模型实例 |
| `patched_deepseek.py` | 修复 DeepSeek 模型在多轮对话中丢失推理内容的问题 |

---

## 核心函数详解

### create_chat_model

```python
def create_chat_model(
    name: str | None = None,
    thinking_enabled: bool = False,
    **kwargs
) -> BaseChatModel:
    """从配置创建聊天模型实例。

    Args:
        name: 要创建的模型名称。如果为 None，则使用配置中的第一个模型。
        thinking_enabled: 是否启用思考/推理模式。
        **kwargs: 额外的模型参数，将覆盖配置文件中的设置。

    Returns:
        配置好的 BaseChatModel 实例。

    Raises:
        ValueError: 如果指定的模型名称不存在于配置中。
        ValueError: 如果启用了思考模式但模型不支持。

    Example:
        >>> # 创建默认模型
        >>> model = create_chat_model()
        
        >>> # 创建指定模型
        >>> model = create_chat_model("gpt-4")
        
        >>> # 启用思考模式
        >>> model = create_chat_model("deepseek-v3", thinking_enabled=True)
        
        >>> # 覆盖配置参数
        >>> model = create_chat_model("gpt-4", temperature=0.5)
    """
```

### 处理流程

```
┌─────────────────────────────────────────────────────────────────┐
│ 1. 获取配置                                                      │
│    config = get_app_config()                                    │
│    model_config = config.get_model_config(name)                  │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│ 2. 解析模型类                                                    │
│    model_class = resolve_class(model_config.use, BaseChatModel) │
│    # 例如: "langchain_openai:ChatOpenAI" -> ChatOpenAI 类        │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│ 3. 准备模型参数                                                  │
│    - 提取配置文件中的参数 (model, api_key, max_tokens 等)          │
│    - 排除元数据字段 (name, display_name, description 等)          │
│    - 合并 thinking 快捷字段到 when_thinking_enabled                │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│ 4. 处理思考模式                                                  │
│    if thinking_enabled and supports_thinking:                   │
│        应用 when_thinking_enabled 配置                           │
│    elif not thinking_enabled and has_thinking_settings:          │
│        禁用思考模式 (设置 type: "disabled")                       │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│ 5. 创建模型实例                                                  │
│    model_instance = model_class(**kwargs, **model_settings)     │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│ 6. 附加 LangSmith 追踪 (可选)                                    │
│    if is_tracing_enabled():                                     │
│        tracer = LangChainTracer(project_name=...)               │
│        model_instance.callbacks.append(tracer)                   │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│ 7. 返回模型实例                                                  │
│    return model_instance                                        │
└─────────────────────────────────────────────────────────────────┘
```

### 关键代码解析

#### 参数提取与过滤

```python
# 从配置中提取模型参数，排除元数据字段
model_settings_from_config = model_config.model_dump(
    exclude_none=True,
    exclude={
        "use",           # 类路径
        "name",          # 模型标识名
        "display_name",  # 显示名称
        "description",  # 描述
        "supports_thinking",       # 能力声明
        "supports_reasoning_effort", # 能力声明
        "when_thinking_enabled",   # 思考模式配置
        "thinking",                # 思考快捷配置
        "supports_vision",         # 能力声明
    },
)
```

#### 思考模式配置合并

```python
# thinking 字段是 when_thinking_enabled["thinking"] 的快捷方式
# 两者可以同时使用，thinking 会覆盖 when_thinking_enabled["thinking"]
has_thinking_settings = (
    model_config.when_thinking_enabled is not None or
    model_config.thinking is not None
)

effective_wte: dict = dict(model_config.when_thinking_enabled) if model_config.when_thinking_enabled else {}
if model_config.thinking is not None:
    merged_thinking = {**(effective_wte.get("thinking") or {}), **model_config.thinking}
    effective_wte = {**effective_wte, "thinking": merged_thinking}
```

#### 禁用思考模式的处理

```python
# 当 thinking_enabled=False 但模型有思考配置时，显式禁用
if not thinking_enabled and has_thinking_settings:
    if effective_wte.get("extra_body", {}).get("thinking", {}).get("type"):
        # OpenAI 兼容网关：thinking 嵌套在 extra_body 下
        kwargs.update({"extra_body": {"thinking": {"type": "disabled"}}})
        kwargs.update({"reasoning_effort": "minimal"})
    elif effective_wte.get("thinking", {}).get("type"):
        # 原生 langchain_anthropic：thinking 是直接构造参数
        kwargs.update({"thinking": {"type": "disabled"}})
```

---

## 支持的模型提供商

### 提供商配置格式

模型配置使用 `use` 字段指定模型类路径，格式为：

```
package.module:ClassName
```

例如：
- `langchain_openai:ChatOpenAI` - OpenAI 模型
- `langchain_anthropic:ChatAnthropic` - Anthropic Claude 模型
- `langchain_google_genai:ChatGoogleGenerativeAI` - Google Gemini 模型
- `src.models.patched_deepseek:PatchedChatDeepSeek` - DeepSeek 补丁版本

### OpenAI

```yaml
- name: gpt-4
  display_name: GPT-4
  use: langchain_openai:ChatOpenAI
  model: gpt-4
  api_key: $OPENAI_API_KEY
  max_tokens: 4096
  temperature: 0.7
  supports_vision: true  # 启用视觉能力
```

**支持的额外参数**：
- `model` - 模型名称
- `api_key` - API 密钥（支持环境变量）
- `base_url` - API 基础 URL（用于自定义端点）
- `max_tokens` - 最大输出 token 数
- `temperature` - 采样温度
- `top_p` - 核采样参数
- `frequency_penalty` - 频率惩罚
- `presence_penalty` - 存在惩罚

### Anthropic Claude

```yaml
- name: claude-3-5-sonnet
  display_name: Claude 3.5 Sonnet
  use: langchain_anthropic:ChatAnthropic
  model: claude-3-5-sonnet-20241022
  api_key: $ANTHROPIC_API_KEY
  max_tokens: 8192
  supports_vision: true
  supports_thinking: true  # 支持思考模式
  when_thinking_enabled:
    thinking:
      type: enabled
```

**思考模式配置说明**：
Anthropic Claude 使用原生 `thinking` 参数，当 `thinking_enabled=True` 时，会传递：
```python
{"thinking": {"type": "enabled"}}
```

### Google Gemini

```yaml
- name: gemini-2.5-pro
  display_name: Gemini 2.5 Pro
  use: langchain_google_genai:ChatGoogleGenerativeAI
  model: gemini-2.5-pro
  google_api_key: $GOOGLE_API_KEY
  max_tokens: 8192
  supports_vision: true
```

**支持的额外参数**：
- `google_api_key` - Google API 密钥
- `model` - 模型名称

### DeepSeek

```yaml
- name: deepseek-v3
  display_name: DeepSeek V3 (Thinking)
  use: src.models.patched_deepseek:PatchedChatDeepSeek  # 使用补丁版本
  model: deepseek-reasoner
  api_key: $DEEPSEEK_API_KEY
  max_tokens: 16384
  supports_thinking: true
  supports_vision: false
  when_thinking_enabled:
    extra_body:
      thinking:
        type: enabled
```

**注意事项**：
- 推荐使用 `PatchedChatDeepSeek` 替代原生 `ChatDeepSeek`
- 思考模式配置使用 `extra_body.thinking` 嵌套结构

### OpenAI 兼容网关

支持任何 OpenAI 兼容的 API 网关：

```yaml
# Kimi K2.5
- name: kimi-k2.5
  display_name: Kimi K2.5
  use: src.models.patched_deepseek:PatchedChatDeepSeek
  model: kimi-k2.5
  api_base: https://api.moonshot.cn/v1
  api_key: $MOONSHOT_API_KEY
  max_tokens: 32768
  supports_thinking: true
  supports_vision: true
  when_thinking_enabled:
    extra_body:
      thinking:
        type: enabled

# 火山引擎 Doubao
- name: doubao-seed-1.8
  display_name: Doubao-Seed-1.8
  use: src.models.patched_deepseek:PatchedChatDeepSeek
  model: doubao-seed-1-8-251228
  api_base: https://ark.cn-beijing.volces.com/api/v3
  api_key: $VOLCENGINE_API_KEY
  supports_thinking: true
  supports_vision: true
  supports_reasoning_effort: true
  when_thinking_enabled:
    extra_body:
      thinking:
        type: enabled

# Novita AI
- name: novita-deepseek-v3.2
  display_name: Novita DeepSeek V3.2
  use: langchain_openai:ChatOpenAI
  model: deepseek/deepseek-v3.2
  api_key: $NOVITA_API_KEY
  base_url: https://api.novita.ai/openai
  max_tokens: 4096
  temperature: 0.7
  supports_thinking: true
  supports_vision: true
  when_thinking_enabled:
    extra_body:
      thinking:
        type: enabled
```

### 能力标志字段

| 字段 | 类型 | 说明 |
|------|------|------|
| `supports_thinking` | bool | 模型是否支持思考/推理模式 |
| `supports_vision` | bool | 模型是否支持图像输入 |
| `supports_reasoning_effort` | bool | 模型是否支持 reasoning_effort 参数 |

---

## 思考模式配置

### 配置结构

思考模式支持两种配置方式：

#### 方式一：`thinking` 快捷字段

```yaml
- name: my-model
  supports_thinking: true
  thinking:
    type: enabled
    budget_tokens: 5000
```

这是 `when_thinking_enabled.thinking` 的快捷方式。

#### 方式二：`when_thinking_enabled` 完整配置

```yaml
- name: my-model
  supports_thinking: true
  when_thinking_enabled:
    thinking:
      type: enabled
      budget_tokens: 5000
    extra_body:
      custom_param: value
```

### 配置合并规则

当同时使用 `thinking` 和 `when_thinking_enabled` 时：

```python
# 合并逻辑
effective_wte = {
    **when_thinking_enabled,  # 基础配置
    "thinking": {
        **when_thinking_enabled.get("thinking", {}),  # 原有 thinking 配置
        **thinking,  # thinking 字段覆盖
    }
}
```

**示例**：

```yaml
- name: deepseek-v3
  supports_thinking: true
  thinking:
    type: enabled
  when_thinking_enabled:
    extra_body:
      thinking:
        budget_tokens: 10000
  # 最终结果：
  # when_thinking_enabled:
  #   extra_body:
  #     thinking:
  #       budget_tokens: 10000
  #   thinking:
  #     type: enabled
```

### 不同提供商的配置差异

| 提供商 | 思考配置路径 | 示例 |
|--------|-------------|------|
| Anthropic | `thinking` (顶级参数) | `thinking: {type: enabled}` |
| OpenAI 兼容 | `extra_body.thinking` | `extra_body: {thinking: {type: enabled}}` |
| DeepSeek | `extra_body.thinking` | `extra_body: {thinking: {type: enabled}}` |

### 禁用思考模式

当 `thinking_enabled=False` 但模型配置了思考参数时，系统会自动禁用：

```python
# OpenAI 兼容网关
if effective_wte.get("extra_body", {}).get("thinking", {}).get("type"):
    kwargs.update({"extra_body": {"thinking": {"type": "disabled"}}})
    kwargs.update({"reasoning_effort": "minimal"})

# Anthropic
elif effective_wte.get("thinking", {}).get("type"):
    kwargs.update({"thinking": {"type": "disabled"}})
```

---

## DeepSeek 补丁

### 问题背景

DeepSeek 系列模型（包括其他兼容 API）在思考模式下会在响应中返回 `reasoning_content` 字段。在多轮对话中，当助手消息回传给 API 时，必须保留这个 `reasoning_content`，否则 API 会报错。

原生的 `langchain_deepseek.ChatDeepSeek` 将 `reasoning_content` 存储在 `additional_kwargs` 中，但在发送请求时不会将其包含在请求 payload 中。

### 解决方案

`PatchedChatDeepSeek` 通过重写 `_get_request_payload` 方法来解决这个问题：

```python
class PatchedChatDeepSeek(ChatDeepSeek):
    """修复 reasoning_content 保留问题的 ChatDeepSeek 版本。"""

    def _get_request_payload(
        self,
        input_: LanguageModelInput,
        *,
        stop: list[str] | None = None,
        **kwargs: Any,
    ) -> dict:
        """获取带有 reasoning_content 保留的请求 payload。"""
        # 获取原始消息
        original_messages = self._convert_input(input_).to_messages()

        # 调用父类获取基础 payload
        payload = super()._get_request_payload(input_, stop=stop, **kwargs)

        # 将 reasoning_content 从 additional_kwargs 恢复到 payload
        payload_messages = payload.get("messages", [])

        if len(payload_messages) == len(original_messages):
            # 按位置匹配
            for payload_msg, orig_msg in zip(payload_messages, original_messages):
                if payload_msg.get("role") == "assistant" and isinstance(orig_msg, AIMessage):
                    reasoning_content = orig_msg.additional_kwargs.get("reasoning_content")
                    if reasoning_content is not None:
                        payload_msg["reasoning_content"] = reasoning_content
        else:
            # 备用方案：按助手消息计数匹配
            ai_messages = [m for m in original_messages if isinstance(m, AIMessage)]
            assistant_payloads = [(i, m) for i, m in enumerate(payload_messages) if m.get("role") == "assistant"]

            for (idx, payload_msg), ai_msg in zip(assistant_payloads, ai_messages):
                reasoning_content = ai_msg.additional_kwargs.get("reasoning_content")
                if reasoning_content is not None:
                    payload_messages[idx]["reasoning_content"] = reasoning_content

        return payload
```

### 使用方式

在配置中使用补丁版本：

```yaml
- name: deepseek-v3
  use: src.models.patched_deepseek:PatchedChatDeepSeek  # 使用补丁版本
  model: deepseek-reasoner
  # ... 其他配置
```

### 适用范围

补丁版本适用于所有需要保留 `reasoning_content` 的 OpenAI 兼容 API：
- DeepSeek 官方 API
- Kimi (Moonshot)
- 火山引擎 Doubao
- 其他 OpenAI 兼容且支持思考模式的 API

---

## LangSmith 追踪

### 自动集成

当 LangSmith 追踪启用时，`create_chat_model` 会自动将 `LangChainTracer` 附加到模型实例：

```python
if is_tracing_enabled():
    try:
        from langchain_core.tracers.langchain import LangChainTracer

        tracing_config = get_tracing_config()
        tracer = LangChainTracer(
            project_name=tracing_config.project,
        )
        existing_callbacks = model_instance.callbacks or []
        model_instance.callbacks = [*existing_callbacks, tracer]
        logger.debug(f"LangSmith tracing attached to model '{name}' (project='{tracing_config.project}')")
    except Exception as e:
        logger.warning(f"Failed to attach LangSmith tracing to model '{name}': {e}")
```

### 配置方式

通过环境变量配置 LangSmith：

```bash
# 启用追踪
export LANGSMITH_TRACING=true

# API 密钥
export LANGSMITH_API_KEY=your-api-key

# 项目名称（可选，默认 "deer-flow"）
export LANGSMITH_PROJECT=my-project

# 端点（可选，默认官方端点）
export LANGSMITH_ENDPOINT=https://api.smith.langchain.com
```

### 环境变量优先级

| 配置项 | 优先级 |
|--------|--------|
| enabled | `LANGSMITH_TRACING` > `LANGCHAIN_TRACING_V2` > `LANGCHAIN_TRACING` |
| api_key | `LANGSMITH_API_KEY` > `LANGCHAIN_API_KEY` |
| project | `LANGSMITH_PROJECT` > `LANGCHAIN_PROJECT` |
| endpoint | `LANGSMITH_ENDPOINT` > `LANGCHAIN_ENDPOINT` |

### 追踪效果

启用后，所有模型调用将在 LangSmith 中显示：
- 输入消息
- 输出响应
- Token 使用量
- 延迟统计
- 错误信息

---

## 配置格式示例

### 基础配置

```yaml
models:
  - name: gpt-4
    display_name: GPT-4
    description: OpenAI GPT-4 model
    use: langchain_openai:ChatOpenAI
    model: gpt-4
    api_key: $OPENAI_API_KEY
    max_tokens: 4096
    temperature: 0.7
```

### 多模型配置

```yaml
models:
  # 默认模型（列表第一个）
  - name: gpt-4
    display_name: GPT-4
    use: langchain_openai:ChatOpenAI
    model: gpt-4
    api_key: $OPENAI_API_KEY

  # 思考模型
  - name: claude-thinking
    display_name: Claude Thinking
    use: langchain_anthropic:ChatAnthropic
    model: claude-3-5-sonnet-20241022
    api_key: $ANTHROPIC_API_KEY
    supports_thinking: true
    thinking:
      type: enabled

  # 视觉模型
  - name: gemini-vision
    display_name: Gemini Vision
    use: langchain_google_genai:ChatGoogleGenerativeAI
    model: gemini-2.5-pro
    google_api_key: $GOOGLE_API_KEY
    supports_vision: true
```

### 环境变量

配置文件支持环境变量替换，格式为 `$ENV_VAR_NAME`：

```yaml
api_key: $OPENAI_API_KEY          # 读取 OPENAI_API_KEY 环境变量
base_url: $CUSTOM_API_ENDPOINT    # 读取 CUSTOM_API_ENDPOINT 环境变量
```

### 完整配置示例

```yaml
models:
  # OpenAI GPT-4
  - name: gpt-4
    display_name: GPT-4
    description: OpenAI GPT-4 model for general tasks
    use: langchain_openai:ChatOpenAI
    model: gpt-4
    api_key: $OPENAI_API_KEY
    max_tokens: 4096
    temperature: 0.7
    supports_vision: true

  # Anthropic Claude with thinking
  - name: claude-3-5-sonnet
    display_name: Claude 3.5 Sonnet
    description: Anthropic Claude with extended thinking
    use: langchain_anthropic:ChatAnthropic
    model: claude-3-5-sonnet-20241022
    api_key: $ANTHROPIC_API_KEY
    max_tokens: 8192
    supports_vision: true
    supports_thinking: true
    thinking:
      type: enabled
      budget_tokens: 10000

  # DeepSeek with thinking (using patched version)
  - name: deepseek-v3
    display_name: DeepSeek V3
    description: DeepSeek reasoning model
    use: src.models.patched_deepseek:PatchedChatDeepSeek
    model: deepseek-reasoner
    api_key: $DEEPSEEK_API_KEY
    max_tokens: 16384
    supports_thinking: true
    supports_vision: false
    supports_reasoning_effort: true
    when_thinking_enabled:
      extra_body:
        thinking:
          type: enabled

  # Google Gemini
  - name: gemini-2.5-pro
    display_name: Gemini 2.5 Pro
    description: Google's latest Gemini model
    use: langchain_google_genai:ChatGoogleGenerativeAI
    model: gemini-2.5-pro
    google_api_key: $GOOGLE_API_KEY
    max_tokens: 8192
    supports_vision: true
```

### ModelConfig 字段说明

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `name` | str | 是 | 模型的唯一标识符，用于代码中引用 |
| `display_name` | str \| None | 否 | 模型的显示名称，用于 UI 展示 |
| `description` | str \| None | 否 | 模型描述 |
| `use` | str | 是 | 模型类路径，格式为 `package:ClassName` |
| `model` | str | 是 | 实际调用的模型名称 |
| `supports_thinking` | bool | 否 | 是否支持思考模式，默认 false |
| `supports_vision` | bool | 否 | 是否支持图像输入，默认 false |
| `supports_reasoning_effort` | bool | 否 | 是否支持 reasoning_effort 参数，默认 false |
| `when_thinking_enabled` | dict \| None | 否 | 启用思考模式时的额外配置 |
| `thinking` | dict \| None | 否 | 思考配置快捷方式，等同于 `when_thinking_enabled.thinking` |
| `*` | Any | 否 | 其他参数将直接传递给模型构造函数 |

---

## 最佳实践

### 1. 使用环境变量管理密钥

```yaml
# 推荐
api_key: $OPENAI_API_KEY

# 不推荐（硬编码）
api_key: sk-xxxx
```

### 2. 为生产环境配置 LangSmith 追踪

```bash
export LANGSMITH_TRACING=true
export LANGSMITH_API_KEY=your-api-key
export LANGSMITH_PROJECT=deer-flow-production
```

### 3. 思考模型配置建议

```yaml
# 推荐：使用 thinking 快捷字段
- name: claude-thinking
  supports_thinking: true
  thinking:
    type: enabled

# 也可以：使用完整配置
- name: deepseek-v3
  supports_thinking: true
  when_thinking_enabled:
    extra_body:
      thinking:
        type: enabled
```

### 4. 多模型按能力组织

```yaml
models:
  # 快速模型（默认）
  - name: gpt-4o-mini
    # ...

  # 深度思考模型
  - name: claude-thinking
    supports_thinking: true
    # ...

  # 视觉模型
  - name: gemini-vision
    supports_vision: true
    # ...
```

### 5. 使用补丁版本处理 DeepSeek 兼容 API

对于所有需要保留 `reasoning_content` 的 API，使用 `PatchedChatDeepSeek`：

```yaml
- name: kimi-thinking
  use: src.models.patched_deepseek:PatchedChatDeepSeek
  # ...
```