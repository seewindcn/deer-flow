# DeerFlow Agents 模块技术文档

## 1. 模块概述

### 1.1 模块职责和功能定位

`/mnt/e/dev/ai/deer-flow/backend/src/agents` 模块是 DeerFlow 框架的核心 Agent 系统，负责：

- **主代理创建与管理**：通过 `lead_agent` 子模块创建和配置主 Agent 实例
- **中间件系统**：提供可扩展的中间件机制，用于在 Agent 执行生命周期的各个阶段注入自定义逻辑
- **记忆系统**：实现用户上下文和对话历史的持久化存储与智能摘要
- **检查点系统**：支持对话状态的持久化存储，支持内存、SQLite、PostgreSQL 三种后端
- **线程状态管理**：定义和管理 Agent 执行过程中的状态数据结构

### 1.2 与其他模块的关系

```
┌─────────────────────────────────────────────────────────────────┐
│                        agents 模块                               │
├─────────────────────────────────────────────────────────────────┤
│  ┌─────────────┐  ┌──────────────┐  ┌───────────────┐          │
│  │ lead_agent  │  │ middlewares/ │  │    memory/    │          │
│  │   (创建)    │──│  (扩展处理)  │──│  (持久化)     │          │
│  └──────┬──────┘  └──────┬───────┘  └───────┬───────┘          │
│         │                │                   │                   │
│         └────────────────┼───────────────────┘                   │
│                          │                                       │
│  ┌───────────────────────┴───────────────────────┐              │
│  │              checkpointer/                     │              │
│  │            (状态检查点)                        │              │
│  └───────────────────────────────────────────────┘              │
└─────────────────────────────────────────────────────────────────┘
           │                    │                    │
           ▼                    ▼                    ▼
    ┌─────────────┐      ┌─────────────┐     ┌─────────────┐
    │   models/   │      │   tools/    │     │   config/   │
    │  (模型工厂) │      │  (工具系统) │     │  (配置管理) │
    └─────────────┘      └─────────────┘     └─────────────┘
```

**依赖关系**：
- **models**：调用 `create_chat_model()` 创建 LLM 实例
- **tools**：调用 `get_available_tools()` 获取可用工具列表
- **config**：读取应用配置、模型配置、记忆配置等
- **sandbox**：集成沙箱中间件实现隔离执行
- **skills**：加载技能配置注入到系统提示词

---

## 2. 目录结构

```
backend/src/agents/
├── __init__.py                    # 模块入口，导出主要 API
├── thread_state.py                # 线程状态定义和数据结构
│
├── lead_agent/                    # 主代理系统
│   ├── __init__.py               # 导出 make_lead_agent
│   ├── agent.py                  # Agent 创建核心逻辑
│   └── prompt.py                 # 系统提示词模板
│
├── middlewares/                   # 中间件系统
│   ├── clarification_middleware.py   # 澄清请求拦截
│   ├── dangling_tool_call_middleware.py  # 悬空工具调用修复
│   ├── memory_middleware.py       # 记忆更新队列
│   ├── subagent_limit_middleware.py   # 子代理并发限制
│   ├── thread_data_middleware.py  # 线程数据目录管理
│   ├── title_middleware.py        # 自动标题生成
│   ├── todo_middleware.py         # Todo 列表管理
│   ├── uploads_middleware.py      # 上传文件注入
│   └── view_image_middleware.py   # 图像查看注入
│
├── memory/                        # 记忆系统
│   ├── __init__.py               # 导出记忆相关 API
│   ├── prompt.py                 # 记忆更新提示词
│   ├── queue.py                  # 记忆更新队列（防抖）
│   └── updater.py                # 记忆读写和更新逻辑
│
└── checkpointer/                  # 检查点系统
    ├── __init__.py               # 导出检查点 API
    ├── provider.py               # 同步检查点工厂
    └── async_provider.py         # 异步检查点工厂
```

---

## 3. 核心组件详解

### 3.1 lead_agent/ 主代理系统

#### 3.1.1 agent.py - Agent 创建核心

**核心函数**：`make_lead_agent(config: RunnableConfig)`

```python
def make_lead_agent(config: RunnableConfig):
    """
    创建并配置主 Agent 实例。
    
    执行流程：
    1. 解析运行时配置（模型名称、思考模式、计划模式等）
    2. 加载 Agent 特定配置（如果有）
    3. 解析模型名称（请求覆盖 -> Agent 配置 -> 全局默认）
    4. 构建中间件链
    5. 应用系统提示词模板
    6. 调用 langchain.agents.create_agent() 创建 Agent
    
    Args:
        config: 运行时配置，包含 configurable 字典
        
    Returns:
        配置完成的 Agent 实例
    """
```

**配置解析优先级**：
```python
# 1. 请求级配置（最高优先级）
requested_model_name = cfg.get("model_name") or cfg.get("model")

# 2. Agent 级配置
agent_model_name = agent_config.model if agent_config else None

# 3. 全局默认配置
default_model_name = app_config.models[0].name
```

