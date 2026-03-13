# DeerFlow Subagents 子代理系统技术文档

## 1. 模块概述

Subagents（子代理）系统是 DeerFlow 的核心组件之一，提供了一种将复杂任务委托给专门化代理执行的机制。通过子代理，主代理可以：

- **上下文隔离**：将探索性和实现性任务分离，保持主对话上下文清晰
- **并行执行**：多个子代理可以同时运行不同任务
- **专门化处理**：不同类型的任务可以分配给最适合的子代理
- **超时控制**：每个子代理都有独立的超时机制

### 架构概览

```
┌─────────────────────────────────────────────────────────────┐
│                     Lead Agent (主代理)                      │
│                                                             │
│  ┌─────────────────────────────────────────────────────┐   │
│  │                    Task Tool                         │   │
│  │  - 解析任务请求                                      │   │
│  │  - 选择子代理类型                                    │   │
│  │  - 启动异步执行                                      │   │
│  │  - 轮询状态并返回结果                                │   │
│  └───────────────────────┬─────────────────────────────┘   │
└──────────────────────────┼──────────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────────┐
│                   SubagentExecutor                          │
│  - 创建子代理实例                                           │
│  - 过滤工具集                                               │
│  - 传递沙箱状态和线程数据                                   │
│  - 执行任务并收集结果                                       │
└─────────────────────────────────────────────────────────────┘
                           │
           ┌───────────────┼───────────────┐
           ▼               ▼               ▼
    ┌─────────────┐ ┌─────────────┐ ┌─────────────┐
    │ general-    │ │    bash     │ │  自定义      │
    │ purpose     │ │   agent     │ │  子代理      │
    └─────────────┘ └─────────────┘ └─────────────┘
```

## 2. 目录结构

```
backend/src/subagents/
├── __init__.py              # 模块入口，导出核心类
├── config.py                # SubagentConfig 配置类
├── executor.py              # SubagentExecutor 执行引擎
├── registry.py              # 子代理注册表
└── builtins/                # 内置子代理
    ├── __init__.py          # 内置子代理注册
    ├── general_purpose.py   # 通用子代理配置
    └── bash_agent.py        # Bash 命令执行子代理

backend/src/config/
└── subagents_config.py      # 运行时配置（从 config.yaml 加载）

backend/src/tools/builtins/
└── task_tool.py             # Task 工具实现

backend/src/agents/middlewares/
└── subagent_limit_middleware.py  # 并发限制中间件
```

## 3. 核心类详解

### 3.1 SubagentConfig（子代理配置）

定义子代理的配置信息，位于 `config.py`：

```python
from dataclasses import dataclass, field


@dataclass
class SubagentConfig:
    """Configuration for a subagent.

    Attributes:
        name: 子代理的唯一标识符。
        description: 描述何时应该使用该子代理。
        system_prompt: 指导子代理行为的系统提示词。
        tools: 允许使用的工具名称列表。如果为 None，继承所有工具。
        disallowed_tools: 禁止使用的工具名称列表。
        model: 使用的模型 - 'inherit' 表示继承父代理的模型。
        max_turns: 停止前最大代理轮次。
        timeout_seconds: 最大执行时间（秒），默认 900（15分钟）。
    """

    name: str
    description: str
    system_prompt: str
    tools: list[str] | None = None
    disallowed_tools: list[str] | None = field(default_factory=lambda: ["task"])
    model: str = "inherit"
    max_turns: int = 50
    timeout_seconds: int = 900
```

**配置项说明：**

| 属性 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `name` | `str` | 必填 | 子代理的唯一标识符 |
| `description` | `str` | 必填 | 描述何时委托给该子代理 |
| `system_prompt` | `str` | 必填 | 子代理的系统提示词 |
| `tools` | `list[str] \| None` | `None` | 允许的工具白名单，`None` 表示继承所有工具 |
| `disallowed_tools` | `list[str] \| None` | `["task"]` | 禁止的工具黑名单 |
| `model` | `str` | `"inherit"` | 模型名称，`"inherit"` 继承父代理模型 |
| `max_turns` | `int` | `50` | 最大执行轮次 |
| `timeout_seconds` | `int` | `900` | 超时时间（秒） |

