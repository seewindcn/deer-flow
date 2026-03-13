# DeerFlow Checkpointer 模块技术文档

## 1. 模块概述

### 1.1 职责和功能定位

`/mnt/e/dev/ai/deer-flow/backend/src/agents/checkpointer` 模块是 DeerFlow 框架的状态持久化组件，负责：

- **状态持久化**：为 LangGraph Agent 提供对话状态的持久化存储
- **多后端支持**：支持 memory、sqlite、postgres 三种存储后端
- **生命周期管理**：提供同步和异步两种资源管理方式
- **配置驱动**：通过 `config.yaml` 配置文件灵活选择后端

### 1.2 核心价值

Checkpointer 模块解决了 Agent 系统的关键问题：

```
┌─────────────────────────────────────────────────────────────────┐
│                      Checkpointer 核心价值                       │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  1. 对话连续性：重启服务后恢复对话状态                              │
│  2. 多实例共享：多个服务实例共享同一对话状态                         │
│  3. 故障恢复：服务崩溃后恢复工作上下文                              │
│  4. 线程隔离：每个线程独立的状态空间                                │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### 1.3 与其他模块的关系

```
┌─────────────────────────────────────────────────────────────────┐
│                    Checkpointer 模块依赖关系                      │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ┌─────────────────┐                                            │
│  │   config 模块    │                                            │
│  │                 │                                            │
│  │ ┌─────────────┐ │     ┌─────────────────────────────────┐   │
│  │ │AppConfig    │ │     │      checkpointer 模块          │   │
│  │ │  ├─...      │ │     │                                 │   │
│  │ │  └─checkpoint│─┼────▶│  ┌───────────┐ ┌──────────────┐│   │
│  │ └─────────────┘ │     │  │ provider  │ │async_provider││   │
│  └─────────────────┘     │  │ (同步)    │ │  (异步)      ││   │
│                          │  └─────┬─────┘ └──────┬───────┘│   │
│                          │        │              │        │   │
│                          └────────┼──────────────┼────────┘   │
│                                   │              │             │
│                                   ▼              ▼             │
│                          ┌────────────────────────────┐       │
│                          │    LangGraph Checkpoint    │       │
│                          │  ┌──────────────────────┐  │       │
│                          │  │ InMemorySaver        │  │       │
│                          │  │ SqliteSaver          │  │       │
│                          │  │ PostgresSaver        │  │       │
│                          │  └──────────────────────┘  │       │
│                          └────────────────────────────┘       │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘

被依赖关系：
┌─────────────────┐     ┌─────────────────┐     ┌─────────────────┐
│   DeerFlowClient│────▶│  get_checkpointer│     │   LangGraph     │
│   (客户端)      │     │  make_checkpointer│────▶│   (Agent 图)    │
└─────────────────┘     └─────────────────┘     └─────────────────┘
```

**主要依赖**：
- `src.config.app_config`：获取应用配置
- `src.config.checkpointer_config`：获取检查点配置
- `src.config.paths`：解析相对路径

**主要被依赖**：
- `src.client`：DeerFlowClient 通过 get_checkpointer() 获取检查点器
- LangGraph Agent：通过 checkpointer 参数持久化状态

---

## 2. 目录结构

```
backend/src/agents/checkpointer/
├── __init__.py              # 模块入口，导出公共 API
├── provider.py              # 同步检查点提供者（单例 + 上下文管理器）
└── async_provider.py        # 异步检查点提供者（异步上下文管理器）
```

### 2.1 文件说明

| 文件 | 行数 | 说明 |
|------|------|------|
| `__init__.py` | 9 | 模块入口，导出 4 个公共函数 |
| `provider.py` | 201 | 同步工厂，包含单例模式和上下文管理器 |
| `async_provider.py` | 109 | 异步工厂，用于 FastAPI 等异步框架 |

### 2.2 公共 API

```python
# __init__.py 导出的公共接口
from .async_provider import make_checkpointer
from .provider import checkpointer_context, get_checkpointer, reset_checkpointer