**关键配置项**：
| 配置项 | 类型 | 默认值 | 说明 |
|--------|------|--------|------|
| `thinking_enabled` | bool | True | 是否启用思考模式 |
| `reasoning_effort` | str | None | 推理努力程度 |
| `model_name` / `model` | str | None | 指定模型名称 |
| `is_plan_mode` | bool | False | 是否启用计划模式 |
| `subagent_enabled` | bool | False | 是否启用子代理 |
| `max_concurrent_subagents` | int | 3 | 最大并发子代理数 |
| `is_bootstrap` | bool | False | 是否为引导 Agent |
| `agent_name` | str | None | 自定义 Agent 名称 |

#### 3.1.2 prompt.py - 系统提示词模板

**核心结构**：

```python
SYSTEM_PROMPT_TEMPLATE = """
<role>
You are {agent_name}, an open-source super agent.
</role>

{soul}                    # Agent 个性（来自 SOUL.md）
{memory_context}          # 记忆上下文注入

<thinking_style>
- 思考策略指导
{subagent_thinking}       # 子代理思考指导（条件注入）
</thinking_style>

<clarification_system>
- 澄清请求系统说明
- 强调 CLARIFY → PLAN → ACT 工作流
</clarification_system>

{skills_section}          # 技能列表（条件注入）
{subagent_section}        # 子代理系统说明（条件注入）

<working_directory>
- 工作目录说明
</working_directory>

<response_style>
- 响应风格指导
</response_style>

<citations>
- 引用格式说明
</citations>

<critical_reminders>
- 关键提醒
{subagent_reminder}       # 子代理提醒（条件注入）
</critical_reminders>
"""
```

**动态部分生成函数**：

```python
def _build_subagent_section(max_concurrent: int) -> str:
    """构建子代理系统提示词，包含并发限制说明"""
    
def get_skills_prompt_section(available_skills: set[str] | None) -> str:
    """生成技能列表提示词部分"""
    
def _get_memory_context(agent_name: str | None) -> str:
    """获取记忆上下文用于注入系统提示词"""
    
def get_agent_soul(agent_name: str | None) -> str:
    """加载 Agent 个性（SOUL.md）"""
```

---

### 3.2 middlewares/ 中间件系统

中间件系统基于 `langchain.agents.middleware.AgentMiddleware` 基类，支持在 Agent 执行生命周期的各个阶段注入自定义逻辑。

#### 3.2.1 中间件执行顺序

```python
# 在 _build_middlewares() 中定义的执行顺序：
middlewares = [
    ThreadDataMiddleware(),           # 1. 线程数据目录初始化
    UploadsMiddleware(),              # 2. 上传文件注入
    SandboxMiddleware(),              # 3. 沙箱环境设置
    DanglingToolCallMiddleware(),     # 4. 悬空工具调用修复
    SummarizationMiddleware(),        # 5. 对话摘要（可选）
    TodoMiddleware(),                 # 6. Todo 列表管理（可选）
    TitleMiddleware(),                # 7. 自动标题生成
    MemoryMiddleware(),               # 8. 记忆更新队列
    ViewImageMiddleware(),            # 9. 图像查看注入（可选）
    SubagentLimitMiddleware(),        # 10. 子代理并发限制（可选）
    ClarificationMiddleware(),        # 11. 澄清请求拦截（最后）
]
```

#### 3.2.2 各中间件详解

##### (1) ThreadDataMiddleware - 线程数据目录管理

```python
class ThreadDataMiddleware(AgentMiddleware[ThreadDataMiddlewareState]):
    """
    为每个线程创建独立的数据目录结构。
    
    创建目录：
    - {base_dir}/threads/{thread_id}/user-data/workspace
    - {base_dir}/threads/{thread_id}/user-data/uploads
    - {base_dir}/threads/{thread_id}/user-data/outputs
    
    生命周期管理：
    - lazy_init=True（默认）：仅计算路径，按需创建目录
    - lazy_init=False：在 before_agent() 中立即创建目录
    """
    
    @override
    def before_agent(self, state, runtime) -> dict | None:
        """在 Agent 执行前初始化线程数据目录"""
        thread_id = runtime.context.get("thread_id")
        paths = self._get_thread_paths(thread_id)
        return {"thread_data": {**paths}}
```

##### (2) UploadsMiddleware - 上传文件注入

```python
class UploadsMiddleware(AgentMiddleware[UploadsMiddlewareState]):
    """
    将上传文件信息注入到 Agent 上下文中。
    
    功能：
    1. 从消息的 additional_kwargs.files 读取文件元数据
    2. 扫描线程上传目录获取历史文件
    3. 在用户消息前添加 <uploaded_files> 块
    
    输出格式：
    <uploaded_files>
    The following files were uploaded in this message:
    - filename.pdf (123.4 KB)
      Path: /mnt/user-data/uploads/filename.pdf
    </uploaded_files>
    """
    
    @override
    def before_agent(self, state, runtime) -> dict | None:
        """在 Agent 执行前注入上传文件信息"""
```

##### (3) DanglingToolCallMiddleware - 悬空工具调用修复

