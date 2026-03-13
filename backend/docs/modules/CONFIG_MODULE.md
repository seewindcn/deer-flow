# DeerFlow Config 模块技术文档

## 1. 模块概述

### 1.1 模块职责和功能定位

`/mnt/e/dev/ai/deer-flow/backend/src/config` 模块是 DeerFlow 框架的配置管理中心，负责：

- **主配置管理**：加载和解析 `config.yaml` 主配置文件
- **扩展配置管理**：管理 MCP 服务器和技能状态配置（`extensions_config.json`）
- **模型配置**：定义和管理 LLM 模型参数和思考模式配置
- **沙箱配置**：配置代码执行沙箱环境和参数
- **记忆配置**：配置记忆系统的存储和行为参数
- **子代理配置**：配置子代理系统的超时和覆盖参数
- **追踪配置**：配置 LangSmith 追踪和监控
- **检查点配置**：配置对话状态持久化后端
- **摘要配置**：配置对话自动摘要行为
- **标题配置**：配置自动标题生成行为
- **路径管理**：统一管理应用数据存储路径

### 1.2 配置文件层次结构

```
DeerFlow 配置层次
│
├── config.yaml (主配置文件)
│   ├── models[]           # LLM 模型配置列表
│   ├── sandbox            # 沙箱配置
│   ├── tools[]            # 工具配置列表
│   ├── tool_groups[]      # 工具组配置
│   ├── skills             # 技能目录配置
│   ├── memory             # 记忆系统配置
│   ├── subagents          # 子代理配置
│   ├── checkpointer       # 检查点配置
│   ├── summarization      # 摘要配置
│   └── title              # 标题生成配置
│
└── extensions_config.json (扩展配置文件)
    ├── mcpServers         # MCP 服务器配置
    └── skills             # 技能状态配置
```

### 1.3 与其他模块的关系

```
┌─────────────────────────────────────────────────────────────────┐
│                        config 模块                               │
├─────────────────────────────────────────────────────────────────┤
│  ┌─────────────┐  ┌──────────────┐  ┌───────────────┐          │
│  │ app_config  │  │ model_config │  │extensions_cfg │          │
│  │  (主配置)   │──│  (模型配置)  │──│  (扩展配置)   │          │
│  └──────┬──────┘  └──────┬───────┘  └───────┬───────┘          │
│         │                │                   │                   │
│         ├────────────────┼───────────────────┤                   │
│         │                │                   │                   │
│  ┌──────┴──────┐  ┌──────┴───────┐  ┌───────┴───────┐          │
│  │sandbox_cfg  │  │ memory_config│  │ tracing_config│          │
│  │(沙箱配置)   │  │ (记忆配置)   │  │ (追踪配置)    │          │
│  └─────────────┘  └──────────────┘  └───────────────┘          │
│                                                                  │
│  ┌─────────────┐  ┌──────────────┐  ┌───────────────┐          │
│  │subagents_cfg│  │checkpointer  │  │summarization  │          │
│  │(子代理配置) │  │_config       │  │_config        │          │
│  └─────────────┘  └──────────────┘  └───────────────┘          │
└─────────────────────────────────────────────────────────────────┘
            │                    │                    │
            ▼                    ▼                    ▼
     ┌─────────────┐      ┌─────────────┐     ┌─────────────┐
     │   agents/   │      │   models/   │     │   sandbox/  │
     │  (Agent系统)│      │  (模型工厂) │     │  (沙箱系统) │
     └─────────────┘      └─────────────┘     └─────────────┘
```

**被依赖关系**：
- **agents**：读取所有配置以创建和配置 Agent
- **models**：读取模型配置创建 LLM 实例
- **sandbox**：读取沙箱配置初始化执行环境
- **mcp**：读取扩展配置加载 MCP 服务器
- **memory**：读取记忆配置管理持久化

---

## 2. 目录结构

```
backend/src/config/
├── __init__.py                    # 模块入口，导出主要 API
├── app_config.py                  # 主配置类和加载器
├── model_config.py                # 模型配置类
├── extensions_config.py           # 扩展配置（MCP + 技能状态）
├── sandbox_config.py              # 沙箱配置类
├── memory_config.py               # 记忆系统配置类
├── subagents_config.py            # 子代理配置类
├── tracing_config.py              # LangSmith 追踪配置类
├── checkpointer_config.py         # 检查点配置类
├── summarization_config.py        # 摘要配置类
├── title_config.py                # 标题生成配置类
├── tool_config.py                 # 工具配置类
├── skills_config.py               # 技能目录配置类
├── agents_config.py               # 自定义 Agent 配置加载器
└── paths.py                       # 路径管理类
```

---

## 3. 主配置 (app_config.py)

### 3.1 AppConfig 类定义

```python
class AppConfig(BaseModel):
    """DeerFlow 应用主配置类
    
    使用 Pydantic 进行配置验证和类型转换。
    支持 extra="allow" 以允许未定义的额外字段。
    """
    
    # 可用的 LLM 模型列表
    models: list[ModelConfig] = Field(
        default_factory=list, 
        description="Available models"
    )
    
    # 沙箱执行环境配置
    sandbox: SandboxConfig = Field(
        description="Sandbox configuration"
    )
    
    # 可用工具列表
    tools: list[ToolConfig] = Field(
        default_factory=list, 
        description="Available tools"
    )
    
    # 工具组配置列表
    tool_groups: list[ToolGroupConfig] = Field(
        default_factory=list, 
        description="Available tool groups"
    )
    
    # 技能目录配置
    skills: SkillsConfig = Field(
        default_factory=SkillsConfig, 
        description="Skills configuration"
    )
    
    # 扩展配置（MCP 服务器和技能状态）
    extensions: ExtensionsConfig = Field(
        default_factory=ExtensionsConfig, 
        description="Extensions configuration (MCP servers and skills state)"
    )
    
    # 检查点配置（可选）
    checkpointer: CheckpointerConfig | None = Field(
        default=None, 
        description="Checkpointer configuration"
    )
    
    # Pydantic 配置
    model_config = ConfigDict(extra="allow", frozen=False)
```

### 3.2 配置加载流程

