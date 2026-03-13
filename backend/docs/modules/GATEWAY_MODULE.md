# DeerFlow Gateway 模块技术文档

## 1. 模块概述

### 1.1 模块职责和功能定位

`/mnt/e/dev/ai/deer-flow/backend/src/gateway` 模块是 DeerFlow 框架的 API 网关层，负责：

- **FastAPI 应用创建与管理**：提供 `create_app()` 工厂函数创建配置完整的 FastAPI 应用
- **RESTful API 端点**：提供模型管理、MCP 配置、记忆管理、技能管理、文件上传、工件服务等 API
- **生命周期管理**：通过 lifespan 上下文管理器处理应用启动和关闭逻辑
- **IM 通道服务**：集成飞书、Slack、Telegram 等 IM 渠道服务
- **健康检查**：提供服务健康状态监控端点

### 1.2 与其他模块的关系

```
┌─────────────────────────────────────────────────────────────────┐
│                        gateway 模块                              │
├─────────────────────────────────────────────────────────────────┤
│  ┌─────────────┐  ┌──────────────┐  ┌───────────────┐          │
│  │   app.py    │  │   config.py  │  │  path_utils   │          │
│  │  (应用创建) │──│  (配置管理)  │──│  (路径工具)   │          │
│  └──────┬──────┘  └──────────────┘  └───────────────┘          │
│         │                                                        │
│         └───────────────────────────────────────────────────────│
│                              │                                   │
│  ┌───────────────────────────┴───────────────────────────────┐  │
│  │                      routers/                               │  │
│  │  ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌─────────┐          │  │
│  │  │ models  │ │   mcp   │ │ memory  │ │ skills  │          │  │
│  │  └─────────┘ └─────────┘ └─────────┘ └─────────┘          │  │
│  │  ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌─────────┐          │  │
│  │  │uploads  │ │artifacts│ │ agents  │ │suggest- │          │  │
│  │  │         │ │         │ │         │ │ ions    │          │  │
│  │  └─────────┘ └─────────┘ └─────────┘ └─────────┘          │  │
│  │  ┌─────────┐                                              │  │
│  │  │channels │                                              │  │
│  │  └─────────┘                                              │  │
│  └───────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
            │                    │                    │
            ▼                    ▼                    ▼
     ┌─────────────┐      ┌─────────────┐     ┌─────────────┐
     │   config/   │      │   agents/   │     │   skills/   │
     │  (配置模块) │      │  (Agent系统)│     │  (技能系统) │
     └─────────────┘      └─────────────┘     └─────────────┘
            │                    │                    │
            ▼                    ▼                    ▼
     ┌─────────────┐      ┌─────────────┐     ┌─────────────┐
     │   models/   │      │   sandbox/  │     │  channels/  │
     │  (模型工厂) │      │  (沙箱系统) │     │  (IM通道)   │
     └─────────────┘      └─────────────┘     └─────────────┘
```

**依赖关系**：
- **config**：读取应用配置、模型配置、扩展配置等
- **agents/memory**：访问记忆数据
- **skills**：加载和管理技能
- **sandbox**：文件上传和沙箱路径解析
- **channels**：IM 通道服务管理
- **models**：创建 LLM 实例（用于建议生成）

**架构说明**：
- Gateway 与 LangGraph Server 是**独立进程**
- LangGraph 请求由 nginx 反向代理处理
- Gateway 提供自定义 API 端点
- MCP 工具初始化在 LangGraph Server 中完成，Gateway 不涉及

---

## 2. 目录结构

```
backend/src/gateway/
├── __init__.py                    # 模块入口，导出主要 API
├── app.py                         # FastAPI 应用创建和生命周期管理
├── config.py                      # Gateway 配置类
├── path_utils.py                  # 虚拟路径解析工具
│
└── routers/                       # API 路由模块
    ├── __init__.py               # 路由模块导出
    ├── models.py                 # 模型管理 API
    ├── mcp.py                    # MCP 配置 API
    ├── memory.py                 # 记忆管理 API
    ├── skills.py                 # 技能管理 API
    ├── uploads.py                # 文件上传 API
    ├── artifacts.py              # 工件服务 API
    ├── agents.py                 # 自定义代理 API
    ├── suggestions.py            # 建议问题 API
    └── channels.py               # IM 通道 API
```

---

## 3. FastAPI 应用创建

### 3.1 app.py - 应用核心

#### 3.1.1 create_app() 函数

```python
def create_app() -> FastAPI:
    """创建并配置 FastAPI 应用实例。
    
    执行流程：
    1. 创建 FastAPI 实例，配置元数据和 OpenAPI 文档
    2. 定义 OpenAPI 标签（用于文档分组）
    3. 注册所有路由模块
    4. 添加健康检查端点
    
    Returns:
        配置完成的 FastAPI 应用实例
    """
    app = FastAPI(
        title="DeerFlow API Gateway",
        description="""
## DeerFlow API Gateway

API Gateway for DeerFlow - A LangGraph-based AI agent backend with sandbox execution capabilities.

### Features

- **Models Management**: Query and retrieve available AI models
- **MCP Configuration**: Manage Model Context Protocol (MCP) server configurations
- **Memory Management**: Access and manage global memory data for personalized conversations
- **Skills Management**: Query and manage skills and their enabled status
- **Artifacts**: Access thread artifacts and generated files
- **Health Monitoring**: System health check endpoints
        """,
        version="0.1.0",
        lifespan=lifespan,              # 生命周期管理器
        docs_url="/docs",               # Swagger UI 路径
        redoc_url="/redoc",             # ReDoc 路径
        openapi_url="/openapi.json",    # OpenAPI Schema 路径
        openapi_tags=[...],             # API 标签定义
    )
    
    # 注册路由
    app.include_router(models.router)
    app.include_router(mcp.router)
    # ... 其他路由
    
    # 健康检查端点
    @app.get("/health", tags=["health"])
    async def health_check() -> dict:
        return {"status": "healthy", "service": "deer-flow-gateway"}
    
    return app
```