```python
class DanglingToolCallMiddleware(AgentMiddleware[AgentState]):
    """
    修复消息历史中的悬空工具调用。
    
    问题场景：
    - AIMessage 包含 tool_calls 但没有对应的 ToolMessage
    - 原因：用户中断、请求取消等
    
    解决方案：
    - 在悬空的 AIMessage 后插入合成 ToolMessage
    - 内容："[Tool call was interrupted and did not return a result.]"
    - 状态：status="error"
    """
    
    @override
    def wrap_model_call(self, request, handler) -> ModelCallResult:
        """在模型调用前修复悬空工具调用"""
        patched = self._build_patched_messages(request.messages)
        if patched is not None:
            request = request.override(messages=patched)
        return handler(request)
```

##### (4) MemoryMiddleware - 记忆更新队列

```python
class MemoryMiddleware(AgentMiddleware[MemoryMiddlewareState]):
    """
    在 Agent 执行后将对话加入记忆更新队列。
    
    功能：
    1. 过滤消息：仅保留用户输入和最终助手响应
    2. 移除 <uploaded_files> 块（会话级别，不持久化）
    3. 加入防抖队列，批量更新记忆
    
    消息过滤逻辑：
    - 保留：HumanMessage（清理后）、AIMessage（无 tool_calls）
    - 移除：ToolMessage、AIMessage（有 tool_calls）
    """
    
    @override
    def after_agent(self, state, runtime) -> dict | None:
        """在 Agent 执行后将对话加入记忆更新队列"""
        filtered_messages = _filter_messages_for_memory(messages)
        queue.add(thread_id=thread_id, messages=filtered_messages, 
                  agent_name=self._agent_name)
```

##### (5) TitleMiddleware - 自动标题生成

```python
class TitleMiddleware(AgentMiddleware[TitleMiddlewareState]):
    """
    在首次对话后自动生成线程标题。
    
    触发条件：
    - title_config.enabled = True
    - 线程尚无标题
    - 至少有一轮完整对话（1 用户消息 + 1 助手响应）
    
    生成方式：
    - 使用轻量级 LLM 模型
    - 基于首条用户消息和助手响应
    - 限制最大字符数和词数
    """
    
    @override
    async def aafter_model(self, state, runtime) -> dict | None:
        """在模型响应后生成标题"""
        if self._should_generate_title(state):
            title = await self._generate_title(state)
            return {"title": title}
```

##### (6) TodoMiddleware - Todo 列表管理

```python
class TodoMiddleware(TodoListMiddleware):
    """
    扩展 TodoListMiddleware，增加上下文丢失检测。
    
    问题场景：
    - 消息历史被截断（如 SummarizationMiddleware）
    - write_todos 工具调用被移出上下文窗口
    - 模型丢失对 Todo 列表的感知
    
    解决方案：
    - 检测 Todo 列表存在但 write_tools 调用不在消息中
    - 注入系统提醒消息，恢复模型对 Todo 的感知
    """
    
    @override
    def before_model(self, state, runtime) -> dict | None:
        """在模型调用前检测并注入 Todo 提醒"""
        if todos_exist and not write_todos_in_messages:
            reminder = HumanMessage(
                name="todo_reminder",
                content="<system_reminder>...todo list...</system_reminder>"
            )
            return {"messages": [reminder]}
```

##### (7) ViewImageMiddleware - 图像查看注入

```python
class ViewImageMiddleware(AgentMiddleware[ViewImageMiddlewareState]):
    """
    在 view_image 工具完成后将图像注入对话。
    
    功能：
    1. 检测上一轮助手消息是否包含 view_image 工具调用
    2. 验证所有工具调用是否已完成
    3. 创建包含图像 base64 数据的 HumanMessage
    4. 使 LLM 能够"看到"并分析图像
    
    条件注入：
    - 仅当模型支持视觉能力时添加
    - 通过 model_config.supports_vision 判断
    """
    
    @override
    def before_model(self, state, runtime) -> dict | None:
        """在模型调用前注入图像详情"""
        if self._should_inject_image_message(state):
            image_content = self._create_image_details_message(state)
            return {"messages": [HumanMessage(content=image_content)]}
```

##### (8) SubagentLimitMiddleware - 子代理并发限制

```python
class SubagentLimitMiddleware(AgentMiddleware[AgentState]):
    """
    强制限制单次响应中的并发子代理调用数。
    
    功能：
    - 当 LLM 生成超过 max_concurrent 个 task 工具调用时
    - 保留前 max_concurrent 个，丢弃其余
    - 比提示词限制更可靠
    
    有效范围：[2, 4]，默认 3
    """
    
    @override
    def after_model(self, state, runtime) -> dict | None:
        """在模型响应后截断超出的 task 调用"""
        task_indices = [i for i, tc in enumerate(tool_calls) 
                        if tc.get("name") == "task"]
        if len(task_indices) > self.max_concurrent:
            truncated_tool_calls = [...]  # 保留前 N 个
            return {"messages": [updated_msg]}
```

##### (9) ClarificationMiddleware - 澄清请求拦截