```python
@classmethod
def from_file(cls, config_path: str | None = None) -> Self:
    """
    从 YAML 文件加载配置的完整流程：
    
    1. 解析配置文件路径（resolve_config_path）
    2. 读取 YAML 文件内容
    3. 解析环境变量引用
    4. 加载子模块配置
    5. 加载扩展配置（extensions_config.json）
    6. 验证并返回配置实例
    """
    # 1. 解析配置文件路径
    resolved_path = cls.resolve_config_path(config_path)
    
    # 2. 读取 YAML 文件
    with open(resolved_path, encoding="utf-8") as f:
        config_data = yaml.safe_load(f)
    
    # 3. 解析环境变量
    config_data = cls.resolve_env_variables(config_data)
    
    # 4. 加载子模块配置（注册到全局单例）
    if "title" in config_data:
        load_title_config_from_dict(config_data["title"])
    
    if "summarization" in config_data:
        load_summarization_config_from_dict(config_data["summarization"])
    
    if "memory" in config_data:
        load_memory_config_from_dict(config_data["memory"])
    
    if "subagents" in config_data:
        load_subagents_config_from_dict(config_data["subagents"])
    
    if "checkpointer" in config_data:
        load_checkpointer_config_from_dict(config_data["checkpointer"])
    
    # 5. 加载扩展配置（独立文件）
    extensions_config = ExtensionsConfig.from_file()
    config_data["extensions"] = extensions_config.model_dump()
    
    # 6. 验证并返回
    result = cls.model_validate(config_data)
    return result
```

### 3.3 配置文件路径解析

```python
@classmethod
def resolve_config_path(cls, config_path: str | None = None) -> Path:
    """
    解析配置文件路径的优先级顺序：
    
    1. 函数参数指定的路径（config_path）
    2. 环境变量 DEER_FLOW_CONFIG_PATH
    3. 当前工作目录下的 config.yaml
    4. 当前工作目录父目录下的 config.yaml
    
    这个设计允许：
    - 测试时传入临时配置路径
    - 生产环境通过环境变量指定配置
    - 开发时自动查找项目根目录的配置文件
    """
    # 优先级 1：函数参数
    if config_path:
        path = Path(config_path)
        if not Path.exists(path):
            raise FileNotFoundError(
                f"Config file specified by param `config_path` not found at {path}"
            )
        return path
    
    # 优先级 2：环境变量
    elif os.getenv("DEER_FLOW_CONFIG_PATH"):
        path = Path(os.getenv("DEER_FLOW_CONFIG_PATH"))
        if not Path.exists(path):
            raise FileNotFoundError(
                f"Config file specified by environment variable "
                f"`DEER_FLOW_CONFIG_PATH` not found at {path}"
            )
        return path
    
    # 优先级 3 & 4：自动查找
    else:
        # 当前目录
        path = Path(os.getcwd()) / "config.yaml"
        if not path.exists():
            # 父目录
            path = Path(os.getcwd()).parent / "config.yaml"
            if not path.exists():
                raise FileNotFoundError(
                    "`config.yaml` file not found at the current "
                    "directory nor its parent directory"
                )
        return path
```

### 3.4 环境变量解析

```python
@classmethod
def resolve_env_variables(cls, config: Any) -> Any:
    """
    递归解析配置中的环境变量引用。
    
    语法：使用 $ 前缀引用环境变量
    例如：$OPENAI_API_KEY 会被解析为 os.getenv("OPENAI_API_KEY")
    
    支持的类型：
    - 字符串：检测 $ 前缀并替换
    - 字典：递归处理所有值
    - 列表：递归处理所有元素
    
    示例配置：
        models:
          - api_key: $OPENAI_API_KEY  # 从环境变量读取
    """
    if isinstance(config, str):
        if config.startswith("$"):
            env_name = config[1:]
            env_value = os.getenv(env_name)
            if env_value is None:
                raise ValueError(
                    f"Environment variable {env_name} not found for config value {config}"
                )
            return env_value
        return config
    
    elif isinstance(config, dict):
        return {k: cls.resolve_env_variables(v) for k, v in config.items()}
    
    elif isinstance(config, list):
        return [cls.resolve_env_variables(item) for item in config]
    
    return config
```

### 3.5 单例模式实现

```python
# 全局配置实例缓存
_app_config: AppConfig | None = None


def get_app_config() -> AppConfig:
    """
    获取应用配置的单例实例。
    
    首次调用时从文件加载，后续调用返回缓存的实例。
    这是 DeerFlow 中访问配置的主要入口点。
    
    Returns:
        AppConfig: 全局配置实例
    
    Example:
        >>> config = get_app_config()
        >>> model = config.get_model_config("gpt-4")
    """
    global _app_config
    if _app_config is None:
        _app_config = AppConfig.from_file()
    return _app_config


def reload_app_config(config_path: str | None = None) -> AppConfig:
    """
    重新加载配置文件并更新缓存实例。
    
    用于配置文件修改后热更新，无需重启应用。
    
    Args:
        config_path: 可选的配置文件路径
    
    Returns:
        新加载的 AppConfig 实例
    """
    global _app_config
    _app_config = AppConfig.from_file(config_path)
    return _app_config


def reset_app_config() -> None:
    """
    重置配置缓存。
    
    清除单例缓存，下次调用 get_app_config() 将重新加载。
    主要用于测试场景。
    """
    global _app_config
    _app_config = None


def set_app_config(config: AppConfig) -> None:
    """
    设置自定义配置实例。
    
    用于测试时注入 mock 配置。
    
    Args:
        config: 要使用的 AppConfig 实例
    """
    global _app_config
    _app_config = config
```

### 3.6 辅助方法

```python
def get_model_config(self, name: str) -> ModelConfig | None:
    """
    根据名称获取模型配置。
    
    Args:
        name: 模型名称（在 config.yaml 中定义）
    
    Returns:
        ModelConfig 或 None（未找到时）
    
    Example:
        >>> config = get_app_config()
        >>> model_cfg = config.get_model_config("gpt-4")
        >>> if model_cfg:
        ...     print(f"Model: {model_cfg.model}")
    """
    return next((model for model in self.models if model.name == name), None)


def get_tool_config(self, name: str) -> ToolConfig | None:
    """根据名称获取工具配置"""
    return next((tool for tool in self.tools if tool.name == name), None)


def get_tool_group_config(self, name: str) -> ToolGroupConfig | None:
    """根据名称获取工具组配置"""
    return next((group for group in self.tool_groups if group.name == name), None)
```

---

## 4. 模型配置 (model_config.py)

### 4.1 ModelConfig 类定义

