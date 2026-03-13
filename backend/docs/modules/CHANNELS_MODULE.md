---

# DeerFlow Channels 模块技术文档

## 1. 模块概述

### 1.1 模块职责和功能定位

Channels 模块是 DeerFlow 框架中的即时通讯（IM）通道集成层，负责：

- **多平台消息接入**：支持飞书（Feishu/Lark）、Slack、Telegram 三大 IM 平台
- **消息双向转发**：将外部 IM 消息转发给 DeerFlow Agent，并将 Agent 响应回传到 IM 平台
- **线程管理**：维护 IM 会话与 LangGraph 线程的映射关系，支持多轮对话
- **文件传输**：支持在 IM 平台和 Agent 之间传递文件附件
- **命令处理**：支持斜杠命令（如 `/new`、`/help` 等）用于会话管理

### 1.2 架构设计

```
┌─────────────────────────────────────────────────────────────────────────┐
│                        Channels 模块架构                                  │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│   ┌─────────────┐   ┌─────────────┐   ┌─────────────┐                  │
│   │   Feishu    │   │   Slack     │   │  Telegram   │                  │
│   │  Channel    │   │  Channel    │   │  Channel    │                  │
│   └──────┬──────┘   └──────┬──────┘   └──────┬──────┘                  │
│          │                 │                 │                          │
│          └────────────┬────┴────────────────┘                          │
│                       │                                                 │
│                       ▼                                                 │
│              ┌─────────────────┐                                        │
│              │   MessageBus    │ ◄── 异步消息队列（解耦）               │
│              │  (Inbound/      │                                        │
│              │   Outbound)     │                                        │
│              └────────┬────────┘                                        │
│                       │                                                 │
│                       ▼                                                 │
│              ┌─────────────────┐      ┌─────────────────┐              │
│              │ ChannelManager  │─────►│  ChannelStore   │              │
│              │  (消息分发器)    │      │  (线程映射存储)  │              │
│              └────────┬────────┘      └─────────────────┘              │
│                       │                                                 │
│                       ▼                                                 │
│              ┌─────────────────┐                                        │
│              │ LangGraph Server│ ◄── runs.wait() API                   │
│              │   (Agent)       │                                        │
│              └─────────────────┘                                        │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### 1.3 与其他模块的关系

**依赖关系**：

| 模块 | 关系说明 |
|------|----------|
| `src.config.app_config` | 配置管理，读取 `config.yaml` 中的 channels 配置 |
| `src.config.paths` | 路径管理，获取存储文件位置 |
| `src.reflection` | 反射工具，动态加载通道类 |
| `langgraph-sdk` | LangGraph 客户端，与 LangGraph Server 通信 |

---

## 2. 目录结构

```
backend/src/channels/
├── __init__.py      # 模块入口，导出公共 API
├── base.py          # 通道抽象基类
├── feishu.py        # 飞书/Lark 通道实现
├── slack.py         # Slack 通道实现
├── telegram.py      # Telegram 通道实现
├── service.py       # 通道服务生命周期管理
├── manager.py       # 消息分发器（核心调度逻辑）
├── message_bus.py   # 异步消息总线
└── store.py         # 线程映射持久化存储
```

**文件职责说明**：

| 文件 | 行数 | 职责 |
|------|------|------|
| `__init__.py` | 16 | 模块公共 API 导出 |
| `base.py` | 108 | 通道抽象基类，定义生命周期和消息处理接口 |
| `feishu.py` | 378 | 飞书 WebSocket 通道实现 |
| `slack.py` | 244 | Slack Socket Mode 通道实现 |
| `telegram.py` | 282 | Telegram 长轮询通道实现 |
| `service.py` | 182 | 通道服务管理，读取配置并启动通道 |
| `manager.py` | 490 | 消息分发器，连接 MessageBus 和 LangGraph |
| `message_bus.py` | 173 | 异步消息队列，解耦通道和调度器 |
| `store.py` | 153 | JSON 文件存储，维护会话到线程的映射 |

---

## 3. 支持的通道类型

### 3.1 Feishu（飞书/Lark）

**连接方式**：WebSocket 长连接（无需公网 IP）

**特点**：
- 使用 `lark-oapi` SDK
- 支持话题回复（Thread）
- 支持表情回应（Reaction）
- 支持交互式卡片消息
- 支持文件和图片上传

**消息流程**：
```
1. 用户发送消息 → Bot 添加 "OK" 表情反应
2. Bot 回复话题："Working on it......"
3. Agent 处理消息并返回结果
4. Bot 回复话题：Agent 响应内容
5. Bot 添加 "DONE" 表情反应到原消息
```

**配置项**：

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `enabled` | bool | 是 | 是否启用通道 |
| `app_id` | string | 是 | 飞书应用 ID |
| `app_secret` | string | 是 | 飞书应用密钥 |

**配置示例**：

```yaml
channels:
  langgraph_url: "http://localhost:2024"
  gateway_url: "http://localhost:8001"
  feishu:
    enabled: true
    app_id: "cli_xxx"
    app_secret: "xxx"
    session:
      assistant_id: "lead_agent"
      config:
        recursion_limit: 100
      context:
        thinking_enabled: true
