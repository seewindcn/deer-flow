# Memory Module 技术文档

## 1. 模块概述

### 1.1 职责和功能定位

Memory 模块是 DeerFlow 的**全局记忆机制**，负责存储和管理用户的上下文信息、对话历史和关键事实。该模块通过 LLM 自动从对话中提取和更新记忆，实现个性化的 Agent 响应。

**核心功能：**

1. **记忆存储**：将用户上下文、对话历史和事实存储在 `memory.json` 文件中
2. **智能更新**：使用 LLM 从对话中自动提取和更新记忆内容
3. **记忆注入**：将相关记忆注入到系统提示词中，实现个性化响应
4. **防抖批处理**：通过队列机制批量处理记忆更新，避免频繁调用 LLM
5. **上传事件过滤**：自动清理文件上传相关的事件记录，避免记录会话级别的临时信息

### 1.2 与其他模块的关系

```
┌─────────────────────────────────────────────────────────────────┐
│                        Memory Module                             │
│  ┌───────────┐  ┌───────────┐  ┌───────────┐  ┌───────────┐    │
│  │  queue.py │  │ updater.py│  │ prompt.py │  │ __init__.py│   │
│  └─────┬─────┘  └─────┬─────┘  └─────┬─────┘  └───────────┘    │
└────────┼──────────────┼──────────────┼──────────────────────────┘
         │              │              │
         ▼              ▼              ▼
┌────────────────┐ ┌─────────────┐ ┌─────────────────────┐
│ MemoryMiddleware│ │ MemoryConfig│ │ Lead Agent Prompt   │
│ (middleware)    │ │ (config)    │ │ (prompt injection)  │
└────────┬────────┘ └──────┬──────┘ └──────────┬──────────┘
         │                 │                    │
         ▼                 ▼                    ▼
┌─────────────────────────────────────────────────────────────┐
│                    Lead Agent (主 Agent)                     │
│  - 执行对话                                                  │
│  - 触发记忆更新                                              │
│  - 接收记忆注入                                              │
└─────────────────────────────────────────────────────────────┘
```

**模块依赖关系：**

| 模块 | 关系 | 说明 |
|------|------|------|
| `src.config.memory_config` | 配置依赖 | 提供 MemoryConfig 配置类 |
| `src.config.paths` | 路径依赖 | 提供记忆文件存储路径 |
| `src.models` | 模型依赖 | 提供 LLM 模型创建能力 |
| `src.agents.middlewares.memory_middleware` | 中间件 | 触发记忆更新队列 |
| `src.agents.lead_agent.prompt` | 提示词 | 注入记忆到系统提示词 |
| `src.gateway.routers.memory` | API 路由 | 提供 HTTP API 访问记忆数据 |

---

## 2. 目录结构

```
backend/src/agents/memory/
├── __init__.py          # 模块入口，导出公共 API
├── queue.py             # 记忆更新队列（防抖机制）
├── updater.py           # 记忆更新器（LLM 调用）
└── prompt.py            # 提示词模板（更新和注入）
```

### 2.1 文件说明

| 文件 | 行数 | 职责 |
|------|------|------|
| `__init__.py` | 44 | 模块入口，定义公共 API 导出 |
| `queue.py` | 195 | 记忆更新队列实现，包含防抖机制 |
| `updater.py` | 384 | 记忆更新器，负责读取/写入/更新记忆 |
| `prompt.py` | 273 | 提示词模板，格式化记忆数据 |

---

## 3. 记忆数据结构

### 3.1 memory.json 完整结构

```json
{
  "version": "1.0",
  "lastUpdated": "2024-01-15T10:30:00Z",
  "user": {
    "workContext": {
      "summary": "核心开发者，DeerFlow 项目（16k+ stars），技术栈：Python/TypeScript",
      "updatedAt": "2024-01-15T10:30:00Z"
    },
    "personalContext": {
      "summary": "双语用户（中英文），偏好简洁响应，对 AI 技术有深入研究",
      "updatedAt": "2024-01-15T10:30:00Z"
    },
    "topOfMind": {
      "summary": "正在进行 DeerFlow 记忆模块开发。同时研究 LangGraph 状态管理优化。关注 Agent 架构设计最佳实践。",
      "updatedAt": "2024-01-15T10:30:00Z"
    }
  },
  "history": {
    "recentMonths": {
      "summary": "最近三个月专注于 DeerFlow 框架开发，实现了记忆系统、MCP 集成、沙箱执行等核心功能。解决了多个并发问题和性能瓶颈。",
      "updatedAt": "2024-01-15T10:30:00Z"
    },
    "earlierContext": {
      "summary": "过去半年主要研究 LLM 应用架构，完成了多个 AI 项目原型。",
      "updatedAt": "2024-01-15T10:30:00Z"
    },
    "longTermBackground": {
      "summary": "资深全栈开发者，拥有丰富的分布式系统和 AI 应用开发经验。",
      "updatedAt": "2024-01-15T10:30:00Z"
    }
  },
  "facts": [
    {
      "id": "fact_abc12345",
      "content": "用户偏好 TypeScript 而非 JavaScript",
      "category": "preference",
      "confidence": 0.9,
      "createdAt": "2024-01-15T10:30:00Z",
      "source": "thread_xyz789"
    },
    {
      "id": "fact_def67890",
      "content": "用户在 DeerFlow 项目中有 16k+ GitHub stars",
      "category": "context",
      "confidence": 0.95,
      "createdAt": "2024-01-14T08:20:00Z",
      "source": "thread_abc123"
    }
  ]
}
```