### 3.2 SubagentStatus（子代理状态）

枚举类，定义子代理执行的状态：

```python
from enum import Enum


class SubagentStatus(Enum):
    """Status of a subagent execution."""

    PENDING = "pending"        # 等待执行
    RUNNING = "running"        # 正在执行
    COMPLETED = "completed"    # 执行完成
    FAILED = "failed"          # 执行失败
    TIMED_OUT = "timed_out"    # 执行超时
```

**状态流转：**

```
PENDING → RUNNING → COMPLETED
                  → FAILED
                  → TIMED_OUT
```

### 3.3 SubagentResult（子代理结果）

数据类，封装子代理执行的结果：

```python
from dataclasses import dataclass
from datetime import datetime
from typing import Any


@dataclass
class SubagentResult:
    """Result of a subagent execution.

    Attributes:
        task_id: 此次执行的唯一标识符。
        trace_id: 分布式追踪 ID（关联父代理和子代理日志）。
        status: 当前执行状态。
        result: 最终结果消息（如果完成）。
        error: 错误消息（如果失败）。
        started_at: 执行开始时间。
        completed_at: 执行完成时间。
        ai_messages: 执行期间生成的完整 AI 消息列表（字典格式）。
    """

    task_id: str
    trace_id: str
    status: SubagentStatus
    result: str | None = None
    error: str | None = None
    started_at: datetime | None = None
    completed_at: datetime | None = None
    ai_messages: list[dict[str, Any]] | None = None

    def __post_init__(self):
        """Initialize mutable defaults."""
        if self.ai_messages is None:
            self.ai_messages = []
```

### 3.4 SubagentExecutor（子代理执行器）

核心执行引擎，负责创建和运行子代理：

```python
class SubagentExecutor:
    """Executor for running subagents."""

    def __init__(
        self,
        config: SubagentConfig,
        tools: list[BaseTool],
        parent_model: str | None = None,
        sandbox_state: SandboxState | None = None,
        thread_data: ThreadDataState | None = None,
        thread_id: str | None = None,
        trace_id: str | None = None,
    ):
        """Initialize the executor.

        Args:
            config: 子代理配置。
            tools: 所有可用工具列表（将被过滤）。
            parent_model: 父代理的模型名称（用于继承）。
            sandbox_state: 来自父代理的沙箱状态。
            thread_data: 来自父代理的线程数据。
            thread_id: 用于沙箱操作的线程 ID。
            trace_id: 来自父代理的分布式追踪 ID。
        """
        self.config = config
        self.parent_model = parent_model
        self.sandbox_state = sandbox_state
        self.thread_data = thread_data
        self.thread_id = thread_id
        self.trace_id = trace_id or str(uuid.uuid4())[:8]

        # 根据配置过滤工具
        self.tools = _filter_tools(
            tools,
            config.tools,
            config.disallowed_tools,
        )

    def execute(self, task: str, result_holder: SubagentResult | None = None) -> SubagentResult:
        """同步执行任务（异步执行的包装器）。

        在新的事件循环中运行异步执行，允许在线程池中使用异步工具。
        """
        # ...

    async def _aexecute(self, task: str, result_holder: SubagentResult | None = None) -> SubagentResult:
        """异步执行任务。"""
        # ...

    def execute_async(self, task: str, task_id: str | None = None) -> str:
        """在后台启动任务执行。

        Args:
            task: 任务描述。
            task_id: 可选的任务 ID。如未提供，将生成随机 UUID。

        Returns:
            可用于后续检查状态的任务 ID。
        """
        # ...
```

**关键方法：**