```python
class ClarificationMiddleware(AgentMiddleware[ClarificationMiddlewareState]):
    """
    拦截 ask_clarification 工具调用，中断执行并呈现问题。
    
    功能：
    1. 检测 ask_clarification 工具调用
    2. 提取问题和元数据
    3. 格式化为用户友好的消息
    4. 返回 Command 中断执行
    
    澄清类型：
    - missing_info: 缺失信息
    - ambiguous_requirement: 模糊需求
    - approach_choice: 方法选择
    - risk_confirmation: 风险确认
    - suggestion: 建议
    """
    
    @override
    def wrap_tool_call(self, request, handler) -> ToolMessage | Command:
        """拦截 ask_clarification 工具调用"""
        if request.tool_call.get("name") == "ask_clarification":
            return self._handle_clarification(request)
        return handler(request)
```

---

### 3.3 memory/ 记忆系统

#### 3.3.1 系统架构

```
┌─────────────────────────────────────────────────────────────┐
│                     Memory System                            │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  ┌──────────────┐    ┌──────────────┐    ┌──────────────┐  │
│  │   prompt.py  │    │   queue.py   │    │  updater.py  │  │
│  │  (提示词)    │    │  (防抖队列)  │    │  (读写更新)  │  │
│  └──────────────┘    └──────────────┘    └──────────────┘  │
│         │                   │                    │          │
│         └───────────────────┼────────────────────┘          │
│                             │                               │
│                    ┌────────┴────────┐                      │
│                    │  memory.json    │                      │
│                    │  (持久化存储)   │                      │
│                    └─────────────────┘                      │
└─────────────────────────────────────────────────────────────┘
```

#### 3.3.2 queue.py - 防抖队列

```python
class MemoryUpdateQueue:
    """
    带防抖机制的记忆更新队列。
    
    功能：
    1. 收集对话上下文
    2. 在防抖窗口内批量处理
    3. 同一线程的新消息替换旧消息
    
    防抖机制：
    - 每次添加消息时重置计时器
    - debounce_seconds 秒后触发处理
    - 避免频繁调用 LLM 更新记忆
    """
    
    def add(self, thread_id: str, messages: list, agent_name: str | None):
        """添加对话到更新队列"""
        # 移除同一线程的旧消息
        self._queue = [c for c in self._queue if c.thread_id != thread_id]
        self._queue.append(context)
        self._reset_timer()  # 重置防抖计时器
```

#### 3.3.3 updater.py - 记忆读写更新

```python
class MemoryUpdater:
    """
    使用 LLM 更新记忆的核心类。
    
    记忆结构：
    {
        "version": "1.0",
        "lastUpdated": "2024-01-01T00:00:00Z",
        "user": {
            "workContext": {"summary": "...", "updatedAt": "..."},
            "personalContext": {"summary": "...", "updatedAt": "..."},
            "topOfMind": {"summary": "...", "updatedAt": "..."}
        },
        "history": {
            "recentMonths": {"summary": "...", "updatedAt": "..."},
            "earlierContext": {"summary": "...", "updatedAt": "..."},
            "longTermBackground": {"summary": "...", "updatedAt": "..."}
        },
        "facts": [
            {
                "id": "fact_xxx",
                "content": "...",
                "category": "preference|knowledge|context|behavior|goal",
                "confidence": 0.9,
                "createdAt": "...",
                "source": "thread_id"
            }
        ]
    }
    """
    
    def update_memory(self, messages, thread_id, agent_name) -> bool:
        """基于对话更新记忆"""
        # 1. 获取当前记忆
        current_memory = get_memory_data(agent_name)
        
        # 2. 格式化对话
        conversation_text = format_conversation_for_update(messages)
        
        # 3. 构建 LLM 提示词
        prompt = MEMORY_UPDATE_PROMPT.format(...)
        
        # 4. 调用 LLM 获取更新
        response = model.invoke(prompt)
        update_data = json.loads(response.content)
        
        # 5. 应用更新
        updated_memory = self._apply_updates(current_memory, update_data)
        
        # 6. 清理上传文件提及
        updated_memory = _strip_upload_mentions_from_memory(updated_memory)
        
        # 7. 保存
        return _save_memory_to_file(updated_memory, agent_name)
```

**缓存机制**：
```python
# 基于 agent_name 的缓存，值：(memory_data, file_mtime)
_memory_cache: dict[str | None, tuple[dict[str, Any], float | None]] = {}

def get_memory_data(agent_name: str | None = None) -> dict[str, Any]:
    """获取记忆数据（带文件修改时间检查的缓存）"""
    current_mtime = file_path.stat().st_mtime if file_path.exists() else None
    cached = _memory_cache.get(agent_name)
    
    # 文件已修改或缓存不存在，重新加载
    if cached is None or cached[1] != current_mtime:
        memory_data = _load_memory_from_file(agent_name)
        _memory_cache[agent_name] = (memory_data, current_mtime)
        return memory_data
    
    return cached[0]
```

---

### 3.4 checkpointer/ 检查点系统

#### 3.4.1 系统架构