```python
class ModelConfig(BaseModel):
    """
    LLM 模型配置类。
    
    支持所有 LangChain 兼容的模型提供商，并通过 extra="allow"
    支持传递任意额外参数给底层模型类。
    """
    
    # 模型唯一标识符
    name: str = Field(
        ..., 
        description="Unique name for the model"
    )
    
    # 显示名称（用于 UI）
    display_name: str | None = Field(
        ..., 
        default_factory=lambda: None, 
        description="Display name for the model"
    )
    
    # 模型描述
    description: str | None = Field(
        ..., 
        default_factory=lambda: None, 
        description="Description for the model"
    )
    
    # LangChain 模型类的导入路径
    use: str = Field(
        ...,
        description="Class path of the model provider(e.g. langchain_openai.ChatOpenAI)",
    )
    
    # 模型标识符（传递给 API）
    model: str = Field(
        ..., 
        description="Model name"
    )
    
    # Pydantic 配置：允许额外字段
    model_config = ConfigDict(extra="allow")
    
    # ========== 思考模式配置 ==========
    
    # 是否支持思考模式
    supports_thinking: bool = Field(
        default_factory=lambda: False, 
        description="Whether the model supports thinking"
    )
    
    # 是否支持推理努力程度控制
    supports_reasoning_effort: bool = Field(
        default_factory=lambda: False, 
        description="Whether the model supports reasoning effort"
    )
    
    # 启用思考时的额外参数
    when_thinking_enabled: dict | None = Field(
        default_factory=lambda: None,
        description="Extra settings to be passed to the model when thinking is enabled",
    )
    
    # 是否支持视觉/图像输入
    supports_vision: bool = Field(
        default_factory=lambda: False, 
        description="Whether the model supports vision/image inputs"
    )
    
    # 思考参数的快捷方式（与 when_thinking_enabled 合并）
    thinking: dict | None = Field(
        default_factory=lambda: None,
        description=(
            "Thinking settings for the model. If provided, these settings will be "
            "passed to the model when thinking is enabled. This is a shortcut for "
            "`when_thinking_enabled` and will be merged with `when_thinking_enabled` "
            "if both are provided."
        ),
    )
```

### 4.2 思考模式配置示例

```yaml
# 支持思考的模型配置示例（DeepSeek）
models:
  - name: deepseek-v3
    display_name: DeepSeek V3
    description: DeepSeek V3 with thinking support
    use: langchain_deepseek:ChatDeepSeek
    model: deepseek-chat
    api_key: $DEEPSEEK_API_KEY
    supports_thinking: true
    supports_reasoning_effort: true
    when_thinking_enabled:
      extra_body:
        thinking:
          type: enabled

# OpenAI 兼容网关配置示例（Novita）
models:
  - name: novita-deepseek-v3.2
    display_name: Novita DeepSeek V3.2
    use: langchain_openai:ChatOpenAI
    model: deepseek/deepseek-v3.2
    api_key: $NOVITA_API_KEY
    base_url: https://api.novita.ai/openai
    supports_thinking: true
    when_thinking_enabled:
      extra_body:
        thinking:
          type: enabled

# 支持视觉的模型配置示例
models:
  - name: gpt-4-vision
    display_name: GPT-4 Vision
    use: langchain_openai:ChatOpenAI
    model: gpt-4-vision-preview
    api_key: $OPENAI_API_KEY
    supports_vision: true
```

---

## 5. 扩展配置 (extensions_config.py)

### 5.1 ExtensionsConfig 类定义

```python
class McpOAuthConfig(BaseModel):
    """MCP 服务器的 OAuth 配置（用于 HTTP/SSE 传输）"""
    
    enabled: bool = Field(
        default=True, 
        description="Whether OAuth token injection is enabled"
    )
    token_url: str = Field(description="OAuth token endpoint URL")
    grant_type: Literal["client_credentials", "refresh_token"] = Field(
        default="client_credentials",
        description="OAuth grant type",
    )
    client_id: str | None = Field(default=None, description="OAuth client ID")
    client_secret: str | None = Field(default=None, description="OAuth client secret")
    refresh_token: str | None = Field(
        default=None, 
        description="OAuth refresh token (for refresh_token grant)"
    )
    scope: str | None = Field(default=None, description="OAuth scope")
    audience: str | None = Field(default=None, description="OAuth audience (provider-specific)")
    token_field: str = Field(
        default="access_token", 
        description="Field name containing access token in token response"
    )
    token_type_field: str = Field(
        default="token_type", 
        description="Field name containing token type in token response"
    )
    expires_in_field: str = Field(
        default="expires_in", 
        description="Field name containing expiry (seconds) in token response"
    )
    default_token_type: str = Field(
        default="Bearer", 
        description="Default token type when missing in token response"
    )
    refresh_skew_seconds: int = Field(
        default=60, 
        description="Refresh token this many seconds before expiry"
    )
    extra_token_params: dict[str, str] = Field(
        default_factory=dict, 
        description="Additional form params sent to token endpoint"
    )


class McpServerConfig(BaseModel):
    """单个 MCP 服务器的配置"""
    
    enabled: bool = Field(
        default=True, 
        description="Whether this MCP server is enabled"
    )
    
    # 传输类型
    type: str = Field(
        default="stdio", 
        description="Transport type: 'stdio', 'sse', or 'http'"
    )
    
    # stdio 类型参数
    command: str | None = Field(
        default=None, 
        description="Command to execute to start the MCP server (for stdio type)"
    )
    args: list[str] = Field(
        default_factory=list, 
        description="Arguments to pass to the command (for stdio type)"
    )
    env: dict[str, str] = Field(
        default_factory=dict, 
        description="Environment variables for the MCP server"
    )
    
    # sse/http 类型参数
    url: str | None = Field(
        default=None, 
        description="URL of the MCP server (for sse or http type)"
    )
    headers: dict[str, str] = Field(
        default_factory=dict, 
        description="HTTP headers to send (for sse or http type)"
    )
    oauth: McpOAuthConfig | None = Field(
        default=None, 
        description="OAuth configuration (for sse or http type)"
    )
    
    # 描述
    description: str = Field(
        default="", 
        description="Human-readable description of what this MCP server provides"
    )


class SkillStateConfig(BaseModel):
    """单个技能的状态配置"""
    
    enabled: bool = Field(
        default=True, 
        description="Whether this skill is enabled"
    )


class ExtensionsConfig(BaseModel):
    """统一的扩展配置（MCP 服务器和技能状态）"""
    
    # MCP 服务器映射（名称 -> 配置）
    mcp_servers: dict[str, McpServerConfig] = Field(
        default_factory=dict,
        description="Map of MCP server name to configuration",
        alias="mcpServers",  # JSON 文件中的键名
    )
    
    # 技能状态映射（名称 -> 状态）
    skills: dict[str, SkillStateConfig] = Field(
        default_factory=dict,
        description="Map of skill name to state configuration",
    )
    
    model_config = ConfigDict(extra="allow", populate_by_name=True)
```

### 5.2 配置文件路径解析