### 3.2 user 部分（用户上下文）

用户上下文描述用户的当前状态，包含三个子部分：

| 字段 | 用途 | 长度建议 | 更新频率 |
|------|------|----------|----------|
| `workContext` | 工作/职业相关信息 | 2-3 句话 | 较低 |
| `personalContext` | 个人偏好、语言、兴趣 | 1-2 句话 | 较低 |
| `topOfMind` | 当前关注点/优先事项 | 3-5 句话（详细段落） | 最高 |

**workContext 示例：**

```json
{
  "workContext": {
    "summary": "核心贡献者，DeerFlow 项目（16k+ stars），主要技术栈：Python, TypeScript, LangGraph",
    "updatedAt": "2024-01-15T10:30:00Z"
  }
}
```

**topOfMind 更新逻辑：**

- 捕获 3-5 个并发关注主题
- 包含：主要工作、并行探索、学习/追踪兴趣
- 更新时整合新焦点，移除已完成/放弃的主题

### 3.3 history 部分（历史上下文）

历史上下文按时间维度组织，用于提供长期视角：

| 字段 | 时间范围 | 用途 | 长度建议 |
|------|----------|------|----------|
| `recentMonths` | 最近 1-3 个月 | 近期活动、技术探索、项目工作 | 4-6 句话或 1-2 段 |
| `earlierContext` | 3-12 个月前 | 历史模式、学习历程、已建立模式 | 3-5 句话或 1 段 |
| `longTermBackground` | 长期/基础 | 核心专业知识、长期兴趣、工作风格 | 2-4 句话 |

### 3.4 facts 数组

事实数组存储具体的、可量化的用户信息：

**事实结构：**

```python
{
    "id": "fact_abc12345",           # 唯一标识符
    "content": "具体事实内容",        # 事实描述
    "category": "preference",        # 分类
    "confidence": 0.9,               # 置信度 (0.0-1.0)
    "createdAt": "2024-01-15T10:30:00Z",  # 创建时间
    "source": "thread_xyz789"        # 来源线程 ID
}
```

**事实分类（category）：**

| 分类 | 说明 | 示例 |
|------|------|------|
| `preference` | 工具、风格、方法偏好 | "用户偏好 TypeScript" |
| `knowledge` | 专业知识、技术掌握 | "精通 LangGraph 状态管理" |
| `context` | 背景事实 | "DeerFlow 项目有 16k+ stars" |
| `behavior` | 工作模式、沟通习惯 | "用户偏好详细的技术解释" |
| `goal` | 目标、学习目标、项目野心 | "正在学习 MCP 协议" |

**置信度级别：**

| 置信度范围 | 含义 | 使用场景 |
|------------|------|----------|
| 0.9-1.0 | 明确陈述的事实 | 用户直接说 "我从事 X" |
| 0.7-0.8 | 从行为/讨论中强烈暗示 | 从代码风格推断偏好 |
| 0.5-0.6 | 推断的模式（谨慎使用） | 仅用于明确的模式识别 |

---

## 4. 记忆更新队列

### 4.1 MemoryUpdateQueue 类

`MemoryUpdateQueue` 实现了带防抖机制的记忆更新队列，确保多个对话更新被批处理。

**类定义：**

```python
class MemoryUpdateQueue:
    """Queue for memory updates with debounce mechanism.

    This queue collects conversation contexts and processes them after
    a configurable debounce period. Multiple conversations received within
    the debounce window are batched together.
    """

    def __init__(self):
        """Initialize the memory update queue."""
        self._queue: list[ConversationContext] = []
        self._lock = threading.Lock()
        self._timer: threading.Timer | None = None
        self._processing = False
```

**核心属性：**

| 属性 | 类型 | 说明 |
|------|------|------|
| `_queue` | `list[ConversationContext]` | 待处理的对话上下文队列 |
| `_lock` | `threading.Lock` | 线程安全锁 |
| `_timer` | `threading.Timer \| None` | 防抖定时器 |
| `_processing` | `bool` | 处理状态标志 |

### 4.2 ConversationContext 数据类

```python
@dataclass
class ConversationContext:
    """Context for a conversation to be processed for memory update."""

    thread_id: str                        # 线程 ID
    messages: list[Any]                   # 对话消息列表
    timestamp: datetime = field(default_factory=datetime.utcnow)  # 时间戳
    agent_name: str | None = None         # Agent 名称（用于 per-agent 记忆）
```

### 4.3 防抖机制实现

**add() 方法 - 添加对话到队列：**

