---

# DeerFlow MCP 模块技术文档

## 1. 模块概述

### 1.1 模块职责和功能定位

MCP (Model Context Protocol) 模块是 DeerFlow 框架中的扩展协议集成层，负责：

- **工具发现与加载**：从 MCP 服务器动态发现和加载 LangChain 工具
- **多传输协议支持**：支持 `stdio`、`sse`、`http` 三种传输协议
- **OAuth 认证**：为 HTTP/SSE 类型的 MCP 服务器提供自动令牌管理和刷新
- **工具缓存**：实现高效的工具缓存机制，避免重复加载
- **配置热更新**：支持配置文件变更检测和自动重新加载

### 1.2 MCP 协议简介

**Model Context Protocol (MCP)** 是由 Anthropic 推出的开放协议，用于在 AI 模型和外部工具/数据源之间建立标准化通信。其主要特点：

- **标准化接口**：定义了工具发现、调用和响应的标准格式
- **多传输支持**：支持 stdio（本地进程）、SSE（Server-Sent Events）、HTTP 等传输方式
- **工具生态**：社区提供了丰富的 MCP 服务器实现（GitHub、PostgreSQL、文件系统等）

```
┌─────────────────┐      MCP Protocol      ┌──────────────────┐
│   DeerFlow      │ ◄──────────────────────► │   MCP Server    │
│   (MCP Client)  │                         │   (Tools)        │
└─────────────────┘                         └──────────────────┘
```

### 1.3 与其他模块的关系

```
┌─────────────────────────────────────────────────────────────────┐
│                        DeerFlow 系统架构                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌──────────────┐    ┌──────────────┐    ┌──────────────┐      │
│  │ Gateway API  │───►│ Extensions   │◄───│ Frontend     │      │
│  │ (mcp.py)     │    │ Config       │    │ (config UI)  │      │
│  └──────┬───────┘    └──────┬───────┘    └──────────────┘      │
│         │                   │                                   │
│         ▼                   ▼                                   │
│  ┌──────────────┐    ┌──────────────┐                         │
│  │ Tools Cache  │◄───│ MCP Module   │                         │
│  │ (cache.py)   │    │ (本模块)      │                         │
│  └──────┬───────┘    └──────┬───────┘                         │
│         │                   │                                   │
│         ▼                   ▼                                   │
│  ┌──────────────────────────────────────┐                      │
│  │         LangGraph Agent              │                      │
│  │         (tools usage)                │                      │
│  └──────────────────────────────────────┘                      │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**依赖关系**：

| 模块 | 关系说明 |
|------|----------|
| `src.config.extensions_config` | 配置管理，提供 MCP 服务器配置定义 |
| `src.tools.tools` | 工具消费者，通过缓存获取 MCP 工具 |
| `src.gateway.routers.mcp` | API 路由，提供配置管理 REST 接口 |
| `langchain-mcp-adapters` | 外部依赖，提供 MCP 客户端实现 |

---

## 2. 目录结构

```
backend/src/mcp/
├── __init__.py      # 模块入口，导出公共 API
├── cache.py         # 工具缓存机制（延迟初始化、过期检测）
├── client.py        # MCP 客户端配置构建
├── oauth.py         # OAuth 令牌管理（获取、缓存、刷新）
└── tools.py         # 工具加载主逻辑
```

**文件职责说明**：

| 文件 | 行数 | 职责 |
|------|------|------|
| `__init__.py` | 14 | 模块公共 API 导出 |
| `cache.py` | 138 | 工具缓存、延迟初始化、配置变更检测 |
| `client.py` | 68 | 服务器参数构建、配置转换 |
| `oauth.py` | 150 | OAuth 令牌管理、工具拦截器 |
| `tools.py` | 66 | 工具加载入口、MultiServerMCPClient 封装 |

---

## 3. 核心组件详解

### 3.1 client.py - MCP 客户端配置构建

此模块负责将用户配置转换为 `langchain-mcp-adapters` 库所需的参数格式。

#### 3.1.1 核心函数：`build_server_params`

```python
def build_server_params(server_name: str, config: McpServerConfig) -> dict[str, Any]:
    """构建单个 MCP 服务器的参数配置。

    根据传输类型构建不同的参数字典：
    - stdio: 需要 command, args, env
    - sse/http: 需要 url, headers

    Args:
        server_name: MCP 服务器名称（用于错误提示）
        config: MCP 服务器配置对象

    Returns:
        适配 langchain-mcp-adapters 的参数字典

    Raises:
        ValueError: 配置缺失或传输类型不支持
    """
    transport_type = config.type or "stdio"
    params: dict[str, Any] = {"transport": transport_type}

    if transport_type == "stdio":
        # stdio 类型：本地进程通信
        if not config.command:
            raise ValueError(f"MCP server '{server_name}' with stdio transport requires 'command' field")
        params["command"] = config.command
        params["args"] = config.args
        # 可选：环境变量注入
        if config.env:
            params["env"] = config.env

    elif transport_type in ("sse", "http"):
        # sse/http 类型：远程服务通信
        if not config.url:
            raise ValueError(f"MCP server '{server_name}' with {transport_type} transport requires 'url' field")
        params["url"] = config.url
        # 可选：自定义 HTTP 头
        if config.headers:
            params["headers"] = config.headers

    else:
        raise ValueError(f"MCP server '{server_name}' has unsupported transport type: {transport_type}")

    return params