```

### 3.2 Slack

**连接方式**：Socket Mode（WebSocket，无需公网 IP）

**特点**：
- 使用 `slack-sdk` SDK
- 支持话题回复（Thread）
- 支持表情回应（Reaction）
- 支持 Markdown 到 mrkdwn 格式转换
- 支持文件上传（v2 API）
- 支持用户白名单

**消息流程**：
```
1. 用户发送消息 → Bot 添加 "eyes" 表情反应
2. Bot 回复话题：":hourglass_flowing_sand: Working on it..."
3. Agent 处理消息并返回结果
4. Bot 回复话题：Agent 响应内容
5. Bot 添加 "white_check_mark" 表情反应（成功）或 "x"（失败）
```

**配置项**：

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `enabled` | bool | 是 | 是否启用通道 |
| `bot_token` | string | 是 | Slack Bot Token（xoxb-...） |
| `app_token` | string | 是 | Slack App-Level Token（xapp-...） |
| `allowed_users` | string[] | 否 | 允许的用户 ID 列表，空=允许所有 |

**配置示例**：

```yaml
channels:
  slack:
    enabled: true
    bot_token: "xoxb-xxx"
    app_token: "xapp-xxx"
    allowed_users:
      - "U1234567890"
```

### 3.3 Telegram

**连接方式**：长轮询（Long Polling，无需公网 IP）

**特点**：
- 使用 `python-telegram-bot` SDK
- 支持消息回复引用（Reply）
- 支持图片（≤10MB）和文件（≤50MB）上传
- 支持用户白名单
- 内置命令处理（`/start`、`/help` 等）

**消息流程**：
```
1. 用户发送消息 → Bot 回复："Working on it..."
2. Agent 处理消息并返回结果
3. Bot 回复：Agent 响应内容
```

**配置项**：

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `enabled` | bool | 是 | 是否启用通道 |
| `bot_token` | string | 是 | Telegram Bot Token（从 @BotFather 获取） |
| `allowed_users` | int[] | 否 | 允许的用户 ID 列表，空=允许所有 |

**配置示例**：

```yaml
channels:
  telegram:
    enabled: true
    bot_token: "123456789:ABCdefGHIjklMNOpqrsTUVwxyz"
    allowed_users:
      - 123456789
```

---

## 4. 通道服务类

### 4.1 ChannelService

**职责**：管理所有通道的生命周期，是模块的入口点。

**核心属性**：

| 属性 | 类型 | 说明 |
|------|------|------|
| `bus` | MessageBus | 消息总线实例 |
| `store` | ChannelStore | 线程映射存储 |
| `manager` | ChannelManager | 消息分发器 |
| `_channels` | dict[str, Channel] | 已启动的通道实例 |

**核心方法**：

```python
class ChannelService:
    @classmethod
    def from_app_config(cls) -> ChannelService:
        """从应用配置创建 ChannelService 实例。"""
        
    async def start(self) -> None:
        """启动管理器和所有启用的通道。"""
        
    async def stop(self) -> None:
        """停止所有通道和管理器。"""
        
    async def restart_channel(self, name: str) -> bool:
        """重启指定通道。"""
        
    def get_status(self) -> dict[str, Any]:
        """返回所有通道的状态信息。"""