```python
@classmethod
def resolve_config_path(cls, config_path: str | None = None) -> Path | None:
    """
    解析扩展配置文件路径的优先级顺序：
    
    1. 函数参数指定的路径（config_path）
    2. 环境变量 DEER_FLOW_EXTENSIONS_CONFIG_PATH
    3. 当前目录下的 extensions_config.json
    4. 父目录下的 extensions_config.json
    5. 当前目录下的 mcp_config.json（向后兼容）
    6. 父目录下的 mcp_config.json（向后兼容）
    7. 返回 None（扩展配置是可选的）
    
    注意：与主配置不同，扩展配置文件不存在时返回 None，
    而不是抛出异常。这是因为扩展配置是可选的。
    """
    if config_path:
        path = Path(config_path)
        if not path.exists():
            raise FileNotFoundError(
                f"Extensions config file specified by param `config_path` not found at {path}"
            )
        return path
    
    elif os.getenv("DEER_FLOW_EXTENSIONS_CONFIG_PATH"):
        path = Path(os.getenv("DEER_FLOW_EXTENSIONS_CONFIG_PATH"))
        if not path.exists():
            raise FileNotFoundError(
                f"Extensions config file specified by environment variable "
                f"`DEER_FLOW_EXTENSIONS_CONFIG_PATH` not found at {path}"
            )
        return path
    
    else:
        # 检查 extensions_config.json
        path = Path(os.getcwd()) / "extensions_config.json"
        if path.exists():
            return path
        
        path = Path(os.getcwd()).parent / "extensions_config.json"
        if path.exists():
            return path
        
        # 向后兼容：检查 mcp_config.json
        path = Path(os.getcwd()) / "mcp_config.json"
        if path.exists():
            return path
        
        path = Path(os.getcwd()).parent / "mcp_config.json"
        if path.exists():
            return path
        
        # 扩展配置是可选的
        return None
```

### 5.3 辅助方法

```python
def get_enabled_mcp_servers(self) -> dict[str, McpServerConfig]:
    """
    获取所有已启用的 MCP 服务器。
    
    Returns:
        仅包含 enabled=True 的服务器字典
    """
    return {
        name: config 
        for name, config in self.mcp_servers.items() 
        if config.enabled
    }


def is_skill_enabled(self, skill_name: str, skill_category: str) -> bool:
    """
    检查技能是否启用。
    
    默认行为：
    - public 和 custom 类别的技能默认启用
    - 其他类别默认禁用
    
    Args:
        skill_name: 技能名称
        skill_category: 技能类别（public, custom 等）
    
    Returns:
        True 如果启用，False 否则
    """
    skill_config = self.skills.get(skill_name)
    if skill_config is None:
        # 默认启用 public & custom 技能
        return skill_category in ("public", "custom")
    return skill_config.enabled
```

---

## 6. 其他配置类

### 6.1 SandboxConfig - 沙箱配置

```python
class VolumeMountConfig(BaseModel):
    """卷挂载配置"""
    
    host_path: str = Field(..., description="Path on the host machine")
    container_path: str = Field(..., description="Path inside the container")
    read_only: bool = Field(default=False, description="Whether the mount is read-only")


class SandboxConfig(BaseModel):
    """
    沙箱执行环境配置。
    
    支持多种执行模式：
    - LocalSandboxProvider：本地直接执行（开发推荐）
    - AioSandboxProvider：Docker 容器隔离（生产推荐）
    """
    
    # 沙箱提供者类的导入路径
    use: str = Field(
        ...,
        description="Class path of the sandbox provider (e.g. src.sandbox.local:LocalSandboxProvider)",
    )
    
    # Docker 镜像
    image: str | None = Field(
        default=None,
        description="Docker image to use for the sandbox container",
    )
    
    # 基础端口
    port: int | None = Field(
        default=None,
        description="Base port for sandbox containers",
    )
    
    # 最大副本数（并发沙箱数）
    replicas: int | None = Field(
        default=None,
        description="Maximum number of concurrent sandbox containers (default: 3). "
                    "When the limit is reached the least-recently-used sandbox is evicted.",
    )
    
    # 容器名称前缀
    container_prefix: str | None = Field(
        default=None,
        description="Prefix for container names",
    )
    
    # 空闲超时（秒）
    idle_timeout: int | None = Field(
        default=None,
        description="Idle timeout in seconds before sandbox is released (default: 600 = 10 minutes). "
                    "Set to 0 to disable.",
    )
    
    # 卷挂载列表
    mounts: list[VolumeMountConfig] = Field(
        default_factory=list,
        description="List of volume mounts to share directories between host and container",
    )
    
    # 环境变量
    environment: dict[str, str] = Field(
        default_factory=dict,
        description="Environment variables to inject into the sandbox container. "
                    "Values starting with $ will be resolved from host environment variables.",
    )
```

**配置示例**：

```yaml
# 本地沙箱（简单设置，适合开发）
sandbox:
  use: src.sandbox.local:LocalSandboxProvider

# Docker 沙箱（隔离安全，适合生产）
sandbox:
  use: src.community.aio_sandbox:AioSandboxProvider
  image: enterprise-public-cn-beijing.cr.volces.com/vefaas-public/all-in-one-sandbox:latest
  port: 8080
  replicas: 3
  container_prefix: deer-flow-sandbox
  idle_timeout: 600
  mounts:
    - host_path: /path/on/host
      container_path: /path/in/container
      read_only: false
  environment:
    MY_API_KEY: $MY_API_KEY  # 从主机环境变量解析

# Kubernetes 沙箱（通过 provisioner）
sandbox:
  use: src.community.aio_sandbox:AioSandboxProvider
  provisioner_url: http://provisioner:8002
```

### 6.2 MemoryConfig - 记忆配置

```python
class MemoryConfig(BaseModel):
    """
    记忆系统配置。
    
    记忆系统用于存储用户上下文、偏好和历史信息，
    并在后续对话中自动注入。
    """
    
    enabled: bool = Field(
        default=True,
        description="Whether to enable memory mechanism",
    )
    
    storage_path: str = Field(
        default="",
        description=(
            "Path to store memory data. "
            "If empty, defaults to `{base_dir}/memory.json`. "
            "Absolute paths are used as-is. "
            "Relative paths are resolved against `Paths.base_dir`."
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

**配置示例**：

```yaml
memory:
  enabled: true
  storage_path: ""  # 默认: {base_dir}/memory.json
  debounce_seconds: 30
  model_name: gpt-4o-mini  # 使用轻量级模型更新记忆
  max_facts: 100
  fact_confidence_threshold: 0.7
  injection_enabled: true
  max_injection_tokens: 2000
```

### 6.3 SubagentsConfig - 子代理配置

```python
class SubagentOverrideConfig(BaseModel):
    """单个子代理的配置覆盖"""
    
    timeout_seconds: int | None = Field(
        default=None,
        ge=1,
        description="Timeout in seconds for this subagent (None = use global default)",
    )