```

#### 3.1.2 核心函数：`build_servers_config`

```python
def build_servers_config(extensions_config: ExtensionsConfig) -> dict[str, dict[str, Any]]:
    """构建所有启用的 MCP 服务器配置。

    遍历配置中的服务器，过滤已启用的服务器，
    并为每个服务器构建参数配置。

    Args:
        extensions_config: 扩展配置对象

    Returns:
        服务器名称到参数配置的映射字典
    """
    enabled_servers = extensions_config.get_enabled_mcp_servers()

    if not enabled_servers:
        logger.info("No enabled MCP servers found")
        return {}

    servers_config = {}
    for server_name, server_config in enabled_servers.items():
        try:
            servers_config[server_name] = build_server_params(server_name, server_config)
            logger.info(f"Configured MCP server: {server_name}")
        except Exception as e:
            # 单个服务器配置失败不影响其他服务器
            logger.error(f"Failed to configure MCP server '{server_name}': {e}")

    return servers_config
```

---

### 3.2 cache.py - 工具缓存机制

此模块实现 MCP 工具的缓存管理，支持延迟初始化和配置变更检测。

#### 3.2.1 全局状态管理

```python
# 缓存状态变量
_mcp_tools_cache: list[BaseTool] | None = None  # 工具列表缓存
_cache_initialized = False                        # 缓存是否已初始化
_initialization_lock = asyncio.Lock()            # 异步初始化锁
_config_mtime: float | None = None               # 配置文件最后修改时间
```

#### 3.2.2 核心函数：`initialize_mcp_tools`

```python
async def initialize_mcp_tools() -> list[BaseTool]:
    """初始化并缓存 MCP 工具。

    此函数应在应用启动时调用一次。
    使用异步锁确保并发安全。

    Returns:
        从所有启用的 MCP 服务器加载的 LangChain 工具列表

    使用示例:
        # 应用启动时
        await initialize_mcp_tools()
    """
    global _mcp_tools_cache, _cache_initialized, _config_mtime

    async with _initialization_lock:  # 防止并发初始化
        if _cache_initialized:
            logger.info("MCP tools already initialized")
            return _mcp_tools_cache or []

        from src.mcp.tools import get_mcp_tools

        logger.info("Initializing MCP tools...")
        _mcp_tools_cache = await get_mcp_tools()      # 加载工具
        _cache_initialized = True
        _config_mtime = _get_config_mtime()            # 记录配置文件修改时间
        logger.info(f"MCP tools initialized: {len(_mcp_tools_cache)} tool(s) loaded")

        return _mcp_tools_cache
```

#### 3.2.3 核心函数：`get_cached_mcp_tools`

```python
def get_cached_mcp_tools() -> list[BaseTool]:
    """获取缓存的 MCP 工具（支持延迟初始化）。

    此函数用于在同步上下文中获取工具。
    如果工具未初始化，会自动进行延迟初始化。
    同时检测配置文件变更，自动触发重新加载。

    Returns:
        缓存的 MCP 工具列表

    特性：
    - 自动检测配置文件变更
    - 支持在运行中的事件循环中初始化
    - 兼容 FastAPI 和 LangGraph Studio 环境
    """
    global _cache_initialized

    # 检测配置文件是否被修改
    if _is_cache_stale():
        logger.info("MCP cache is stale, resetting for re-initialization...")
        reset_mcp_tools_cache()

    if not _cache_initialized:
        logger.info("MCP tools not initialized, performing lazy initialization...")
        try:
            loop = asyncio.get_event_loop()
            if loop.is_running():
                # 事件循环正在运行（如 LangGraph Studio）
                # 在线程池中创建新的事件循环
                import concurrent.futures
                with concurrent.futures.ThreadPoolExecutor() as executor:
                    future = executor.submit(asyncio.run, initialize_mcp_tools())
                    future.result()
            else:
                # 事件循环未运行，直接使用当前循环
                loop.run_until_complete(initialize_mcp_tools())
        except RuntimeError:
            # 不存在事件循环，创建一个
            asyncio.run(initialize_mcp_tools())
        except Exception as e:
            logger.error(f"Failed to lazy-initialize MCP tools: {e}")
            return []

    return _mcp_tools_cache or []
```

#### 3.2.4 缓存失效检测

```python
def _is_cache_stale() -> bool:
    """检测缓存是否因配置文件变更而失效。

    通过比较配置文件的修改时间来判断是否需要重新加载。

    Returns:
        True 表示缓存已失效，需要重新初始化
    """
    global _config_mtime

    if not _cache_initialized:
        return False  # 未初始化，不算失效

    current_mtime = _get_config_mtime()

    # 无法获取修改时间时不判定失效
    if _config_mtime is None or current_mtime is None:
        return False

    # 配置文件被修改后判定失效
    if current_mtime > _config_mtime:
        logger.info(f"MCP config file has been modified (mtime: {_config_mtime} -> {current_mtime})")
        return True

    return False