```python
def add(self, thread_id: str, messages: list[Any], agent_name: str | None = None) -> None:
    """Add a conversation to the update queue.

    Args:
        thread_id: The thread ID.
        messages: The conversation messages.
        agent_name: If provided, memory is stored per-agent. If None, uses global memory.
    """
    config = get_memory_config()
    if not config.enabled:
        return

    context = ConversationContext(
        thread_id=thread_id,
        messages=messages,
        agent_name=agent_name,
    )

    with self._lock:
        # Check if this thread already has a pending update
        # If so, replace it with the newer one
        self._queue = [c for c in self._queue if c.thread_id != thread_id]
        self._queue.append(context)

        # Reset or start the debounce timer
        self._reset_timer()

    print(f"Memory update queued for thread {thread_id}, queue size: {len(self._queue)}")
```

**_reset_timer() 方法 - 重置防抖定时器：**

```python
def _reset_timer(self) -> None:
    """Reset the debounce timer."""
    config = get_memory_config()

    # Cancel existing timer if any
    if self._timer is not None:
        self._timer.cancel()

    # Start new timer
    self._timer = threading.Timer(
        config.debounce_seconds,
        self._process_queue,
    )
    self._timer.daemon = True
    self._timer.start()

    print(f"Memory update timer set for {config.debounce_seconds}s")
```

**防抖机制流程图：**

```
用户对话结束
     │
     ▼
MemoryMiddleware.after_agent()
     │
     ▼
queue.add(thread_id, messages)
     │
     ├─── 检查配置是否启用
     ├─── 创建 ConversationContext
     ├─── 移除同 thread_id 的旧更新
     ├─── 添加新上下文到队列
     └─── 重置防抖定时器
           │
           ▼
     等待 debounce_seconds
           │
           ▼
     _process_queue()
           │
           ├─── 批量处理队列中的所有上下文
           └─── 调用 MemoryUpdater.update_memory()
```

### 4.4 批处理逻辑

```python
def _process_queue(self) -> None:
    """Process all queued conversation contexts."""
    # Import here to avoid circular dependency
    from src.agents.memory.updater import MemoryUpdater

    with self._lock:
        if self._processing:
            # Already processing, reschedule
            self._reset_timer()
            return

        if not self._queue:
            return

        self._processing = True
        contexts_to_process = self._queue.copy()
        self._queue.clear()
        self._timer = None

    print(f"Processing {len(contexts_to_process)} queued memory updates")

    try:
        updater = MemoryUpdater()

        for context in contexts_to_process:
            try:
                print(f"Updating memory for thread {context.thread_id}")
                success = updater.update_memory(
                    messages=context.messages,
                    thread_id=context.thread_id,
                    agent_name=context.agent_name,
                )
                if success:
                    print(f"Memory updated successfully for thread {context.thread_id}")
                else:
                    print(f"Memory update skipped/failed for thread {context.thread_id}")
            except Exception as e:
                print(f"Error updating memory for thread {context.thread_id}: {e}")

            # Small delay between updates to avoid rate limiting
            if len(contexts_to_process) > 1:
                time.sleep(0.5)

    finally:
        with self._lock:
            self._processing = False
```

### 4.5 全局单例

```python
# Global singleton instance
_memory_queue: MemoryUpdateQueue | None = None
_queue_lock = threading.Lock()


def get_memory_queue() -> MemoryUpdateQueue:
    """Get the global memory update queue singleton.

    Returns:
        The memory update queue instance.
    """
    global _memory_queue
    with _queue_lock:
        if _memory_queue is None:
            _memory_queue = MemoryUpdateQueue()
        return _memory_queue


def reset_memory_queue() -> None:
    """Reset the global memory queue.

    This is useful for testing.
    """
    global _memory_queue
    with _queue_lock:
        if _memory_queue is not None:
            _memory_queue.clear()
        _memory_queue = None
```

---

## 5. 记忆更新器

### 5.1 MemoryUpdater 类

`MemoryUpdater` 是核心的记忆更新器，负责通过 LLM 更新记忆数据。

**类定义：**

```python
class MemoryUpdater:
    """Updates memory using LLM based on conversation context."""

    def __init__(self, model_name: str | None = None):
        """Initialize the memory updater.

        Args:
            model_name: Optional model name to use. If None, uses config or default.
        """
        self._model_name = model_name

    def _get_model(self):
        """Get the model for memory updates."""
        config = get_memory_config()
        model_name = self._model_name or config.model_name
        return create_chat_model(name=model_name, thinking_enabled=False)
```

### 5.2 update_memory() 方法