class SubagentsAppConfig(BaseModel):
    """
    子代理系统配置。
    
    支持全局默认超时和每个子代理的单独覆盖。
    """
    
    timeout_seconds: int = Field(
        default=900,  # 15 分钟
        ge=1,
        description="Default timeout in seconds for all subagents (default: 900 = 15 minutes)",
    )
    
    agents: dict[str, SubagentOverrideConfig] = Field(
        default_factory=dict,
        description="Per-agent configuration overrides keyed by agent name",
    )
    
    def get_timeout_for(self, agent_name: str) -> int:
        """
        获取特定子代理的有效超时时间。
        
        优先级：代理级覆盖 > 全局默认
        
        Args:
            agent_name: 子代理名称
        
        Returns:
            超时时间（秒）
        """
        override = self.agents.get(agent_name)
        if override is not None and override.timeout_seconds is not None:
            return override.timeout_seconds
        return self.timeout_seconds
```

**配置示例**：

```yaml
subagents:
  timeout_seconds: 900  # 全局默认：15 分钟
  agents:
    general-purpose:
      timeout_seconds: 1800  # 通用代理：30 分钟
    bash:
      timeout_seconds: 120   # Bash 代理：2 分钟
```

### 6.4 TracingConfig - 追踪配置

```python
class TracingConfig(BaseModel):
    """
    LangSmith 追踪配置。
    
    所有配置都从环境变量读取，支持 LANGSMITH_* 和旧版 LANGCHAIN_* 变量。
    LANGSMITH_* 变量优先级更高。
    """
    
    enabled: bool = Field(...)
    api_key: str | None = Field(...)
    project: str = Field(...)
    endpoint: str = Field(...)
    
    @property
    def is_configured(self) -> bool:
        """检查追踪是否完全配置（启用且有 API key）"""
        return self.enabled and bool(self.api_key)


# 环境变量优先级：
# enabled  : LANGSMITH_TRACING > LANGCHAIN_TRACING_V2 > LANGCHAIN_TRACING
# api_key  : LANGSMITH_API_KEY  > LANGCHAIN_API_KEY
# project  : LANGSMITH_PROJECT  > LANGCHAIN_PROJECT   (默认: "deer-flow")
# endpoint : LANGSMITH_ENDPOINT > LANGCHAIN_ENDPOINT  (默认: https://api.smith.langchain.com)
```

**环境变量示例**：

```bash
# LangSmith 配置
export LANGSMITH_TRACING=true
export LANGSMITH_API_KEY=lsv2_pt_xxxxx
export LANGSMITH_PROJECT=deer-flow

# 或使用旧版变量名
export LANGCHAIN_TRACING_V2=true
export LANGCHAIN_API_KEY=lsv2_pt_xxxxx
```

### 6.5 CheckpointerConfig - 检查点配置

```python
CheckpointerType = Literal["memory", "sqlite", "postgres"]


class CheckpointerConfig(BaseModel):
    """
    LangGraph 状态持久化检查点配置。
    
    支持三种后端：
    - memory：进程内存（重启丢失）
    - sqlite：本地文件（需要 langgraph-checkpoint-sqlite）
    - postgres：PostgreSQL（需要 langgraph-checkpoint-postgres）
    """
    
    type: CheckpointerType = Field(
        description="Checkpointer backend type. "
        "'memory' is in-process only (lost on restart). "
        "'sqlite' persists to a local file. "
        "'postgres' persists to PostgreSQL."
    )
    
    connection_string: str | None = Field(
        default=None,
        description="Connection string for sqlite (file path) or postgres (DSN). "
        "Required for sqlite and postgres types.",
    )
```

**配置示例**：

```yaml
# 内存检查点（默认，适合测试）
checkpointer:
  type: memory

# SQLite 检查点（适合单机部署）
checkpointer:
  type: sqlite
  connection_string: .deer-flow/checkpoints.db

# PostgreSQL 检查点（适合生产集群）
checkpointer:
  type: postgres
  connection_string: postgresql://user:pass@localhost:5432/deerflow
```

### 6.6 SummarizationConfig - 摘要配置

```python
ContextSizeType = Literal["fraction", "tokens", "messages"]


class ContextSize(BaseModel):
    """上下文大小规格"""
    
    type: ContextSizeType = Field(description="Type of context size specification")
    value: int | float = Field(description="Value for the context size specification")
    
    def to_tuple(self) -> tuple[ContextSizeType, int | float]:
        """转换为 SummarizationMiddleware 期望的元组格式"""
        return (self.type, self.value)


class SummarizationConfig(BaseModel):
    """
    自动对话摘要配置。
    
    当上下文窗口达到阈值时，自动摘要旧对话以节省 token。
    """
    
    enabled: bool = Field(
        default=False,
        description="Whether to enable automatic conversation summarization",
    )
    
    model_name: str | None = Field(
        default=None,
        description="Model name to use for summarization (None = use a lightweight model)",
    )
    
    trigger: ContextSize | list[ContextSize] | None = Field(
        default=None,
        description="One or more thresholds that trigger summarization",
    )
    
    keep: ContextSize = Field(
        default_factory=lambda: ContextSize(type="messages", value=20),
        description="Context retention policy after summarization",
    )
    
    trim_tokens_to_summarize: int | None = Field(
        default=4000,
        description="Maximum tokens to keep when preparing messages for summarization",
    )
    
    summary_prompt: str | None = Field(
        default=None,
        description="Custom prompt template for generating summaries",
    )
```

**配置示例**：

```yaml
summarization:
  enabled: true
  model_name: gpt-4o-mini
  trigger:
    - type: messages
      value: 50        # 50 条消息时触发
    - type: tokens
      value: 4000      # 或 4000 tokens 时触发
  keep:
    type: messages
    value: 20          # 保留最近 20 条消息
  trim_tokens_to_summarize: 4000
```

### 6.7 TitleConfig - 标题配置

```python
class TitleConfig(BaseModel):
    """
    自动线程标题生成配置。
    
    在首次对话后自动生成简洁的标题。
    """
    
    enabled: bool = Field(
        default=True,
        description="Whether to enable automatic title generation",
    )
    
    max_words: int = Field(
        default=6,
        ge=1,
        le=20,
        description="Maximum number of words in the generated title",
    )
    
    max_chars: int = Field(
        default=60,
        ge=10,
        le=200,
        description="Maximum number of characters in the generated title",
    )
    
    model_name: str | None = Field(
        default=None,
        description="Model name to use for title generation (None = use default model)",
    )
    
    prompt_template: str = Field(
        default=(
            "Generate a concise title (max {max_words} words) for this conversation.\n"
            "User: {user_msg}\n"
            "Assistant: {assistant_msg}\n\n"
            "Return ONLY the title, no quotes, no explanation."
        ),
        description="Prompt template for title generation",
    )
```

**配置示例**：

```yaml
title:
  enabled: true
  max_words: 6
  max_chars: 60
  model_name: gpt-4o-mini  # 使用轻量级模型生成标题