```

---

### 3.3 oauth.py - OAuth 认证模块

此模块为 HTTP/SSE 类型的 MCP 服务器提供 OAuth 令牌管理。

#### 3.3.1 数据类：`_OAuthToken`

```python
@dataclass
class _OAuthToken:
    """缓存的 OAuth 令牌数据结构。"""
    access_token: str      # 访问令牌
    token_type: str        # 令牌类型（如 "Bearer"）
    expires_at: datetime   # 过期时间（UTC）
```

#### 3.3.2 核心类：`OAuthTokenManager`

```python
class OAuthTokenManager:
    """MCP 服务器的 OAuth 令牌管理器。

    负责：
    - 令牌获取
    - 令牌缓存
    - 自动刷新（在过期前刷新）
    """

    def __init__(self, oauth_by_server: dict[str, McpOAuthConfig]):
        """
        Args:
            oauth_by_server: 服务器名称到 OAuth 配置的映射
        """
        self._oauth_by_server = oauth_by_server
        self._tokens: dict[str, _OAuthToken] = {}
        # 每个服务器独立的锁，防止并发刷新
        self._locks: dict[str, asyncio.Lock] = {
            name: asyncio.Lock() for name in oauth_by_server
        }

    @classmethod
    def from_extensions_config(cls, extensions_config: ExtensionsConfig) -> "OAuthTokenManager":
        """从扩展配置创建令牌管理器。

        Args:
            extensions_config: 扩展配置对象

        Returns:
            配置好的 OAuthTokenManager 实例
        """
        oauth_by_server: dict[str, McpOAuthConfig] = {}
        for server_name, server_config in extensions_config.get_enabled_mcp_servers().items():
            if server_config.oauth and server_config.oauth.enabled:
                oauth_by_server[server_name] = server_config.oauth
        return cls(oauth_by_server)

    async def get_authorization_header(self, server_name: str) -> str | None:
        """获取指定服务器的 Authorization 头。

        实现逻辑：
        1. 检查缓存中是否有有效令牌
        2. 如果令牌即将过期，异步刷新
        3. 返回 "Bearer <token>" 格式的头

        Args:
            server_name: MCP 服务器名称

        Returns:
            Authorization 头字符串，如 "Bearer xxx"；无 OAuth 配置时返回 None
        """
        oauth = self._oauth_by_server.get(server_name)
        if not oauth:
            return None

        token = self._tokens.get(server_name)
        if token and not self._is_expiring(token, oauth):
            # 缓存命中且未过期
            return f"{token.token_type} {token.access_token}"

        lock = self._locks[server_name]
        async with lock:  # 双重检查锁定模式
            token = self._tokens.get(server_name)
            if token and not self._is_expiring(token, oauth):
                return f"{token.token_type} {token.access_token}"

            # 获取新令牌
            fresh = await self._fetch_token(oauth)
            self._tokens[server_name] = fresh
            logger.info(f"Refreshed OAuth access token for MCP server: {server_name}")
            return f"{fresh.token_type} {fresh.access_token}"

    @staticmethod
    def _is_expiring(token: _OAuthToken, oauth: McpOAuthConfig) -> bool:
        """检查令牌是否即将过期。

        Args:
            token: 令牌对象
            oauth: OAuth 配置（包含 refresh_skew_seconds）

        Returns:
            True 表示令牌即将过期，需要刷新
        """
        now = datetime.now(UTC)
        # 在过期前 refresh_skew_seconds 秒判定为即将过期
        return token.expires_at <= now + timedelta(seconds=max(oauth.refresh_skew_seconds, 0))

    async def _fetch_token(self, oauth: McpOAuthConfig) -> _OAuthToken:
        """从 OAuth 服务器获取新令牌。

        支持 grant_type:
        - client_credentials: 客户端凭据模式
        - refresh_token: 刷新令牌模式

        Args:
            oauth: OAuth 配置对象

        Returns:
            新的 OAuth 令牌对象

        Raises:
            ValueError: 配置不完整或响应格式错误
        """
        import httpx

        data: dict[str, str] = {
            "grant_type": oauth.grant_type,
            **oauth.extra_token_params,
        }

        # 可选参数
        if oauth.scope:
            data["scope"] = oauth.scope
        if oauth.audience:
            data["audience"] = oauth.audience

        # 根据授权类型添加参数
        if oauth.grant_type == "client_credentials":
            if not oauth.client_id or not oauth.client_secret:
                raise ValueError("OAuth client_credentials requires client_id and client_secret")
            data["client_id"] = oauth.client_id
            data["client_secret"] = oauth.client_secret

        elif oauth.grant_type == "refresh_token":
            if not oauth.refresh_token:
                raise ValueError("OAuth refresh_token grant requires refresh_token")
            data["refresh_token"] = oauth.refresh_token
            if oauth.client_id:
                data["client_id"] = oauth.client_id
            if oauth.client_secret:
                data["client_secret"] = oauth.client_secret

        else:
            raise ValueError(f"Unsupported OAuth grant type: {oauth.grant_type}")

        # 发送令牌请求
        async with httpx.AsyncClient(timeout=15.0) as client:
            response = await client.post(oauth.token_url, data=data)
            response.raise_for_status()
            payload = response.json()

        # 解析响应
        access_token = payload.get(oauth.token_field)
        if not access_token:
            raise ValueError(f"OAuth token response missing '{oauth.token_field}'")

        token_type = str(payload.get(oauth.token_type_field, oauth.default_token_type) or oauth.default_token_type)

        # 计算过期时间
        expires_in_raw = payload.get(oauth.expires_in_field, 3600)
        try:
            expires_in = int(expires_in_raw)
        except (TypeError, ValueError):
            expires_in = 3600  # 默认 1 小时

        expires_at = datetime.now(UTC) + timedelta(seconds=max(expires_in, 1))
        return _OAuthToken(access_token=access_token, token_type=token_type, expires_at=expires_at)