__all__ = [
    "get_checkpointer",      # 获取同步单例
    "reset_checkpointer",    # 重置同步单例
    "checkpointer_context",  # 同步上下文管理器
    "make_checkpointer",     # 异步上下文管理器
]
```

---

## 3. 支持的后端类型

### 3.1 后端类型概述

Checkpointer 模块支持三种存储后端，满足不同场景需求：

```
┌─────────────────────────────────────────────────────────────────────┐
│                        Checkpointer 后端选择                         │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  ┌──────────────────────────────────────────────────────────────┐   │
│  │                      memory (内存)                            │   │
│  │                                                              │   │
│  │  特点：进程内存存储，重启丢失                                    │   │
│  │  场景：开发测试、无持久化需求                                    │   │
│  │  依赖：无额外依赖                                              │   │
│  │  实现：langgraph.checkpoint.memory.InMemorySaver              │   │
│  └──────────────────────────────────────────────────────────────┘   │
│                                                                      │
│  ┌──────────────────────────────────────────────────────────────┐   │
│  │                      sqlite (SQLite)                          │   │
│  │                                                              │   │
│  │  特点：本地文件存储，单机持久化                                  │   │
│  │  场景：单机部署、轻量级生产环境                                  │   │
│  │  依赖：langgraph-checkpoint-sqlite                            │   │
│  │  实现：langgraph.checkpoint.sqlite.SqliteSaver                │   │
│  │         langgraph.checkpoint.sqlite.aio.AsyncSqliteSaver      │   │
│  └──────────────────────────────────────────────────────────────┘   │
│                                                                      │
│  ┌──────────────────────────────────────────────────────────────┐   │
│  │                    postgres (PostgreSQL)                      │   │
│  │                                                              │   │
│  │  特点：数据库存储，支持多实例共享                                │   │
│  │  场景：生产集群、高可用部署                                      │   │
│  │  依赖：langgraph-checkpoint-postgres + psycopg[binary]        │   │
│  │  实现：langgraph.checkpoint.postgres.PostgresSaver            │   │
│  │         langgraph.checkpoint.postgres.aio.AsyncPostgresSaver  │   │
│  └──────────────────────────────────────────────────────────────┘   │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

### 3.2 后端对比

| 特性 | memory | sqlite | postgres |
|------|--------|--------|----------|
| 持久化 | 否 | 是 | 是 |
| 多实例共享 | 否 | 否 | 是 |
| 额外依赖 | 无 | langgraph-checkpoint-sqlite | langgraph-checkpoint-postgres + psycopg |
| 配置复杂度 | 最低 | 低 | 中 |
| 性能 | 最高 | 高 | 中 |
| 适用场景 | 开发/测试 | 单机生产 | 集群生产 |

### 3.3 依赖安装

```bash
# SQLite 后端
uv add langgraph-checkpoint-sqlite

# PostgreSQL 后端
uv add langgraph-checkpoint-postgres "psycopg[binary]" psycopg-pool
```

---

## 4. 同步检查点提供者 (provider.py)

### 4.1 模块设计

同步提供者提供两种使用方式：

```
┌─────────────────────────────────────────────────────────────────┐
│                    同步 Checkpointer 使用方式                     │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  方式一：全局单例模式                                             │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │                                                          │   │
│  │  cp = get_checkpointer()                                 │   │
│  │  # 在整个应用生命周期内复用同一个实例                          │   │
│  │  # 连接在进程退出时关闭                                    │   │
│  │                                                          │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                  │
│  方式二：上下文管理器模式                                         │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │                                                          │   │
│  │  with checkpointer_context() as cp:                      │   │
│  │      # 每次进入创建新连接                                   │   │
│  │      # 退出时自动关闭连接                                   │   │
│  │      graph.invoke(input, config={...})                   │   │
│  │                                                          │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### 4.2 错误消息常量

```python
# provider.py 中的错误消息常量

SQLITE_INSTALL = (
    "langgraph-checkpoint-sqlite is required for the SQLite checkpointer. "
    "Install it with: uv add langgraph-checkpoint-sqlite"
)

POSTGRES_INSTALL = (
    "langgraph-checkpoint-postgres is required for the PostgreSQL checkpointer. "
    "Install it with: uv add langgraph-checkpoint-postgres psycopg[binary] psycopg-pool"
)

POSTGRES_CONN_REQUIRED = "checkpointer.connection_string is required for the postgres backend"
```

### 4.3 SQLite 连接字符串解析

```python
def _resolve_sqlite_conn_str(raw: str) -> str:
    """
    将 SQLite 连接字符串解析为可用格式。
    
    处理三种情况：
    1. ":memory:" - 内存数据库，原样返回
    2. "file:..." - URI 格式，原样返回
    3. 文件路径 - 解析为绝对路径
    
    Args:
        raw: 原始连接字符串
        
    Returns:
        解析后的连接字符串
        
    Example:
        >>> _resolve_sqlite_conn_str(":memory:")
        ':memory:'
        >>> _resolve_sqlite_conn_str(".deer-flow/checkpoints.db")
        '/home/user/.deer-flow/checkpoints.db'
    """
    if raw == ":memory:" or raw.startswith("file:"):
        return raw
    return str(resolve_path(raw))