```

---

## 7. 路径管理 (paths.py)

### 7.1 Paths 类定义

```python
# 沙箱内虚拟路径前缀
VIRTUAL_PATH_PREFIX = "/mnt/user-data"


class Paths:
    """
    DeerFlow 应用数据路径的集中管理类。
    
    目录布局（主机侧）：
        {base_dir}/
        ├── memory.json              # 全局记忆文件
        ├── USER.md                  # 全局用户配置文件
        ├── agents/
        │   └── {agent_name}/
        │       ├── config.yaml      # Agent 配置
        │       ├── SOUL.md          # Agent 个性文件
        │       └── memory.json      # Agent 记忆文件
        └── threads/
            └── {thread_id}/
                └── user-data/       # 沙箱挂载目录
                    ├── workspace/   # 工作区
                    ├── uploads/     # 上传文件
                    └── outputs/     # 输出文件
    
    BaseDir 解析优先级：
        1. 构造函数参数 base_dir
        2. 环境变量 DEER_FLOW_HOME
        3. 本地开发：cwd/.deer-flow（当 cwd 是 backend/ 时）
        4. 默认：$HOME/.deer-flow
    """
    
    def __init__(self, base_dir: str | Path | None = None) -> None:
        self._base_dir = Path(base_dir).resolve() if base_dir is not None else None
    
    @property
    def base_dir(self) -> Path:
        """根目录（所有应用数据的父目录）"""
        if self._base_dir is not None:
            return self._base_dir
        
        if env_home := os.getenv("DEER_FLOW_HOME"):
            return Path(env_home).resolve()
        
        cwd = Path.cwd()
        if cwd.name == "backend" or (cwd / "pyproject.toml").exists():
            return cwd / ".deer-flow"
        
        return Path.home() / ".deer-flow"
    
    @property
    def host_base_dir(self) -> Path:
        """
        主机可见的基础目录（用于 Docker 卷挂载）。
        
        当在 Docker 中运行并挂载 Docker socket 时（DooD），
        Docker 守护进程在主机上运行，需要使用主机侧路径。
        通过 DEER_FLOW_HOST_BASE_DIR 环境变量设置。
        """
        if env := os.getenv("DEER_FLOW_HOST_BASE_DIR"):
            return Path(env)
        return self.base_dir
    
    @property
    def memory_file(self) -> Path:
        """记忆文件路径：{base_dir}/memory.json"""
        return self.base_dir / "memory.json"
    
    @property
    def user_md_file(self) -> Path:
        """全局用户配置文件路径：{base_dir}/USER.md"""
        return self.base_dir / "USER.md"
    
    @property
    def agents_dir(self) -> Path:
        """自定义 Agent 根目录：{base_dir}/agents/"""
        return self.base_dir / "agents"
    
    def agent_dir(self, name: str) -> Path:
        """特定 Agent 目录：{base_dir}/agents/{name}/"""
        return self.agents_dir / name.lower()
    
    def agent_memory_file(self, name: str) -> Path:
        """Agent 记忆文件：{base_dir}/agents/{name}/memory.json"""
        return self.agent_dir(name) / "memory.json"
    
    def thread_dir(self, thread_id: str) -> Path:
        """
        线程数据目录：{base_dir}/threads/{thread_id}/
        
        包含沙箱挂载的 user-data/ 子目录。
        验证 thread_id 只包含安全字符以防止目录遍历攻击。
        """
        if not _SAFE_THREAD_ID_RE.match(thread_id):
            raise ValueError(
                f"Invalid thread_id {thread_id!r}: only alphanumeric characters, "
                "hyphens, and underscores are allowed."
            )
        return self.base_dir / "threads" / thread_id
    
    def sandbox_work_dir(self, thread_id: str) -> Path:
        """沙箱工作区目录（主机路径）"""
        return self.thread_dir(thread_id) / "user-data" / "workspace"
    
    def sandbox_uploads_dir(self, thread_id: str) -> Path:
        """沙箱上传目录（主机路径）"""
        return self.thread_dir(thread_id) / "user-data" / "uploads"
    
    def sandbox_outputs_dir(self, thread_id: str) -> Path:
        """沙箱输出目录（主机路径）"""
        return self.thread_dir(thread_id) / "user-data" / "outputs"
    
    def sandbox_user_data_dir(self, thread_id: str) -> Path:
        """沙箱用户数据根目录（主机路径）"""
        return self.thread_dir(thread_id) / "user-data"
    
    def ensure_thread_dirs(self, thread_id: str) -> None:
        """
        创建线程的所有标准沙箱目录。
        
        目录以 0o777 权限创建，允许沙箱容器（可能以不同 UID 运行）
        写入卷挂载路径。
        """
        for d in [
            self.sandbox_work_dir(thread_id),
            self.sandbox_uploads_dir(thread_id),
            self.sandbox_outputs_dir(thread_id),
        ]:
            d.mkdir(parents=True, exist_ok=True)
            d.chmod(0o777)
    
    def resolve_virtual_path(self, thread_id: str, virtual_path: str) -> Path:
        """
        将沙箱虚拟路径解析为主机文件系统路径。
        
        Args:
            thread_id: 线程 ID
            virtual_path: 沙箱内路径（如 /mnt/user-data/outputs/report.pdf）
        
        Returns:
            解析后的主机文件系统绝对路径
        
        Raises:
            ValueError: 路径不以预期前缀开头或检测到路径遍历
        """
        stripped = virtual_path.lstrip("/")
        prefix = VIRTUAL_PATH_PREFIX.lstrip("/")
        
        if stripped != prefix and not stripped.startswith(prefix + "/"):
            raise ValueError(f"Path must start with /{prefix}")
        
        relative = stripped[len(prefix) :].lstrip("/")
        base = self.sandbox_user_data_dir(thread_id).resolve()
        actual = (base / relative).resolve()
        
        try:
            actual.relative_to(base)
        except ValueError:
            raise ValueError("Access denied: path traversal detected")
        
        return actual


# 单例访问函数
def get_paths() -> Paths:
    """返回全局 Paths 单例（延迟初始化）"""
    global _paths
    if _paths is None:
        _paths = Paths()
    return _paths


def resolve_path(path: str) -> Path:
    """
    解析路径为绝对路径。
    
    相对路径相对于应用基础目录解析。
    绝对路径原样返回（规范化后）。
    """
    p = Path(path)
    if not p.is_absolute():
        p = get_paths().base_dir / path
    return p.resolve()
```

---

## 8. 工具和技能配置

### 8.1 ToolConfig - 工具配置

```python
class ToolGroupConfig(BaseModel):
    """工具组配置"""
    
    name: str = Field(..., description="Unique name for the tool group")
    model_config = ConfigDict(extra="allow")