```

#### 3.3.3 工具拦截器构建

```python
def build_oauth_tool_interceptor(extensions_config: ExtensionsConfig) -> Any | None:
    """构建 OAuth 工具拦截器，自动注入 Authorization 头。

    此拦截器会在每次工具调用时检查是否有对应的 OAuth 配置，
    如果有则自动获取并注入令牌。

    Args:
        extensions_config: 扩展配置对象

    Returns:
        异步拦截器函数，或 None（无 OAuth 配置时）

    拦截器签名:
        async def interceptor(request: Any, handler: Any) -> Any
    """
    token_manager = OAuthTokenManager.from_extensions_config(extensions_config)
    if not token_manager.has_oauth_servers():
        return None

    async def oauth_interceptor(request: Any, handler: Any) -> Any:
        # 获取 Authorization 头
        header = await token_manager.get_authorization_header(request.server_name)
        if not header:
            return await handler(request)

        # 更新请求头
        updated_headers = dict(request.headers or {})
        updated_headers["Authorization"] = header
        return await handler(request.override(headers=updated_headers))

    return oauth_interceptor
```

---

### 3.4 tools.py - 工具加载主逻辑

此模块是 MCP 工具加载的入口点，整合所有组件完成工具发现和加载。

```python
async def get_mcp_tools() -> list[BaseTool]:
    """从所有启用的 MCP 服务器获取工具。

    执行流程：
    1. 检查 langchain-mcp-adapters 是否安装
    2. 从文件加载最新配置
    3. 构建服务器配置
    4. 注入 OAuth 初始头（用于工具发现阶段）
    5. 创建 MCP 客户端
    6. 获取所有工具

    Returns:
        LangChain 工具列表

    Note:
        使用 ExtensionsConfig.from_file() 而非 get_extensions_config()
        确保总是从磁盘读取最新配置，反映 Gateway API 的修改。
    """
    try:
        from langchain_mcp_adapters.client import MultiServerMCPClient
    except ImportError:
        logger.warning("langchain-mcp-adapters not installed. Install it to enable MCP tools")
        return []

    # 从文件加载最新配置（而非缓存）
    extensions_config = ExtensionsConfig.from_file()
    servers_config = build_servers_config(extensions_config)

    if not servers_config:
        logger.info("No enabled MCP servers configured")
        return []

    try:
        logger.info(f"Initializing MCP client with {len(servers_config)} server(s)")

        # 为需要 OAuth 的服务器注入初始 Authorization 头
        initial_oauth_headers = await get_initial_oauth_headers(extensions_config)
        for server_name, auth_header in initial_oauth_headers.items():
            if server_name not in servers_config:
                continue
            if servers_config[server_name].get("transport") in ("sse", "http"):
                existing_headers = dict(servers_config[server_name].get("headers", {}))
                existing_headers["Authorization"] = auth_header
                servers_config[server_name]["headers"] = existing_headers

        # 构建工具拦截器（用于工具调用时的令牌注入）
        tool_interceptors = []
        oauth_interceptor = build_oauth_tool_interceptor(extensions_config)
        if oauth_interceptor is not None:
            tool_interceptors.append(oauth_interceptor)

        # 创建多服务器客户端
        client = MultiServerMCPClient(servers_config, tool_interceptors=tool_interceptors)

        # 获取所有工具
        tools = await client.get_tools()
        logger.info(f"Successfully loaded {len(tools)} tool(s) from MCP servers")

        return tools

    except Exception as e:
        logger.error(f"Failed to load MCP tools: {e}", exc_info=True)
        return []