| 方法 | 说明 |
|------|------|
| `execute()` | 同步执行任务，返回完整结果 |
| `execute_async()` | 后台异步执行，返回 task_id |
| `_aexecute()` | 内部异步执行实现 |
| `_create_agent()` | 创建子代理实例 |
| `_build_initial_state()` | 构建初始状态 |

## 4. 内置子代理

### 4.1 general-purpose（通用子代理）

适用于需要复杂推理和多步骤操作的任务：

```python
# backend/src/subagents/builtins/general_purpose.py

from src.subagents.config import SubagentConfig

GENERAL_PURPOSE_CONFIG = SubagentConfig(
    name="general-purpose",
    description="""A capable agent for complex, multi-step tasks that require both exploration and action.

Use this subagent when:
- The task requires both exploration and modification
- Complex reasoning is needed to interpret results
- Multiple dependent steps must be executed
- The task would benefit from isolated context management

Do NOT use for simple, single-step operations.""",
    system_prompt="""You are a general-purpose subagent working on a delegated task...

<guidelines>
- Focus on completing the delegated task efficiently
- Use available tools as needed to accomplish the goal
- Think step by step but act decisively
- If you encounter issues, explain them clearly in your response
- Return a concise summary of what you accomplished
- Do NOT ask for clarification - work with the information provided
</guidelines>

<output_format>
When you complete the task, provide:
1. A brief summary of what was accomplished
2. Key findings or results
3. Any relevant file paths, data, or artifacts created
4. Issues encountered (if any)
5. Citations: Use `[citation:Title](URL)` format for external sources
</output_format>
""",
    tools=None,  # 继承所有工具
    disallowed_tools=["task", "ask_clarification", "present_files"],  # 防止嵌套
    model="inherit",
    max_turns=50,
)
```

**适用场景：**
- 需要探索和修改的任务
- 需要复杂推理来解释结果
- 多个依赖步骤的执行
- 需要隔离上下文管理的任务

### 4.2 bash（命令执行子代理）

专门用于执行 Bash 命令：

```python
# backend/src/subagents/builtins/bash_agent.py

from src.subagents.config import SubagentConfig

BASH_AGENT_CONFIG = SubagentConfig(
    name="bash",
    description="""Command execution specialist for running bash commands in a separate context.

Use this subagent when:
- You need to run a series of related bash commands
- Terminal operations like git, npm, docker, etc.
- Command output is verbose and would clutter main context
- Build, test, or deployment operations

Do NOT use for simple single commands - use bash tool directly instead.""",
    system_prompt="""You are a bash command execution specialist...

<guidelines>
- Execute commands one at a time when they depend on each other
- Use parallel execution when commands are independent
- Report both stdout and stderr when relevant
- Handle errors gracefully and explain what went wrong
- Use absolute paths for file operations
- Be cautious with destructive operations (rm, overwrite, etc.)
</guidelines>
""",
    tools=["bash", "ls", "read_file", "write_file", "str_replace"],  # 仅沙箱工具
    disallowed_tools=["task", "ask_clarification", "present_files"],
    model="inherit",
    max_turns=30,
)
```

**适用场景：**
- 执行一系列相关的 bash 命令
- git、npm、docker 等终端操作
- 命令输出冗长，会干扰主上下文
- 构建、测试或部署操作

## 5. 执行流程

### 5.1 完整执行流程