```python
def update_memory(self, messages: list[Any], thread_id: str | None = None, agent_name: str | None = None) -> bool:
    """Update memory based on conversation messages.

    Args:
        messages: List of conversation messages.
        thread_id: Optional thread ID for tracking source.
        agent_name: If provided, updates per-agent memory. If None, updates global memory.

    Returns:
        True if update was successful, False otherwise.
    """
    config = get_memory_config()
    if not config.enabled:
        return False

    if not messages:
        return False

    try:
        # Get current memory
        current_memory = get_memory_data(agent_name)

        # Format conversation for prompt
        conversation_text = format_conversation_for_update(messages)

        if not conversation_text.strip():
            return False

        # Build prompt
        prompt = MEMORY_UPDATE_PROMPT.format(
            current_memory=json.dumps(current_memory, indent=2),
            conversation=conversation_text,
        )

        # Call LLM
        model = self._get_model()
        response = model.invoke(prompt)
        response_text = str(response.content).strip()

        # Parse response
        # Remove markdown code blocks if present
        if response_text.startswith("```"):
            lines = response_text.split("\n")
            response_text = "\n".join(lines[1:-1] if lines[-1] == "```" else lines[1:])

        update_data = json.loads(response_text)

        # Apply updates
        updated_memory = self._apply_updates(current_memory, update_data, thread_id)

        # Strip file-upload mentions from all summaries before saving.
        updated_memory = _strip_upload_mentions_from_memory(updated_memory)

        # Save
        return _save_memory_to_file(updated_memory, agent_name)

    except json.JSONDecodeError as e:
        print(f"Failed to parse LLM response for memory update: {e}")
        return False
    except Exception as e:
        print(f"Memory update failed: {e}")
        return False
```

### 5.3 LLM 调用流程

```
update_memory()
       │
       ├─── 1. 获取当前记忆数据
       │         get_memory_data(agent_name)
       │
       ├─── 2. 格式化对话
       │         format_conversation_for_update(messages)
       │
       ├─── 3. 构建 LLM 提示词
       │         MEMORY_UPDATE_PROMPT.format(current_memory, conversation)
       │
       ├─── 4. 调用 LLM 模型
       │         model.invoke(prompt)
       │
       ├─── 5. 解析 JSON 响应
       │         json.loads(response_text)
       │
       ├─── 6. 应用更新
       │         _apply_updates(current_memory, update_data, thread_id)
       │
       ├─── 7. 清理上传事件
       │         _strip_upload_mentions_from_memory(updated_memory)
       │
       └─── 8. 保存到文件
                 _save_memory_to_file(updated_memory, agent_name)
```

### 5.4 _apply_updates() 方法

```python
def _apply_updates(
    self,
    current_memory: dict[str, Any],
    update_data: dict[str, Any],
    thread_id: str | None = None,
) -> dict[str, Any]:
    """Apply LLM-generated updates to memory.

    Args:
        current_memory: Current memory data.
        update_data: Updates from LLM.
        thread_id: Optional thread ID for tracking.

    Returns:
        Updated memory data.
    """
    config = get_memory_config()
    now = datetime.utcnow().isoformat() + "Z"

    # Update user sections
    user_updates = update_data.get("user", {})
    for section in ["workContext", "personalContext", "topOfMind"]:
        section_data = user_updates.get(section, {})
        if section_data.get("shouldUpdate") and section_data.get("summary"):
            current_memory["user"][section] = {
                "summary": section_data["summary"],
                "updatedAt": now,
            }

    # Update history sections
    history_updates = update_data.get("history", {})
    for section in ["recentMonths", "earlierContext", "longTermBackground"]:
        section_data = history_updates.get(section, {})
        if section_data.get("shouldUpdate") and section_data.get("summary"):
            current_memory["history"][section] = {
                "summary": section_data["summary"],
                "updatedAt": now,
            }

    # Remove facts
    facts_to_remove = set(update_data.get("factsToRemove", []))
    if facts_to_remove:
        current_memory["facts"] = [f for f in current_memory.get("facts", []) if f.get("id") not in facts_to_remove]

    # Add new facts
    new_facts = update_data.get("newFacts", [])
    for fact in new_facts:
        confidence = fact.get("confidence", 0.5)
        if confidence >= config.fact_confidence_threshold:
            fact_entry = {
                "id": f"fact_{uuid.uuid4().hex[:8]}",
                "content": fact.get("content", ""),
                "category": fact.get("category", "context"),
                "confidence": confidence,
                "createdAt": now,
                "source": thread_id or "unknown",
            }
            current_memory["facts"].append(fact_entry)

    # Enforce max facts limit
    if len(current_memory["facts"]) > config.max_facts:
        # Sort by confidence and keep top ones
        current_memory["facts"] = sorted(
            current_memory["facts"],
            key=lambda f: f.get("confidence", 0),
            reverse=True,
        )[: config.max_facts]

    return current_memory
```

### 5.5 上传事件清理

为了避免将会话级别的文件上传事件持久化到长期记忆中，模块实现了两层清理机制：

**1. 对话消息过滤（memory_middleware）：**

```python
def _filter_messages_for_memory(messages: list[Any]) -> list[Any]:
    """Filter messages to keep only user inputs and final assistant responses.

    This filters out:
    - Tool messages (intermediate tool call results)
    - AI messages with tool_calls (intermediate steps, not final responses)
    - The <uploaded_files> block injected by UploadsMiddleware into human messages
    """
    _UPLOAD_BLOCK_RE = re.compile(r"<uploaded_files>[\s\S]*?</uploaded_files>\n*", re.IGNORECASE)

    filtered = []
    skip_next_ai = False
    for msg in messages:
        msg_type = getattr(msg, "type", None)

        if msg_type == "human":
            content = getattr(msg, "content", "")
            # ... 处理逻辑
            if "<uploaded_files>" in content_str:
                stripped = _UPLOAD_BLOCK_RE.sub("", content_str).strip()
                if not stripped:
                    # 上传-only 消息，跳过
                    skip_next_ai = True
                    continue
                # 保留用户的真实问题
                clean_msg = copy(msg)
                clean_msg.content = stripped
                filtered.append(clean_msg)
        # ...
    return filtered