```
┌─────────────────────────────────────────────────────────────┐
│                    Checkpointer System                       │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  ┌─────────────────┐         ┌─────────────────┐           │
│  │  provider.py    │         │ async_provider  │           │
│  │  (同步工厂)     │         │    .py          │           │
│  │                 │         │  (异步工厂)     │           │
│  └────────┬────────┘         └────────┬────────┘           │
│           │                           │                     │
│           │  ┌────────────────────────┼────────────────┐   │
│           │  │                        │                │   │
│           ▼  ▼                        ▼                ▼   │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐        │
│  │   memory    │  │   sqlite    │  │  postgres   │        │
│  │ InMemory    │  │ SqliteSaver │  │PostgresSaver│        │
│  │   Saver     │  │             │  │             │        │
│  └─────────────┘  └─────────────┘  └─────────────┘        │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

#### 3.4.2 provider.py - 同步检查点工厂

```python
# 全局单例
_checkpointer: Checkpointer | None = None
_checkpointer_ctx = None

def get_checkpointer() -> Checkpointer:
    """
    获取全局同步检查点单例。
    
    返回：
    - 配置的检查点实例
    - 未配置时返回 InMemorySaver
    """
    if _checkpointer is not None:
        return _checkpointer
    
    config = get_checkpointer_config()
    if config is None:
        return InMemorySaver()
    
    _checkpointer_ctx = _sync_checkpointer_cm(config)
    _checkpointer = _checkpointer_ctx.__enter__()
    return _checkpointer

@contextlib.contextmanager
def checkpointer_context() -> Iterator[Checkpointer]:
    """
    同步上下文管理器，每次创建新连接。
    
    用法：
        with checkpointer_context() as cp:
            graph.invoke(input, config={"configurable": {"thread_id": "1"}})
    """
```

#### 3.4.3 async_provider.py - 异步检查点工厂

```python
@contextlib.asynccontextmanager
async def make_checkpointer() -> AsyncIterator[Checkpointer]:
    """
    异步上下文管理器，用于长时间运行的服务器。
    
    用法（FastAPI lifespan）：
        async with make_checkpointer() as checkpointer:
            app.state.checkpointer = checkpointer
    
    资源在进入时打开，退出时关闭，无全局状态。
    """
    config = get_app_config()
    if config.checkpointer is None:
        yield InMemorySaver()
        return
    
    async with _async_checkpointer(config.checkpointer) as saver:
        yield saver
```

---

## 4. 核心类和数据结构

### 4.1 ThreadState 线程状态

```python
class SandboxState(TypedDict):
    """沙箱状态"""
    sandbox_id: NotRequired[str | None]

class ThreadDataState(TypedDict):
    """线程数据状态"""
    workspace_path: NotRequired[str | None]
    uploads_path: NotRequired[str | None]
    outputs_path: NotRequired[str | None]

class ViewedImageData(TypedDict):
    """已查看图像数据"""
    base64: str
    mime_type: str

class ThreadState(AgentState):
    """
    线程状态，继承自 LangChain AgentState。
    
    字段说明：
    - sandbox: 沙箱状态（沙箱 ID）
    - thread_data: 线程数据目录路径
    - title: 线程标题
    - artifacts: 产物列表（带合并去重）
    - todos: Todo 列表
    - uploaded_files: 上传文件列表
    - viewed_images: 已查看图像字典（带合并）
    """
    sandbox: NotRequired[SandboxState | None]
    thread_data: NotRequired[ThreadDataState | None]
    title: NotRequired[str | None]
    artifacts: Annotated[list[str], merge_artifacts]
    todos: NotRequired[list | None]
    uploaded_files: NotRequired[list[dict] | None]
    viewed_images: Annotated[dict[str, ViewedImageData], merge_viewed_images]
```

### 4.2 状态合并器

```python
def merge_artifacts(existing: list[str] | None, new: list[str] | None) -> list[str]:
    """
    产物列表合并器 - 合并并去重。
    
    特点：
    - 使用 dict.fromkeys 保持顺序
    - 新元素追加到末尾
    """
    if existing is None:
        return new or []
    if new is None:
        return existing
    return list(dict.fromkeys(existing + new))

def merge_viewed_images(
    existing: dict[str, ViewedImageData] | None, 
    new: dict[str, ViewedImageData] | None
) -> dict[str, ViewedImageData]:
    """
    已查看图像字典合并器。
    
    特殊情况：
    - new = {} 时清空所有图像
    - 允许中间件处理后清空 viewed_images 状态
    """
    if existing is None:
        return new or {}
    if new is None:
        return existing
    if len(new) == 0:  # 空字典表示清空
        return {}
    return {**existing, **new}  # 新值覆盖旧值