#### 3.1.2 OpenAPI 标签定义

```python
openapi_tags=[
    {
        "name": "models",
        "description": "Operations for querying available AI models and their configurations",
    },
    {
        "name": "mcp",
        "description": "Manage Model Context Protocol (MCP) server configurations",
    },
    {
        "name": "memory",
        "description": "Access and manage global memory data for personalized conversations",
    },
    {
        "name": "skills",
        "description": "Manage skills and their configurations",
    },
    {
        "name": "artifacts",
        "description": "Access and download thread artifacts and generated files",
    },
    {
        "name": "uploads",
        "description": "Upload and manage user files for threads",
    },
    {
        "name": "agents",
        "description": "Create and manage custom agents with per-agent config and prompts",
    },
    {
        "name": "suggestions",
        "description": "Generate follow-up question suggestions for conversations",
    },
    {
        "name": "channels",
        "description": "Manage IM channel integrations (Feishu, Slack, Telegram)",
    },
    {
        "name": "health",
        "description": "Health check and system status endpoints",
    },
]
```

#### 3.1.3 lifespan 生命周期管理

```python
@asynccontextmanager
async def lifespan(app: FastAPI) -> AsyncGenerator[None, None]:
    """应用生命周期处理器。
    
    启动时执行：
    1. 加载并验证应用配置
    2. 记录启动日志
    3. 启动 IM 通道服务（如已配置）
    
    关闭时执行：
    1. 停止 IM 通道服务
    2. 记录关闭日志
    
    注意事项：
    - MCP 工具初始化不在 Gateway 中进行
    - MCP 工具由 LangGraph Server（独立进程）初始化
    - Gateway 和 LangGraph Server 拥有独立的缓存
    """
    # === 启动阶段 ===
    try:
        get_app_config()
        logger.info("Configuration loaded successfully")
    except Exception as e:
        error_msg = f"Failed to load configuration during gateway startup: {e}"
        logger.exception(error_msg)
        raise RuntimeError(error_msg) from e
    
    config = get_gateway_config()
    logger.info(f"Starting API Gateway on {config.host}:{config.port}")
    
    # 启动 IM 通道服务
    try:
        from src.channels.service import start_channel_service
        channel_service = await start_channel_service()
        logger.info("Channel service started: %s", channel_service.get_status())
    except Exception:
        logger.exception("No IM channels configured or channel service failed to start")
    
    yield  # 应用运行中
    
    # === 关闭阶段 ===
    try:
        from src.channels.service import stop_channel_service
        await stop_channel_service()
    except Exception:
        logger.exception("Failed to stop channel service")
    logger.info("Shutting down API Gateway")
```

**生命周期流程图**：

```
┌─────────────────────────────────────────────────────────────┐
│                    应用启动                                  │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│              1. 加载应用配置 (get_app_config)                │
│                 - 验证必要的环境变量                         │
│                 - 加载 config.yaml                           │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│              2. 加载 Gateway 配置                            │
│                 - host, port, cors_origins                  │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│              3. 启动 IM 通道服务 (可选)                      │
│                 - 读取 channels 配置                         │
│                 - 初始化飞书/Slack/Telegram 客户端           │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                    yield (应用运行)                          │
│                 处理 HTTP 请求                               │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│              4. 停止 IM 通道服务                             │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                    应用关闭                                  │
└─────────────────────────────────────────────────────────────┘
```

### 3.2 config.py - Gateway 配置

```python
class GatewayConfig(BaseModel):
    """API Gateway 配置类。
    
    属性：
        host: 绑定主机地址
        port: 绑定端口
        cors_origins: 允许的 CORS 源列表
    """
    host: str = Field(default="0.0.0.0", description="Host to bind the gateway server")
    port: int = Field(default=8001, description="Port to bind the gateway server")
    cors_origins: list[str] = Field(
        default_factory=lambda: ["http://localhost:3000"],
        description="Allowed CORS origins"
    )


_gateway_config: GatewayConfig | None = None


def get_gateway_config() -> GatewayConfig:
    """获取 Gateway 配置（单例模式）。
    
    支持环境变量覆盖：
    - GATEWAY_HOST: 主机地址
    - GATEWAY_PORT: 端口号
    - CORS_ORIGINS: CORS 源（逗号分隔）
    
    Returns:
        GatewayConfig 配置实例
    """
    global _gateway_config
    if _gateway_config is None:
        cors_origins_str = os.getenv("CORS_ORIGINS", "http://localhost:3000")
        _gateway_config = GatewayConfig(
            host=os.getenv("GATEWAY_HOST", "0.0.0.0"),
            port=int(os.getenv("GATEWAY_PORT", "8001")),
            cors_origins=cors_origins_str.split(","),
        )
    return _gateway_config
```