```
┌────────────────────────────────────────────────────────────────┐
│ 1. Task Tool 调用                                              │
│    - 解析参数（description, prompt, subagent_type）             │
│    - 获取子代理配置                                             │
│    - 应用 config.yaml 覆盖                                      │
└─────────────────────┬──────────────────────────────────────────┘
                      │
                      ▼
┌────────────────────────────────────────────────────────────────┐
│ 2. 创建 SubagentExecutor                                       │
│    - 提取父代理上下文（sandbox_state, thread_data）             │
│    - 过滤工具集（移除 task 工具防止嵌套）                       │
│    - 生成 trace_id                                              │
└─────────────────────┬──────────────────────────────────────────┘
                      │
                      ▼
┌────────────────────────────────────────────────────────────────┐
│ 3. execute_async() 启动后台任务                                │
│    - 创建 SubagentResult（PENDING 状态）                        │
│    - 存入 _background_tasks                                     │
│    - 提交到 _scheduler_pool                                     │
└─────────────────────┬──────────────────────────────────────────┘
                      │
                      ▼
┌────────────────────────────────────────────────────────────────┐
│ 4. 调度器线程执行                                               │
│    - 更新状态为 RUNNING                                         │
│    - 提交到 _execution_pool（带超时）                           │
└─────────────────────┬──────────────────────────────────────────┘
                      │
                      ▼
┌────────────────────────────────────────────────────────────────┐
│ 5. 执行线程运行                                                 │
│    - 创建子代理实例                                             │
│    - 构建初始状态                                               │
│    - 流式执行（astream）                                        │
│    - 收集 AI 消息                                               │
└─────────────────────┬──────────────────────────────────────────┘
                      │
                      ▼
┌────────────────────────────────────────────────────────────────┐
│ 6. Task Tool 轮询                                              │
│    - 每 5 秒检查状态                                            │
│    - 发送 task_running 事件（包含新消息）                       │
│    - 等待完成/失败/超时                                         │
│    - 返回最终结果                                               │
│    - 清理后台任务                                               │
└────────────────────────────────────────────────────────────────┘
```

### 5.2 Task Tool 实现详解

Task Tool 是子代理系统的入口点：

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
    """Delegate a task to a specialized subagent that runs in its own context.

    Args:
        description: 任务的简短描述（3-5 个词），用于日志/显示。
        prompt: 子代理的任务描述。要具体明确。
        subagent_type: 子代理类型。
        max_turns: 可选的最大代理轮次。
    """
    # 1. 获取子代理配置
    config = get_subagent_config(subagent_type)

    # 2. 应用配置覆盖
    overrides = {}
    if max_turns is not None:
        overrides["max_turns"] = max_turns
    if overrides:
        config = replace(config, **overrides)

    # 3. 提取父代理上下文
    sandbox_state = runtime.state.get("sandbox")
    thread_data = runtime.state.get("thread_data")
    thread_id = runtime.context.get("thread_id")

    # 4. 获取可用工具（禁用 task 工具防止嵌套）
    tools = get_available_tools(model_name=parent_model, subagent_enabled=False)

    # 5. 创建执行器
    executor = SubagentExecutor(
        config=config,
        tools=tools,
        parent_model=parent_model,
        sandbox_state=sandbox_state,
        thread_data=thread_data,
        thread_id=thread_id,
        trace_id=trace_id,
    )

    # 6. 启动后台执行
    task_id = executor.execute_async(prompt, task_id=tool_call_id)

    # 7. 轮询等待完成
    while True:
        result = get_background_task_result(task_id)
        # ... 检查状态并发送事件 ...
        if result.status == SubagentStatus.COMPLETED:
            cleanup_background_task(task_id)
            return f"Task Succeeded. Result: {result.result}"
```

## 6. 线程池管理

### 6.1 双层线程池架构

系统使用两个独立的线程池：

```python
# backend/src/subagents/executor.py

# 后台任务存储
_background_tasks: dict[str, SubagentResult] = {}
_background_tasks_lock = threading.Lock()

# 调度器线程池 - 用于任务调度和编排
_scheduler_pool = ThreadPoolExecutor(max_workers=3, thread_name_prefix="subagent-scheduler-")

# 执行器线程池 - 用于实际子代理执行（支持超时）
_execution_pool = ThreadPoolExecutor(max_workers=3, thread_name_prefix="subagent-exec-")