```

**2. 记忆内容清理（updater）：**

```python
# 正则表达式匹配上传事件句子
_UPLOAD_SENTENCE_RE = re.compile(
    r"[^.!?]*\b(?:"
    r"upload(?:ed|ing)?(?:\s+\w+){0,3}\s+(?:file|files?|document|documents?|attachment|attachments?)"
    r"|file\s+upload"
    r"|/mnt/user-data/uploads/"
    r"|<uploaded_files>"
    r")[^.!?]*[.!?]?\s*",
    re.IGNORECASE,
)


def _strip_upload_mentions_from_memory(memory_data: dict[str, Any]) -> dict[str, Any]:
    """Remove sentences about file uploads from all memory summaries and facts.

    Uploaded files are session-scoped; persisting upload events in long-term
    memory causes the agent to search for non-existent files in future sessions.
    """
    # Scrub summaries in user/history sections
    for section in ("user", "history"):
        section_data = memory_data.get(section, {})
        for _key, val in section_data.items():
            if isinstance(val, dict) and "summary" in val:
                cleaned = _UPLOAD_SENTENCE_RE.sub("", val["summary"]).strip()
                cleaned = re.sub(r"  +", " ", cleaned)
                val["summary"] = cleaned

    # Also remove any facts that describe upload events
    facts = memory_data.get("facts", [])
    if facts:
        memory_data["facts"] = [f for f in facts if not _UPLOAD_SENTENCE_RE.search(f.get("content", ""))]

    return memory_data
```

**保留 vs 删除示例：**

| 内容 | 结果 | 原因 |
|------|------|------|
| "User uploaded a test file" | 删除 | 上传事件 |
| "User works with CSV files" | 保留 | 工作习惯，非上传事件 |
| "User prefers PDF export" | 保留 | 偏好，非上传事件 |
| "/mnt/user-data/uploads/abc/file.txt" | 删除 | 上传路径 |

---

## 6. 记忆缓存机制

### 6.1 get_memory_data() 函数

记忆模块使用基于文件修改时间的缓存机制，确保数据一致性：

```python
# Per-agent memory cache: keyed by agent_name (None = global)
# Value: (memory_data, file_mtime)
_memory_cache: dict[str | None, tuple[dict[str, Any], float | None]] = {}


def get_memory_data(agent_name: str | None = None) -> dict[str, Any]:
    """Get the current memory data (cached with file modification time check).

    The cache is automatically invalidated if the memory file has been modified
    since the last load, ensuring fresh data is always returned.

    Args:
        agent_name: If provided, loads per-agent memory. If None, loads global memory.

    Returns:
        The memory data dictionary.
    """
    file_path = _get_memory_file_path(agent_name)

    # Get current file modification time
    try:
        current_mtime = file_path.stat().st_mtime if file_path.exists() else None
    except OSError:
        current_mtime = None

    cached = _memory_cache.get(agent_name)

    # Invalidate cache if file has been modified or doesn't exist
    if cached is None or cached[1] != current_mtime:
        memory_data = _load_memory_from_file(agent_name)
        _memory_cache[agent_name] = (memory_data, current_mtime)
        return memory_data

    return cached[0]
```

### 6.2 文件修改时间检测

```
首次调用 get_memory_data()
         │
         ▼
  缓存未命中
         │
         ├─── 读取文件
         ├─── 获取文件 mtime
         └─── 存入缓存: {agent_name: (data, mtime)}
         │
         ▼
    返回数据

后续调用 get_memory_data()
         │
         ▼
  缓存命中？检查 mtime
         │
    ┌────┴────┐
    │         │
    ▼         ▼
 mtime 相同  mtime 不同
    │         │
    ▼         ▼
 返回缓存   重新加载文件
```

### 6.3 reload_memory_data() 函数

强制重新加载记忆数据：

```python
def reload_memory_data(agent_name: str | None = None) -> dict[str, Any]:
    """Reload memory data from file, forcing cache invalidation.

    Args:
        agent_name: If provided, reloads per-agent memory. If None, reloads global memory.

    Returns:
        The reloaded memory data dictionary.
    """
    file_path = _get_memory_file_path(agent_name)
    memory_data = _load_memory_from_file(agent_name)

    try:
        mtime = file_path.stat().st_mtime if file_path.exists() else None
    except OSError:
        mtime = None

    _memory_cache[agent_name] = (memory_data, mtime)
    return memory_data