### 3.3 path_utils.py - 路径解析工具

```python
VIRTUAL_PATH_PREFIX = "/mnt/user-data"


def resolve_thread_virtual_path(thread_id: str, virtual_path: str) -> Path:
    """解析沙箱虚拟路径到实际文件系统路径。
    
    虚拟路径是 Agent 在沙箱内看到的路径，例如：
    - /mnt/user-data/outputs/file.txt
    
    实际路径是宿主机上的路径，例如：
    - {base_dir}/threads/{thread_id}/user-data/outputs/file.txt
    
    Args:
        thread_id: 线程 ID
        virtual_path: 沙箱内的虚拟路径
        
    Returns:
        解析后的实际文件系统路径
        
    Raises:
        HTTPException: 
            - 400: 路径格式无效
            - 403: 检测到路径遍历攻击
    
    安全检查：
    - 验证路径以 /mnt/user-data 开头
    - 检测路径遍历攻击（如 ../../etc/passwd）
    """
    try:
        return get_paths().resolve_virtual_path(thread_id, virtual_path)
    except ValueError as e:
        status = 403 if "traversal" in str(e) else 400
        raise HTTPException(status_code=status, detail=str(e))
```

---

## 4. 路由详解

### 4.1 models.py - 模型 API

提供 AI 模型的查询和管理接口。

#### 数据模型

```python
class ModelResponse(BaseModel):
    """模型信息响应模型"""
    name: str                          # 模型唯一标识
    display_name: str | None           # 显示名称
    description: str | None            # 模型描述
    supports_thinking: bool            # 是否支持思考模式
    supports_reasoning_effort: bool    # 是否支持推理努力程度调整


class ModelsListResponse(BaseModel):
    """模型列表响应模型"""
    models: list[ModelResponse]
```

#### API 端点

```python
router = APIRouter(prefix="/api", tags=["models"])


@router.get("/models", response_model=ModelsListResponse)
async def list_models() -> ModelsListResponse:
    """获取所有可用模型列表。
    
    返回配置文件中定义的所有模型，排除敏感字段（如 API 密钥）。
    
    Example Response:
        {
            "models": [
                {
                    "name": "gpt-4",
                    "display_name": "GPT-4",
                    "description": "OpenAI GPT-4 model",
                    "supports_thinking": false,
                    "supports_reasoning_effort": false
                },
                {
                    "name": "claude-3-opus",
                    "display_name": "Claude 3 Opus",
                    "description": "Anthropic Claude 3 Opus model",
                    "supports_thinking": true,
                    "supports_reasoning_effort": true
                }
            ]
        }
    """


@router.get("/models/{model_name}", response_model=ModelResponse)
async def get_model(model_name: str) -> ModelResponse:
    """获取指定模型的详细信息。
    
    Args:
        model_name: 模型名称
        
    Raises:
        HTTPException: 404 - 模型未找到
    """
```

---

### 4.2 mcp.py - MCP 配置 API

管理 Model Context Protocol (MCP) 服务器配置。

#### 数据模型

```python
class McpOAuthConfigResponse(BaseModel):
    """MCP OAuth 配置"""
    enabled: bool = True
    token_url: str                           # OAuth Token 端点
    grant_type: Literal["client_credentials", "refresh_token"]
    client_id: str | None
    client_secret: str | None
    refresh_token: str | None
    scope: str | None
    audience: str | None
    # ... 其他 OAuth 配置


class McpServerConfigResponse(BaseModel):
    """MCP 服务器配置"""
    enabled: bool = True
    type: str = "stdio"                      # stdio | sse | http
    command: str | None                      # stdio 命令
    args: list[str] = []                     # 命令参数
    env: dict[str, str] = {}                 # 环境变量
    url: str | None                          # sse/http URL
    headers: dict[str, str] = {}             # HTTP 头
    oauth: McpOAuthConfigResponse | None     # OAuth 配置
    description: str = ""                    # 描述


class McpConfigResponse(BaseModel):
    """MCP 配置响应"""
    mcp_servers: dict[str, McpServerConfigResponse]


class McpConfigUpdateRequest(BaseModel):
    """MCP 配置更新请求"""
    mcp_servers: dict[str, McpServerConfigResponse]
```

#### API 端点

```python
router = APIRouter(prefix="/api", tags=["mcp"])


@router.get("/mcp/config", response_model=McpConfigResponse)
async def get_mcp_configuration() -> McpConfigResponse:
    """获取当前 MCP 配置。
    
    Example Response:
        {
            "mcp_servers": {
                "github": {
                    "enabled": true,
                    "type": "stdio",
                    "command": "npx",
                    "args": ["-y", "@modelcontextprotocol/server-github"],
                    "env": {"GITHUB_TOKEN": "ghp_xxx"},
                    "description": "GitHub MCP server"
                }
            }
        }
    """


@router.put("/mcp/config", response_model=McpConfigResponse)
async def update_mcp_configuration(request: McpConfigUpdateRequest) -> McpConfigResponse:
    """更新 MCP 配置。
    
    执行流程：
    1. 确定 extensions_config.json 路径
    2. 加载当前配置（保留 skills 配置）
    3. 合并新的 MCP 配置
    4. 写入文件
    5. 重新加载配置缓存
    
    注意：
    - LangGraph Server 会通过 mtime 检测配置变更
    - 自动重新初始化 MCP 工具
    """
```