MAX_CONCURRENT_SUBAGENTS = 3
```

**架构说明：**

```
┌─────────────────────────────────────────────────────────────────┐
│                      Task Tool 调用                              │
└─────────────────────────┬───────────────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────────────┐
│                   _scheduler_pool                               │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐             │
│  │ scheduler-1 │  │ scheduler-2 │  │ scheduler-3 │             │
│  └──────┬──────┘  └──────┬──────┘  └──────┬──────┘             │
│         │                │                │                    │
│         ▼                ▼                ▼                    │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │              _execution_pool                             │   │
│  │  ┌───────────┐  ┌───────────┐  ┌───────────┐           │   │
│  │  │  exec-1   │  │  exec-2   │  │  exec-3   │           │   │
│  │  │ (timeout) │  │ (timeout) │  │ (timeout) │           │   │
│  │  └───────────┘  └───────────┘  └───────────┘           │   │
│  └─────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────┘
```

### 6.2 任务生命周期管理

```python
def execute_async(self, task: str, task_id: str | None = None) -> str:
    """启动后台任务执行。"""
    # 1. 创建任务 ID
    if task_id is None:
        task_id = str(uuid.uuid4())[:8]

    # 2. 创建初始结果（PENDING 状态）
    result = SubagentResult(
        task_id=task_id,
        trace_id=self.trace_id,
        status=SubagentStatus.PENDING,
    )

    # 3. 存入全局字典
    with _background_tasks_lock:
        _background_tasks[task_id] = result

    # 4. 定义调度器任务
    def run_task():
        with _background_tasks_lock:
            _background_tasks[task_id].status = SubagentStatus.RUNNING
            _background_tasks[task_id].started_at = datetime.now()
            result_holder = _background_tasks[task_id]

        try:
            # 提交到执行池（带超时）
            execution_future = _execution_pool.submit(
                self.execute, task, result_holder
            )
            # 等待执行完成或超时
            exec_result = execution_future.result(timeout=self.config.timeout_seconds)
            # 更新结果
            # ...
        except FuturesTimeoutError:
            # 处理超时
            # ...
        except Exception as e:
            # 处理异常
            # ...

    # 5. 提交到调度器池
    _scheduler_pool.submit(run_task)
    return task_id
```

### 6.3 并发限制中间件

```python
# backend/src/agents/middlewares/subagent_limit_middleware.py

class SubagentLimitMiddleware(AgentMiddleware[AgentState]):
    """截断单个模型响应中过多的 'task' 工具调用。

    当 LLM 在一次响应中生成超过 max_concurrent 个并行 task 工具调用时，
    此中间件只保留前 max_concurrent 个，丢弃其余的。
    """

    def __init__(self, max_concurrent: int = MAX_CONCURRENT_SUBAGENTS):
        super().__init__()
        self.max_concurrent = _clamp_subagent_limit(max_concurrent)  # 限制在 [2, 4]

    def _truncate_task_calls(self, state: AgentState) -> dict | None:
        messages = state.get("messages", [])
        last_msg = messages[-1]

        tool_calls = getattr(last_msg, "tool_calls", None)
        if not tool_calls:
            return None

        # 计算需要丢弃的 task 工具调用
        task_indices = [i for i, tc in enumerate(tool_calls) if tc.get("name") == "task"]
        if len(task_indices) <= self.max_concurrent:
            return None

        # 保留前 max_concurrent 个 task 调用
        indices_to_drop = set(task_indices[self.max_concurrent:])
        truncated_tool_calls = [
            tc for i, tc in enumerate(tool_calls) if i not in indices_to_drop
        ]

        # 替换消息
        updated_msg = last_msg.model_copy(update={"tool_calls": truncated_tool_calls})
        return {"messages": [updated_msg]}
```

## 7. Task 工具

### 7.1 工具定义

```python
@tool("task", parse_docstring=True)
def task_tool(
    runtime: ToolRuntime[ContextT, ThreadState],
    description: str,
    prompt: str,
    subagent_type: Literal["general-purpose", "bash"],
    tool_call_id: Annotated[str, InjectedToolCallId],
    max_turns: int | None = None,
) -> str:
    """Delegate a task to a specialized subagent that runs in its own context.

    Available subagent types:
    - **general-purpose**: 适用于需要探索和行动的复杂多步骤任务。
    - **bash**: 命令执行专家，用于运行 bash 命令。

    When to use this tool:
    - 需要多个步骤或工具的复杂任务
    - 产生冗长输出的任务
    - 需要将上下文与主对话隔离的情况
    - 并行研究或探索任务

    When NOT to use this tool:
    - 简单的单步操作（直接使用工具）
    - 需要用户交互或澄清的任务
    """