```

---

## 5. 核心函数

### 5.1 make_lead_agent() 完整实现

```python
def make_lead_agent(config: RunnableConfig):
    """
    创建主 Agent 实例的工厂函数。
    
    执行流程：
    ┌─────────────────────────────────────────────────────────┐
    │ 1. 解析运行时配置                                        │
    │    - thinking_enabled, reasoning_effort                 │
    │    - model_name, is_plan_mode, subagent_enabled         │
    │    - is_bootstrap, agent_name                           │
    └─────────────────────────────────────────────────────────┘
                              │
                              ▼
    ┌─────────────────────────────────────────────────────────┐
    │ 2. 加载 Agent 配置（非 bootstrap）                       │
    │    - load_agent_config(agent_name)                      │
    │    - 获取 Agent 特定的模型和工具组配置                   │
    └─────────────────────────────────────────────────────────┘
                              │
                              ▼
    ┌─────────────────────────────────────────────────────────┐
    │ 3. 解析模型名称                                          │
    │    优先级：请求 > Agent 配置 > 全局默认                  │
    │    验证模型是否支持 thinking（如启用）                   │
    └─────────────────────────────────────────────────────────┘
                              │
                              ▼
    ┌─────────────────────────────────────────────────────────┐
    │ 4. 注入 LangSmith 元数据                                 │
    │    - agent_name, model_name, thinking_enabled           │
    │    - reasoning_effort, is_plan_mode, subagent_enabled   │
    └─────────────────────────────────────────────────────────┘
                              │
                              ▼
    ┌─────────────────────────────────────────────────────────┐
    │ 5. 创建 Agent                                            │
    │    - create_chat_model() 创建模型                        │
    │    - get_available_tools() 获取工具                      │
    │    - _build_middlewares() 构建中间件链                   │
    │    - apply_prompt_template() 应用提示词                  │
    └─────────────────────────────────────────────────────────┘
    
    Returns:
        配置完成的 Agent 实例
    """
    from src.tools import get_available_tools
    from src.tools.builtins import setup_agent
    
    cfg = config.get("configurable", {})
    
    # 1. 解析运行时配置
    thinking_enabled = cfg.get("thinking_enabled", True)
    reasoning_effort = cfg.get("reasoning_effort", None)
    requested_model_name = cfg.get("model_name") or cfg.get("model")
    is_plan_mode = cfg.get("is_plan_mode", False)
    subagent_enabled = cfg.get("subagent_enabled", False)
    max_concurrent_subagents = cfg.get("max_concurrent_subagents", 3)
    is_bootstrap = cfg.get("is_bootstrap", False)
    agent_name = cfg.get("agent_name")
    
    # 2. 加载 Agent 配置
    agent_config = load_agent_config(agent_name) if not is_bootstrap else None
    agent_model_name = agent_config.model if agent_config and agent_config.model else _resolve_model_name()
    
    # 3. 解析模型名称
    model_name = requested_model_name or agent_model_name
    model_config = app_config.get_model_config(model_name)
    
    # 4. 注入元数据
    config["metadata"].update({...})
    
    # 5. 创建 Agent
    if is_bootstrap:
        # Bootstrap Agent（用于创建自定义 Agent）
        return create_agent(
            model=create_chat_model(name=model_name, thinking_enabled=thinking_enabled),
            tools=get_available_tools(...) + [setup_agent],
            middleware=_build_middlewares(config, model_name=model_name),
            system_prompt=apply_prompt_template(subagent_enabled=..., available_skills={"bootstrap"}),
            state_schema=ThreadState,
        )
    
    # 默认 Agent
    return create_agent(
        model=create_chat_model(name=model_name, thinking_enabled=thinking_enabled, reasoning_effort=reasoning_effort),
        tools=get_available_tools(model_name=model_name, groups=agent_config.tool_groups if agent_config else None, subagent_enabled=subagent_enabled),
        middleware=_build_middlewares(config, model_name=model_name, agent_name=agent_name),
        system_prompt=apply_prompt_template(subagent_enabled=subagent_enabled, max_concurrent_subagents=max_concurrent_subagents, agent_name=agent_name),
        state_schema=ThreadState,
    )
```

### 5.2 中间件构建函数

```python
def _build_middlewares(config: RunnableConfig, model_name: str | None, agent_name: str | None = None):
    """
    构建中间件链。
    
    执行顺序（按添加顺序）：
    1. ThreadDataMiddleware - 线程数据目录
    2. UploadsMiddleware - 上传文件注入
    3. SandboxMiddleware - 沙箱环境
    4. DanglingToolCallMiddleware - 悬空工具调用修复
    5. SummarizationMiddleware - 对话摘要（可选）
    6. TodoMiddleware - Todo 列表（计划模式）
    7. TitleMiddleware - 自动标题
    8. MemoryMiddleware - 记忆更新
    9. ViewImageMiddleware - 图像注入（视觉模型）
    10. SubagentLimitMiddleware - 子代理限制（子代理模式）
    11. ClarificationMiddleware - 澄清拦截（最后）
    """
    middlewares = [
        ThreadDataMiddleware(),
        UploadsMiddleware(),
        SandboxMiddleware(),
        DanglingToolCallMiddleware(),
    ]
    
    # 可选中间件
    if summarization_middleware := _create_summarization_middleware():
        middlewares.append(summarization_middleware)
    
    if is_plan_mode:
        middlewares.append(_create_todo_list_middleware(is_plan_mode))
    
    middlewares.append(TitleMiddleware())
    middlewares.append(MemoryMiddleware(agent_name=agent_name))
    
    if model_config and model_config.supports_vision:
        middlewares.append(ViewImageMiddleware())
    
    if subagent_enabled:
        middlewares.append(SubagentLimitMiddleware(max_concurrent=max_concurrent_subagents))
    
    middlewares.append(ClarificationMiddleware())  # 必须最后
    
    return middlewares