class ToolConfig(BaseModel):
    """工具配置"""
    
    name: str = Field(..., description="Unique name for the tool")
    group: str = Field(..., description="Group name for the tool")
    use: str = Field(
        ...,
        description="Variable name of the tool provider (e.g. src.sandbox.tools:bash_tool)",
    )
    model_config = ConfigDict(extra="allow")
```

**配置示例**：

```yaml
# 工具组定义
tool_groups:
  - name: web           # Web 浏览和搜索
  - name: file:read     # 只读文件操作
  - name: file:write    # 写文件操作
  - name: bash          # Shell 命令执行

# 工具定义
tools:
  - name: web_search
    group: web
    use: src.community.tavily.tools:web_search_tool
    max_results: 5
    # api_key: $TAVILY_API_KEY
```

### 8.2 SkillsConfig - 技能配置

```python
class SkillsConfig(BaseModel):
    """技能系统配置"""
    
    path: str | None = Field(
        default=None,
        description="Path to skills directory. If not specified, defaults to ../skills",
    )
    
    container_path: str = Field(
        default="/mnt/skills",
        description="Path where skills are mounted in the sandbox container",
    )
    
    def get_skills_path(self) -> Path:
        """获取解析后的技能目录路径"""
        if self.path:
            path = Path(self.path)
            if not path.is_absolute():
                path = Path.cwd() / path
            return path.resolve()
        else:
            from src.skills.loader import get_skills_root_path
            return get_skills_root_path()
    
    def get_skill_container_path(self, skill_name: str, category: str = "public") -> str:
        """获取技能在沙箱容器中的完整路径"""
        return f"{self.container_path}/{category}/{skill_name}"
```

---

## 9. 自定义 Agent 配置 (agents_config.py)

### 9.1 AgentConfig 类和加载器

```python
SOUL_FILENAME = "SOUL.md"
AGENT_NAME_PATTERN = re.compile(r"^[A-Za-z0-9-]+$")


class AgentConfig(BaseModel):
    """自定义 Agent 配置"""
    
    name: str
    description: str = ""
    model: str | None = None           # Agent 专用模型
    tool_groups: list[str] | None = None  # 工具组白名单


def load_agent_config(name: str | None) -> AgentConfig | None:
    """
    从目录加载自定义 Agent 配置。
    
    目录结构：
        {base_dir}/agents/{name}/
        ├── config.yaml    # Agent 配置
        └── SOUL.md        # Agent 个性文件
    
    Args:
        name: Agent 名称
    
    Returns:
        AgentConfig 实例或 None
    
    Raises:
        FileNotFoundError: 目录或配置文件不存在
        ValueError: 配置解析失败
    """
    if name is None:
        return None
    
    if not AGENT_NAME_PATTERN.match(name):
        raise ValueError(
            f"Invalid agent name '{name}'. Must match pattern: {AGENT_NAME_PATTERN.pattern}"
        )
    
    agent_dir = get_paths().agent_dir(name)
    config_file = agent_dir / "config.yaml"
    
    if not agent_dir.exists():
        raise FileNotFoundError(f"Agent directory not found: {agent_dir}")
    
    if not config_file.exists():
        raise FileNotFoundError(f"Agent config not found: {config_file}")
    
    with open(config_file, encoding="utf-8") as f:
        data: dict[str, Any] = yaml.safe_load(f) or {}
    
    # 从目录名设置名称（如果配置文件中未指定）
    if "name" not in data:
        data["name"] = name
    
    # 过滤未知字段
    known_fields = set(AgentConfig.model_fields.keys())
    data = {k: v for k, v in data.items() if k in known_fields}
    
    return AgentConfig(**data)


def load_agent_soul(agent_name: str | None) -> str | None:
    """
    加载 Agent 的 SOUL.md 文件。
    
    SOUL.md 定义 Agent 的个性、价值观和行为准则，
    注入到系统提示词中作为额外上下文。
    """
    agent_dir = get_paths().agent_dir(agent_name) if agent_name else get_paths().base_dir
    soul_path = agent_dir / SOUL_FILENAME
    if not soul_path.exists():
        return None
    content = soul_path.read_text(encoding="utf-8").strip()
    return content or None


def list_custom_agents() -> list[AgentConfig]:
    """
    扫描并列出所有有效的自定义 Agent。
    
    Returns:
        AgentConfig 列表
    """
    agents_dir = get_paths().agents_dir
    
    if not agents_dir.exists():
        return []
    
    agents: list[AgentConfig] = []
    
    for entry in sorted(agents_dir.iterdir()):
        if not entry.is_dir():
            continue
        
        config_file = entry / "config.yaml"
        if not config_file.exists():
            logger.debug(f"Skipping {entry.name}: no config.yaml")
            continue
        
        try:
            agent_cfg = load_agent_config(entry.name)
            agents.append(agent_cfg)
        except Exception as e:
            logger.warning(f"Skipping agent '{entry.name}': {e}")
    
    return agents
```

---

## 10. 配置格式完整示例

### 10.1 config.yaml 完整示例

```yaml
# ============================================================
# DeerFlow 主配置文件
# ============================================================

# ==================== 模型配置 ====================
models:
  # OpenAI GPT-4
  - name: gpt-4
    display_name: GPT-4
    description: OpenAI GPT-4 model
    use: langchain_openai:ChatOpenAI
    model: gpt-4
    api_key: $OPENAI_API_KEY
    max_tokens: 4096
    temperature: 0.7

  # OpenAI GPT-4 Vision
  - name: gpt-4-vision
    display_name: GPT-4 Vision
    use: langchain_openai:ChatOpenAI
    model: gpt-4-vision-preview
    api_key: $OPENAI_API_KEY
    supports_vision: true

  # DeepSeek V3（支持思考模式）
  - name: deepseek-v3
    display_name: DeepSeek V3
    use: langchain_deepseek:ChatDeepSeek
    model: deepseek-chat
    api_key: $DEEPSEEK_API_KEY
    supports_thinking: true
    supports_reasoning_effort: true
    when_thinking_enabled:
      extra_body:
        thinking:
          type: enabled

  # OpenAI 兼容网关（Novita）
  - name: novita-deepseek-v3.2
    display_name: Novita DeepSeek V3.2
    use: langchain_openai:ChatOpenAI
    model: deepseek/deepseek-v3.2
    api_key: $NOVITA_API_KEY
    base_url: https://api.novita.ai/openai
    supports_thinking: true
    when_thinking_enabled:
      extra_body:
        thinking:
          type: enabled

# ==================== 沙箱配置 ====================
# 本地沙箱（开发推荐）
sandbox:
  use: src.sandbox.local:LocalSandboxProvider