```

### 7.2 流式事件

Task 工具通过 `get_stream_writer()` 发送事件：

```python
writer = get_stream_writer()

# 任务开始事件
writer({
    "type": "task_started",
    "task_id": task_id,
    "description": description
})

# 任务运行中事件（每条新 AI 消息）
writer({
    "type": "task_running",
    "task_id": task_id,
    "message": message_dict,
    "message_index": i + 1,
    "total_messages": current_message_count
})

# 任务完成事件
writer({
    "type": "task_completed",
    "task_id": task_id,
    "result": result.result
})

# 任务失败事件
writer({
    "type": "task_failed",
    "task_id": task_id,
    "error": result.error
})

# 任务超时事件
writer({
    "type": "task_timed_out",
    "task_id": task_id,
    "error": result.error
})
```

### 7.3 轮询机制

```python
# 轮询参数
poll_count = 0
last_status = None
last_message_count = 0
max_poll_count = (config.timeout_seconds + 60) // 5  # 超时 + 60秒缓冲

while True:
    result = get_background_task_result(task_id)

    # 发送新消息的事件
    current_message_count = len(result.ai_messages)
    if current_message_count > last_message_count:
        for i in range(last_message_count, current_message_count):
            writer({"type": "task_running", ...})
        last_message_count = current_message_count

    # 检查终端状态
    if result.status == SubagentStatus.COMPLETED:
        cleanup_background_task(task_id)
        return f"Task Succeeded. Result: {result.result}"
    elif result.status == SubagentStatus.FAILED:
        cleanup_background_task(task_id)
        return f"Task failed. Error: {result.error}"
    elif result.status == SubagentStatus.TIMED_OUT:
        cleanup_background_task(task_id)
        return f"Task timed out. Error: {result.error}"

    # 等待下一次轮询
    time.sleep(5)
    poll_count += 1
```

## 8. 状态传递

### 8.1 ThreadState 结构

子代理继承父代理的状态：

```python
# backend/src/agents/thread_state.py

class SandboxState(TypedDict):
    sandbox_id: NotRequired[str | None]


class ThreadDataState(TypedDict):
    workspace_path: NotRequired[str | None]
    uploads_path: NotRequired[str | None]
    outputs_path: NotRequired[str | None]


class ThreadState(AgentState):
    sandbox: NotRequired[SandboxState | None]
    thread_data: NotRequired[ThreadDataState | None]
    title: NotRequired[str | None]
    artifacts: Annotated[list[str], merge_artifacts]
    todos: NotRequired[list | None]
    uploaded_files: NotRequired[list[dict] | None]
    viewed_images: Annotated[dict[str, ViewedImageData], merge_viewed_images]
```

### 8.2 状态传递流程

```python
# Task Tool 提取父代理状态
sandbox_state = runtime.state.get("sandbox")
thread_data = runtime.state.get("thread_data")
thread_id = runtime.context.get("thread_id")

# 创建执行器时传递状态
executor = SubagentExecutor(
    config=config,
    tools=tools,
    parent_model=parent_model,
    sandbox_state=sandbox_state,    # 传递沙箱状态
    thread_data=thread_data,        # 传递线程数据
    thread_id=thread_id,            # 传递线程 ID
    trace_id=trace_id,              # 传递追踪 ID
)

# 构建子代理初始状态
def _build_initial_state(self, task: str) -> dict[str, Any]:
    state = {
        "messages": [HumanMessage(content=task)],
    }

    # 传递沙箱和线程数据
    if self.sandbox_state is not None:
        state["sandbox"] = self.sandbox_state
    if self.thread_data is not None:
        state["thread_data"] = self.thread_data

    return state
