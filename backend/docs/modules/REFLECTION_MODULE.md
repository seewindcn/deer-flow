# Reflection 模块技术文档

> 本文档详细介绍 DeerFlow 的反射模块，包括变量解析、类解析、模块映射提示等核心功能。

## 目录

- [模块概述](#模块概述)
- [目录结构](#目录结构)
- [变量解析](#变量解析)
- [类解析](#类解析)
- [模块映射提示](#模块映射提示)
- [错误处理](#错误处理)
- [使用示例](#使用示例)
- [最佳实践](#最佳实践)

---

## 模块概述

### 职责和功能定位

`src.reflection` 模块是 DeerFlow 框架的核心反射工具系统，负责：

1. **动态变量解析**：通过字符串路径动态导入并获取模块中的变量、函数或对象
2. **动态类解析**：通过字符串路径动态导入并验证类类型
3. **类型安全验证**：在解析时验证变量类型和类的继承关系
4. **友好的错误提示**：提供可操作的依赖安装提示

### 与其他模块的关系

```
┌─────────────────────────────────────────────────────────────────┐
│                       reflection 模块                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌──────────────────┐         ┌──────────────────┐            │
│  │   resolvers.py   │         │    __init__.py   │            │
│  │  (核心解析逻辑)   │◄────────┤    (公共 API)    │            │
│  └──────────────────┘         └──────────────────┘            │
│           │                                                     │
│           │                                                     │
│           ▼                                                     │
│  ┌──────────────────────────────────────────────────┐          │
│  │           MODULE_TO_PACKAGE_HINTS                │          │
│  │    (依赖缺失时的友好提示映射)                       │          │
│  └──────────────────────────────────────────────────┘          │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
                        │
                        │ 被以下模块使用
                        ▼
┌─────────────────────────────────────────────────────────────────┐
│                        使用方模块                                │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌──────────────────┐  ┌──────────────────┐  ┌──────────────┐ │
│  │  models/factory  │  │  tools/tools.py  │  │  sandbox/    │ │
│  │  (模型类解析)     │  │  (工具实例解析)   │  │  (沙箱提供者) │ │
│  └──────────────────┘  └──────────────────┘  └──────────────┘ │
│                                                                 │
│  ┌──────────────────┐                                          │
│  │ channels/service │                                          │
│  │  (通道类解析)     │                                          │
│  └──────────────────┘                                          │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**依赖关系**：

| 使用方模块 | 使用函数 | 用途 |
|-----------|---------|------|
| `src.models.factory` | `resolve_class` | 解析模型类（如 `ChatOpenAI`） |
| `src.tools.tools` | `resolve_variable` | 解析工具实例（如 `bash_tool`） |
| `src.sandbox.sandbox_provider` | `resolve_class` | 解析沙箱提供者类 |
| `src.channels.service` | `resolve_class` | 解析通道实现类 |

---

## 目录结构

```
backend/src/reflection/
├── __init__.py              # 模块入口，导出公共 API
└── resolvers.py             # 核心解析器实现
```

### 文件说明

| 文件 | 职责 |
|------|------|
| `__init__.py` | 模块公共 API 导出，暴露 `resolve_class` 和 `resolve_variable` 函数 |
| `resolvers.py` | 核心解析逻辑实现，包含类型验证和错误处理 |

---

## 变量解析

### resolve_variable() 函数

从模块路径动态解析变量（可以是函数、类实例、常量等任何 Python 对象）。

#### 函数签名

```python
def resolve_variable[T](
    variable_path: str,
    expected_type: type[T] | tuple[type, ...] | None = None,
) -> T:
    """Resolve a variable from a path.

    Args:
        variable_path: The path to the variable (e.g. "parent_package_name.sub_package_name.module_name:variable_name").
        expected_type: Optional type or tuple of types to validate the resolved variable against.
            If provided, uses isinstance() to check if the variable is an instance of the expected type(s).

    Returns:
        The resolved variable.

    Raises:
        ImportError: If the module path is invalid or the attribute doesn't exist.
        ValueError: If the resolved variable doesn't pass the validation checks.
    """
```

### 路径格式说明

变量路径使用冒号分隔符格式：

```
module.path:variable_name
    │          │
    │          └── 变量名（函数、类实例、常量等）
    │
    └──────────── 模块导入路径
```

**示例**：

```
src.sandbox.tools:bash_tool              # 解析 src.sandbox.tools 模块中的 bash_tool 变量
langchain_openai:ChatOpenAI              # 解析 langchain_openai 模块中的 ChatOpenAI 类
src.community.tavily.tools:web_search    # 解析自定义模块中的工具
```

### 类型验证

`resolve_variable` 支持可选的类型验证：

#### 单一类型验证

```python
from langchain.tools import BaseTool
from src.reflection import resolve_variable

# 验证解析结果是 BaseTool 的实例
tool = resolve_variable(
    "src.sandbox.tools:bash_tool",
    expected_type=BaseTool
)
```

#### 多类型验证

```python
# 验证解析结果是多种类型之一
result = resolve_variable(
    "some.module:variable",
    expected_type=(str, int, float)  # 允许 str、int 或 float
)
```

#### 无类型验证

```python
# 不进行类型验证，返回任意类型
variable = resolve_variable("some.module:any_variable")
```

### 解析流程

```
┌─────────────────────────────────────────────────────────────────┐
│                    resolve_variable(path, type)                  │
└─────────────────────────────────────────────────────────────────┘
                               │
                               ▼
┌─────────────────────────────────────────────────────────────────┐
│  1. 解析路径                                                      │
│     - 使用 rsplit(":", 1) 分离模块路径和变量名                     │
│     - 格式验证：必须有且仅有一个冒号                                │
└─────────────────────────────────────────────────────────────────┘
                               │
                               ▼
┌─────────────────────────────────────────────────────────────────┐
│  2. 导入模块                                                      │
│     - 使用 import_module() 动态导入                                │
│     - 捕获 ImportError 并生成友好提示                               │
└─────────────────────────────────────────────────────────────────┘
                               │
                               ▼
┌─────────────────────────────────────────────────────────────────┐
│  3. 获取变量                                                      │
│     - 使用 getattr() 获取模块属性                                  │
│     - 属性不存在时抛出 ImportError                                 │
└─────────────────────────────────────────────────────────────────┘
                               │
                               ▼
┌─────────────────────────────────────────────────────────────────┐
│  4. 类型验证（可选）                                               │
│     - 如果提供了 expected_type，使用 isinstance() 验证            │
│     - 验证失败时抛出 ValueError                                    │
└─────────────────────────────────────────────────────────────────┘
                               │
                               ▼
┌─────────────────────────────────────────────────────────────────┐
│  5. 返回变量                                                      │
└─────────────────────────────────────────────────────────────────┘
```

---

## 类解析

### resolve_class() 函数

从模块路径动态解析类并进行类型验证。

#### 函数签名

```python
def resolve_class[T](class_path: str, base_class: type[T] | None = None) -> type[T]:
    """Resolve a class from a module path and class name.

    Args:
        class_path: The path to the class (e.g. "langchain_openai:ChatOpenAI").
        base_class: The base class to check if the resolved class is a subclass of.

    Returns:
        The resolved class.

    Raises:
        ImportError: If the module path is invalid or the attribute doesn't exist.
        ValueError: If the resolved object is not a class or not a subclass of base_class.
    """
```

### 路径格式说明

类路径使用相同的冒号分隔符格式：

```
module.path:ClassName
     │          │
     │          └── 类名（大驼峰命名）
     │
     └──────────── 模块导入路径
```

**示例**：

```
langchain_openai:ChatOpenAI              # OpenAI 聊天模型类
langchain_anthropic:ChatAnthropic        # Anthropic 聊天模型类
langchain_google_genai:ChatGoogleGenerativeAI  # Google 生成式 AI 类
src.sandbox.providers:LocalSandboxProvider      # 本地沙箱提供者类
```

### 基类验证

`resolve_class` 支持可选的基类验证：

#### 验证类继承关系

```python
from langchain.chat_models import BaseChatModel
from src.reflection import resolve_class

# 解析并验证是 BaseChatModel 的子类
model_class = resolve_class(
    "langchain_openai:ChatOpenAI",
    base_class=BaseChatModel
)

# 可以直接实例化
model = model_class(model="gpt-4", temperature=0.7)
```

#### 无基类验证

```python
# 仅验证是类类型，不检查继承关系
some_class = resolve_class("some.module:SomeClass")

# 验证是类但 base_class=None 效果相同
some_class = resolve_class("some.module:SomeClass", base_class=None)
```

### 解析流程

```
┌─────────────────────────────────────────────────────────────────┐
│                    resolve_class(path, base)                     │
└─────────────────────────────────────────────────────────────────┘
                               │
                               ▼
┌─────────────────────────────────────────────────────────────────┐
│  1. 调用 resolve_variable(path, expected_type=type)              │
│     - 复用变量解析逻辑                                             │
│     - 验证解析结果是 type 类型（即是一个类）                        │
└─────────────────────────────────────────────────────────────────┘
                               │
                               ▼
┌─────────────────────────────────────────────────────────────────┐
│  2. 验证类类型                                                    │
│     - 确保解析结果是 type 实例（是一个类，不是实例）                 │
│     - 不是类时抛出 ValueError                                     │
└─────────────────────────────────────────────────────────────────┘
                               │
                               ▼
┌─────────────────────────────────────────────────────────────────┐
│  3. 基类验证（可选）                                               │
│     - 如果提供了 base_class，使用 issubclass() 验证继承关系         │
│     - 验证失败时抛出 ValueError                                    │
└─────────────────────────────────────────────────────────────────┘
                               │
                               ▼
┌─────────────────────────────────────────────────────────────────┐
│  4. 返回类对象                                                    │
└─────────────────────────────────────────────────────────────────┘
```

### resolve_class 与 resolve_variable 的区别

| 特性 | `resolve_variable` | `resolve_class` |
|------|-------------------|-----------------|
| 返回类型 | 任意对象（T） | 类对象（type[T]） |
| 类型验证 | `isinstance(obj, expected_type)` | `isinstance(cls, type)` + `issubclass(cls, base_class)` |
| 用途 | 解析实例、函数、常量等 | 解析可实例化的类 |
| 验证时机 | 解析后验证 | 解析后双重验证（类类型 + 继承关系） |

---

## 模块映射提示

### MODULE_TO_PACKAGE_HINTS 映射表

当模块导入失败时，提供友好的依赖安装提示。

```python
# backend/src/reflection/resolvers.py

MODULE_TO_PACKAGE_HINTS = {
    "langchain_google_genai": "langchain-google-genai",
    "langchain_anthropic": "langchain-anthropic",
    "langchain_openai": "langchain-openai",
    "langchain_deepseek": "langchain-deepseek",
}
```

### 映射规则

| 模块名 | PyPI 包名 | 说明 |
|-------|----------|------|
| `langchain_google_genai` | `langchain-google-genai` | Google Generative AI 集成 |
| `langchain_anthropic` | `langchain-anthropic` | Anthropic Claude 集成 |
| `langchain_openai` | `langchain-openai` | OpenAI GPT 集成 |
| `langchain_deepseek` | `langchain-deepseek` | DeepSeek 模型集成 |

### _build_missing_dependency_hint() 函数

构建缺失依赖的友好提示信息。

```python
def _build_missing_dependency_hint(module_path: str, err: ImportError) -> str:
    """Build an actionable hint when module import fails.
    
    Args:
        module_path: 模块路径（如 "langchain_openai.chat_models"）
        err: 原始导入错误
        
    Returns:
        友好的错误提示信息，包含安装命令
    """
```

#### 提示生成逻辑

```
┌─────────────────────────────────────────────────────────────────┐
│           _build_missing_dependency_hint(module_path, err)        │
└─────────────────────────────────────────────────────────────────┘
                               │
                               ▼
┌─────────────────────────────────────────────────────────────────┐
│  1. 提取模块根名                                                  │
│     module_root = module_path.split(".", 1)[0]                   │
│     例如: "langchain_openai.chat_models" → "langchain_openai"     │
└─────────────────────────────────────────────────────────────────┘
                               │
                               ▼
┌─────────────────────────────────────────────────────────────────┐
│  2. 确定缺失模块名                                                │
│     missing_module = err.name 或 module_root                     │
│     优先使用错误中的模块名                                         │
└─────────────────────────────────────────────────────────────────┘
                               │
                               ▼
┌─────────────────────────────────────────────────────────────────┐
│  3. 查找包名映射                                                  │
│     - 先查找 MODULE_TO_PACKAGE_HINTS[module_root]                │
│     - 未找到则查找 MODULE_TO_PACKAGE_HINTS[missing_module]       │
│     - 仍未找到则使用 missing_module.replace("_", "-")             │
└─────────────────────────────────────────────────────────────────┘
                               │
                               ▼
┌─────────────────────────────────────────────────────────────────┐
│  4. 生成提示信息                                                  │
│     f"Missing dependency '{missing_module}'. Install it with      │
│      `uv add {package_name}` (or `pip install {package_name}`),   │
│      then restart DeerFlow."                                      │
└─────────────────────────────────────────────────────────────────┘
```

#### 示例输出

```python
# 场景1: langchain_openai 缺失
>>> _build_missing_dependency_hint("langchain_openai", ImportError(...))
"Missing dependency 'langchain_openai'. Install it with `uv add langchain-openai` (or `pip install langchain-openai`), then restart DeerFlow."

# 场景2: 未知模块
>>> _build_missing_dependency_hint("custom_module", ImportError(...))
"Missing dependency 'custom-module'. Install it with `uv add custom-module` (or `pip install custom-module`), then restart DeerFlow."
```

---

## 错误处理

### 导入错误处理

#### 路径格式错误

```python
from src.reflection import resolve_variable

# 错误: 缺少冒号
try:
    resolve_variable("langchain_openai.ChatOpenAI")  # 错误格式
except ImportError as e:
    # 错误信息:
    # "langchain_openai.ChatOpenAI doesn't look like a variable path. 
    #  Example: parent_package_name.sub_package_name.module_name:variable_name"
    pass
```

#### 模块不存在

```python
# 错误: 模块不存在
try:
    resolve_variable("nonexistent_module:SomeClass")
except ImportError as e:
    # 错误信息:
    # "Could not import module nonexistent_module. 
    #  Missing dependency 'nonexistent-module'. Install it with `uv add nonexistent-module`..."
    pass
```

#### 属性不存在

```python
# 错误: 模块中不存在指定属性
try:
    resolve_variable("langchain_openai:NonExistentClass")
except ImportError as e:
    # 错误信息:
    # "Module langchain_openai does not define a NonExistentClass attribute/class"
    pass
```

### 类型错误处理

#### 类型验证失败

```python
from langchain.tools import BaseTool
from src.reflection import resolve_variable

# 错误: 类型不匹配
try:
    resolve_variable("langchain_openai:ChatOpenAI", expected_type=BaseTool)
except ValueError as e:
    # 错误信息:
    # "langchain_openai:ChatOpenAI is not an instance of BaseTool, got type"
    pass
```

#### 基类验证失败

```python
from langchain.chat_models import BaseChatModel
from src.reflection import resolve_class

# 错误: 不是指定基类的子类
try:
    resolve_class("some.module:SomeClass", base_class=BaseChatModel)
except ValueError as e:
    # 错误信息:
    # "some.module:SomeClass is not a subclass of BaseChatModel"
    pass
```

#### 不是类类型

```python
# 错误: 解析结果不是类
try:
    resolve_class("src.sandbox.tools:bash_tool")  # bash_tool 是实例，不是类
except ValueError as e:
    # 错误信息:
    # "src.sandbox.tools:bash_tool is not a valid class"
    pass
```

### 错误处理最佳实践

```python
from src.reflection import resolve_class, resolve_variable
from langchain.chat_models import BaseChatModel
from langchain.tools import BaseTool

def safe_resolve_model(model_path: str) -> BaseChatModel | None:
    """安全解析模型类，返回 None 而非抛出异常。"""
    try:
        return resolve_class(model_path, BaseChatModel)
    except ImportError as e:
        logger.error(f"无法导入模型类: {e}")
        return None
    except ValueError as e:
        logger.error(f"类型验证失败: {e}")
        return None

def safe_resolve_tool(tool_path: str) -> BaseTool | None:
    """安全解析工具实例。"""
    try:
        return resolve_variable(tool_path, BaseTool)
    except ImportError as e:
        logger.error(f"无法导入工具: {e}")
        return None
    except ValueError as e:
        logger.error(f"工具类型不匹配: {e}")
        return None
```

---

## 使用示例

### 解析模型类

在模型工厂中使用 `resolve_class` 解析聊天模型：

```python
# backend/src/models/factory.py

from langchain.chat_models import BaseChatModel
from src.reflection import resolve_class

def create_chat_model(name: str | None = None, **kwargs) -> BaseChatModel:
    """从配置创建聊天模型实例。
    
    Args:
        name: 模型名称，None 则使用第一个配置的模型
        **kwargs: 传递给模型构造函数的额外参数
        
    Returns:
        BaseChatModel 实例
    """
    config = get_app_config()
    
    # 获取模型配置
    if name is None:
        name = config.models[0].name
    model_config = config.get_model_config(name)
    if model_config is None:
        raise ValueError(f"Model {name} not found in config")
    
    # 解析模型类
    # model_config.use 格式如: "langchain_openai:ChatOpenAI"
    model_class = resolve_class(model_config.use, BaseChatModel)
    
    # 准备模型参数
    model_settings = model_config.model_dump(
        exclude_none=True,
        exclude={"use", "name", "display_name", "description", ...}
    )
    
    # 实例化模型
    return model_class(**kwargs, **model_settings)
```

**配置示例**：

```yaml
# config.yaml
models:
  - name: gpt-4
    display_name: GPT-4
    use: langchain_openai:ChatOpenAI  # 模块路径:类名
    model: gpt-4
    temperature: 0.7
    
  - name: claude-3
    display_name: Claude 3
    use: langchain_anthropic:ChatAnthropic
    model: claude-3-opus-20240229
    temperature: 0.7
    
  - name: gemini-pro
    display_name: Gemini Pro
    use: langchain_google_genai:ChatGoogleGenerativeAI
    model: gemini-pro
```

### 解析工具函数

在工具加载系统中使用 `resolve_variable` 解析工具实例：

```python
# backend/src/tools/tools.py

from langchain.tools import BaseTool
from src.reflection import resolve_variable

def get_available_tools(groups: list[str] | None = None) -> list[BaseTool]:
    """获取所有可用工具。
    
    Args:
        groups: 工具组过滤列表，None 表示加载所有工具
        
    Returns:
        工具实例列表
    """
    config = get_app_config()
    
    # 从配置加载工具
    # tool.use 格式如: "src.sandbox.tools:bash_tool"
    loaded_tools = [
        resolve_variable(tool.use, BaseTool)
        for tool in config.tools
        if groups is None or tool.group in groups
    ]
    
    return loaded_tools
```

**配置示例**：

```yaml
# config.yaml
tools:
  - name: bash
    group: bash
    use: src.sandbox.tools:bash_tool  # 模块路径:工具变量名
    
  - name: read_file
    group: file:read
    use: src.sandbox.tools:read_file_tool
    
  - name: web_search
    group: web
    use: src.community.tavily.tools:web_search_tool
    max_results: 5
```

### 解析沙箱提供者

在沙箱系统中使用 `resolve_class` 解析沙箱提供者类：

```python
# backend/src/sandbox/sandbox_provider.py

from src.reflection import resolve_class
from src.sandbox.sandbox import Sandbox

class SandboxProvider(ABC):
    """沙箱提供者抽象基类"""
    
    @abstractmethod
    def acquire(self, thread_id: str | None = None) -> str:
        """获取沙箱环境"""
        pass
    
    @abstractmethod
    def get(self, sandbox_id: str) -> Sandbox | None:
        """获取沙箱实例"""
        pass
    
    @abstractmethod
    def release(self, sandbox_id: str) -> None:
        """释放沙箱环境"""
        pass


def get_sandbox_provider(**kwargs) -> SandboxProvider:
    """获取沙箱提供者单例。"""
    global _default_sandbox_provider
    
    if _default_sandbox_provider is None:
        config = get_app_config()
        
        # 解析沙箱提供者类
        # config.sandbox.use 格式如: "src.sandbox.providers:LocalSandboxProvider"
        cls = resolve_class(config.sandbox.use, SandboxProvider)
        
        # 实例化提供者
        _default_sandbox_provider = cls(**kwargs)
    
    return _default_sandbox_provider
```

**配置示例**：

```yaml
# config.yaml
sandbox:
  use: src.sandbox.providers:LocalSandboxProvider
  # 或使用 Docker:
  # use: src.sandbox.providers:DockerSandboxProvider
```

### 解析通道实现

在通道服务中使用 `resolve_class` 动态加载通道类：

```python
# backend/src/channels/service.py

from src.reflection import resolve_class

async def create_channel(import_path: str, config: dict) -> Channel:
    """动态创建通道实例。
    
    Args:
        import_path: 类路径（如 "src.channels.impl:WebSocketChannel"）
        config: 通道配置
        
    Returns:
        通道实例
    """
    # 解析通道类（不验证基类，允许灵活实现）
    channel_cls = resolve_class(import_path, base_class=None)
    
    # 实例化
    return channel_cls(**config)
```

### 高级用法

#### 多类型验证

```python
from src.reflection import resolve_variable

# 验证可以是多种类型之一
processor = resolve_variable(
    "myapp.processors:default_processor",
    expected_type=(str, int, Callable)  # 字符串、整数或可调用对象
)
```

#### 泛型类型支持

```python
from typing import TypeVar
from src.reflection import resolve_class

T = TypeVar("T")

def create_instance(class_path: str, *args, **kwargs) -> T:
    """通用的实例创建辅助函数。"""
    cls = resolve_class(class_path)
    return cls(*args, **kwargs)

# 使用
model = create_instance("langchain_openai:ChatOpenAI", model="gpt-4")
```

#### 配置驱动的类加载

```python
from typing import Any
from src.reflection import resolve_class

class PluginManager:
    """插件管理器，支持配置驱动的插件加载。"""
    
    def __init__(self, config: dict[str, Any]):
        self.plugins = {}
        
        for name, plugin_config in config.items():
            # 动态加载插件类
            plugin_cls = resolve_class(
                plugin_config["class"],
                base_class=Plugin  # 确保是 Plugin 的子类
            )
            
            # 实例化插件
            self.plugins[name] = plugin_cls(**plugin_config.get("options", {}))
```

---

## 最佳实践

### 1. 始终使用类型验证

```python
# ✅ 推荐：明确类型验证
tool = resolve_variable("src.sandbox.tools:bash_tool", BaseTool)

# ❌ 不推荐：无类型验证，可能导致运行时错误
tool = resolve_variable("src.sandbox.tools:bash_tool")
```

### 2. 使用基类验证确保类型安全

```python
# ✅ 推荐：验证继承关系
model_cls = resolve_class("langchain_openai:ChatOpenAI", BaseChatModel)

# ❌ 不推荐：无验证，可能加载错误的类
model_cls = resolve_class("langchain_openai:ChatOpenAI")
```

### 3. 适当处理异常

```python
# ✅ 推荐：捕获特定异常
try:
    model_cls = resolve_class(config.use, BaseChatModel)
except ImportError as e:
    logger.error(f"无法导入模型: {e}")
    raise ConfigurationError(f"无效的模型配置: {config.use}") from e
except ValueError as e:
    logger.error(f"模型类型错误: {e}")
    raise ConfigurationError(f"模型类不是 BaseChatModel 的子类") from e

# ❌ 不推荐：过于宽泛的异常捕获
try:
    model_cls = resolve_class(config.use, BaseChatModel)
except Exception as e:
    pass  # 隐藏了所有错误
```

### 4. 扩展 MODULE_TO_PACKAGE_HINTS

当添加新的可选依赖时，更新映射表：

```python
# backend/src/reflection/resolvers.py

MODULE_TO_PACKAGE_HINTS = {
    "langchain_google_genai": "langchain-google-genai",
    "langchain_anthropic": "langchain-anthropic",
    "langchain_openai": "langchain-openai",
    "langchain_deepseek": "langchain-deepseek",
    # 添加新的依赖映射
    "langchain_community": "langchain-community",
    "langchain_mcp_adapters": "langchain-mcp-adapters",
}
```

### 5. 路径命名约定

```python
# ✅ 推荐：使用完整的模块路径
"src.sandbox.tools:bash_tool"          # 内置模块
"langchain_openai:ChatOpenAI"          # 第三方库

# ❌ 避免：相对路径或不完整路径
".tools:bash_tool"                     # 相对导入不稳定
"bash_tool"                            # 缺少模块路径
```

### 6. 配置文件组织

```yaml
# config.yaml

# ✅ 推荐：清晰的路径注释
models:
  - name: gpt-4
    use: langchain_openai:ChatOpenAI  # 格式: 模块路径:类名
    
tools:
  - name: bash
    group: bash
    use: src.sandbox.tools:bash_tool  # 格式: 模块路径:变量名
```

---

## 附录

### API 参考

#### resolve_variable

```python
def resolve_variable[T](
    variable_path: str,
    expected_type: type[T] | tuple[type, ...] | None = None,
) -> T:
    """从路径解析变量。

    Args:
        variable_path: 变量路径（格式: "module.path:variable_name"）
        expected_type: 可选的期望类型或类型元组，用于验证

    Returns:
        解析的变量

    Raises:
        ImportError: 模块路径无效或属性不存在
        ValueError: 变量类型验证失败
    """
```

#### resolve_class

```python
def resolve_class[T](class_path: str, base_class: type[T] | None = None) -> type[T]:
    """从模块路径解析类。

    Args:
        class_path: 类路径（格式: "module.path:ClassName"）
        base_class: 可选的基类，验证解析类是否为其子类

    Returns:
        解析的类对象

    Raises:
        ImportError: 模块路径无效或属性不存在
        ValueError: 解析对象不是类或不是指定基类的子类
    """
```

### 相关文件索引

```
backend/src/reflection/
├── __init__.py           # 公共 API 导出
└── resolvers.py          # 核心解析实现

# 使用方
backend/src/models/factory.py         # resolve_class 模型加载
backend/src/tools/tools.py            # resolve_variable 工具加载
backend/src/sandbox/sandbox_provider.py # resolve_class 沙箱提供者
backend/src/channels/service.py       # resolve_class 通道实现
```

### 版本历史

| 版本 | 日期 | 变更说明 |
|------|------|----------|
| 1.0 | 2026-03 | 初始文档版本 |