# Docker 沙箱（生产推荐）
# sandbox:
#   use: src.community.aio_sandbox:AioSandboxProvider
#   image: enterprise-public-cn-beijing.cr.volces.com/vefaas-public/all-in-one-sandbox:latest
#   port: 8080
#   replicas: 3
#   container_prefix: deer-flow-sandbox
#   idle_timeout: 600
#   mounts:
#     - host_path: /path/on/host
#       container_path: /path/in/container
#       read_only: false
#   environment:
#     MY_API_KEY: $MY_API_KEY

# ==================== 工具配置 ====================
tool_groups:
  - name: web
  - name: file:read
  - name: file:write
  - name: bash

tools:
  - name: web_search
    group: web
    use: src.community.tavily.tools:web_search_tool
    max_results: 5
    # api_key: $TAVILY_API_KEY

  - name: web_fetch
    group: web
    use: src.community.jina.tools:web_fetch_tool

# ==================== 技能配置 ====================
skills:
  # 主机路径（可选，默认：../skills）
  # path: /custom/path/to/skills
  # 容器挂载路径（默认：/mnt/skills）
  container_path: /mnt/skills

# ==================== 记忆配置 ====================
memory:
  enabled: true
  storage_path: ""  # 默认: {base_dir}/memory.json
  debounce_seconds: 30
  model_name: gpt-4o-mini
  max_facts: 100
  fact_confidence_threshold: 0.7
  injection_enabled: true
  max_injection_tokens: 2000

# ==================== 子代理配置 ====================
subagents:
  timeout_seconds: 900  # 全局默认：15 分钟
  agents:
    general-purpose:
      timeout_seconds: 1800  # 30 分钟
    bash:
      timeout_seconds: 120  # 2 分钟

# ==================== 检查点配置 ====================
# 内存检查点（默认）
# checkpointer:
#   type: memory

# SQLite 检查点
# checkpointer:
#   type: sqlite
#   connection_string: .deer-flow/checkpoints.db

# PostgreSQL 检查点
# checkpointer:
#   type: postgres
#   connection_string: postgresql://user:pass@localhost:5432/deerflow

# ==================== 摘要配置 ====================
summarization:
  enabled: true
  model_name: gpt-4o-mini
  trigger:
    - type: messages
      value: 50
    - type: tokens
      value: 4000
  keep:
    type: messages
    value: 20
  trim_tokens_to_summarize: 4000

# ==================== 标题配置 ====================
title:
  enabled: true
  max_words: 6
  max_chars: 60
  model_name: gpt-4o-mini
```

### 10.2 extensions_config.json 完整示例

```json
{
  "mcpServers": {
    "filesystem": {
      "enabled": true,
      "type": "stdio",
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-filesystem", "/path/to/allowed/dir"],
      "env": {
        "NODE_OPTIONS": "--max-old-space-size=4096"
      },
      "description": "Filesystem access MCP server"
    },
    
    "brave-search": {
      "enabled": true,
      "type": "stdio",
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-brave-search"],
      "env": {
        "BRAVE_API_KEY": "$BRAVE_API_KEY"
      },
      "description": "Brave Search MCP server"
    },
    
    "remote-server": {
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
        "client_id": "my-client-id",
        "client_secret": "$OAUTH_CLIENT_SECRET",
        "scope": "read write",
        "refresh_skew_seconds": 60
      },
      "description": "Remote MCP server with OAuth"
    }
  },
  
  "skills": {
    "web-search": {
      "enabled": true
    },
    "code-gen": {
      "enabled": false
    },
    "data-analysis": {
      "enabled": true
    }
  }
}
```

---

## 11. 环境变量汇总

| 变量名 | 用途 | 配置位置 |
|--------|------|----------|
| `OPENAI_API_KEY` | OpenAI API 密钥 | config.yaml: `$OPENAI_API_KEY` |
| `ANTHROPIC_API_KEY` | Anthropic API 密钥 | config.yaml: `$ANTHROPIC_API_KEY` |
| `DEEPSEEK_API_KEY` | DeepSeek API 密钥 | config.yaml: `$DEEPSEEK_API_KEY` |
| `NOVITA_API_KEY` | Novita API 密钥 | config.yaml: `$NOVITA_API_KEY` |
| `TAVILY_API_KEY` | Tavily 搜索 API 密钥 | config.yaml: `$TAVILY_API_KEY` |
| `DEER_FLOW_CONFIG_PATH` | 主配置文件路径 | AppConfig.resolve_config_path() |
| `DEER_FLOW_EXTENSIONS_CONFIG_PATH` | 扩展配置文件路径 | ExtensionsConfig.resolve_config_path() |
| `DEER_FLOW_HOME` | 应用数据根目录 | Paths.base_dir |
| `DEER_FLOW_HOST_BASE_DIR` | 主机侧数据目录（Docker 用） | Paths.host_base_dir |
| `LANGSMITH_TRACING` | 启用 LangSmith 追踪 | TracingConfig |
| `LANGSMITH_API_KEY` | LangSmith API 密钥 | TracingConfig |
| `LANGSMITH_PROJECT` | LangSmith 项目名 | TracingConfig |
| `LANGSMITH_ENDPOINT` | LangSmith 端点 | TracingConfig |
| `LANGCHAIN_TRACING_V2` | 旧版追踪启用 | TracingConfig（向后兼容） |
| `LANGCHAIN_API_KEY` | 旧版 API 密钥 | TracingConfig（向后兼容） |

---

## 12. 最佳实践

### 12.1 配置文件位置

```bash
# 推荐的项目结构
deer-flow/
├── config.yaml           # 主配置（项目根目录）
├── extensions_config.json # 扩展配置（项目根目录）
├── .env                  # 环境变量（不提交）
├── .env.example          # 环境变量模板（提交）
├── backend/
│   └── ...
└── frontend/
    └── ...
```

### 12.2 敏感信息处理

```yaml
# 错误：硬编码 API 密钥
models:
  - api_key: sk-xxxxxxxxxxxx  # 不要这样做！

# 正确：使用环境变量引用
models:
  - api_key: $OPENAI_API_KEY  # 从 .env 读取
```

### 12.3 配置热更新

```python
# 在运行时重新加载配置
from src.config import reload_app_config, reload_extensions_config

# 重新加载主配置
new_config = reload_app_config()

# 重新加载扩展配置
new_extensions = reload_extensions_config()
```

### 12.4 测试中的配置管理

```python
import pytest
from src.config import set_app_config, reset_app_config, AppConfig

@pytest.fixture
def mock_config():
    """为测试提供 mock 配置"""
    config = AppConfig(
        models=[...],
        sandbox=SandboxConfig(use="src.sandbox.local:LocalSandboxProvider"),
    )
    set_app_config(config)
    yield config
    reset_app_config()  # 清理
```