```

**单例访问**：

```python
# 启动服务
service = await start_channel_service()

# 获取服务实例
service = get_channel_service()

# 停止服务
await stop_channel_service()
```

### 4.2 ChannelManager

**职责**：核心消息分发器，连接 MessageBus 和 LangGraph Server。

**核心属性**：

| 属性 | 类型 | 说明 |
|------|------|------|
| `bus` | MessageBus | 消息总线 |
| `store` | ChannelStore | 线程映射存储 |
| `_langgraph_url` | str | LangGraph Server 地址 |
| `_gateway_url` | str | Gateway API 地址 |
| `_assistant_id` | str | 默认 Assistant ID |
| `_max_concurrency` | int | 最大并发数（默认 5） |

**核心方法**：

```python
class ChannelManager:
    async def start(self) -> None:
        """启动分发循环。"""
        
    async def stop(self) -> None:
        """停止分发循环。"""
        
    async def _dispatch_loop(self) -> None:
        """分发循环主体，从 MessageBus 消费入站消息。"""
        
    async def _handle_chat(self, msg: InboundMessage) -> None:
        """处理聊天消息，调用 LangGraph Agent。"""
        
    async def _handle_command(self, msg: InboundMessage) -> None:
        """处理斜杠命令。"""
```

**支持的命令**：

| 命令 | 功能 |
|------|------|
| `/new` | 创建新的对话线程 |
| `/status` | 显示当前线程状态 |
| `/models` | 列出可用模型 |
| `/memory` | 显示记忆状态 |
| `/help` | 显示帮助信息 |

### 4.3 MessageBus

**职责**：异步消息队列，解耦通道和调度器。

**消息类型**：

```python
class InboundMessageType(StrEnum):
    """入站消息类型。"""
    CHAT = "chat"      # 普通聊天消息
    COMMAND = "command"  # 斜杠命令
```

**数据类**：

```python
@dataclass
class InboundMessage:
    """从 IM 通道到 Agent 的消息。"""
    channel_name: str          # 通道名称（"feishu"/"slack"/"telegram"）
    chat_id: str               # 平台会话 ID
    user_id: str               # 平台用户 ID
    text: str                  # 消息文本
    msg_type: InboundMessageType  # 消息类型
    thread_ts: str | None      # 平台话题 ID（用于回复）
    topic_id: str | None       # 对话主题 ID（映射到 DeerFlow 线程）
    files: list[dict]          # 文件附件列表
    metadata: dict             # 额外元数据

@dataclass
class OutboundMessage:
    """从 Agent 到 IM 通道的消息。"""
    channel_name: str          # 目标通道名称
    chat_id: str               # 目标会话 ID
    thread_id: str             # DeerFlow 线程 ID
    text: str                  # 响应文本
    artifacts: list[str]       # 产物路径列表
    attachments: list[ResolvedAttachment]  # 解析后的文件附件
    is_final: bool             # 是否为最终响应
    thread_ts: str | None      # 平台话题 ID

@dataclass
class ResolvedAttachment:
    """解析后的文件附件。"""
    virtual_path: str          # 虚拟路径
    actual_path: Path          # 实际文件路径
    filename: str              # 文件名
    mime_type: str             # MIME 类型
    size: int                  # 文件大小（字节）
    is_image: bool             # 是否为图片
```

**核心方法**：

```python
class MessageBus:
    async def publish_inbound(self, msg: InboundMessage) -> None:
        """发布入站消息到队列。"""
        
    async def get_inbound(self) -> InboundMessage:
        """从队列获取入站消息（阻塞）。"""
        
    def subscribe_outbound(self, callback: Callable) -> None:
        """订阅出站消息。"""
        
    async def publish_outbound(self, msg: OutboundMessage) -> None:
        """发布出站消息到所有订阅者。"""