```

---

## 4. 服务器配置

### 4.1 stdio 类型配置

**适用场景**：本地 MCP 服务器进程通信

```json
{
  "mcpServers": {
    "filesystem": {
      "enabled": true,
      "type": "stdio",
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-filesystem", "/path/to/allowed/files"],
      "env": {
        "NODE_OPTIONS": "--max-old-space-size=4096"
      },
      "description": "本地文件系统访问"
    }
  }
}
```

**配置字段说明**：

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `enabled` | bool | 否 | 是否启用（默认 true） |
| `type` | string | 否 | 传输类型，默认 "stdio" |
| `command` | string | **是** | 启动命令 |
| `args` | string[] | 否 | 命令参数 |
| `env` | object | 否 | 环境变量映射 |
| `description` | string | 否 | 描述信息 |

### 4.2 sse 类型配置

**适用场景**：Server-Sent Events 远程服务器

```json
{
  "mcpServers": {
    "remote-mcp": {
      "enabled": true,
      "type": "sse",
      "url": "https://api.example.com/mcp/sse",
      "headers": {
        "X-Custom-Header": "value"
      },
      "oauth": {
        "enabled": true,
        "token_url": "https://auth.example.com/oauth/token",
        "grant_type": "client_credentials",
        "client_id": "your-client-id",
        "client_secret": "your-client-secret",
        "scope": "read write",
        "refresh_skew_seconds": 120
      },
      "description": "远程 MCP 服务（SSE）"
    }
  }
}
```

### 4.3 http 类型配置

**适用场景**：HTTP 远程服务器

```json
{
  "mcpServers": {
    "api-mcp": {
      "enabled": true,
      "type": "http",
      "url": "https://api.example.com/mcp",
      "headers": {
        "X-API-Key": "your-api-key"
      },
      "oauth": {
        "enabled": true,
        "token_url": "https://auth.example.com/oauth/token",
        "grant_type": "refresh_token",
        "refresh_token": "$MY_REFRESH_TOKEN",
        "client_id": "your-client-id",
        "client_secret": "$CLIENT_SECRET"
      },
      "description": "HTTP MCP 服务"
    }
  }
}
```

**OAuth 配置字段说明**：

| 字段 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `enabled` | bool | true | 是否启用 OAuth |
| `token_url` | string | - | 令牌端点 URL |
| `grant_type` | string | "client_credentials" | 授权类型 |
| `client_id` | string | - | 客户端 ID |
| `client_secret` | string | - | 客户端密钥 |
| `refresh_token` | string | - | 刷新令牌（refresh_token 模式） |
| `scope` | string | - | OAuth scope |
| `audience` | string | - | OAuth audience |
| `token_field` | string | "access_token" | 响应中令牌字段名 |
| `token_type_field` | string | "token_type" | 响应中令牌类型字段名 |
| `expires_in_field` | string | "expires_in" | 响应中过期时间字段名 |
| `default_token_type` | string | "Bearer" | 默认令牌类型 |
| `refresh_skew_seconds` | int | 60 | 提前刷新秒数 |
| `extra_token_params` | object | {} | 额外令牌请求参数 |

---

## 5. 工具加载机制

### 5.1 工具发现流程

```
┌─────────────────────────────────────────────────────────────────┐
│                        工具发现流程                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  1. 配置加载                                                    │
│     └── ExtensionsConfig.from_file()                            │
│         └── 读取 extensions_config.json                         │
│         └── 解析环境变量 ($VAR -> 实际值)                        │
│                                                                 │
│  2. 服务器配置构建                                              │
│     └── build_servers_config()                                  │
│         └── 过滤 enabled=true 的服务器                          │
│         └── build_server_params() 构建参数                      │
│                                                                 │
│  3. OAuth 初始化（可选）                                        │
│     └── get_initial_oauth_headers()                             │
│         └── 为每个需要 OAuth 的服务器获取令牌                   │
│         └── 注入到服务器配置的 headers                          │
│                                                                 │
│  4. 客户端创建                                                  │
│     └── MultiServerMCPClient(servers_config)                    │
│         └── langchain-mcp-adapters 库                           │
│         └── 连接各 MCP 服务器                                   │
│         └── 发现工具列表                                        │
│                                                                 │
│  5. 工具获取                                                    │
│     └── client.get_tools()                                      │
│         └── 返回 LangChain BaseTool 列表                        │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 5.2 工具缓存机制

```
┌─────────────────────────────────────────────────────────────────┐
│                        缓存机制流程                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  get_cached_mcp_tools()                                         │
│         │                                                       │
│         ▼                                                       │
│  ┌──────────────┐                                               │
│  │ 检查配置文件 │                                               │
│  │ 修改时间     │                                               │
│  └──────┬───────┘                                               │
│         │                                                       │
│         ▼                                                       │
│  ┌──────────────┐     是     ┌──────────────┐                   │
│  │ 缓存失效？   │───────────►│ 重置缓存     │                   │
│  └──────┬───────┘            └──────┬───────┘                   │
│         │ 否                         │                           │
│         ▼                            │                           │
│  ┌──────────────┐                    │                           │
│  │ 已初始化？   │                    │                           │
│  └──────┬───────┘                    │                           │
│         │                            │                           │
│    ┌────┴────┐                       │                           │
│    │         │                       │                           │
│   否        是                       │                           │
│    │         │                       │                           │
│    ▼         │                       │                           │
│  ┌────────────┐                      │                           │
│  │ 延迟初始化 │                      │                           │
│  │ (异步)     │                      │                           │
│  └─────┬──────┘                      │                           │
│        │                             │                           │
│        └─────────────┬───────────────┘                           │
│                      │                                           │
│                      ▼                                           │
│               返回缓存工具列表                                   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 5.3 缓存失效检测

**触发条件**：

1. **配置文件修改时间变更**：通过 `os.path.getmtime()` 检测
2. **手动重置**：调用 `reset_mcp_tools_cache()`

**实现细节**：

```python
# 检测逻辑
def _is_cache_stale() -> bool:
    current_mtime = _get_config_mtime()
    if _config_mtime is None or current_mtime is None:
        return False
    return current_mtime > _config_mtime  # 文件被修改