```

---

## 7. 系统提示词注入

### 7.1 注入流程

记忆数据通过 `_get_memory_context()` 函数注入到 Agent 的系统提示词中：

```python
def _get_memory_context(agent_name: str | None = None) -> str:
    """Get memory context for injection into system prompt.

    Args:
        agent_name: If provided, loads per-agent memory. If None, loads global memory.

    Returns:
        Formatted memory context string wrapped in XML tags, or empty string if disabled.
    """
    try:
        from src.agents.memory import format_memory_for_injection, get_memory_data
        from src.config.memory_config import get_memory_config

        config = get_memory_config()
        if not config.enabled or not config.injection_enabled:
            return ""

        memory_data = get_memory_data(agent_name)
        memory_content = format_memory_for_injection(memory_data, max_tokens=config.max_injection_tokens)

        if not memory_content.strip():
            return ""

        return f"""<memory>
{memory_content}
</memory>
"""
    except Exception as e:
        print(f"Failed to load memory context: {e}")
        return ""
```

### 7.2 format_memory_for_injection() 函数

```python
def format_memory_for_injection(memory_data: dict[str, Any], max_tokens: int = 2000) -> str:
    """Format memory data for injection into system prompt.

    Args:
        memory_data: The memory data dictionary.
        max_tokens: Maximum tokens to use (counted via tiktoken for accuracy).

    Returns:
        Formatted memory string for system prompt injection.
    """
    if not memory_data:
        return ""

    sections = []

    # Format user context
    user_data = memory_data.get("user", {})
    if user_data:
        user_sections = []

        work_ctx = user_data.get("workContext", {})
        if work_ctx.get("summary"):
            user_sections.append(f"Work: {work_ctx['summary']}")

        personal_ctx = user_data.get("personalContext", {})
        if personal_ctx.get("summary"):
            user_sections.append(f"Personal: {personal_ctx['summary']}")

        top_of_mind = user_data.get("topOfMind", {})
        if top_of_mind.get("summary"):
            user_sections.append(f"Current Focus: {top_of_mind['summary']}")

        if user_sections:
            sections.append("User Context:\n" + "\n".join(f"- {s}" for s in user_sections))

    # Format history
    history_data = memory_data.get("history", {})
    if history_data:
        history_sections = []

        recent = history_data.get("recentMonths", {})
        if recent.get("summary"):
            history_sections.append(f"Recent: {recent['summary']}")

        earlier = history_data.get("earlierContext", {})
        if earlier.get("summary"):
            history_sections.append(f"Earlier: {earlier['summary']}")

        if history_sections:
            sections.append("History:\n" + "\n".join(f"- {s}" for s in history_sections))

    if not sections:
        return ""

    result = "\n\n".join(sections)

    # Use accurate token counting with tiktoken
    token_count = _count_tokens(result)
    if token_count > max_tokens:
        # Truncate to fit within token limit
        char_per_token = len(result) / token_count
        target_chars = int(max_tokens * char_per_token * 0.95)  # 95% to leave margin
        result = result[:target_chars] + "\n..."

    return result
```

### 7.3 注入后的系统提示词示例

```xml
<memory>
User Context:
- Work: 核心开发者，DeerFlow 项目（16k+ stars），技术栈：Python/TypeScript
- Personal: 双语用户（中英文），偏好简洁响应，对 AI 技术有深入研究
- Current Focus: 正在进行 DeerFlow 记忆模块开发。同时研究 LangGraph 状态管理优化。

History:
- Recent: 最近三个月专注于 DeerFlow 框架开发，实现了记忆系统、MCP 集成、沙箱执行等核心功能。
- Earlier: 过去半年主要研究 LLM 应用架构，完成了多个 AI 项目原型。
</memory>
```

### 7.4 apply_prompt_template() 中的使用

```python
def apply_prompt_template(
    subagent_enabled: bool = False,
    max_concurrent_subagents: int = 3,
    *,
    agent_name: str | None = None,
    available_skills: set[str] | None = None
) -> str:
    # Get memory context
    memory_context = _get_memory_context(agent_name)

    # ... 其他提示词构建

    # 记忆上下文被注入到系统提示词中
    # 具体位置在 LEAD_AGENT_PROMPT 模板中
```

---

## 8. 配置项

### 8.1 MemoryConfig 配置类

```python
class MemoryConfig(BaseModel):
    """Configuration for global memory mechanism."""

    enabled: bool = Field(
        default=True,
        description="Whether to enable memory mechanism",
    )
    storage_path: str = Field(
        default="",
        description=(
            "Path to store memory data. "
            "If empty, defaults to `{base_dir}/memory.json` (see Paths.memory_file). "
            "Absolute paths are used as-is. "
            "Relative paths are resolved against `Paths.base_dir` "
            "(not the backend working directory). "
        ),
    )
    debounce_seconds: int = Field(
        default=30,
        ge=1,
        le=300,
        description="Seconds to wait before processing queued updates (debounce)",
    )
    model_name: str | None = Field(
        default=None,
        description="Model name to use for memory updates (None = use default model)",
    )
    max_facts: int = Field(
        default=100,
        ge=10,
        le=500,
        description="Maximum number of facts to store",
    )
    fact_confidence_threshold: float = Field(
        default=0.7,
        ge=0.0,
        le=1.0,
        description="Minimum confidence threshold for storing facts",
    )
    injection_enabled: bool = Field(
        default=True,
        description="Whether to inject memory into system prompt",
    )
    max_injection_tokens: int = Field(
        default=2000,
        ge=100,
        le=8000,
        description="Maximum tokens to use for memory injection",
    )