```

### 4.4 同步上下文管理器实现

```python
@contextlib.contextmanager
def _sync_checkpointer_cm(config: CheckpointerConfig) -> Iterator[Checkpointer]:
    """
    创建和销毁同步检查点的内部上下文管理器。
    
    根据配置类型创建对应的 Checkpointer 实例：
    
    - memory: 创建 InMemorySaver
    - sqlite: 创建 SqliteSaver 并调用 setup()
    - postgres: 创建 PostgresSaver 并调用 setup()
    
    Args:
        config: 检查点配置
        
    Yields:
        Checkpointer 实例
        
    Raises:
        ImportError: 缺少必要的依赖包
        ValueError: 配置错误（如缺少 connection_string）
    """
    # ========== memory 后端 ==========
    if config.type == "memory":
        from langgraph.checkpoint.memory import InMemorySaver
        
        logger.info("Checkpointer: using InMemorySaver (in-process, not persistent)")
        yield InMemorySaver()
        return
    
    # ========== sqlite 后端 ==========
    if config.type == "sqlite":
        try:
            from langgraph.checkpoint.sqlite import SqliteSaver
        except ImportError as exc:
            raise ImportError(SQLITE_INSTALL) from exc
        
        # 解析连接字符串（支持相对路径）
        conn_str = _resolve_sqlite_conn_str(config.connection_string or "store.db")
        
        # 使用 from_conn_string 创建并自动清理
        with SqliteSaver.from_conn_string(conn_str) as saver:
            saver.setup()  # 创建必要的表结构
            logger.info("Checkpointer: using SqliteSaver (%s)", conn_str)
            yield saver
        return
    
    # ========== postgres 后端 ==========
    if config.type == "postgres":
        try:
            from langgraph.checkpoint.postgres import PostgresSaver
        except ImportError as exc:
            raise ImportError(POSTGRES_INSTALL) from exc
        
        # PostgreSQL 必须提供连接字符串
        if not config.connection_string:
            raise ValueError(POSTGRES_CONN_REQUIRED)
        
        with PostgresSaver.from_conn_string(config.connection_string) as saver:
            saver.setup()  # 创建必要的表结构
            logger.info("Checkpointer: using PostgresSaver")
            yield saver
        return
    
    # 未知类型
    raise ValueError(f"Unknown checkpointer type: {config.type!r}")
```

### 4.5 单例模式实现

```python
# 全局单例状态
_checkpointer: Checkpointer | None = None
_checkpointer_ctx = None  # 保持连接活跃的上下文管理器


def get_checkpointer() -> Checkpointer:
    """
    获取全局同步检查点单例。
    
    首次调用时根据配置创建实例，后续调用返回缓存的实例。
    当配置中未设置 checkpointer 时，返回 InMemorySaver。
    
    Returns:
        Checkpointer 实例
        
    Raises:
        ImportError: 缺少必要的依赖包
        ValueError: 配置错误
        
    Example:
        >>> from src.agents.checkpointer import get_checkpointer
        >>> cp = get_checkpointer()
        >>> graph.invoke(input, config={"configurable": {"thread_id": "123"}})
    """
    global _checkpointer, _checkpointer_ctx
    
    # 已有实例，直接返回
    if _checkpointer is not None:
        return _checkpointer
    
    # 确保应用配置已加载
    from src.config.app_config import _app_config
    from src.config.checkpointer_config import get_checkpointer_config
    
    if _app_config is None:
        try:
            get_app_config()
        except FileNotFoundError:
            # 测试环境可能没有 config.yaml
            pass
    
    # 获取检查点配置
    config = get_checkpointer_config()
    
    # 未配置时使用 InMemorySaver
    if config is None:
        from langgraph.checkpoint.memory import InMemorySaver
        
        logger.info("Checkpointer: using InMemorySaver (in-process, not persistent)")
        _checkpointer = InMemorySaver()
        return _checkpointer
    
    # 创建检查点实例并保持上下文
    _checkpointer_ctx = _sync_checkpointer_cm(config)
    _checkpointer = _checkpointer_ctx.__enter__()
    
    return _checkpointer