```

**跨进程同步**：

由于 Gateway API 和 LangGraph Server 运行在不同进程：

```
┌─────────────────┐         修改配置          ┌─────────────────┐
│   Gateway API   │ ─────────────────────►   │ extensions_     │
│  (进程 1)       │                          │ config.json     │
└─────────────────┘                          └────────┬────────┘
                                                       │
                                                       │ mtime 变更
                                                       ▼
                                              ┌─────────────────┐
                                              │  LangGraph      │
                                              │  Server         │
                                              │  (进程 2)       │
                                              │                 │
                                              │ 检测到变更后    │
                                              │ 自动重新加载    │
                                              └─────────────────┘
```

---

## 6. OAuth 认证

### 6.1 令牌管理流程

```
┌─────────────────────────────────────────────────────────────────┐
│                      OAuth 令牌管理流程                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  get_authorization_header(server_name)                          │
│         │                                                       │
│         ▼                                                       │
│  ┌──────────────────┐                                           │
│  │ 检查 OAuth 配置  │                                           │
│  │ 是否存在        │                                           │
│  └────────┬─────────┘                                           │
│           │                                                      │
│      不存在│存在                                                 │
│           │                                                      │
│           ▼                                                      │
│  ┌──────────────────┐                                           │
│  │ 检查缓存令牌    │                                           │
│  │ 是否有效        │                                           │
│  └────────┬─────────┘                                           │
│           │                                                      │
│    有效   │   即将过期/不存在                                    │
│           │                                                      │
│           ▼                                                      │
│  ┌──────────────────┐     双重检查锁定                          │
│  │ 获取异步锁      │◄────────────────┐                          │
│  └────────┬─────────┘                 │                          │
│           │                           │                          │
│           ▼                           │                          │
│  ┌──────────────────┐                 │                          │
│  │ 再次检查缓存    │──有效───────────┘                          │
│  └────────┬─────────┘                                            │
│           │ 无效                                                 │
│           ▼                                                      │
│  ┌──────────────────┐                                           │
│  │ _fetch_token()  │                                           │
│  │ 向 OAuth 服务   │                                           │
│  │ 请求新令牌      │                                           │
│  └────────┬─────────┘                                           │
│           │                                                      │
│           ▼                                                      │
│  ┌──────────────────┐                                           │
│  │ 缓存新令牌      │                                           │
│  │ 记录过期时间    │                                           │
│  └────────┬─────────┘                                           │
│           │                                                      │
│           ▼                                                      │
│  返回 "Bearer <token>"                                           │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 6.2 令牌自动刷新

**刷新策略**：在令牌过期前 `refresh_skew_seconds` 秒（默认 60 秒）自动刷新。

```python
@staticmethod
def _is_expiring(token: _OAuthToken, oauth: McpOAuthConfig) -> bool:
    now = datetime.now(UTC)
    # 在过期前 refresh_skew_seconds 秒判定为即将过期
    return token.expires_at <= now + timedelta(seconds=max(oauth.refresh_skew_seconds, 0))
```

**示例时间线**：

```
令牌生命周期：
─────────────────────────────────────────────────────────► 时间
│                    │                    │              
0s              3540s              3600s              
(获取)          (提前60s判定)      (实际过期)         
                 触发刷新                              
```

### 6.3 工具拦截器

**作用**：在每次工具调用时自动注入 OAuth Authorization 头。

```python
async def oauth_interceptor(request: Any, handler: Any) -> Any:
    """
    拦截器执行流程：
    1. 从请求中获取 server_name
    2. 获取对应的 Authorization 头
    3. 如果有，更新请求头
    4. 调用原始处理器
    """
    header = await token_manager.get_authorization_header(request.server_name)
    if not header:
        return await handler(request)

    updated_headers = dict(request.headers or {})
    updated_headers["Authorization"] = header
    return await handler(request.override(headers=updated_headers))
```

**两阶段 OAuth 注入**：

| 阶段 | 时机 | 目的 |
|------|------|------|
| 初始化阶段 | `get_initial_oauth_headers()` | 工具发现时建立连接 |
| 调用阶段 | `oauth_interceptor` | 每次工具调用时刷新令牌 |

---