```

### 8.2 配置项说明

| 配置项 | 类型 | 默认值 | 范围 | 说明 |
|--------|------|--------|------|------|
| `enabled` | bool | `True` | - | 是否启用记忆机制 |
| `storage_path` | str | `""` | - | 记忆文件存储路径，空则使用默认路径 |
| `debounce_seconds` | int | `30` | 1-300 | 防抖等待秒数 |
| `model_name` | str \| None | `None` | - | 记忆更新使用的模型名称 |
| `max_facts` | int | `100` | 10-500 | 最大存储事实数量 |
| `fact_confidence_threshold` | float | `0.7` | 0.0-1.0 | 事实存储的最低置信度阈值 |
| `injection_enabled` | bool | `True` | - | 是否注入记忆到系统提示词 |
| `max_injection_tokens` | int | `2000` | 100-8000 | 记忆注入的最大 token 数 |

### 8.3 配置加载

```python
# Global configuration instance
_memory_config: MemoryConfig = MemoryConfig()


def get_memory_config() -> MemoryConfig:
    """Get the current memory configuration."""
    return _memory_config


def set_memory_config(config: MemoryConfig) -> None:
    """Set the memory configuration."""
    global _memory_config
    _memory_config = config


def load_memory_config_from_dict(config_dict: dict) -> None:
    """Load memory configuration from a dictionary."""
    global _memory_config
    _memory_config = MemoryConfig(**config_dict)
```

### 8.4 YAML 配置示例

```yaml
# config.yaml
memory:
  enabled: true
  storage_path: ""  # 使用默认路径
  debounce_seconds: 30
  model_name: null  # 使用默认模型
  max_facts: 100
  fact_confidence_threshold: 0.7
  injection_enabled: true
  max_injection_tokens: 2000
```

---

## 9. 文件存储路径

### 9.1 路径解析逻辑

```python
def _get_memory_file_path(agent_name: str | None = None) -> Path:
    """Get the path to the memory file.

    Args:
        agent_name: If provided, returns the per-agent memory file path.
                    If None, returns the global memory file path.

    Returns:
        Path to the memory file.
    """
    if agent_name is not None:
        return get_paths().agent_memory_file(agent_name)

    config = get_memory_config()
    if config.storage_path:
        p = Path(config.storage_path)
        # Absolute path: use as-is; relative path: resolve against base_dir
        return p if p.is_absolute() else get_paths().base_dir / p
    return get_paths().memory_file
```

### 9.2 存储位置

| 类型 | 路径 | 说明 |
|------|------|------|
| 全局记忆 | `{base_dir}/memory.json` | 默认全局记忆文件 |
| Agent 记忆 | `{base_dir}/agents/{agent_name}/memory.json` | Per-Agent 记忆文件 |

**base_dir 解析优先级：**

1. 构造函数参数 `base_dir`
2. 环境变量 `DEER_FLOW_HOME`
3. 本地开发回退：`cwd/.deer-flow`（当 cwd 是 backend/ 目录时）
4. 默认：`$HOME/.deer-flow`

---

## 10. API 接口

### 10.1 HTTP API 端点

| 端点 | 方法 | 说明 |
|------|------|------|
| `/api/memory` | GET | 获取全局记忆数据 |
| `/api/memory/reload` | POST | 重新加载记忆数据 |
| `/api/memory/config` | GET | 获取记忆配置 |
| `/api/memory/status` | GET | 获取记忆状态（配置+数据） |

### 10.2 响应模型

**MemoryResponse:**

```python
class MemoryResponse(BaseModel):
    """Response model for memory data."""
    version: str = Field(default="1.0", description="Memory schema version")
    lastUpdated: str = Field(default="", description="Last update timestamp")
    user: UserContext = Field(default_factory=UserContext)
    history: HistoryContext = Field(default_factory=HistoryContext)
    facts: list[Fact] = Field(default_factory=list)
```

**MemoryConfigResponse:**

```python
class MemoryConfigResponse(BaseModel):
    """Response model for memory configuration."""
    enabled: bool = Field(..., description="Whether memory is enabled")
    storage_path: str = Field(..., description="Path to memory storage file")
    debounce_seconds: int = Field(..., description="Debounce time for memory updates")
    max_facts: int = Field(..., description="Maximum number of facts to store")
    fact_confidence_threshold: float = Field(..., description="Minimum confidence threshold for facts")
    injection_enabled: bool = Field(..., description="Whether memory injection is enabled")
    max_injection_tokens: int = Field(..., description="Maximum tokens for memory injection")
```

---

## 11. 使用示例

### 11.1 基本使用

```python
from src.agents.memory import (
    get_memory_data,
    format_memory_for_injection,
    update_memory_from_conversation,
    get_memory_queue,
)

# 获取记忆数据
memory_data = get_memory_data()
print(f"当前关注点: {memory_data['user']['topOfMind']['summary']}")

# 格式化记忆用于注入
memory_text = format_memory_for_injection(memory_data, max_tokens=2000)