def reset_checkpointer() -> None:
    """
    重置同步单例，强制下次调用时重新创建。
    
    关闭所有打开的后端连接，清除缓存的实例。
    用于测试或配置变更后重新初始化。
    
    Example:
        >>> reset_checkpointer()
        >>> cp = get_checkpointer()  # 将创建新实例
    """
    global _checkpointer, _checkpointer_ctx
    
    if _checkpointer_ctx is not None:
        try:
            _checkpointer_ctx.__exit__(None, None, None)
        except Exception:
            logger.warning("Error during checkpointer cleanup", exc_info=True)
        _checkpointer_ctx = None
    
    _checkpointer = None
```

### 4.6 公共上下文管理器

```python
@contextlib.contextmanager
def checkpointer_context() -> Iterator[Checkpointer]:
    """
    同步上下文管理器，每次创建独立的检查点实例。
    
    与 get_checkpointer() 不同，此方法不缓存实例，
    每次 with 块都会创建和销毁独立连接。
    适用于 CLI 脚本或测试场景，确保资源确定性的清理。
    
    Yields:
        Checkpointer 实例（未配置时返回 InMemorySaver）
        
    Example:
        >>> with checkpointer_context() as cp:
        ...     result = graph.invoke(
        ...         {"messages": [HumanMessage(content="Hello")]},
        ...         config={"configurable": {"thread_id": "thread-123"}}
        ...     )
    """
    config = get_app_config()
    
    # 未配置时使用 InMemorySaver
    if config.checkpointer is None:
        from langgraph.checkpoint.memory import InMemorySaver
        
        yield InMemorySaver()
        return
    
    # 使用配置的后端
    with _sync_checkpointer_cm(config.checkpointer) as saver:
        yield saver
```

---

## 5. 异步检查点提供者 (async_provider.py)

### 5.1 模块设计

异步提供者专为异步框架设计（如 FastAPI），提供正确的资源清理：

```
┌─────────────────────────────────────────────────────────────────┐
│                    异步 Checkpointer 使用方式                     │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  异步上下文管理器模式（推荐）                                       │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │                                                          │   │
│  │  async with make_checkpointer() as checkpointer:         │   │
│  │      app.state.checkpointer = checkpointer               │   │
│  │      # 在应用生命周期内保持连接                              │   │
│  │                                                          │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                  │
│  FastAPI 生命周期间集成：                                         │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │                                                          │   │
│  │  @asynccontextmanager                                    │   │
│  │  async def lifespan(app: FastAPI):                       │   │
│  │      async with make_checkpointer() as cp:               │   │
│  │          app.state.checkpointer = cp                     │   │
│  │          yield                                           │   │
│  │      # 退出时自动清理资源                                   │   │
│  │                                                          │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### 5.2 异步上下文管理器实现

```python
@contextlib.asynccontextmanager
async def _async_checkpointer(config) -> AsyncIterator[Checkpointer]:
    """
    创建和销毁异步检查点的内部上下文管理器。
    
    使用异步版本的 Saver：
    - memory: InMemorySaver（同步，可直接使用）
    - sqlite: AsyncSqliteSaver
    - postgres: AsyncPostgresSaver
    
    Args:
        config: 检查点配置
        
    Yields:
        Checkpointer 实例
        
    Raises:
        ImportError: 缺少必要的依赖包
        ValueError: 配置错误
    """
    # ========== memory 后端 ==========
    if config.type == "memory":
        from langgraph.checkpoint.memory import InMemorySaver
        
        yield InMemorySaver()
        return
    
    # ========== sqlite 后端 ==========
    if config.type == "sqlite":
        try:
            from langgraph.checkpoint.sqlite.aio import AsyncSqliteSaver
        except ImportError as exc:
            raise ImportError(SQLITE_INSTALL) from exc
        
        import pathlib
        
        conn_str = _resolve_sqlite_conn_str(config.connection_string or "store.db")
        
        # 为文件路径创建父目录
        if conn_str != ":memory:" and not conn_str.startswith("file:"):
            pathlib.Path(conn_str).parent.mkdir(parents=True, exist_ok=True)
        
        async with AsyncSqliteSaver.from_conn_string(conn_str) as saver:
            await saver.setup()  # 异步创建表结构
            yield saver
        return
    
    # ========== postgres 后端 ==========
    if config.type == "postgres":
        try:
            from langgraph.checkpoint.postgres.aio import AsyncPostgresSaver
        except ImportError as exc:
            raise ImportError(POSTGRES_INSTALL) from exc
        
        if not config.connection_string:
            raise ValueError(POSTGRES_CONN_REQUIRED)
        
        async with AsyncPostgresSaver.from_conn_string(config.connection_string) as saver:
            await saver.setup()  # 异步创建表结构
            yield saver
        return
    
    raise ValueError(f"Unknown checkpointer type: {config.type!r}")
```