```

### 4.4 ChannelStore

**职责**：持久化 IM 会话到 DeerFlow 线程的映射关系。

**存储格式**（JSON 文件）：

```json
{
  "feishu:oc_xxx:om_xxx": {
    "thread_id": "uuid-xxx",
    "user_id": "ou_xxx",
    "created_at": 1700000000.0,
    "updated_at": 1700000000.0
  }
}
```

**键格式**：
- 基础键：`<channel_name>:<chat_id>`
- 话题键：`<channel_name>:<chat_id>:<topic_id>`

**核心方法**：

```python
class ChannelStore:
    def get_thread_id(self, channel_name: str, chat_id: str, 
                       topic_id: str | None = None) -> str | None:
        """查询 DeerFlow 线程 ID。"""
        
    def set_thread_id(self, channel_name: str, chat_id: str, 
                      thread_id: str, topic_id: str | None = None,
                      user_id: str = "") -> None:
        """设置线程映射。"""
        
    def remove(self, channel_name: str, chat_id: str, 
               topic_id: str | None = None) -> bool:
        """移除映射。"""
        
    def list_entries(self, channel_name: str | None = None) -> list[dict]:
        """列出所有映射。"""
```

---

## 5. 通道接口

### 5.1 Channel 基类

所有通道实现必须继承此抽象基类：

```python
class Channel(ABC):
    """IM 通道抽象基类。"""
    
    def __init__(self, name: str, bus: MessageBus, config: dict[str, Any]) -> None:
        """初始化通道。
        
        Args:
            name: 通道名称（如 "feishu"、"slack"、"telegram"）
            bus: 消息总线实例
            config: 通道配置字典
        """
        
    @property
    def is_running(self) -> bool:
        """通道是否正在运行。"""
        
    @abstractmethod
    async def start(self) -> None:
        """启动通道，开始监听外部平台消息。"""
        
    @abstractmethod
    async def stop(self) -> None:
        """优雅停止通道。"""
        
    @abstractmethod
    async def send(self, msg: OutboundMessage) -> None:
        """发送消息到外部平台。
        
        Args:
            msg: 出站消息对象
        """
        
    async def send_file(self, msg: OutboundMessage, 
                        attachment: ResolvedAttachment) -> bool:
        """上传文件到外部平台（可选实现）。
        
        Returns:
            True 表示上传成功，False 表示不支持或失败
        """
```

### 5.2 通道注册机制

通道通过注册表实现延迟加载：

```python
# service.py
_CHANNEL_REGISTRY: dict[str, str] = {
    "feishu": "src.channels.feishu:FeishuChannel",
    "slack": "src.channels.slack:SlackChannel",
    "telegram": "src.channels.telegram:TelegramChannel",
}
```

**动态加载**：

```python
from src.reflection import resolve_class

channel_cls = resolve_class("src.channels.feishu:FeishuChannel")
channel = channel_cls(bus=bus, config=config)
```

---

## 6. 配置项

### 6.1 完整配置示例

```yaml
# config.yaml
channels:
  # LangGraph Server 地址
  langgraph_url: "http://localhost:2024"
  
  # Gateway API 地址
  gateway_url: "http://localhost:8001"
  
  # 默认会话配置（所有通道共享）
  session:
    assistant_id: "lead_agent"
    config:
      recursion_limit: 100
    context:
      thinking_enabled: true
      is_plan_mode: false
      subagent_enabled: false
  
  # 飞书通道配置
  feishu:
    enabled: true
    app_id: "cli_xxx"
    app_secret: "xxx"
    # 通道级别会话配置（覆盖默认）
    session:
      assistant_id: "custom_agent"
      config:
        recursion_limit: 200
      context:
        thinking_enabled: false
  
  # Slack 通道配置
  slack:
    enabled: true
    bot_token: "xoxb-xxx"
    app_token: "xapp-xxx"
    allowed_users:
      - "U1234567890"
    # 用户级别会话配置
    session:
      users:
        "U1234567890":
          assistant_id: "vip_agent"
  
  # Telegram 通道配置
  telegram:
    enabled: true
    bot_token: "123456789:ABCdef"
    allowed_users:
      - 123456789