---

### 4.3 memory.py - 记忆 API

访问和管理全局记忆数据。

#### 数据模型

```python
class ContextSection(BaseModel):
    """上下文区块"""
    summary: str          # 摘要内容
    updatedAt: str        # 更新时间


class UserContext(BaseModel):
    """用户上下文"""
    workContext: ContextSection       # 工作上下文
    personalContext: ContextSection   # 个人上下文
    topOfMind: ContextSection         # 当前关注


class HistoryContext(BaseModel):
    """历史上下文"""
    recentMonths: ContextSection      # 最近几个月
    earlierContext: ContextSection    # 更早的上下文
    longTermBackground: ContextSection # 长期背景


class Fact(BaseModel):
    """记忆事实"""
    id: str                 # 事实 ID
    content: str            # 事实内容
    category: str           # 类别：preference|knowledge|context|behavior|goal
    confidence: float       # 置信度 (0-1)
    createdAt: str          # 创建时间
    source: str             # 来源线程 ID


class MemoryResponse(BaseModel):
    """记忆数据响应"""
    version: str = "1.0"
    lastUpdated: str
    user: UserContext
    history: HistoryContext
    facts: list[Fact]


class MemoryConfigResponse(BaseModel):
    """记忆配置响应"""
    enabled: bool
    storage_path: str
    debounce_seconds: int
    max_facts: int
    fact_confidence_threshold: float
    injection_enabled: bool
    max_injection_tokens: int


class MemoryStatusResponse(BaseModel):
    """记忆状态响应"""
    config: MemoryConfigResponse
    data: MemoryResponse
```

#### API 端点

```python
router = APIRouter(prefix="/api", tags=["memory"])


@router.get("/memory", response_model=MemoryResponse)
async def get_memory() -> MemoryResponse:
    """获取当前全局记忆数据。
    
    包含：
    - 用户上下文（工作、个人、当前关注）
    - 历史上下文（近期、早期、长期）
    - 事实列表
    """


@router.post("/memory/reload", response_model=MemoryResponse)
async def reload_memory() -> MemoryResponse:
    """从存储文件重新加载记忆数据。
    
    用于文件被外部修改后刷新内存缓存。
    """


@router.get("/memory/config", response_model=MemoryConfigResponse)
async def get_memory_config_endpoint() -> MemoryConfigResponse:
    """获取记忆系统配置。"""


@router.get("/memory/status", response_model=MemoryStatusResponse)
async def get_memory_status() -> MemoryStatusResponse:
    """获取记忆状态（配置 + 数据）。"""
```

---

### 4.4 skills.py - 技能 API

管理技能列表、启用状态和安装。

#### 数据模型

```python
class SkillResponse(BaseModel):
    """技能信息响应"""
    name: str                # 技能名称
    description: str         # 描述
    license: str | None      # 许可证
    category: str            # 类别：public | custom
    enabled: bool            # 是否启用


class SkillsListResponse(BaseModel):
    """技能列表响应"""
    skills: list[SkillResponse]


class SkillUpdateRequest(BaseModel):
    """技能更新请求"""
    enabled: bool


class SkillInstallRequest(BaseModel):
    """技能安装请求"""
    thread_id: str           # 线程 ID
    path: str                # .skill 文件的虚拟路径


class SkillInstallResponse(BaseModel):
    """技能安装响应"""
    success: bool
    skill_name: str
    message: str
```

#### 技能验证

```python
# 允许的 SKILL.md frontmatter 属性
ALLOWED_FRONTMATTER_PROPERTIES = {"name", "description", "license", "allowed-tools", "metadata"}


def _validate_skill_frontmatter(skill_dir: Path) -> tuple[bool, str, str | None]:
    """验证技能目录的 SKILL.md frontmatter。
    
    验证规则：
    1. 必须存在 SKILL.md 文件
    2. 必须包含 YAML frontmatter
    3. frontmatter 必须是字典
    4. 不允许意外的属性
    5. 必须有 name 和 description
    6. name 必须是 hyphen-case（小写字母、数字、连字符）
    7. name 不能以连字符开头/结尾，不能有连续连字符
    8. name 最大 64 字符
    9. description 最大 1024 字符，不能包含 <>
    
    Returns:
        (is_valid, message, skill_name)
    """
```

#### API 端点

```python
router = APIRouter(prefix="/api", tags=["skills"])


@router.get("/skills", response_model=SkillsListResponse)
async def list_skills() -> SkillsListResponse:
    """获取所有技能列表。
    
    包括 public 和 custom 目录下的所有技能。
    """


@router.get("/skills/{skill_name}", response_model=SkillResponse)
async def get_skill(skill_name: str) -> SkillResponse:
    """获取指定技能详情。"""


@router.put("/skills/{skill_name}", response_model=SkillResponse)
async def update_skill(skill_name: str, request: SkillUpdateRequest) -> SkillResponse:
    """更新技能启用状态。
    
    修改 extensions_config.json 中的 skills 配置。
    """


@router.post("/skills/install", response_model=SkillInstallResponse)
async def install_skill(request: SkillInstallRequest) -> SkillInstallResponse:
    """从 .skill 文件安装技能。
    
    .skill 文件是 ZIP 压缩包，包含：
    - SKILL.md（必需）
    - scripts/（可选）
    - references/（可选）
    - assets/（可选）
    
    安装流程：
    1. 解析虚拟路径获取实际文件
    2. 验证文件存在且为 ZIP 格式
    3. 解压到临时目录
    4. 验证 SKILL.md frontmatter
    5. 复制到 custom 技能目录
    
    Raises:
        HTTPException:
            - 400: 无效的 .skill 文件
            - 403: 路径遍历攻击
            - 404: 文件未找到
            - 409: 技能已存在
    """
```