### 5.3 公共异步上下文管理器

```python
@contextlib.asynccontextmanager
async def make_checkpointer() -> AsyncIterator[Checkpointer]:
    """
    异步上下文管理器，为调用者生命周期提供检查点。
    
    资源在进入时打开，退出时关闭，无全局状态。
    当配置中未设置 checkpointer 时，返回 InMemorySaver。
    
    Yields:
        Checkpointer 实例
        
    Example:
        >>> async with make_checkpointer() as checkpointer:
        ...     app.state.checkpointer = checkpointer
    """
    config = get_app_config()
    
    # 未配置时使用 InMemorySaver
    if config.checkpointer is None:
        from langgraph.checkpoint.memory import InMemorySaver
        
        yield InMemorySaver()
        return
    
    # 使用配置的后端
    async with _async_checkpointer(config.checkpointer) as saver:
        yield saver
```

---

## 6. 配置项

### 6.1 CheckpointerConfig 配置类

```python
# src/config/checkpointer_config.py

from typing import Literal
from pydantic import BaseModel, Field

CheckpointerType = Literal["memory", "sqlite", "postgres"]


class CheckpointerConfig(BaseModel):
    """
    LangGraph 状态持久化检查点配置。
    
    支持三种后端类型：
    - memory: 进程内存存储（重启丢失）
    - sqlite: 本地文件存储（需要 langgraph-checkpoint-sqlite）
    - postgres: PostgreSQL 数据库存储（需要 langgraph-checkpoint-postgres）
    """
    
    type: CheckpointerType = Field(
        description="Checkpointer backend type. "
        "'memory' is in-process only (lost on restart). "
        "'sqlite' persists to a local file (requires langgraph-checkpoint-sqlite). "
        "'postgres' persists to PostgreSQL (requires langgraph-checkpoint-postgres)."
    )
    
    connection_string: str | None = Field(
        default=None,
        description="Connection string for sqlite (file path) or postgres (DSN). "
        "Required for sqlite and postgres types. "
        "For sqlite, use a file path like '.deer-flow/checkpoints.db' or ':memory:' for in-memory. "
        "For postgres, use a DSN like 'postgresql://user:pass@localhost:5432/db'.",
    )
```

### 6.2 全局配置管理

```python
# 全局配置实例 — None 表示未配置检查点
_checkpointer_config: CheckpointerConfig | None = None


def get_checkpointer_config() -> CheckpointerConfig | None:
    """获取当前检查点配置，未配置时返回 None"""
    return _checkpointer_config


def set_checkpointer_config(config: CheckpointerConfig | None) -> None:
    """设置检查点配置"""
    global _checkpointer_config
    _checkpointer_config = config


def load_checkpointer_config_from_dict(config_dict: dict) -> None:
    """从字典加载检查点配置"""
    global _checkpointer_config
    _checkpointer_config = CheckpointerConfig(**config_dict)
```

### 6.3 配置加载流程

```python
# src/config/app_config.py 中的加载逻辑

@classmethod
def from_file(cls, config_path: str | None = None) -> Self:
    # ... 其他配置加载 ...
    
    # 加载检查点配置（如果存在）
    if "checkpointer" in config_data:
        load_checkpointer_config_from_dict(config_data["checkpointer"])
    
    # ...
```

---

## 7. 使用示例

### 7.1 配置示例

#### 7.1.1 内存检查点（默认）

```yaml
# config.yaml

# 方式一：不配置 checkpointer（默认使用内存）
# checkpointer 配置完全省略

# 方式二：显式配置内存检查点
checkpointer:
  type: memory
```

#### 7.1.2 SQLite 检查点

```yaml
# config.yaml

checkpointer:
  type: sqlite
  connection_string: .deer-flow/checkpoints.db  # 相对路径（相对于 base_dir）
  # connection_string: /var/data/checkpoints.db  # 绝对路径
  # connection_string: ":memory:"  # 内存模式（用于测试）
```