```

### 6.2 配置层级说明

会话配置支持三级覆盖：

```
默认配置（session）
    │
    ├── 通道级别（channels.<name>.session）
    │       │
    │       └── 用户级别（channels.<name>.session.users.<user_id>）
```

**合并优先级**：用户级别 > 通道级别 > 默认配置

### 6.3 配置字段说明

#### 全局配置

| 字段 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `langgraph_url` | string | `http://localhost:2024` | LangGraph Server 地址 |
| `gateway_url` | string | `http://localhost:8001` | Gateway API 地址 |
| `session` | object | 见下方 | 默认会话配置 |

#### session 配置

| 字段 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `assistant_id` | string | `lead_agent` | Agent ID |
| `config` | object | `{"recursion_limit": 100}` | LangGraph 运行配置 |
| `context` | object | 见下方 | 运行上下文 |

#### context 配置

| 字段 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `thinking_enabled` | bool | `true` | 是否启用思考模式 |
| `is_plan_mode` | bool | `false` | 是否为规划模式 |
| `subagent_enabled` | bool | `false` | 是否启用子 Agent |

---

## 7. 执行流程

### 7.1 消息处理完整流程

```
┌─────────────────────────────────────────────────────────────────────────┐
│                          消息处理完整流程                                 │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  1. 用户在 IM 平台发送消息                                               │
│         │                                                               │
│         ▼                                                               │
│  ┌─────────────────┐                                                    │
│  │ Channel (SDK)   │ ◄── Feishu/Slack/Telegram SDK 回调                 │
│  │ _on_message()   │                                                    │
│  └────────┬────────┘                                                    │
│           │                                                             │
│           ▼                                                             │
│  ┌─────────────────┐                                                    │
│  │ 创建 InboundMsg │                                                    │
│  │ - channel_name  │                                                    │
│  │ - chat_id       │                                                    │
│  │ - topic_id      │ ◄── 用于线程映射                                   │
│  │ - text          │                                                    │
│  └────────┬────────┘                                                    │
│           │                                                             │
│           ▼                                                             │
│  ┌─────────────────┐                                                    │
│  │ MessageBus      │                                                    │
│  │ publish_inbound │                                                    │
│  └────────┬────────┘                                                    │
│           │                                                             │
│           ▼                                                             │
│  ┌─────────────────┐                                                    │
│  │ ChannelManager  │ ◄── _dispatch_loop() 消费队列                      │
│  │ _handle_message │                                                    │
│  └────────┬────────┘                                                    │
│           │                                                             │
│           ▼                                                             │
│  ┌─────────────────┐                                                    │
│  │ 查询/创建线程   │ ◄── ChannelStore.get_thread_id()                   │
│  │                 │     或 client.threads.create()                     │
│  └────────┬────────┘                                                    │
│           │                                                             │
│           ▼                                                             │
│  ┌─────────────────┐                                                    │
│  │ LangGraph SDK   │                                                    │
│  │ runs.wait()     │ ◄── 调用 Agent                                     │
│  └────────┬────────┘                                                    │
│           │                                                             │
│           ▼                                                             │
│  ┌─────────────────┐                                                    │
│  │ 提取响应文本    │ ◄── _extract_response_text()                       │
│  │ 提取产物列表    │ ◄── _extract_artifacts()                           │
│  └────────┬────────┘                                                    │
│           │                                                             │
│           ▼                                                             │
│  ┌─────────────────┐                                                    │
│  │ 解析文件附件    │ ◄── _resolve_attachments()                         │
│  │ (安全路径验证)  │                                                    │
│  └────────┬────────┘                                                    │
│           │                                                             │
│           ▼                                                             │
│  ┌─────────────────┐                                                    │
│  │ 创建 OutboundMsg│                                                    │
│  │ publish_outbound│                                                    │
│  └────────┬────────┘                                                    │
│           │                                                             │
│           ▼                                                             │
│  ┌─────────────────┐                                                    │
│  │ Channel         │ ◄── _on_outbound() 回调                            │
│  │ send()          │                                                    │
│  │ send_file()     │ ◄── 上传文件附件                                   │
│  └────────┬────────┘                                                    │
│           │                                                             │
│           ▼                                                             │
│  用户在 IM 平台收到回复                                                  │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### 7.2 线程映射机制

```
┌─────────────────────────────────────────────────────────────────────────┐
│                          线程映射机制                                    │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  IM 平台消息                    DeerFlow 线程                           │
│  ─────────────                 ─────────────                           │
│                                                                         │
│  用户 A 发送新消息              → 创建新线程 thread-1                    │
│  chat_id: C1, topic_id: T1     │                                       │
│                                │ ChannelStore: C1:T1 → thread-1        │
│                                │                                       │
│  用户 A 在同一话题回复          → 复用线程 thread-1                     │
│  chat_id: C1, topic_id: T1     │                                       │
│                                │ 查询: C1:T1 → thread-1 (命中)         │
│                                │                                       │
│  用户 A 开始新话题              → 创建新线程 thread-2                   │
│  chat_id: C1, topic_id: T2     │                                       │
│                                │ ChannelStore: C1:T2 → thread-2        │
│                                │                                       │
│  用户 B 发送消息                → 创建新线程 thread-3                   │
│  chat_id: C2, topic_id: T3     │                                       │
│                                │ ChannelStore: C2:T3 → thread-3        │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### 7.3 安全机制