---

### 4.5 uploads.py - 文件上传 API

处理用户文件上传，支持自动转换为 Markdown。

#### 配置

```python
# 可转换为 Markdown 的文件扩展名
CONVERTIBLE_EXTENSIONS = {
    ".pdf", ".ppt", ".pptx",
    ".xls", ".xlsx",
    ".doc", ".docx",
}
```

#### 数据模型

```python
class UploadResponse(BaseModel):
    """上传响应"""
    success: bool
    files: list[dict[str, str]]  # 文件信息列表
    message: str
```

#### 辅助函数

```python
def get_uploads_dir(thread_id: str) -> Path:
    """获取线程的上传目录。
    
    路径：{base_dir}/threads/{thread_id}/user-data/uploads/
    """


async def convert_file_to_markdown(file_path: Path) -> Path | None:
    """使用 markitdown 将文件转换为 Markdown。
    
    支持格式：PDF, PPT, Excel, Word
    
    生成同名 .md 文件。
    """
```

#### API 端点

```python
router = APIRouter(prefix="/api/threads/{thread_id}/uploads", tags=["uploads"])


@router.post("", response_model=UploadResponse)
async def upload_files(thread_id: str, files: list[UploadFile] = File(...)) -> UploadResponse:
    """上传多个文件到线程的上传目录。
    
    处理流程：
    1. 安全检查文件名（防止路径遍历）
    2. 保存原始文件
    3. 对于可转换格式，生成 Markdown 版本
    4. 同步到沙箱（非 local 沙箱）
    
    返回的文件信息包括：
    - filename: 文件名
    - size: 文件大小
    - path: 实际文件系统路径
    - virtual_path: 沙箱内虚拟路径
    - artifact_url: HTTP 访问 URL
    - markdown_file: Markdown 文件名（如已转换）
    - markdown_path: Markdown 文件路径
    - markdown_virtual_path: Markdown 虚拟路径
    - markdown_artifact_url: Markdown 访问 URL
    """


@router.get("/list", response_model=dict)
async def list_uploaded_files(thread_id: str) -> dict:
    """列出线程上传目录中的所有文件。"""


@router.delete("/{filename}")
async def delete_uploaded_file(thread_id: str, filename: str) -> dict:
    """删除上传的文件。
    
    安全检查：确保文件在上传目录内。
    """
```

---

### 4.6 artifacts.py - 工件服务 API

提供 Agent 生成工件的访问服务。

#### 辅助函数

```python
def is_text_file_by_content(path: Path, sample_size: int = 8192) -> bool:
    """通过内容检测是否为文本文件。
    
    文本文件不包含 null 字节。
    """


def _extract_file_from_skill_archive(zip_path: Path, internal_path: str) -> bytes | None:
    """从 .skill ZIP 压缩包中提取文件。
    
    支持访问 .skill 文件内的资源，例如：
    /api/threads/{id}/artifacts/mnt/user-data/outputs/my.skill/SKILL.md
    """
```

#### API 端点

```python
router = APIRouter(prefix="/api", tags=["artifacts"])


@router.get("/threads/{thread_id}/artifacts/{path:path}")
async def get_artifact(thread_id: str, path: str, request: Request) -> FileResponse:
    """获取工件文件。
    
    路径格式：
    - 普通文件：mnt/user-data/outputs/file.txt
    - .skill 内文件：mnt/user-data/outputs/my.skill/SKILL.md
    
    响应类型自动检测：
    - HTML：渲染为 HTML
    - 文本：纯文本响应
    - 二进制：带 Content-Disposition 的响应
    
    查询参数：
    - download=true：强制下载（attachment）
    
    安全检查：
    - 验证路径以 /mnt/user-data 开头
    - 检测路径遍历攻击
    
    缓存：
    - .skill 内文件缓存 5 分钟
    """
```

---

### 4.7 agents.py - 自定义代理 API

提供自定义 Agent 的 CRUD 操作。

#### 数据模型

```python
class AgentResponse(BaseModel):
    """代理响应"""
    name: str                          # 代理名称
    description: str = ""              # 描述
    model: str | None = None           # 模型覆盖
    tool_groups: list[str] | None      # 工具组白名单
    soul: str | None = None            # SOUL.md 内容


class AgentsListResponse(BaseModel):
    """代理列表响应"""
    agents: list[AgentResponse]


class AgentCreateRequest(BaseModel):
    """代理创建请求"""
    name: str                          # 必须匹配 ^[A-Za-z0-9-]+$
    description: str = ""
    model: str | None = None
    tool_groups: list[str] | None = None
    soul: str = ""                     # SOUL.md 内容


class AgentUpdateRequest(BaseModel):
    """代理更新请求"""
    description: str | None = None
    model: str | None = None
    tool_groups: list[str] | None = None
    soul: str | None = None


class UserProfileResponse(BaseModel):
    """用户配置响应"""
    content: str | None = None         # USER.md 内容


class UserProfileUpdateRequest(BaseModel):
    """用户配置更新请求"""
    content: str = ""
```