#### 7.1.3 PostgreSQL 检查点

```yaml
# config.yaml

checkpointer:
  type: postgres
  connection_string: postgresql://user:password@localhost:5432/deerflow
  # 或使用环境变量
  # connection_string: $DATABASE_URL
```

### 7.2 同步使用示例

#### 7.2.1 使用全局单例

```python
from langchain_core.messages import HumanMessage
from src.agents.checkpointer import get_checkpointer
from src.agents import create_agent

# 获取检查点（首次调用创建，后续复用）
checkpointer = get_checkpointer()

# 创建带检查点的 Agent
agent = create_agent(
    model=chat_model,
    tools=tools,
    checkpointer=checkpointer,
)

# 执行对话（状态自动持久化）
result = agent.invoke(
    {"messages": [HumanMessage(content="你好")]},
    config={"configurable": {"thread_id": "user-123"}},
)

# 后续对话可以恢复状态
result2 = agent.invoke(
    {"messages": [HumanMessage(content="继续")]},
    config={"configurable": {"thread_id": "user-123"}},
)
```

#### 7.2.2 使用上下文管理器

```python
from src.agents.checkpointer import checkpointer_context

# 每次创建独立连接，退出时自动清理
with checkpointer_context() as checkpointer:
    agent = create_agent(
        model=chat_model,
        tools=tools,
        checkpointer=checkpointer,
    )
    
    result = agent.invoke(
        {"messages": [HumanMessage(content="Hello")]},
        config={"configurable": {"thread_id": "thread-1"}},
    )
# 退出 with 块时自动关闭连接
```

#### 7.2.3 CLI 脚本示例

```python
#!/usr/bin/env python3
"""CLI 工具示例：使用检查点进行对话"""

import sys
from langchain_core.messages import HumanMessage
from src.agents.checkpointer import checkpointer_context
from src.agents import create_agent
from src.models import create_chat_model


def main():
    thread_id = sys.argv[1] if len(sys.argv) > 1 else "default"
    
    # 使用上下文管理器确保资源清理
    with checkpointer_context() as checkpointer:
        agent = create_agent(
            model=create_chat_model(name="gpt-4"),
            checkpointer=checkpointer,
        )
        
        print(f"Thread ID: {thread_id}")
        print("Type 'quit' to exit")
        
        while True:
            user_input = input("You: ")
            if user_input.lower() == "quit":
                break
            
            result = agent.invoke(
                {"messages": [HumanMessage(content=user_input)]},
                config={"configurable": {"thread_id": thread_id}},
            )
            
            print(f"Assistant: {result['messages'][-1].content}")


if __name__ == "__main__":
    main()
```

### 7.3 异步使用示例

#### 7.3.1 FastAPI 集成

```python
from contextlib import asynccontextmanager
from fastapi import FastAPI
from src.agents.checkpointer import make_checkpointer
from src.agents import create_agent
from src.models import create_chat_model


@asynccontextmanager
async def lifespan(app: FastAPI):
    """FastAPI 应用生命周期管理"""
    # 启动时创建检查点
    async with make_checkpointer() as checkpointer:
        app.state.checkpointer = checkpointer
        
        # 预创建 Agent（可选）
        app.state.agent = create_agent(
            model=create_chat_model(name="gpt-4"),
            checkpointer=checkpointer,
        )
        
        yield  # 应用运行中
    
    # 关闭时自动清理资源


app = FastAPI(lifespan=lifespan)


@app.post("/chat/{thread_id}")
async def chat(thread_id: str, message: str):
    """对话接口"""
    from langchain_core.messages import HumanMessage
    
    result = await app.state.agent.ainvoke(
        {"messages": [HumanMessage(content=message)]},
        config={"configurable": {"thread_id": thread_id}},
    )
    
    return {"response": result["messages"][-1].content}
```

#### 7.3.2 独立异步脚本

```python
import asyncio
from langchain_core.messages import HumanMessage
from src.agents.checkpointer import make_checkpointer
from src.agents import create_agent
from src.models import create_chat_model


async def main():
    async with make_checkpointer() as checkpointer:
        agent = create_agent(
            model=create_chat_model(name="gpt-4"),
            checkpointer=checkpointer,
        )
        
        # 执行多轮对话
        for i in range(3):
            result = await agent.ainvoke(
                {"messages": [HumanMessage(content=f"消息 {i+1}")]},
                config={"configurable": {"thread_id": "demo-thread"}},
            )
            print(f"回复 {i+1}: {result['messages'][-1].content}")


if __name__ == "__main__":
    asyncio.run(main())
```