# 手动更新记忆
from langchain_core.messages import HumanMessage, AIMessage
messages = [
    HumanMessage(content="我正在学习 LangGraph"),
    AIMessage(content="很好的选择！LangGraph 是构建 Agent 应用的优秀框架。"),
]
update_memory_from_conversation(messages, thread_id="test_thread")
```

### 11.2 通过队列更新

```python
from src.agents.memory import get_memory_queue

# 获取队列实例
queue = get_memory_queue()

# 添加对话到队列（防抖处理）
queue.add(
    thread_id="thread_123",
    messages=messages,
    agent_name=None,  # 使用全局记忆
)

# 强制立即处理队列
queue.flush()

# 查看队列状态
print(f"待处理数量: {queue.pending_count}")
print(f"是否正在处理: {queue.is_processing}")
```

### 11.3 Per-Agent 记忆

```python
from src.agents.memory import get_memory_data, update_memory_from_conversation

# 获取特定 Agent 的记忆
agent_memory = get_memory_data(agent_name="code_reviewer")

# 更新特定 Agent 的记忆
update_memory_from_conversation(
    messages=messages,
    thread_id="thread_456",
    agent_name="code_reviewer",
)
```

---

## 12. 测试覆盖

### 12.1 上传事件过滤测试

```python
class TestFilterMessagesForMemory:
    def test_upload_only_turn_is_excluded(self):
        """纯上传消息（无真实问题）及其配对的 AI 响应都应被排除"""
        msgs = [
            _human(_UPLOAD_BLOCK),
            _ai("I have read the file. It says: Hello."),
        ]
        result = _filter_messages_for_memory(msgs)
        assert result == []

    def test_upload_with_real_question_preserves_question(self):
        """上传消息中包含真实问题时，保留问题（移除上传块）"""
        combined = _UPLOAD_BLOCK + "\n\nWhat does this file contain?"
        msgs = [
            _human(combined),
            _ai("The file contains: Hello DeerFlow."),
        ]
        result = _filter_messages_for_memory(msgs)

        assert len(result) == 2
        assert "<uploaded_files>" not in result[0].content
        assert "What does this file contain?" in result[0].content

    def test_tool_messages_are_excluded(self):
        """工具消息必须被过滤掉"""
        msgs = [
            _human("Search for something"),
            _ai("Calling search tool", tool_calls=[{"name": "search", "id": "1", "args": {}}]),
            ToolMessage(content="Search results", tool_call_id="1"),
            _ai("Here are the results."),
        ]
        result = _filter_messages_for_memory(msgs)
        ai_msgs = [m for m in result if m.type == "ai"]
        assert len(ai_msgs) == 1
        assert ai_msgs[0].content == "Here are the results."
```

### 12.2 记忆内容清理测试

```python
class TestStripUploadMentionsFromMemory:
    def test_upload_event_sentence_removed_from_summary(self):
        """上传事件句子从摘要中移除"""
        mem = self._make_memory(
            "User is interested in AI. User uploaded a test file for verification purposes. User prefers concise answers."
        )
        result = _strip_upload_mentions_from_memory(mem)
        summary = result["user"]["topOfMind"]["summary"]
        assert "uploaded a test file" not in summary
        assert "User is interested in AI" in summary

    def test_legitimate_csv_mention_is_preserved(self):
        """'User works with CSV files' 不应被删除——这不是上传事件"""
        mem = self._make_memory("User regularly works with CSV files for data analysis.")
        result = _strip_upload_mentions_from_memory(mem)
        assert "CSV files" in result["user"]["topOfMind"]["summary"]

    def test_upload_fact_removed_from_facts(self):
        """上传相关事实应被移除"""
        facts = [
            {"content": "User uploaded a file titled secret.txt", "category": "behavior"},
            {"content": "User prefers dark mode", "category": "preference"},
        ]
        mem = self._make_memory("summary", facts=facts)
        result = _strip_upload_mentions_from_memory(mem)
        remaining = [f["content"] for f in result["facts"]]
        assert "User prefers dark mode" in remaining
        assert not any("uploaded a file" in c for c in remaining)
```

---

## 13. 设计考量

### 13.1 为什么使用防抖机制？

1. **避免频繁 LLM 调用**：用户可能在短时间内进行多轮对话，防抖可以批量处理
2. **成本控制**：减少 LLM API 调用次数
3. **数据一致性**：避免并发更新冲突

### 13.2 为什么需要上传事件过滤？

1. **会话隔离**：上传的文件是会话级别的，不会在其他会话中存在
2. **避免幻觉**：防止 Agent 在后续会话中尝试访问不存在的文件
3. **保持记忆质量**：只保留有意义的信息

### 13.3 为什么使用文件修改时间缓存？

1. **自动失效**：文件被外部修改时自动重新加载
2. **性能优化**：避免频繁文件读取
3. **数据一致性**：确保总是获取最新的记忆数据

---

## 14. 扩展阅读

- [MEMORY_UPDATE_PROMPT 完整提示词](#promptpy-文件内容)
- [MemoryMiddleware 中间件文档](../middlewares/MIDDLEWARES_MODULE.md)
- [配置系统文档](../../docs/modules/CONFIG_MODULE.md)
- [Paths 路径管理文档](../../docs/modules/CONFIG_MODULE.md#paths-类)