```

---

## 6. 执行流程

### 6.1 Agent 创建流程

```
┌─────────────────────────────────────────────────────────────┐
│                    请求到达 Gateway                          │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│              调用 make_lead_agent(config)                   │
└─────────────────────────────────────────────────────────────┘
                              │
        ┌─────────────────────┼─────────────────────┐
        │                     │                     │
        ▼                     ▼                     ▼
┌───────────────┐   ┌───────────────┐   ┌───────────────┐
│ 解析配置      │   │ 加载 Agent    │   │ 解析模型      │
│ - thinking    │   │ 配置          │   │ 名称          │
│ - plan_mode   │   │ - tool_groups │   │ - 请求优先    │
│ - subagent    │   │ - model       │   │ - 验证能力    │
└───────────────┘   └───────────────┘   └───────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                  构建中间件链                                │
│  [ThreadData] → [Uploads] → [Sandbox] → [Dangling] →       │
│  [Summarization?] → [Todo?] → [Title] → [Memory] →         │
│  [ViewImage?] → [SubagentLimit?] → [Clarification]         │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                  应用系统提示词                              │
│  - Agent 个性（SOUL.md）                                    │
│  - 记忆上下文                                               │
│  - 技能列表                                                 │
│  - 子代理说明（可选）                                       │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│              调用 create_agent() 返回实例                    │
└─────────────────────────────────────────────────────────────┘
```

### 6.2 中间件执行顺序

```
请求生命周期中的中间件调用点：

┌─────────────────────────────────────────────────────────────┐
│                    before_agent                             │
│  ┌─────────────────────────────────────────────────────┐   │
│  │ ThreadDataMiddleware: 初始化线程目录                  │   │
│  │ UploadsMiddleware: 注入上传文件信息                   │   │
│  │ SandboxMiddleware: 设置沙箱环境                       │   │
│  └─────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                    before_model                             │
│  ┌─────────────────────────────────────────────────────┐   │
│  │ TodoMiddleware: 注入 Todo 提醒（如上下文丢失）        │   │
│  │ ViewImageMiddleware: 注入图像详情                     │   │
│  └─────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                    wrap_model_call                          │
│  ┌─────────────────────────────────────────────────────┐   │
│  │ DanglingToolCallMiddleware: 修复悬空工具调用          │   │
│  └─────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                    LLM 调用                                 │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                    after_model                              │
│  ┌─────────────────────────────────────────────────────┐   │
│  │ SubagentLimitMiddleware: 截断超出的 task 调用         │   │
│  └─────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                    wrap_tool_call                           │
│  ┌─────────────────────────────────────────────────────┐   │
│  │ ClarificationMiddleware: 拦截 ask_clarification      │   │
│  └─────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                    after_agent                              │
│  ┌─────────────────────────────────────────────────────┐   │
│  │ TitleMiddleware: 生成线程标题                         │   │
│  │ MemoryMiddleware: 队列记忆更新                        │   │
│  └─────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────┘
```

### 6.3 状态流转

```
ThreadState 状态流转：

┌─────────────────────────────────────────────────────────────┐
│                    初始状态                                  │
│  messages: [HumanMessage]                                   │
│  其他字段: None 或空                                         │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼ ThreadDataMiddleware
┌─────────────────────────────────────────────────────────────┐
│  thread_data: {                                             │
│    workspace_path: "/mnt/user-data/workspace",              │
│    uploads_path: "/mnt/user-data/uploads",                  │
│    outputs_path: "/mnt/user-data/outputs"                   │
│  }                                                          │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼ UploadsMiddleware
┌─────────────────────────────────────────────────────────────┐
│  uploaded_files: [                                          │
│    {filename, size, path, extension}                        │
│  ]                                                          │
│  messages: [<uploaded_files>... + 原始内容]                  │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼ Agent 执行
┌─────────────────────────────────────────────────────────────┐
│  messages: [..., AIMessage, ToolMessage, ...]               │
│  artifacts: ["file1.md", "file2.py"]  # 合并累积            │
│  viewed_images: {"path": {base64, mime_type}}  # 合并累积   │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼ TitleMiddleware
┌─────────────────────────────────────────────────────────────┐
│  title: "Generated Thread Title"                            │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼ MemoryMiddleware
┌─────────────────────────────────────────────────────────────┐
│  （无状态变更，触发异步记忆更新）                            │
└─────────────────────────────────────────────────────────────┘
```

---

## 7. 配置项

### 7.1 Agent 配置

```yaml
# config.yaml
agents:
  - name: "custom-agent"
    model: "gpt-4"              # Agent 专用模型
    tool_groups:                # 工具组白名单
      - "file_operations"
      - "web_search"
```

### 7.2 记忆配置

```yaml
memory:
  enabled: true                 # 是否启用记忆
  storage_path: "memory.json"   # 存储路径
  model_name: "gpt-4o-mini"     # 记忆更新模型
  debounce_seconds: 5           # 防抖延迟
  max_facts: 100                # 最大事实数
  fact_confidence_threshold: 0.7  # 事实置信度阈值
  injection_enabled: true       # 是否注入到提示词
  max_injection_tokens: 2000    # 最大注入 token 数