```

### 8.3 中间件配置

子代理使用最小化的中间件来确保工具可以访问沙箱和线程数据：

```python
def _create_agent(self):
    # 子代理需要最小化中间件以确保工具可以访问沙箱和 thread_data
    # 这些中间件将复用父代理的沙箱/thread_data
    from src.agents.middlewares.thread_data_middleware import ThreadDataMiddleware
    from src.sandbox.middleware import SandboxMiddleware

    middlewares = [
        ThreadDataMiddleware(lazy_init=True),  # 计算线程路径
        SandboxMiddleware(lazy_init=True),    # 复用父代理的沙箱（无需重新获取）
    ]

    return create_agent(
        model=model,
        tools=self.tools,
        middleware=middlewares,
        system_prompt=self.config.system_prompt,
        state_schema=ThreadState,
    )
```

## 9. 配置项

### 9.1 运行时配置

通过 `config.yaml` 可以覆盖子代理的超时设置：

```yaml
# config.yaml
subagents:
  # 全局默认超时（秒）
  timeout_seconds: 900

  # 按子代理类型覆盖
  agents:
    general-purpose:
      timeout_seconds: 1200  # 20 分钟
    bash:
      timeout_seconds: 600   # 10 分钟
```

### 9.2 配置类定义

```python
# backend/src/config/subagents_config.py

from pydantic import BaseModel, Field


class SubagentOverrideConfig(BaseModel):
    """每个子代理的配置覆盖。"""

    timeout_seconds: int | None = Field(
        default=None,
        ge=1,
        description="此子代理的超时时间（秒），None 表示使用全局默认",
    )


class SubagentsAppConfig(BaseModel):
    """子代理系统配置。"""

    timeout_seconds: int = Field(
        default=900,
        ge=1,
        description="所有子代理的默认超时时间（秒）",
    )
    agents: dict[str, SubagentOverrideConfig] = Field(
        default_factory=dict,
        description="按子代理名称的配置覆盖",
    )

    def get_timeout_for(self, agent_name: str) -> int:
        """获取特定子代理的有效超时时间。"""
        override = self.agents.get(agent_name)
        if override is not None and override.timeout_seconds is not None:
            return override.timeout_seconds
        return self.timeout_seconds
```

### 9.3 注册表与配置覆盖

```python
# backend/src/subagents/registry.py

def get_subagent_config(name: str) -> SubagentConfig | None:
    """获取子代理配置（应用 config.yaml 覆盖）。"""
    config = BUILTIN_SUBAGENTS.get(name)
    if config is None:
        return None

    # 应用 config.yaml 的超时覆盖
    from src.config.subagents_config import get_subagents_app_config

    app_config = get_subagents_app_config()
    effective_timeout = app_config.get_timeout_for(name)
    if effective_timeout != config.timeout_seconds:
        config = replace(config, timeout_seconds=effective_timeout)

    return config
```

## 10. 工具过滤机制

### 10.1 过滤函数

```python
def _filter_tools(
    all_tools: list[BaseTool],
    allowed: list[str] | None,
    disallowed: list[str] | None,
) -> list[BaseTool]:
    """根据子代理配置过滤工具。

    Args:
        all_tools: 所有可用工具列表。
        allowed: 工具名称白名单。如果提供，只包含这些工具。
        disallowed: 工具名称黑名单。这些工具总是被排除。

    Returns:
        过滤后的工具列表。
    """
    filtered = all_tools

    # 应用白名单
    if allowed is not None:
        allowed_set = set(allowed)
        filtered = [t for t in filtered if t.name in allowed_set]

    # 应用黑名单
    if disallowed is not None:
        disallowed_set = set(disallowed)
        filtered = [t for t in filtered if t.name not in disallowed_set]

    return filtered