### 7.4 DeerFlowClient 集成示例

```python
from src.client import DeerFlowClient
from src.agents.checkpointer import get_checkpointer

# 方式一：自动使用全局检查点
client = DeerFlowClient()
# 内部调用 get_checkpointer() 获取检查点

# 方式二：显式指定检查点
checkpointer = get_checkpointer()
client = DeerFlowClient(checkpointer=checkpointer)

# 方式三：禁用检查点（传 None 会回退到全局检查点）
# 要真正禁用，需要不传参数（使用默认行为）
```

### 7.5 测试示例

```python
import pytest
from unittest.mock import MagicMock, patch
from src.agents.checkpointer import get_checkpointer, reset_checkpointer
from src.config.checkpointer_config import set_checkpointer_config


@pytest.fixture(autouse=True)
def reset_state():
    """每个测试前后重置状态"""
    set_checkpointer_config(None)
    reset_checkpointer()
    yield
    set_checkpointer_config(None)
    reset_checkpointer()


def test_memory_checkpointer():
    """测试内存检查点"""
    from langgraph.checkpoint.memory import InMemorySaver
    from src.config.checkpointer_config import load_checkpointer_config_from_dict
    
    load_checkpointer_config_from_dict({"type": "memory"})
    cp = get_checkpointer()
    
    assert isinstance(cp, InMemorySaver)


def test_checkpointer_singleton():
    """测试单例模式"""
    from src.config.checkpointer_config import load_checkpointer_config_from_dict
    
    load_checkpointer_config_from_dict({"type": "memory"})
    
    cp1 = get_checkpointer()
    cp2 = get_checkpointer()
    
    assert cp1 is cp2  # 同一实例
    
    reset_checkpointer()
    cp3 = get_checkpointer()
    
    assert cp1 is not cp3  # 重置后是新实例


@pytest.mark.anyio
async def test_async_checkpointer():
    """测试异步检查点"""
    from langgraph.checkpoint.memory import InMemorySaver
    from src.agents.checkpointer import make_checkpointer
    
    async with make_checkpointer() as checkpointer:
        assert isinstance(checkpointer, InMemorySaver)
```

---

## 8. 最佳实践

### 8.1 后端选择指南

```
┌─────────────────────────────────────────────────────────────────┐
│                      后端选择决策树                               │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  开始                                                            │
│    │                                                             │
│    ▼                                                             │
│  需要持久化？ ────────── 否 ──────▶ 使用 memory                   │
│    │                                                             │
│   是                                                              │
│    │                                                             │
│    ▼                                                             │
│  多实例部署？ ────────── 否 ──────▶ 使用 sqlite                   │
│    │                                                             │
│   是                                                              │
│    │                                                             │
│    ▼                                                             │
│  使用 postgres                                                   │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### 8.2 使用场景建议

| 场景 | 推荐后端 | 推荐方式 |
|------|----------|----------|
| 本地开发 | memory 或 sqlite | get_checkpointer() |
| 单元测试 | memory | checkpointer_context() |
| CLI 工具 | sqlite | checkpointer_context() |
| 单实例服务 | sqlite | get_checkpointer() |
| FastAPI 服务 | postgres | make_checkpointer() |
| 多实例集群 | postgres | make_checkpointer() |

### 8.3 注意事项

1. **资源清理**：
   - 同步代码使用 `checkpointer_context()` 或在退出前调用 `reset_checkpointer()`
   - 异步代码使用 `make_checkpointer()` 确保连接正确关闭

2. **线程安全**：
   - `get_checkpointer()` 返回的单例是线程安全的
   - 每个线程应使用不同的 `thread_id`

3. **配置变更**：
   - 配置变更后需调用 `reset_checkpointer()` 重新加载

4. **PostgreSQL 连接池**：
   - 生产环境建议配置连接池参数
   - 注意连接字符串格式

---

## 9. 错误处理

### 9.1 常见错误

| 错误类型 | 原因 | 解决方案 |
|----------|------|----------|
| `ImportError: langgraph-checkpoint-sqlite` | SQLite 后端依赖未安装 | `uv add langgraph-checkpoint-sqlite` |
| `ImportError: langgraph-checkpoint-postgres` | PostgreSQL 后端依赖未安装 | `uv add langgraph-checkpoint-postgres "psycopg[binary]"` |
| `ValueError: connection_string is required` | PostgreSQL 未配置连接字符串 | 添加 `connection_string` 配置 |
| `ValueError: Unknown checkpointer type` | 配置了不支持的类型 | 使用 `memory`、`sqlite` 或 `postgres` |

### 9.2 错误处理示例

```python
from src.agents.checkpointer import get_checkpointer, reset_checkpointer
from src.config.checkpointer_config import load_checkpointer_config_from_dict