```

### 7.3 检查点配置

```yaml
checkpointer:
  type: "sqlite"                # memory | sqlite | postgres
  connection_string: "store.db" # 连接字符串
```

### 7.4 标题配置

```yaml
title:
  enabled: true
  max_words: 5
  max_chars: 50
  prompt_template: |
    Generate a concise title...
```

### 7.5 摘要配置

```yaml
summarization:
  enabled: true
  model_name: "gpt-4o-mini"
  trigger: [[10, "message"]]    # 触发条件
  keep: [5, "message"]          # 保留数量
```

---

## 8. 设计模式

### 8.1 工厂模式

**应用场景**：`make_lead_agent()` 函数

```python
def make_lead_agent(config: RunnableConfig):
    """工厂函数，根据配置创建不同类型的 Agent"""
    if is_bootstrap:
        return create_agent(...)  # Bootstrap Agent
    return create_agent(...)      # 默认 Agent
```

### 8.2 中间件模式

**应用场景**：整个中间件系统

```python
class AgentMiddleware(Generic[StateT]):
    """中间件基类，定义生命周期钩子"""
    
    def before_agent(self, state, runtime) -> dict | None: ...
    def after_agent(self, state, runtime) -> dict | None: ...
    def before_model(self, state, runtime) -> dict | None: ...
    def after_model(self, state, runtime) -> dict | None: ...
    def wrap_model_call(self, request, handler) -> ModelCallResult: ...
    def wrap_tool_call(self, request, handler) -> ToolMessage | Command: ...
```

**优点**：
- 开闭原则：新增功能无需修改核心代码
- 单一职责：每个中间件专注一个功能
- 可配置性：根据运行时配置动态组装

### 8.3 单例模式

**应用场景**：检查点和记忆队列

```python
# 全局单例
_checkpointer: Checkpointer | None = None
_memory_queue: MemoryUpdateQueue | None = None

def get_checkpointer() -> Checkpointer:
    """获取全局检查点单例"""
    global _checkpointer
    if _checkpointer is None:
        _checkpointer = _create_checkpointer()
    return _checkpointer

def get_memory_queue() -> MemoryUpdateQueue:
    """获取全局记忆队列单例"""
    global _memory_queue
    if _memory_queue is None:
        _memory_queue = MemoryUpdateQueue()
    return _memory_queue
```

### 8.4 策略模式

**应用场景**：检查点后端选择

```python
@contextlib.contextmanager
def _sync_checkpointer_cm(config: CheckpointerConfig):
    """根据配置类型选择不同策略"""
    if config.type == "memory":
        yield InMemorySaver()
    elif config.type == "sqlite":
        with SqliteSaver.from_conn_string(conn_str) as saver:
            yield saver
    elif config.type == "postgres":
        with PostgresSaver.from_conn_string(conn_str) as saver:
            yield saver
```

### 8.5 观察者模式

**应用场景**：记忆更新队列

```python
class MemoryUpdateQueue:
    """观察者模式的变体 - 防抖队列"""
    
    def add(self, thread_id, messages, agent_name):
        """添加观察者（对话上下文）"""
        self._queue.append(context)
        self._reset_timer()  # 通知处理
    
    def _process_queue(self):
        """处理所有待处理的观察者"""
        for context in contexts_to_process:
            updater.update_memory(...)
```

### 8.6 模板方法模式

**应用场景**：系统提示词生成

```python
SYSTEM_PROMPT_TEMPLATE = """
<role>...</role>
{soul}
{memory_context}
{skills_section}
{subagent_section}
...
"""

def apply_prompt_template(...) -> str:
    """模板方法，填充动态部分"""
    return SYSTEM_PROMPT_TEMPLATE.format(
        agent_name=agent_name or "DeerFlow 2.0",
        soul=get_agent_soul(agent_name),
        memory_context=_get_memory_context(agent_name),
        skills_section=get_skills_prompt_section(available_skills),
        subagent_section=_build_subagent_section(n) if subagent_enabled else "",
        ...
    )
```

### 8.7 装饰器模式

**应用场景**：`wrap_model_call` 和 `wrap_tool_call`

```python
class DanglingToolCallMiddleware(AgentMiddleware):
    """装饰模型调用，添加悬空工具调用修复功能"""
    
    @override
    def wrap_model_call(self, request, handler):
        patched = self._build_patched_messages(request.messages)
        if patched is not None:
            request = request.override(messages=patched)
        return handler(request)  # 调用原始处理器
```

---

## 9. 总结

DeerFlow 的 `agents` 模块是一个设计精良的 Agent 系统，具有以下特点：

1. **高度可配置**：通过配置文件和运行时参数灵活控制 Agent 行为
2. **中间件架构**：采用责任链模式，支持功能扩展和定制
3. **记忆系统**：智能的用户上下文管理和对话摘要
4. **多后端支持**：检查点系统支持内存、SQLite、PostgreSQL
5. **状态管理**：使用 TypedDict 和 Annotated 实现类型安全的状态合并
6. **异步支持**：全面支持同步和异步操作

该模块是 DeerFlow 框架的核心，为上层应用提供了强大而灵活的 Agent 能力。