```

### 10.2 过滤策略

| 子代理类型 | 白名单 | 黑名单 | 效果 |
|-----------|--------|--------|------|
| `general-purpose` | `None` | `["task", "ask_clarification", "present_files"]` | 继承所有工具，禁止嵌套和澄清 |
| `bash` | `["bash", "ls", "read_file", "write_file", "str_replace"]` | `["task", "ask_clarification", "present_files"]` | 仅沙箱工具 |

## 11. 模型继承

```python
def _get_model_name(config: SubagentConfig, parent_model: str | None) -> str | None:
    """解析子代理的模型名称。

    Args:
        config: 子代理配置。
        parent_model: 父代理的模型名称。

    Returns:
        要使用的模型名称，None 表示使用默认。
    """
    if config.model == "inherit":
        return parent_model
    return config.model
```

**模型选择优先级：**
1. 如果 `config.model` 不是 `"inherit"`，使用指定的模型
2. 如果 `config.model` 是 `"inherit"`，继承父代理的模型
3. 如果父代理模型不可用，使用默认模型

## 12. 分布式追踪

每个子代理执行都有一个 `trace_id`，用于关联父代理和子代理的日志：

```python
# 生成或继承 trace_id
trace_id = metadata.get("trace_id") or str(uuid.uuid4())[:8]

# 日志中使用 trace_id
logger.info(f"[trace={self.trace_id}] Subagent {self.config.name} starting execution")
```

**日志示例：**
```
[trace=a1b2c3d4] Subagent general-purpose starting async execution with max_turns=50
[trace=a1b2c3d4] Subagent general-purpose captured AI message #1
[trace=a1b2c3d4] Task abc12345 status: running
[trace=a1b2c3d4] Subagent general-purpose completed async execution
[trace=a1b2c3d4] Task abc12345 completed after 12 polls
```

## 13. 错误处理

### 13.1 执行失败

```python
except Exception as e:
    logger.exception(f"[trace={self.trace_id}] Subagent {self.config.name} async execution failed")
    result.status = SubagentStatus.FAILED
    result.error = str(e)
    result.completed_at = datetime.now()
```

### 13.2 超时处理

```python
try:
    exec_result = execution_future.result(timeout=self.config.timeout_seconds)
except FuturesTimeoutError:
    logger.error(f"[trace={self.trace_id}] Subagent {self.config.name} execution timed out")
    result.status = SubagentStatus.TIMED_OUT
    result.error = f"Execution timed out after {self.config.timeout_seconds} seconds"
    result.completed_at = datetime.now()
    execution_future.cancel()  # 尽力取消
```

### 13.3 清理机制

```python
def cleanup_background_task(task_id: str) -> None:
    """从后台任务中移除已完成的任务。

    只有处于终端状态的任务才会被清理，以避免与后台执行器的竞争条件。
    """
    with _background_tasks_lock:
        result = _background_tasks.get(task_id)
        if result is None:
            return

        is_terminal_status = result.status in {
            SubagentStatus.COMPLETED,
            SubagentStatus.FAILED,
            SubagentStatus.TIMED_OUT,
        }
        if is_terminal_status or result.completed_at is not None:
            del _background_tasks[task_id]
```

## 14. 最佳实践

### 14.1 选择正确的子代理类型

- **general-purpose**：需要探索和行动的复杂任务
- **bash**：命令密集型操作，输出冗长的任务

### 14.2 设置合理的超时

```yaml
# config.yaml
subagents:
  timeout_seconds: 900  # 默认 15 分钟
  agents:
    general-purpose:
      timeout_seconds: 1200  # 复杂任务给更多时间
    bash:
      timeout_seconds: 300   # 命令执行通常更快
```

### 14.3 避免嵌套

系统默认禁止子代理调用 `task` 工具（在 `disallowed_tools` 中包含 `task`）。这防止了无限递归。

### 14.4 监控和调试

使用 `trace_id` 跟踪整个执行链：
```python
logger.info(f"[trace={trace_id}] Task {task_id} started")
logger.info(f"[trace={trace_id}] Subagent {config.name} captured AI message #{count}")
```