## 7. 执行流程

### 7.1 完整工具加载流程

```
┌─────────────────────────────────────────────────────────────────────────┐
│                          完整工具加载流程                                │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  应用启动                                                                │
│       │                                                                 │
│       ▼                                                                 │
│  ┌─────────────────────────────────────┐                               │
│  │ initialize_mcp_tools()              │                               │
│  │ (main.py / debug.py)                │                               │
│  └──────────────────┬──────────────────┘                               │
│                     │                                                   │
│                     ▼                                                   │
│  ┌─────────────────────────────────────┐                               │
│  │ get_mcp_tools()                     │                               │
│  │ 1. 加载 ExtensionsConfig            │                               │
│  │ 2. build_servers_config()           │                               │
│  │ 3. get_initial_oauth_headers()      │ ◄── OAuth 阶段 1             │
│  │ 4. build_oauth_tool_interceptor()   │                               │
│  │ 5. MultiServerMCPClient()           │                               │
│  │ 6. client.get_tools()               │                               │
│  └──────────────────┬──────────────────┘                               │
│                     │                                                   │
│                     ▼                                                   │
│  ┌─────────────────────────────────────┐                               │
│  │ 缓存工具列表                        │                               │
│  │ _mcp_tools_cache = tools            │                               │
│  │ _config_mtime = file.mtime          │                               │
│  └──────────────────┬──────────────────┘                               │
│                     │                                                   │
│                     ▼                                                   │
│  ┌─────────────────────────────────────┐                               │
│  │ Agent 运行时                        │                               │
│  │ get_available_tools()               │                               │
│  │  └── get_cached_mcp_tools()         │ ◄── 检查缓存有效性          │
│  │      └── 返回缓存工具               │                               │
│  └─────────────────────────────────────┘                               │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### 7.2 OAuth 完整流程

```
┌─────────────────────────────────────────────────────────────────────────┐
│                          OAuth 完整流程                                  │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  1. 工具发现阶段（初始连接）                                            │
│     ┌──────────────────────────────────────────────────────────┐       │
│     │ get_initial_oauth_headers()                              │       │
│     │   │                                                      │       │
│     │   ▼                                                      │       │
│     │ OAuthTokenManager.from_extensions_config()               │       │
│     │   │                                                      │       │
│     │   ▼                                                      │       │
│     │ for server in oauth_servers:                            │       │
│     │   │                                                      │       │
│     │   ▼                                                      │       │
│     │ _fetch_token()  ──────────────────────────────►          │       │
│     │   │                    OAuth Server                      │       │
│     │   │                    token_url                         │       │
│     │   │                                                      │       │
│     │   ▼                                                      │       │
│     │ 注入 Authorization 头到 servers_config[server]["headers"]│       │
│     └──────────────────────────────────────────────────────────┘       │
│                                                                         │
│  2. 工具调用阶段（运行时刷新）                                          │
│     ┌──────────────────────────────────────────────────────────┐       │
│     │ MCP Tool Call                                            │       │
│     │   │                                                      │       │
│     │   ▼                                                      │       │
│     │ oauth_interceptor(request, handler)                     │       │
│     │   │                                                      │       │
│     │   ▼                                                      │       │
│     │ get_authorization_header(server_name)                    │       │
│     │   │                                                      │       │
│     │   ├── 缓存有效 ──► 返回缓存令牌                          │       │
│     │   │                                                      │       │
│     │   └── 缓存过期 ──► _fetch_token() ──► 缓存新令牌        │       │
│     │                                                          │       │
│     │ request.override(headers={..., "Authorization": ...})   │       │
│     │   │                                                      │       │
│     │   ▼                                                      │       │
│     │ handler(request)  ──────────────────────────►            │       │
│     │                    MCP Server                            │       │
│     │                    (with valid token)                    │       │
│     └──────────────────────────────────────────────────────────┘       │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 8. 配置项

### 8.1 MCP 服务器配置格式

**完整配置示例**：

```json
{
  "mcpServers": {
    "filesystem": {
      "enabled": true,
      "type": "stdio",
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-filesystem", "/path/to/files"],
      "env": {},
      "description": "本地文件系统访问"
    },
    "github": {
      "enabled": true,
      "type": "stdio",
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-github"],
      "env": {
        "GITHUB_TOKEN": "$GITHUB_TOKEN"
      },
      "description": "GitHub 仓库操作"
    },
    "postgres": {
      "enabled": false,
      "type": "stdio",
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-postgres", "postgresql://localhost/mydb"],
      "env": {},
      "description": "PostgreSQL 数据库访问"
    },
    "remote-api": {
      "enabled": true,
      "type": "http",
      "url": "https://api.example.com/mcp",
      "headers": {
        "X-Custom-Header": "value"
      },
      "oauth": {
        "enabled": true,
        "token_url": "https://auth.example.com/oauth/token",
        "grant_type": "client_credentials",
        "client_id": "your-client-id",
        "client_secret": "$CLIENT_SECRET",
        "scope": "read write",
        "refresh_skew_seconds": 120
      },
      "description": "远程 API MCP 服务"
    }
  },
  "skills": {}
}
```