**文件路径验证**：

```python
# 只允许访问 /mnt/user-data/outputs/ 下的文件
_OUTPUTS_VIRTUAL_PREFIX = "/mnt/user-data/outputs/"

def _resolve_attachments(thread_id: str, artifacts: list[str]) -> list[ResolvedAttachment]:
    for virtual_path in artifacts:
        # 1. 检查路径前缀
        if not virtual_path.startswith(_OUTPUTS_VIRTUAL_PREFIX):
            logger.warning("rejected non-outputs artifact path: %s", virtual_path)
            continue
            
        # 2. 解析实际路径
        actual = paths.resolve_virtual_path(thread_id, virtual_path)
        
        # 3. 验证路径不会逃逸到 outputs 目录之外
        actual.resolve().relative_to(outputs_dir)
        
        # 4. 验证文件存在
        if not actual.is_file():
            continue
```

---

## 8. 扩展指南

### 8.1 添加新通道

1. **创建通道类**：

```python
# src/channels/my_channel.py
from src.channels.base import Channel
from src.channels.message_bus import MessageBus, OutboundMessage

class MyChannel(Channel):
    def __init__(self, bus: MessageBus, config: dict) -> None:
        super().__init__(name="my_channel", bus=bus, config=config)
        
    async def start(self) -> None:
        # 初始化 SDK，注册回调
        pass
        
    async def stop(self) -> None:
        # 清理资源
        pass
        
    async def send(self, msg: OutboundMessage) -> None:
        # 发送消息到平台
        pass
```

2. **注册通道**：

```python
# src/channels/service.py
_CHANNEL_REGISTRY: dict[str, str] = {
    ...
    "my_channel": "src.channels.my_channel:MyChannel",
}
```

3. **添加配置**：

```yaml
# config.yaml
channels:
  my_channel:
    enabled: true
    api_key: "xxx"
```

### 8.2 最佳实践

- **异步处理**：所有 SDK 回调应使用 `asyncio.run_coroutine_threadsafe()` 调度到主事件循环
- **错误处理**：捕获所有异常，避免影响其他通道
- **重试机制**：发送失败时实现指数退避重试
- **日志记录**：记录关键操作和错误信息
- **资源清理**：在 `stop()` 中正确释放 SDK 资源

---

## 9. 依赖项

| 包名 | 用途 | 安装命令 |
|------|------|----------|
| `lark-oapi` | 飞书 SDK | `uv add lark-oapi` |
| `slack-sdk` | Slack SDK | `uv add slack-sdk` |
| `python-telegram-bot` | Telegram SDK | `uv add python-telegram-bot` |
| `markdown-to-mrkdwn` | Markdown 转 Slack 格式 | `uv add markdown-to-mrkdwn` |
| `langgraph-sdk` | LangGraph 客户端 | 已安装 |