#### 验证函数

```python
AGENT_NAME_PATTERN = re.compile(r"^[A-Za-z0-9-]+$")


def _validate_agent_name(name: str) -> None:
    """验证代理名称。
    
    规则：
    - 仅允许字母、数字、连字符
    - 不区分大小写（存储时转为小写）
    
    Raises:
        HTTPException: 422 - 名称无效
    """


def _normalize_agent_name(name: str) -> str:
    """规范化代理名称为小写。"""
```

#### API 端点

```python
router = APIRouter(prefix="/api", tags=["agents"])


@router.get("/agents", response_model=AgentsListResponse)
async def list_agents() -> AgentsListResponse:
    """列出所有自定义代理。"""


@router.get("/agents/check")
async def check_agent_name(name: str) -> dict:
    """检查代理名称是否可用。
    
    Returns:
        {"available": true/false, "name": "<normalized>"}
    """


@router.get("/agents/{name}", response_model=AgentResponse)
async def get_agent(name: str) -> AgentResponse:
    """获取代理详情（包含 SOUL.md）。"""


@router.post("/agents", response_model=AgentResponse, status_code=201)
async def create_agent_endpoint(request: AgentCreateRequest) -> AgentResponse:
    """创建自定义代理。
    
    创建内容：
    - config.yaml：代理配置
    - SOUL.md：代理个性
    
    目录结构：
    {base_dir}/agents/{name}/
    ├── config.yaml
    └── SOUL.md
    
    Raises:
        HTTPException:
            - 409: 代理已存在
            - 422: 名称无效
    """


@router.put("/agents/{name}", response_model=AgentResponse)
async def update_agent(name: str, request: AgentUpdateRequest) -> AgentResponse:
    """更新自定义代理。"""


@router.delete("/agents/{name}", status_code=204)
async def delete_agent(name: str) -> None:
    """删除自定义代理。"""


# === 用户配置 ===

@router.get("/user-profile", response_model=UserProfileResponse)
async def get_user_profile() -> UserProfileResponse:
    """获取全局用户配置（USER.md）。"""


@router.put("/user-profile", response_model=UserProfileResponse)
async def update_user_profile(request: UserProfileUpdateRequest) -> UserProfileResponse:
    """更新全局用户配置。
    
    USER.md 会被注入到所有自定义代理的系统提示词中。
    """
```

---

### 4.8 suggestions.py - 建议问题 API

基于对话上下文生成后续问题建议。

#### 数据模型

```python
class SuggestionMessage(BaseModel):
    """对话消息"""
    role: str       # user | assistant
    content: str    # 消息内容


class SuggestionsRequest(BaseModel):
    """建议请求"""
    messages: list[SuggestionMessage]   # 近期对话
    n: int = 3                          # 建议数量 (1-5)
    model_name: str | None = None       # 可选模型覆盖


class SuggestionsResponse(BaseModel):
    """建议响应"""
    suggestions: list[str]
```

#### 辅助函数

```python
def _strip_markdown_code_fence(text: str) -> str:
    """移除 Markdown 代码围栏。"""


def _parse_json_string_list(text: str) -> list[str] | None:
    """从文本中解析 JSON 字符串列表。"""


def _format_conversation(messages: list[SuggestionMessage]) -> str:
    """格式化对话用于提示词。"""
```

#### API 端点

```python
router = APIRouter(prefix="/api", tags=["suggestions"])


@router.post("/threads/{thread_id}/suggestions", response_model=SuggestionsResponse)
async def generate_suggestions(thread_id: str, request: SuggestionsRequest) -> SuggestionsResponse:
    """生成后续问题建议。
    
    使用 LLM 根据对话上下文生成用户可能想问的问题。
    
    提示词要求：
    - 生成 EXACTLY n 个问题
    - 问题与对话相关
    - 使用用户相同的语言
    - 每个问题简洁（<= 20 词 / <= 40 中文字符）
    - 输出 JSON 数组格式
    """
```

---

### 4.9 channels.py - IM 通道 API

管理 IM 集成服务（飞书、Slack、Telegram）。

#### 数据模型

```python
class ChannelStatusResponse(BaseModel):
    """通道状态响应"""
    service_running: bool
    channels: dict[str, dict]   # 通道名称 -> 状态信息


class ChannelRestartResponse(BaseModel):
    """通道重启响应"""
    success: bool
    message: str
```

#### API 端点

```python
router = APIRouter(prefix="/api/channels", tags=["channels"])


@router.get("/", response_model=ChannelStatusResponse)
async def get_channels_status() -> ChannelStatusResponse:
    """获取所有 IM 通道状态。
    
    返回：
    - service_running: 服务是否运行
    - channels: 各通道状态
        - enabled: 是否启用
        - running: 是否运行中
    """


@router.post("/{name}/restart", response_model=ChannelRestartResponse)
async def restart_channel(name: str) -> ChannelRestartResponse:
    """重启指定 IM 通道。
    
    支持的通道：
    - feishu（飞书）
    - slack
    - telegram
    
    Raises:
        HTTPException: 503 - 通道服务未运行
    """
```

---

## 5. API 端点列表

### 5.1 完整端点表格