try:
    # 配置 PostgreSQL 后端
    load_checkpointer_config_from_dict({
        "type": "postgres",
        "connection_string": "postgresql://localhost/db"
    })
    
    checkpointer = get_checkpointer()
    
except ImportError as e:
    print(f"缺少依赖: {e}")
    # 回退到内存模式
    load_checkpointer_config_from_dict({"type": "memory"})
    checkpointer = get_checkpointer()
    
except ValueError as e:
    print(f"配置错误: {e}")
    raise
```

---

## 10. 源码参考

### 10.1 完整模块源码

#### `__init__.py`

```python
from .async_provider import make_checkpointer
from .provider import checkpointer_context, get_checkpointer, reset_checkpointer

__all__ = [
    "get_checkpointer",
    "reset_checkpointer",
    "checkpointer_context",
    "make_checkpointer",
]
```

#### `provider.py` (关键部分)

```python
"""Sync checkpointer factory.

Provides a **sync singleton** and a **sync context manager** for LangGraph
graph compilation and CLI tools.

Supported backends: memory, sqlite, postgres.
"""

from __future__ import annotations

import contextlib
import logging
from collections.abc import Iterator

from langgraph.types import Checkpointer

from src.config.app_config import get_app_config
from src.config.checkpointer_config import CheckpointerConfig
from src.config.paths import resolve_path

logger = logging.getLogger(__name__)

# Error message constants
SQLITE_INSTALL = "langgraph-checkpoint-sqlite is required..."
POSTGRES_INSTALL = "langgraph-checkpoint-postgres is required..."
POSTGRES_CONN_REQUIRED = "checkpointer.connection_string is required..."


def _resolve_sqlite_conn_str(raw: str) -> str:
    if raw == ":memory:" or raw.startswith("file:"):
        return raw
    return str(resolve_path(raw))


@contextlib.contextmanager
def _sync_checkpointer_cm(config: CheckpointerConfig) -> Iterator[Checkpointer]:
    # ... 实现见上文 ...


_checkpointer: Checkpointer | None = None
_checkpointer_ctx = None


def get_checkpointer() -> Checkpointer:
    # ... 实现见上文 ...


def reset_checkpointer() -> None:
    # ... 实现见上文 ...


@contextlib.contextmanager
def checkpointer_context() -> Iterator[Checkpointer]:
    # ... 实现见上文 ...
```

#### `async_provider.py` (关键部分)

```python
"""Async checkpointer factory.

Provides an **async context manager** for long-running async servers.
"""

from __future__ import annotations

import contextlib
import logging
from collections.abc import AsyncIterator

from langgraph.types import Checkpointer

from src.agents.checkpointer.provider import (
    POSTGRES_CONN_REQUIRED,
    POSTGRES_INSTALL,
    SQLITE_INSTALL,
    _resolve_sqlite_conn_str,
)
from src.config.app_config import get_app_config

logger = logging.getLogger(__name__)


@contextlib.asynccontextmanager
async def _async_checkpointer(config) -> AsyncIterator[Checkpointer]:
    # ... 实现见上文 ...


@contextlib.asynccontextmanager
async def make_checkpointer() -> AsyncIterator[Checkpointer]:
    # ... 实现见上文 ...
```

---

## 11. 总结

Checkpointer 模块是 DeerFlow 框架中实现状态持久化的核心组件：

1. **灵活的后端选择**：支持 memory、sqlite、postgres 三种后端
2. **双模式支持**：同步单例和异步上下文管理器
3. **配置驱动**：通过 config.yaml 灵活配置
4. **资源安全**：自动管理连接生命周期

该模块的设计遵循了简洁、可测试、可扩展的原则，为 LangGraph Agent 提供了可靠的状态持久化能力。