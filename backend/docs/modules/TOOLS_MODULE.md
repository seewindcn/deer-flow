# Tools 模块技术文档

> 本文档详细介绍 DeerFlow 的工具系统，包括工具注册机制、加载流程、内置工具实现以及 MCP 工具集成。

## 目录

- [模块概述](#模块概述)
- [目录结构](#目录结构)
- [工具注册机制](#工具注册机制)
- [工具加载机制](#工具加载机制)
- [内置工具详解](#内置工具详解)
- [MCP 工具集成](#mcp-工具集成)
- [工具配置](#工具配置)
- [工具开发指南](#工具开发指南)

---

## 模块概述

### 职责和功能定位

`src.tools` 模块是 DeerFlow Agent 框架的核心工具管理系统，负责：

1. **工具统一管理**：提供统一的工具注册、加载和获取接口
2. **内置工具提供**：实现核心内置工具（文件展示、用户澄清、图片查看、任务委托）
3. **MCP 工具集成**：集成 Model Context Protocol 工具缓存机制
4. **工具分组过滤**：支持按工具组进行工具过滤和权限控制
5. **动态工具加载**：通过配置文件动态加载外部工具

### 与其他模块的关系

```
┌─────────────────────────────────────────────────────────────────┐
│                         tools 模块                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌──────────────┐    ┌──────────────┐    ┌──────────────┐     │
│  │   tools.py   │    │   builtins/  │    │    __init__  │     │
│  │  (核心加载)   │◄───┤  (内置工具)   │◄───┤    (导出)    │     │
│  └──────────────┘    └──────────────┘    └──────────────┘     │
│         │                    │                                  │
│         │                    │                                  │
│         ▼                    ▼                                  │
│  ┌──────────────┐    ┌──────────────┐                          │
│  │   config/    │    │    mcp/      │                          │
│  │ (工具配置)    │    │ (MCP工具缓存) │                          │
│  └──────────────┘    └──────────────┘                          │
│         │                                                      │
│         ▼                                                      │
│  ┌──────────────┐                                              │
│  │  reflection/ │                                              │
│  │ (变量解析器)  │                                              │
│  └──────────────┘                                              │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
                              │
                              │ 提供工具给
                              ▼
        ┌─────────────────────────────────────┐
        │           agents/ 模块               │
        │   (Lead Agent, Mobile Agent 等)      │
        └─────────────────────────────────────┘
```

**依赖关系**：

| 依赖模块 | 用途 |
|---------|------|
| `src.config` | 获取工具配置、模型配置 |
| `src.reflection` | 动态解析工具变量路径 |
| `src.mcp.cache` | 获取缓存的 MCP 工具 |
| `src.agents.thread_state` | 访问线程状态和沙箱环境 |
| `src.sandbox` | 沙箱执行和文件操作 |
| `src.subagents` | 子 Agent 任务委托 |

---

## 目录结构

```
backend/src/tools/
├── __init__.py                 # 模块入口，导出 get_available_tools
├── tools.py                    # 核心工具加载逻辑
└── builtins/                   # 内置工具目录
    ├── __init__.py             # 导出所有内置工具
    ├── clarification_tool.py   # 用户澄清工具
    ├── present_file_tool.py    # 文件展示工具
    ├── view_image_tool.py      # 图片查看工具
    ├── task_tool.py            # 子 Agent 任务委托工具
    └── setup_agent_tool.py     # Agent 设置工具
```

### 文件说明

| 文件 | 职责 |
|------|------|
| `__init__.py` | 模块公共 API 导出 |
| `tools.py` | 工具加载核心逻辑，`get_available_tools()` 函数实现 |
| `builtins/__init__.py` | 内置工具统一导出 |
| `builtins/clarification_tool.py` | 实现 `ask_clarification_tool` |
| `builtins/present_file_tool.py` | 实现 `present_file_tool` |
| `builtins/view_image_tool.py` | 实现 `view_image_tool` |
| `builtins/task_tool.py` | 实现 `task_tool` 子 Agent 委托 |
| `builtins/setup_agent_tool.py` | 实现 `setup_agent` Agent 创建 |

---

## 工具注册机制

### @tool 装饰器

DeerFlow 使用 LangChain 的 `@tool` 装饰器来注册工具。该装饰器将普通 Python 函数转换为 LangChain `BaseTool` 实例。

#### 基本用法

```python
from langchain.tools import tool

@tool("tool_name", parse_docstring=True)
def my_tool(param1: str, param2: int) -> str:
    """工具描述（会被解析为工具说明）
    
    Args:
        param1: 参数1说明
        param2: 参数2说明
        
    Returns:
        返回值说明
    """
    return f"Result: {param1}, {param2}"
```

#### 装饰器参数

| 参数 | 类型 | 说明 |
|------|------|------|
| `name` | `str` | 工具名称，用于 LLM 调用识别 |
| `parse_docstring` | `bool` | 是否解析 docstring 生成参数描述 |
| `return_direct` | `bool` | 是否直接返回结果给用户（跳过后续处理） |

### 工具函数定义

#### 标准工具函数签名

```python
from typing import Annotated
from langchain.tools import InjectedToolCallId, ToolRuntime, tool
from langgraph.types import Command
from langgraph.typing import ContextT

from src.agents.thread_state import ThreadState

@tool("my_tool", parse_docstring=True)
def my_tool(
    runtime: ToolRuntime[ContextT, ThreadState],  # 运行时上下文
    param1: str,                                   # 业务参数
    tool_call_id: Annotated[str, InjectedToolCallId],  # 工具调用ID（注入）
) -> Command | str:
    """工具功能描述
    
    详细说明何时使用此工具...
    
    Args:
        param1: 参数说明
    """
    # 工具实现逻辑
    return Command(update={"messages": [...]})
```

#### 参数类型说明

**运行时参数**：

```python
runtime: ToolRuntime[ContextT, ThreadState]
```

提供访问：
- `runtime.state` - 当前线程状态（包含沙箱、线程数据等）
- `runtime.context` - 运行时上下文（包含 thread_id、agent_name 等）
- `runtime.config` - 配置信息

**注入参数**：

```python
tool_call_id: Annotated[str, InjectedToolCallId]
```

LangChain 自动注入的工具调用标识符，用于：
- 返回 `Command` 时构建 `ToolMessage`
- 跟踪工具调用链

**返回类型**：

| 类型 | 用途 |
|------|------|
| `str` | 直接返回文本结果 |
| `Command` | 更新状态或返回消息 |

### 工具分类

DeerFlow 将工具分为三类：

```python
# backend/src/tools/tools.py

# 1. 基础内置工具（始终加载）
BUILTIN_TOOLS = [
    present_file_tool,
    ask_clarification_tool,
]

# 2. 子 Agent 工具（可选加载）
SUBAGENT_TOOLS = [
    task_tool,
]

# 3. 配置加载工具（从 config.yaml 加载）
# 由 get_available_tools() 动态加载
```

---

## 工具加载机制

### get_available_tools() 函数

核心工具加载函数，根据配置和运行时参数返回可用工具列表。

```python
def get_available_tools(
    groups: list[str] | None = None,
    include_mcp: bool = True,
    model_name: str | None = None,
    subagent_enabled: bool = False,
) -> list[BaseTool]:
    """获取所有可用工具。
    
    Args:
        groups: 工具组过滤列表。None 表示加载所有工具
        include_mcp: 是否包含 MCP 工具
        model_name: 模型名称，用于判断是否添加视觉工具
        subagent_enabled: 是否启用子 Agent 工具
        
    Returns:
        可用工具列表
    """
```

### 加载流程

```
┌────────────────────────────────────────────────────────────┐
│                  get_available_tools()                      │
└────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌────────────────────────────────────────────────────────────┐
│  1. 加载配置工具 (config.yaml → tools)                       │
│     - 读取 AppConfig.tools                                   │
│     - 根据 groups 过滤                                       │
│     - 使用 resolve_variable 解析工具变量路径                  │
└────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌────────────────────────────────────────────────────────────┐
│  2. 加载 MCP 工具（如果 include_mcp=True）                   │
│     - 检查 ExtensionsConfig 启用的 MCP 服务器                │
│     - 获取缓存的 MCP 工具                                     │
│     - 自动处理缓存过期和重新初始化                             │
└────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌────────────────────────────────────────────────────────────┐
│  3. 组装内置工具                                             │
│     - 添加 BUILTIN_TOOLS                                     │
│     - 如果 subagent_enabled=True，添加 SUBAGENT_TOOLS        │
│     - 如果模型支持视觉，添加 view_image_tool                   │
└────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌────────────────────────────────────────────────────────────┐
│  4. 返回合并后的工具列表                                      │
│     loaded_tools + builtin_tools + mcp_tools                │
└────────────────────────────────────────────────────────────┘
```

### 工具过滤和分组

#### 工具组配置

```yaml
# config.yaml
tool_groups:
  - name: web           # Web 搜索和抓取工具
  - name: file:read     # 文件读取工具
  - name: file:write    # 文件写入工具
  - name: bash          # Bash 执行工具

tools:
  - name: web_search
    group: web
    use: src.community.tavily.tools:web_search_tool
    
  - name: read_file
    group: file:read
    use: src.sandbox.tools:read_file_tool
```

#### 工具组过滤逻辑

```python
# 只加载特定组的工具
tools = get_available_tools(groups=["web", "file:read"])
# 结果：web_search + read_file（+ 内置工具）

# 加载所有工具
tools = get_available_tools(groups=None)
# 结果：所有配置工具 + 内置工具
```

### 工具变量解析

使用 `resolve_variable` 动态加载工具：

```python
from src.reflection import resolve_variable

# 解析工具路径格式：module_path:variable_name
# 例如：src.sandbox.tools:bash_tool
tool = resolve_variable(tool_config.use, BaseTool)
```

**路径格式**：

```
package.subpackage.module:variable_name
     │              │           │
     │              │           └── 变量名（函数或对象）
     │              └───────────── 模块路径
     └──────────────────────────── 包路径
```

---

## 内置工具详解

### 1. present_file_tool

**文件展示工具** - 将文件呈现给用户查看。

```python
# backend/src/tools/builtins/present_file_tool.py

@tool("present_files", parse_docstring=True)
def present_file_tool(
    runtime: ToolRuntime[ContextT, ThreadState],
    filepaths: list[str],
    tool_call_id: Annotated[str, InjectedToolCallId],
) -> Command:
    """将文件展示给用户查看。
    
    用途：
    - 将创建的文件提供给用户查看、下载或交互
    - 批量展示相关文件
    
    不适用场景：
    - 仅读取文件内容进行处理（使用 read_file）
    - 临时文件或中间文件
    
    Args:
        filepaths: 文件路径列表（仅支持 /mnt/user-data/outputs 目录）
    """
```

**路径规范化**：

```python
def _normalize_presented_filepath(
    runtime: ToolRuntime[ContextT, ThreadState],
    filepath: str,
) -> str:
    """规范化文件路径到虚拟路径。
    
    支持两种输入：
    1. 虚拟路径：/mnt/user-data/outputs/report.md
    2. 实际路径：/app/backend/.deer-flow/threads/<thread>/user-data/outputs/report.md
    
    返回统一的虚拟路径格式。
    """
```

**状态更新**：

```python
return Command(
    update={
        "artifacts": normalized_paths,  # 添加到 artifacts 状态
        "messages": [ToolMessage("Successfully presented files", tool_call_id=tool_call_id)],
    },
)
```

### 2. ask_clarification_tool

**用户澄清工具** - 当需要用户输入才能继续时询问用户。

```python
# backend/src/tools/builtins/clarification_tool.py

@tool("ask_clarification", parse_docstring=True, return_direct=True)
def ask_clarification_tool(
    question: str,
    clarification_type: Literal[
        "missing_info",
        "ambiguous_requirement",
        "approach_choice",
        "risk_confirmation",
        "suggestion",
    ],
    context: str | None = None,
    options: list[str] | None = None,
) -> str:
    """向用户请求澄清。
    
    使用场景：
    - 缺失信息：需要的详细信息未提供
    - 需求模糊：存在多种合理解释
    - 方法选择：多种有效方法需要用户偏好
    - 风险操作：危险操作需要明确确认
    - 建议提议：有推荐但需要用户批准
    
    Args:
        question: 澄清问题
        clarification_type: 澄清类型
        context: 需要澄清的原因背景
        options: 选项列表（用于 approach_choice 或 suggestion）
    """
```

**澄清类型**：

| 类型 | 说明 | 示例场景 |
|------|------|----------|
| `missing_info` | 缺失必要信息 | 缺少文件路径、URL |
| `ambiguous_requirement` | 需求模糊 | 多种解释方式 |
| `approach_choice` | 方法选择 | 多种实现方案 |
| `risk_confirmation` | 风险确认 | 删除文件、生产环境操作 |
| `suggestion` | 建议确认 | 推荐方案需批准 |

**中间件拦截**：

```python
# 注意：实际逻辑由 ClarificationMiddleware 处理
# 该工具调用会被中间件拦截并中断执行，向用户展示问题
return "Clarification request processed by middleware"
```

### 3. view_image_tool

**图片查看工具** - 读取图片文件供模型视觉能力使用。

```python
# backend/src/tools/builtins/view_image_tool.py

@tool("view_image", parse_docstring=True)
def view_image_tool(
    runtime: ToolRuntime[ContextT, ThreadState],
    image_path: str,
    tool_call_id: Annotated[str, InjectedToolCallId],
) -> Command:
    """读取图片文件。
    
    Args:
        image_path: 图片绝对路径（支持 jpg/jpeg/png/webp）
    """
```

**处理流程**：

```python
# 1. 替换虚拟路径
actual_path = replace_virtual_path(image_path, thread_data)

# 2. 验证路径和文件
if not path.is_absolute():
    return error("Path must be absolute")
if not path.exists():
    return error("Image file not found")
if path.suffix.lower() not in {".jpg", ".jpeg", ".png", ".webp"}:
    return error("Unsupported image format")

# 3. 读取并编码
image_base64 = base64.b64encode(image_data).decode("utf-8")

# 4. 更新状态
return Command(
    update={
        "viewed_images": {image_path: {"base64": image_base64, "mime_type": mime_type}},
        "messages": [ToolMessage("Successfully read image", tool_call_id=tool_call_id)],
    }
)
```

**启用条件**：

```python
# tools.py 中根据模型配置决定是否添加
model_config = config.get_model_config(model_name)
if model_config is not None and model_config.supports_vision:
    builtin_tools.append(view_image_tool)
```

### 4. task_tool

**子 Agent 任务委托工具** - 将任务委托给专门的子 Agent。

```python
# backend/src/tools/builtins/task_tool.py

@tool("task", parse_docstring=True)
def task_tool(
    runtime: ToolRuntime[ContextT, ThreadState],
    description: str,
    prompt: str,
    subagent_type: Literal["general-purpose", "bash"],
    tool_call_id: Annotated[str, InjectedToolCallId],
    max_turns: int | None = None,
) -> str:
    """将任务委托给专门的子 Agent 执行。
    
    子 Agent 类型：
    - general-purpose: 通用 Agent，适合复杂多步骤任务
    - bash: Bash 命令执行专家，适合 Git 操作、构建过程
    
    使用场景：
    - 复杂多步骤任务
    - 产生大量输出的任务
    - 需要隔离上下文的任务
    - 并行研究或探索任务
    
    Args:
        description: 任务简短描述（3-5 个词）
        prompt: 详细任务描述
        subagent_type: 子 Agent 类型
        max_turns: 最大轮次限制
    """
```

**执行流程**：

```python
# 1. 获取子 Agent 配置
config = get_subagent_config(subagent_type)

# 2. 构建配置覆盖
if skills_section:
    overrides["system_prompt"] = config.system_prompt + "\n\n" + skills_section
if max_turns is not None:
    overrides["max_turns"] = max_turns

# 3. 创建执行器
executor = SubagentExecutor(
    config=config,
    tools=tools,                    # 子 Agent 不含 task_tool（防止嵌套）
    parent_model=parent_model,
    sandbox_state=sandbox_state,
    thread_data=thread_data,
    thread_id=thread_id,
)

# 4. 异步执行并轮询
task_id = executor.execute_async(prompt)

# 5. 后端轮询（每 5 秒检查一次）
while True:
    result = get_background_task_result(task_id)
    # 发送状态更新事件
    writer({"type": "task_running", ...})
    # 检查完成状态
    if result.status in (COMPLETED, FAILED, TIMED_OUT):
        break
    time.sleep(5)
```

**状态事件**：

```python
# 任务开始
{"type": "task_started", "task_id": "...", "description": "..."}

# 任务运行中（每个新消息）
{"type": "task_running", "task_id": "...", "message": ..., "message_index": 1}

# 任务完成
{"type": "task_completed", "task_id": "...", "result": "..."}

# 任务失败
{"type": "task_failed", "task_id": "...", "error": "..."}

# 任务超时
{"type": "task_timed_out", "task_id": "..."}
```

### 5. setup_agent

**Agent 设置工具** - 创建自定义 Agent。

```python
# backend/src/tools/builtins/setup_agent_tool.py

@tool
def setup_agent(
    soul: str,
    description: str,
    runtime: ToolRuntime,
) -> Command:
    """设置自定义 DeerFlow Agent。
    
    Args:
        soul: SOUL.md 内容，定义 Agent 的个性和行为
        description: Agent 功能的一行描述
    """
```

**创建流程**：

```python
# 1. 确定目录
agent_dir = paths.agent_dir(agent_name) if agent_name else paths.base_dir

# 2. 创建 config.yaml
config_data = {"name": agent_name, "description": description}
with open(config_file, "w") as f:
    yaml.dump(config_data, f)

# 3. 创建 SOUL.md
soul_file.write_text(soul, encoding="utf-8")

# 4. 返回状态更新
return Command(update={"created_agent_name": agent_name, ...})
```

---

## MCP 工具集成

### MCP 工具缓存机制

MCP (Model Context Protocol) 工具通过缓存机制集成：

```python
# backend/src/mcp/cache.py

_mcp_tools_cache: list[BaseTool] | None = None
_cache_initialized = False
_config_mtime: float | None = None  # 配置文件修改时间

async def initialize_mcp_tools() -> list[BaseTool]:
    """初始化并缓存 MCP 工具（应用启动时调用）。"""
    global _mcp_tools_cache, _cache_initialized, _config_mtime
    
    _mcp_tools_cache = await get_mcp_tools()
    _cache_initialized = True
    _config_mtime = _get_config_mtime()
    
    return _mcp_tools_cache

def get_cached_mcp_tools() -> list[BaseTool]:
    """获取缓存的 MCP 工具（支持懒加载和缓存过期检测）。"""
    # 检查缓存是否过期（配置文件修改）
    if _is_cache_stale():
        reset_mcp_tools_cache()
    
    # 如果未初始化，自动初始化
    if not _cache_initialized:
        # 处理不同事件循环场景
        ...
    
    return _mcp_tools_cache or []
```

### 缓存过期检测

```python
def _is_cache_stale() -> bool:
    """检查配置文件是否已修改，缓存是否过期。"""
    current_mtime = _get_config_mtime()
    if _config_mtime is None or current_mtime is None:
        return False
    return current_mtime > _config_mtime
```

### 工具加载集成

```python
# backend/src/tools/tools.py

# 获取缓存的 MCP 工具
mcp_tools = []
if include_mcp:
    try:
        from src.config.extensions_config import ExtensionsConfig
        from src.mcp.cache import get_cached_mcp_tools
        
        extensions_config = ExtensionsConfig.from_file()
        if extensions_config.get_enabled_mcp_servers():
            mcp_tools = get_cached_mcp_tools()
            logger.info(f"Using {len(mcp_tools)} cached MCP tool(s)")
    except ImportError:
        logger.warning("MCP module not available...")
    except Exception as e:
        logger.error(f"Failed to get cached MCP tools: {e}")
```

### 注意事项

> **重要**：使用 `ExtensionsConfig.from_file()` 而非 `config.extensions` 来确保总是读取磁盘上的最新配置。这是因为 Gateway API 在独立进程中运行，配置更新需要立即反映。

---

## 工具配置

### config.yaml 工具配置格式

```yaml
# ============================================================================
# 工具组配置
# ============================================================================
tool_groups:
  - name: web           # Web 工具组
  - name: file:read      # 文件读取组
  - name: file:write     # 文件写入组
  - name: bash           # Bash 执行组

# ============================================================================
# 工具配置
# ============================================================================
tools:
  # Web 搜索工具（需要 Tavily API key）
  - name: web_search
    group: web
    use: src.community.tavily.tools:web_search_tool
    max_results: 5
    # api_key: $TAVILY_API_KEY

  # Web 抓取工具
  - name: web_fetch
    group: web
    use: src.community.jina_ai.tools:web_fetch_tool
    timeout: 10

  # 图片搜索工具
  - name: image_search
    group: web
    use: src.community.image_search.tools:image_search_tool
    max_results: 5

  # 文件读取工具
  - name: ls
    group: file:read
    use: src.sandbox.tools:ls_tool

  - name: read_file
    group: file:read
    use: src.sandbox.tools:read_file_tool

  # 文件写入工具
  - name: write_file
    group: file:write
    use: src.sandbox.tools:write_file_tool

  - name: str_replace
    group: file:write
    use: src.sandbox.tools:str_replace_tool

  # Bash 执行工具
  - name: bash
    group: bash
    use: src.sandbox.tools:bash_tool
```

### ToolConfig 数据模型

```python
# backend/src/config/tool_config.py

class ToolGroupConfig(BaseModel):
    """工具组配置"""
    name: str = Field(..., description="工具组唯一名称")


class ToolConfig(BaseModel):
    """工具配置"""
    name: str = Field(..., description="工具唯一名称")
    group: str = Field(..., description="工具所属组名")
    use: str = Field(
        ...,
        description="工具变量路径（如 src.sandbox.tools:bash_tool）",
    )
    # 其他额外配置字段会保留并传递给工具
```

### 工具配置继承

工具配置支持额外的自定义字段：

```yaml
tools:
  - name: web_search
    group: web
    use: src.community.tavily.tools:web_search_tool
    max_results: 5          # 自定义字段
    search_depth: advanced   # 自定义字段
    api_key: $TAVILY_API_KEY # 环境变量引用
```

这些额外字段会在工具初始化时传递给工具实例。

---

## 工具开发指南

### 创建新工具

#### 1. 定义工具函数

```python
# src/tools/builtins/my_tool.py

from typing import Annotated
from langchain.tools import InjectedToolCallId, ToolRuntime, tool
from langchain_core.messages import ToolMessage
from langgraph.types import Command
from langgraph.typing import ContextT

from src.agents.thread_state import ThreadState


@tool("my_tool", parse_docstring=True)
def my_tool(
    runtime: ToolRuntime[ContextT, ThreadState],
    input_param: str,
    tool_call_id: Annotated[str, InjectedToolCallId],
) -> Command | str:
    """工具功能的简短描述。
    
    详细说明工具的使用场景和最佳实践。
    
    When to use:
    - 场景1
    - 场景2
    
    When NOT to use:
    - 不适用场景1
    
    Args:
        input_param: 参数说明
    """
    # 1. 访问运行时状态
    thread_id = runtime.context.get("thread_id")
    sandbox_state = runtime.state.get("sandbox")
    
    # 2. 执行工具逻辑
    result = do_something(input_param)
    
    # 3. 返回结果
    if needs_state_update:
        return Command(
            update={
                "my_state": result,
                "messages": [ToolMessage(f"Result: {result}", tool_call_id=tool_call_id)],
            }
        )
    return f"Result: {result}"
```

#### 2. 注册到内置工具

```python
# src/tools/builtins/__init__.py

from .my_tool import my_tool

__all__ = [
    # ... 其他工具
    "my_tool",
]
```

#### 3. 添加到工具列表

```python
# src/tools/tools.py

from src.tools.builtins import my_tool

BUILTIN_TOOLS = [
    present_file_tool,
    ask_clarification_tool,
    my_tool,  # 添加新工具
]
```

### 创建配置加载工具

#### 1. 在独立模块定义工具

```python
# src/community/my_tools/tool.py

from langchain.tools import tool

@tool("custom_tool", parse_docstring=True)
def custom_tool(query: str, max_results: int = 5) -> str:
    """自定义工具描述。
    
    Args:
        query: 查询内容
        max_results: 最大结果数
    """
    return f"Processed: {query}"
```

#### 2. 在 config.yaml 配置

```yaml
tools:
  - name: custom_tool
    group: custom
    use: src.community.my_tools.tool:custom_tool
    max_results: 10
```

### 工具最佳实践

#### 文档规范

```python
@tool("my_tool", parse_docstring=True)
def my_tool(...):
    """一行简短描述。
    
    详细功能说明...
    
    When to use:
    - 明确的使用场景列表
    
    When NOT to use:
    - 明确的不适用场景
    
    Args:
        param: 参数说明（支持类型推断）
    """
```

#### 错误处理

```python
@tool("safe_tool", parse_docstring=True)
def safe_tool(
    runtime: ToolRuntime[ContextT, ThreadState],
    file_path: str,
    tool_call_id: Annotated[str, InjectedToolCallId],
) -> Command:
    """安全文件处理工具。"""
    try:
        # 验证输入
        if not file_path:
            return Command(
                update={"messages": [ToolMessage("Error: file_path is required", tool_call_id=tool_call_id)]}
            )
        
        # 执行操作
        result = process_file(file_path)
        return Command(
            update={"messages": [ToolMessage(f"Success: {result}", tool_call_id=tool_call_id)]}
        )
    except FileNotFoundError as e:
        return Command(
            update={"messages": [ToolMessage(f"Error: File not found: {file_path}", tool_call_id=tool_call_id)]}
        )
    except Exception as e:
        logger.error(f"Tool failed: {e}", exc_info=True)
        return Command(
            update={"messages": [ToolMessage(f"Error: {str(e)}", tool_call_id=tool_call_id)]}
        )
```

#### 状态更新

```python
# 使用 Command 更新多个状态字段
return Command(
    update={
        "artifacts": ["file1.md", "file2.md"],
        "viewed_images": {"path": {"base64": "...", "mime_type": "..."}},
        "messages": [ToolMessage("Success", tool_call_id=tool_call_id)],
    }
)
```

---

## 附录

### 工具分类汇总

| 类别 | 工具 | 加载条件 |
|------|------|----------|
| 基础内置 | `present_file_tool` | 始终加载 |
| 基础内置 | `ask_clarification_tool` | 始终加载 |
| 条件内置 | `view_image_tool` | 模型支持视觉 |
| 子 Agent | `task_tool` | `subagent_enabled=True` |
| 配置工具 | `bash_tool`, `read_file_tool` 等 | 从 config.yaml 加载 |
| MCP 工具 | 动态加载 | `include_mcp=True` 且有启用的服务器 |

### 相关文件索引

```
backend/src/tools/
├── __init__.py           # 导出 get_available_tools
├── tools.py              # 工具加载核心逻辑
└── builtins/
    ├── __init__.py
    ├── clarification_tool.py  # ask_clarification_tool
    ├── present_file_tool.py   # present_file_tool
    ├── view_image_tool.py     # view_image_tool
    ├── task_tool.py           # task_tool
    └── setup_agent_tool.py    # setup_agent

backend/src/config/
├── app_config.py         # AppConfig.tools 配置
└── tool_config.py        # ToolConfig, ToolGroupConfig

backend/src/reflection/
└── resolvers.py          # resolve_variable 工具加载

backend/src/mcp/
└── cache.py              # MCP 工具缓存
```

### 版本历史

| 版本 | 日期 | 变更说明 |
|------|------|----------|
| 1.0 | 2026-03 | 初始文档版本 |