| 路径 | 方法 | 标签 | 描述 |
|------|------|------|------|
| `/health` | GET | health | 健康检查 |
| `/api/models` | GET | models | 获取模型列表 |
| `/api/models/{model_name}` | GET | models | 获取模型详情 |
| `/api/mcp/config` | GET | mcp | 获取 MCP 配置 |
| `/api/mcp/config` | PUT | mcp | 更新 MCP 配置 |
| `/api/memory` | GET | memory | 获取记忆数据 |
| `/api/memory/reload` | POST | memory | 重新加载记忆 |
| `/api/memory/config` | GET | memory | 获取记忆配置 |
| `/api/memory/status` | GET | memory | 获取记忆状态 |
| `/api/skills` | GET | skills | 获取技能列表 |
| `/api/skills/{skill_name}` | GET | skills | 获取技能详情 |
| `/api/skills/{skill_name}` | PUT | skills | 更新技能状态 |
| `/api/skills/install` | POST | skills | 安装技能 |
| `/api/threads/{thread_id}/uploads` | POST | uploads | 上传文件 |
| `/api/threads/{thread_id}/uploads/list` | GET | uploads | 列出上传文件 |
| `/api/threads/{thread_id}/uploads/{filename}` | DELETE | uploads | 删除上传文件 |
| `/api/threads/{thread_id}/artifacts/{path:path}` | GET | artifacts | 获取工件文件 |
| `/api/threads/{thread_id}/suggestions` | POST | suggestions | 生成建议问题 |
| `/api/agents` | GET | agents | 获取代理列表 |
| `/api/agents/check` | GET | agents | 检查代理名称 |
| `/api/agents` | POST | agents | 创建代理 |
| `/api/agents/{name}` | GET | agents | 获取代理详情 |
| `/api/agents/{name}` | PUT | agents | 更新代理 |
| `/api/agents/{name}` | DELETE | agents | 删除代理 |
| `/api/user-profile` | GET | agents | 获取用户配置 |
| `/api/user-profile` | PUT | agents | 更新用户配置 |
| `/api/channels/` | GET | channels | 获取通道状态 |
| `/api/channels/{name}/restart` | POST | channels | 重启通道 |

### 5.2 端点分类统计

| 分类 | 端点数 | 功能描述 |
|------|--------|----------|
| health | 1 | 健康检查 |
| models | 2 | 模型查询 |
| mcp | 2 | MCP 配置管理 |
| memory | 4 | 记忆管理 |
| skills | 4 | 技能管理 |
| uploads | 3 | 文件上传 |
| artifacts | 1 | 工件访问 |
| suggestions | 1 | 建议生成 |
| agents | 8 | 代理管理 |
| channels | 2 | IM 通道 |
| **总计** | **28** | |

---

## 6. 中间件配置

### 6.1 CORS 配置

**重要说明**：CORS 由 nginx 处理，FastAPI 层不添加 CORS 中间件。

```python
# app.py 中的注释
# CORS is handled by nginx - no need for FastAPI middleware
```

**Gateway 配置**（仅供参考）：

```python
class GatewayConfig(BaseModel):
    cors_origins: list[str] = Field(
        default_factory=lambda: ["http://localhost:3000"],
        description="Allowed CORS origins"
    )
```

**nginx 配置示例**：

```nginx
# CORS headers for gateway
add_header 'Access-Control-Allow-Origin' '$http_origin' always;
add_header 'Access-Control-Allow-Methods' 'GET, POST, PUT, DELETE, OPTIONS' always;
add_header 'Access-Control-Allow-Headers' 'Authorization, Content-Type' always;
add_header 'Access-Control-Allow-Credentials' 'true' always;
```

### 6.2 日志配置

```python
# app.py
logging.basicConfig(
    level=logging.INFO,
    format="%(asctime)s - %(name)s - %(levelname)s - %(message)s",
    datefmt="%Y-%m-%d %H:%M:%S",
)
logger = logging.getLogger(__name__)
```

---

## 7. 启动和运行

### 7.1 开发模式运行

#### 使用 Makefile

```bash
# 在项目根目录
make dev    # 启动所有服务（包括 Gateway）
```

#### 单独运行 Gateway

```bash
# 在 backend/ 目录
make gateway    # 使用 uv 运行 Gateway（端口 8001）
```

#### 直接使用 uvicorn

```bash
cd backend/
uv run uvicorn src.gateway.app:app --host 0.0.0.0 --port 8001 --reload
```

### 7.2 生产部署

#### 使用 Gunicorn + Uvicorn

```bash
gunicorn src.gateway.app:app \
    --workers 4 \
    --worker-class uvicorn.workers.UvicornWorker \
    --bind 0.0.0.0:8001 \
    --timeout 120
```

#### 使用 Docker

```dockerfile
FROM python:3.12-slim

WORKDIR /app
COPY backend/ .

RUN pip install uv && uv sync

EXPOSE 8001

CMD ["uv", "run", "uvicorn", "src.gateway.app:app", "--host", "0.0.0.0", "--port", "8001"]
```

### 7.3 环境变量

| 变量名 | 默认值 | 说明 |
|--------|--------|------|
| `GATEWAY_HOST` | `0.0.0.0` | Gateway 绑定主机 |
| `GATEWAY_PORT` | `8001` | Gateway 绑定端口 |
| `CORS_ORIGINS` | `http://localhost:3000` | CORS 允许源（逗号分隔） |
| `DEER_FLOW_HOME` | `~/.deer-flow` | 应用数据目录 |