### 8.2 extensions_config.json 说明

**配置文件位置查找优先级**：

1. 环境变量 `DEER_FLOW_EXTENSIONS_CONFIG_PATH` 指定的路径
2. 当前目录下的 `extensions_config.json`
3. 当前目录的父目录下的 `extensions_config.json`
4. 当前目录下的 `mcp_config.json`（向后兼容）
5. 当前目录的父目录下的 `mcp_config.json`（向后兼容）

**环境变量解析**：

配置文件中的环境变量占位符（如 `$GITHUB_TOKEN`）会自动解析：

```python
# 示例配置
"env": {
  "GITHUB_TOKEN": "$GITHUB_TOKEN"  # 自动从环境变量读取
}

# 解析逻辑
if value.startswith("$"):
    env_value = os.getenv(value[1:])
    config[key] = env_value or ""
```

### 8.3 Gateway API 接口

| 端点 | 方法 | 说明 |
|------|------|------|
| `/api/mcp/config` | GET | 获取当前 MCP 配置 |
| `/api/mcp/config` | PUT | 更新 MCP 配置（写入文件并重新加载） |

**GET 响应示例**：

```json
{
  "mcp_servers": {
    "github": {
      "enabled": true,
      "type": "stdio",
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-github"],
      "env": {},
      "url": null,
      "headers": {},
      "oauth": null,
      "description": "GitHub MCP server"
    }
  }
}
```

---

## 9. 模块 API 参考

### 9.1 公共导出

```python
# __init__.py 导出的 API
from .cache import get_cached_mcp_tools, initialize_mcp_tools, reset_mcp_tools_cache
from .client import build_server_params, build_servers_config
from .tools import get_mcp_tools

__all__ = [
    "build_server_params",       # 构建单个服务器参数
    "build_servers_config",      # 构建所有服务器配置
    "get_mcp_tools",             # 异步获取 MCP 工具
    "initialize_mcp_tools",     # 初始化并缓存工具
    "get_cached_mcp_tools",     # 获取缓存工具（自动延迟初始化）
    "reset_mcp_tools_cache",    # 重置缓存
]
```

### 9.2 使用示例

**应用启动时初始化**：

```python
from src.mcp import initialize_mcp_tools

async def main():
    # 初始化 MCP 工具
    await initialize_mcp_tools()
    # ... 其他初始化
```

**获取缓存工具**：

```python
from src.mcp import get_cached_mcp_tools

def get_available_tools():
    builtin_tools = [...]
    mcp_tools = get_cached_mcp_tools()  # 自动延迟初始化
    return builtin_tools + mcp_tools
```

**重置缓存**：

```python
from src.mcp import reset_mcp_tools_cache

# 测试后重置
def tearDown():
    reset_mcp_tools_cache()
```

---

## 10. 错误处理

### 10.1 配置验证错误

| 错误 | 触发条件 | 解决方案 |
|------|----------|----------|
| `requires 'command' field` | stdio 类型缺少 command | 添加 command 字段 |
| `requires 'url' field` | sse/http 类型缺少 url | 添加 url 字段 |
| `unsupported transport type` | 未知传输类型 | 使用 stdio/sse/http |

### 10.2 OAuth 错误

| 错误 | 触发条件 | 解决方案 |
|------|----------|----------|
| `requires client_id and client_secret` | client_credentials 缺少凭据 | 添加 client_id 和 client_secret |
| `requires refresh_token` | refresh_token 模式缺少 token | 添加 refresh_token |
| `Unsupported OAuth grant type` | 未知授权类型 | 使用 client_credentials 或 refresh_token |
| `missing 'access_token'` | 响应格式错误 | 检查 token_field 配置 |

### 10.3 运行时错误处理

```python
# tools.py 中的错误处理
try:
    from langchain_mcp_adapters.client import MultiServerMCPClient
except ImportError:
    logger.warning("langchain-mcp-adapters not installed...")
    return []

try:
    # ... 工具加载
except Exception as e:
    logger.error(f"Failed to load MCP tools: {e}", exc_info=True)
    return []  # 返回空列表，不阻塞应用启动
```

---

## 11. 依赖关系

### 11.1 外部依赖

| 包名 | 用途 | 必需 |
|------|------|------|
| `langchain-mcp-adapters` | MCP 客户端实现 | 是（可选功能） |
| `langchain-core` | BaseTool 类型 | 是 |
| `httpx` | OAuth HTTP 请求 | 是（OAuth 功能） |
| `pydantic` | 配置模型验证 | 是 |

### 11.2 内部依赖

```
src/mcp/
    │
    ├── src/config/extensions_config.py  (配置定义)
    │       ├── ExtensionsConfig
    │       ├── McpServerConfig
    │       └── McpOAuthConfig
    │
    └── src/tools/tools.py  (工具消费者)
            └── get_cached_mcp_tools()
```

---

*文档生成时间：2026-03-12*
*DeerFlow MCP Module v1.0*