### 7.4 端口分配

| 服务 | 端口 | 说明 |
|------|------|------|
| LangGraph Server | 2024 | Agent 执行服务 |
| Gateway API | 8001 | RESTful API |
| Frontend | 3000 | Next.js 前端 |
| nginx | 2026 | 统一入口 |

### 7.5 架构图

```
                         ┌─────────────────────────────────┐
                         │          用户浏览器              │
                         └───────────────┬─────────────────┘
                                         │
                                         ▼
                         ┌─────────────────────────────────┐
                         │        nginx (Port 2026)        │
                         │        统一入口 + CORS           │
                         └───────────────┬─────────────────┘
                                         │
                    ┌────────────────────┼────────────────────┐
                    │                    │                    │
                    ▼                    ▼                    ▼
         ┌──────────────────┐ ┌──────────────────┐ ┌──────────────────┐
         │   Frontend       │ │   LangGraph      │ │    Gateway       │
         │   (Port 3000)    │ │   (Port 2024)    │ │   (Port 8001)    │
         │                  │ │                  │ │                  │
         │   Next.js 16     │ │   Agent 执行     │ │   RESTful API    │
         │   React 19       │ │   LangGraph      │ │   FastAPI        │
         └──────────────────┘ └──────────────────┘ └──────────────────┘
                    │                    │                    │
                    └────────────────────┴────────────────────┘
                                         │
                                         ▼
                         ┌─────────────────────────────────┐
                         │       共享资源                   │
                         │  - config.yaml                  │
                         │  - extensions_config.json       │
                         │  - .deer-flow/ (数据目录)        │
                         └─────────────────────────────────┘
```

---

## 8. 错误处理

### 8.1 标准 HTTP 错误码

| 状态码 | 说明 | 使用场景 |
|--------|------|----------|
| 200 | 成功 | GET/PUT 成功 |
| 201 | 已创建 | POST 创建成功 |
| 204 | 无内容 | DELETE 成功 |
| 400 | 错误请求 | 参数无效、格式错误 |
| 403 | 禁止访问 | 路径遍历攻击 |
| 404 | 未找到 | 资源不存在 |
| 409 | 冲突 | 资源已存在 |
| 422 | 无法处理 | 验证失败 |
| 500 | 服务器错误 | 内部错误 |
| 503 | 服务不可用 | 依赖服务未运行 |

### 8.2 错误响应格式

```json
{
    "detail": "错误描述信息"
}
```

### 8.3 常见错误示例

```python
# 404 - 资源未找到
raise HTTPException(status_code=404, detail=f"Model '{model_name}' not found")

# 400 - 参数无效
raise HTTPException(status_code=400, detail="File must have .skill extension")

# 403 - 路径遍历
raise HTTPException(status_code=403, detail="Access denied: path traversal detected")

# 409 - 冲突
raise HTTPException(status_code=409, detail=f"Agent '{name}' already exists")

# 422 - 验证失败
raise HTTPException(status_code=422, detail=f"Invalid agent name '{name}'")

# 500 - 内部错误
raise HTTPException(status_code=500, detail=f"Failed to update configuration: {str(e)}")

# 503 - 服务不可用
raise HTTPException(status_code=503, detail="Channel service is not running")
```

---

## 9. 测试

### 9.1 测试命令

```bash
# 运行所有测试
cd backend/
make test

# 运行 Gateway 相关测试
PYTHONPATH=. uv run pytest tests/gateway/ -v
```

### 9.2 API 测试示例

```bash
# 健康检查
curl http://localhost:8001/health

# 获取模型列表
curl http://localhost:8001/api/models

# 获取 MCP 配置
curl http://localhost:8001/api/mcp/config

# 获取记忆数据
curl http://localhost:8001/api/memory

# 获取技能列表
curl http://localhost:8001/api/skills

# 上传文件
curl -X POST http://localhost:8001/api/threads/test-thread/uploads \
    -F "files=@test.txt"

# 获取工件
curl http://localhost:8001/api/threads/test-thread/artifacts/mnt/user-data/outputs/test.txt

# 获取代理列表
curl http://localhost:8001/api/agents

# 获取通道状态
curl http://localhost:8001/api/channels/
```

---

## 10. 最佳实践

### 10.1 路由设计

- 每个路由模块使用独立的 `APIRouter`
- 使用 `prefix` 参数定义路由前缀
- 使用 `tags` 参数分组 API 文档
- 定义详细的 `response_model` 和 `summary`

### 10.2 错误处理

- 使用 `HTTPException` 抛出错误
- 提供清晰的错误描述
- 记录详细的错误日志

### 10.3 安全考虑

- 验证所有用户输入
- 使用路径解析工具防止路径遍历
- 敏感信息（API Key）不暴露在响应中
- CORS 由 nginx 统一处理

### 10.4 性能优化

- 使用单例模式缓存配置
- 文件操作使用异步 I/O
- 合理设置缓存头
- 大文件使用流式响应

---

## 11. 参考资源

- [FastAPI 官方文档](https://fastapi.tiangolo.com/)
- [Pydantic 文档](https://docs.pydantic.dev/)
- [Uvicorn 文档](https://www.uvicorn.org/)
- [DeerFlow 主文档](../../../AGENTS.md)
- [配置模块文档](./CONFIG_MODULE.md)
- [Agent 模块文档](./AGENTS_MODULE.md)