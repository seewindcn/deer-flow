# 🦌 DeerFlow - 2.0

<a href="https://trendshift.io/repositories/14699" target="_blank"><img src="https://trendshift.io/api/badge/repositories/14699" alt="bytedance%2Fdeer-flow | Trendshift" style="width: 250px; height: 55px;" width="250" height="55"/></a>
> 2026年2月28日，DeerFlow 在 2.0 版本发布后登顶 GitHub Trending 🏆 第一名。万分感谢我们出色的社区 —— 是你们创造了这一切！💪🔥

DeerFlow（**D**eep **E**xploration and **E**fficient **R**esearch **Flow**，深度探索与高效研究流）是一个开源的**超级 Agent 框架**，通过编排**子 Agent**、**记忆**和**沙箱**来完成几乎所有任务 —— 由**可扩展的技能**驱动。

https://github.com/user-attachments/assets/a8bcadc4-e040-4cf2-8fda-dd768b999c18

> [!NOTE]
> **DeerFlow 2.0 是一次从零开始的重新设计。** 它与 v1 版本没有共享任何代码。如果你在寻找原始的 Deep Research 框架，它维护在 [`1.x` 分支](https://github.com/bytedance/deer-flow/tree/main-1.x) 上 —— 欢迎继续贡献。活跃开发已转移到 2.0。

## 官方网站

在我们的官方网站了解更多信息并查看**真实演示**。

**[deerflow.tech](https://deerflow.tech/)**

## InfoQuest

DeerFlow 新集成了字节跳动自主研发的智能搜索和爬取工具集 —— [InfoQuest（支持免费在线体验）](https://docs.byteplus.com/en/docs/InfoQuest/What_is_Info_Quest)

<a href="https://docs.byteplus.com/en/docs/InfoQuest/What_is_Info_Quest" target="_blank">
  <img 
    src="https://sf16-sg.tiktokcdn.com/obj/eden-sg/hubseh7bsbps/20251208-160108.png"   alt="InfoQuest_banner" 
  />
</a>

---

## 目录

- [🦌 DeerFlow - 2.0](#-deerflow---20)
  - [官方网站](#官方网站)
  - [InfoQuest](#infoquest)
  - [目录](#目录)
  - [快速开始](#快速开始)
    - [配置](#配置)
    - [运行应用](#运行应用)
      - [方式一：Docker（推荐）](#方式一docker推荐)
      - [方式二：本地开发](#方式二本地开发)
    - [高级配置](#高级配置)
      - [沙箱模式](#沙箱模式)
      - [MCP 服务器](#mcp-服务器)
      - [IM 通道](#im-通道)
  - [从深度研究到超级 Agent 框架](#从深度研究到超级-agent-框架)
  - [核心特性](#核心特性)
    - [技能与工具](#技能与工具)
      - [Claude Code 集成](#claude-code-集成)
    - [子 Agent](#子-agent)
    - [沙箱与文件系统](#沙箱与文件系统)
    - [上下文工程](#上下文工程)
    - [长期记忆](#长期记忆)
  - [推荐模型](#推荐模型)
  - [嵌入式 Python 客户端](#嵌入式-python-客户端)
  - [文档](#文档)
  - [贡献](#贡献)
  - [许可证](#许可证)
  - [致谢](#致谢)
    - [核心贡献者](#核心贡献者)
  - [Star 历史](#star-历史)

## 快速开始

### 配置

1. **克隆 DeerFlow 仓库**

   ```bash
   git clone https://github.com/bytedance/deer-flow.git
   cd deer-flow
   ```

2. **生成本地配置文件**

   在项目根目录（`deer-flow/`）下运行：

   ```bash
   make config
   ```

   此命令根据提供的示例模板创建本地配置文件。

3. **配置你偏好的模型**

   编辑 `config.yaml` 并至少定义一个模型：

   ```yaml
   models:
     - name: gpt-4                       # 内部标识符
       display_name: GPT-4               # 人类可读名称
       use: langchain_openai:ChatOpenAI  # LangChain 类路径
       model: gpt-4                      # API 模型标识符
       api_key: $OPENAI_API_KEY          # API 密钥（推荐：使用环境变量）
       max_tokens: 4096                  # 每次请求的最大 token 数
       temperature: 0.7                  # 采样温度

     - name: openrouter-gemini-2.5-flash
       display_name: Gemini 2.5 Flash (OpenRouter)
       use: langchain_openai:ChatOpenAI
       model: google/gemini-2.5-flash-preview
       api_key: $OPENAI_API_KEY          # OpenRouter 在此处仍使用 OpenAI 兼容的字段名
       base_url: https://openrouter.ai/api/v1
   ```

   OpenRouter 和类似的 OpenAI 兼容网关应使用 `langchain_openai:ChatOpenAI` 并配置 `base_url`。如果你更喜欢使用特定提供商的环境变量名，请显式地将 `api_key` 指向该变量（例如 `api_key: $OPENROUTER_API_KEY`）。

4. **为配置的模型设置 API 密钥**

   选择以下方式之一：

- 方式 A：编辑项目根目录下的 `.env` 文件（推荐）


   ```bash
   TAVILY_API_KEY=your-tavily-api-key
   OPENAI_API_KEY=your-openai-api-key
   # 当你的配置使用 langchain_openai:ChatOpenAI + base_url 时，OpenRouter 也使用 OPENAI_API_KEY
   # 根据需要添加其他提供商的密钥
   INFOQUEST_API_KEY=your-infoquest-api-key
   ```

- 方式 B：在 shell 中导出环境变量

   ```bash
   export OPENAI_API_KEY=your-openai-api-key
   ```

- 方式 C：直接编辑 `config.yaml`（生产环境不推荐）

   ```yaml
   models:
     - name: gpt-4
       api_key: your-actual-api-key-here  # 替换占位符
   ```

### 运行应用

#### 方式一：Docker（推荐）

**开发环境**（热重载，源码挂载）：

```bash
make docker-init    # 拉取沙箱镜像（仅需一次或镜像更新时）
make docker-start   # 启动服务（自动从 config.yaml 检测沙箱模式）
```

当 `config.yaml` 使用 provisioner 模式（`sandbox.use: deerflow.community.aio_sandbox:AioSandboxProvider` 配合 `provisioner_url`）时，`make docker-start` 才会启动 `provisioner`。

**生产环境**（本地构建镜像，挂载运行时配置和数据）：

```bash
make up     # 构建镜像并启动所有生产服务
make down   # 停止并移除容器
```

> [!NOTE]
> LangGraph agent 服务器目前通过 `langgraph dev`（开源 CLI 服务器）运行。

访问地址：http://localhost:2026

详细的 Docker 开发指南请参阅 [CONTRIBUTING.md](CONTRIBUTING.md)。

#### 方式二：本地开发

如果你更喜欢在本地运行服务：

前置条件：先完成上面的"配置"步骤（`make config` 和模型 API 密钥）。`make dev` 需要有效的配置文件（默认为项目根目录下的 `config.yaml`；可通过 `DEER_FLOW_CONFIG_PATH` 覆盖）。

1. **检查前置条件**：
   ```bash
   make check  # 验证 Node.js 22+、pnpm、uv、nginx
   ```

2. **安装依赖**：
   ```bash
   make install  # 安装后端 + 前端依赖
   ```

3. **（可选）预拉取沙箱镜像**：
   ```bash
   # 如果使用 Docker/容器沙箱，推荐执行
   make setup-sandbox
   ```

4. **启动服务**：
   ```bash
   make dev
   ```

5. **访问地址**：http://localhost:2026

### 高级配置

#### 沙箱模式

DeerFlow 支持多种沙箱执行模式：
- **本地执行**（直接在主机上运行沙箱代码）
- **Docker 执行**（在隔离的 Docker 容器中运行沙箱代码）
- **Kubernetes 上的 Docker 执行**（通过 provisioner 服务在 Kubernetes Pod 中运行沙箱代码）

对于 Docker 开发，服务启动遵循 `config.yaml` 的沙箱模式。在本地/Docker 模式下，不会启动 `provisioner`。

请参阅[沙箱配置指南](backend/docs/CONFIGURATION.md#sandbox)来配置你偏好的模式。

#### MCP 服务器

DeerFlow 支持可配置的 MCP 服务器和技能来扩展其能力。
对于 HTTP/SSE MCP 服务器，支持 OAuth 令牌流程（`client_credentials`、`refresh_token`）。
详细说明请参阅 [MCP 服务器指南](backend/docs/MCP_SERVER.md)。

#### IM 通道

DeerFlow 支持从即时通讯应用接收任务。通道在配置后自动启动 —— 所有通道都不需要公网 IP。

| 通道 | 传输方式 | 难度 |
|------|----------|------|
| Telegram | Bot API（长轮询） | 简单 |
| Slack | Socket Mode | 中等 |
| 飞书 / Lark | WebSocket | 中等 |

**在 `config.yaml` 中配置：**

```yaml
channels:
  # LangGraph 服务器 URL（默认：http://localhost:2024）
  langgraph_url: http://localhost:2024
  # Gateway API URL（默认：http://localhost:8001）
  gateway_url: http://localhost:8001

  # 可选：所有移动通道的全局会话默认值
  session:
    assistant_id: lead_agent
    config:
      recursion_limit: 100
    context:
      thinking_enabled: true
      is_plan_mode: false
      subagent_enabled: false

  feishu:
    enabled: true
    app_id: $FEISHU_APP_ID
    app_secret: $FEISHU_APP_SECRET

  slack:
    enabled: true
    bot_token: $SLACK_BOT_TOKEN     # xoxb-...
    app_token: $SLACK_APP_TOKEN     # xapp-...（Socket Mode）
    allowed_users: []              # 空 = 允许所有用户

  telegram:
    enabled: true
    bot_token: $TELEGRAM_BOT_TOKEN
    allowed_users: []              # 空 = 允许所有用户

    # 可选：按通道/用户设置会话配置
    session:
      assistant_id: mobile_agent
      context:
        thinking_enabled: false
      users:
        "123456789":
          assistant_id: vip_agent
          config:
            recursion_limit: 150
          context:
            thinking_enabled: true
            subagent_enabled: true
```

在 `.env` 文件中设置相应的 API 密钥：

```bash
# Telegram
TELEGRAM_BOT_TOKEN=123456789:ABCdefGHIjklMNOpqrSTUvwxYZ

# Slack
SLACK_BOT_TOKEN=xoxb-...
SLACK_APP_TOKEN=xapp-...

# 飞书 / Lark
FEISHU_APP_ID=cli_xxxx
FEISHU_APP_SECRET=your_app_secret
```

**Telegram 设置**

1. 与 [@BotFather](https://t.me/BotFather) 对话，发送 `/newbot`，然后复制 HTTP API 令牌。
2. 在 `.env` 中设置 `TELEGRAM_BOT_TOKEN` 并在 `config.yaml` 中启用该通道。

**Slack 设置**

1. 在 [api.slack.com/apps](https://api.slack.com/apps) 创建 Slack App → Create New App → From scratch。
2. 在 **OAuth & Permissions** 下，添加 Bot Token Scopes：`app_mentions:read`、`chat:write`、`im:history`、`im:read`、`im:write`、`files:write`。
3. 启用 **Socket Mode** → 生成一个 App-Level Token（`xapp-…`），包含 `connections:write` scope。
4. 在 **Event Subscriptions** 下，订阅 bot 事件：`app_mention`、`message.im`。
5. 在 `.env` 中设置 `SLACK_BOT_TOKEN` 和 `SLACK_APP_TOKEN`，并在 `config.yaml` 中启用该通道。

**飞书 / Lark 设置**

1. 在[飞书开放平台](https://open.feishu.cn/)创建应用 → 启用**机器人**能力。
2. 添加权限：`im:message`、`im:message.p2p_msg:readonly`、`im:resource`。
3. 在**事件**下，订阅 `im.message.receive_v1` 并选择**长连接**模式。
4. 复制 App ID 和 App Secret。在 `.env` 中设置 `FEISHU_APP_ID` 和 `FEISHU_APP_SECRET`，并在 `config.yaml` 中启用该通道。

**命令**

通道连接后，你可以直接从聊天中与 DeerFlow 交互：

| 命令 | 描述 |
|------|------|
| `/new` | 开始新对话 |
| `/status` | 显示当前线程信息 |
| `/models` | 列出可用模型 |
| `/memory` | 查看记忆 |
| `/help` | 显示帮助 |

> 不带命令前缀的消息被视为普通聊天 —— DeerFlow 会创建线程并以对话方式响应。

## 从深度研究到超级 Agent 框架

DeerFlow 最初是一个深度研究框架 —— 社区让它走得更远。自发布以来，开发者将其扩展到研究之外：构建数据管道、生成幻灯片、启动仪表板、自动化内容工作流。这些是我们从未预料到的用途。

这告诉我们一件重要的事情：DeerFlow 不仅仅是一个研究工具。它是一个**框架** —— 一个为 Agent 提供实际完成工作所需基础设施的运行时。

所以我们从头重建了它。

DeerFlow 2.0 不再是一个你需要自己组装的框架。它是一个超级 Agent 框架 —— 开箱即用，完全可扩展。基于 LangGraph 和 LangChain 构建，它自带 Agent 所需的一切：文件系统、记忆、技能、沙箱执行，以及规划和生成子 Agent 来处理复杂多步骤任务的能力。

直接使用。或者拆解它，让它成为你的。

## 核心特性

### 技能与工具

技能是让 DeerFlow 能够做**几乎所有事情**的关键。

一个标准的 Agent Skill 是一个结构化的能力模块 —— 一个定义工作流程、最佳实践和参考支持资源的 Markdown 文件。DeerFlow 内置了研究、报告生成、幻灯片创建、网页、图像和视频生成等技能。但真正的力量在于可扩展性：添加你自己的技能，替换内置技能，或将它们组合成复合工作流。

技能是按需加载的 —— 只有在任务需要时才加载，而不是一次性全部加载。这保持了上下文窗口的精简，使 DeerFlow 即使在使用对 token 敏感的模型时也能良好工作。

当通过 Gateway 安装 `.skill` 归档文件时，DeerFlow 接受标准可选的前置元数据（如 `version`、`author` 和 `compatibility`），而不是拒绝其他有效的外部技能。

工具遵循相同的理念。DeerFlow 附带核心工具集 —— 网页搜索、网页抓取、文件操作、bash 执行 —— 并通过 MCP 服务器和 Python 函数支持自定义工具。替换任何东西。添加任何东西。

Gateway 生成的后续建议现在在解析 JSON 数组响应之前会规范化纯字符串模型输出和块/列表样式的富内容，因此特定提供商的内容包装器不会静默丢弃建议。

```
# 沙箱容器内的路径
/mnt/skills/public
├── research/SKILL.md
├── report-generation/SKILL.md
├── slide-creation/SKILL.md
├── web-page/SKILL.md
└── image-generation/SKILL.md

/mnt/skills/custom
└── your-custom-skill/SKILL.md      ← 你的
```

#### Claude Code 集成

`claude-to-deerflow` 技能让你直接从 [Claude Code](https://docs.anthropic.com/en/docs/claude-code) 与运行中的 DeerFlow 实例交互。发送研究任务、检查状态、管理线程 —— 无需离开终端。

**安装技能**：

```bash
npx skills add https://github.com/bytedance/deer-flow --skill claude-to-deerflow
```

然后确保 DeerFlow 正在运行（默认在 `http://localhost:2026`），在 Claude Code 中使用 `/claude-to-deerflow` 命令。

**可以做什么**：
- 向 DeerFlow 发送消息并获取流式响应
- 选择执行模式：flash（快速）、standard、pro（规划）、ultra（子 Agent）
- 检查 DeerFlow 健康状态，列出模型/技能/Agent
- 管理线程和对话历史
- 上传文件进行分析

**环境变量**（可选，用于自定义端点）：

```bash
DEERFLOW_URL=http://localhost:2026            # 统一代理基础 URL
DEERFLOW_GATEWAY_URL=http://localhost:2026    # Gateway API
DEERFLOW_LANGGRAPH_URL=http://localhost:2026/api/langgraph  # LangGraph API
```

完整 API 参考请参阅 [`skills/public/claude-to-deerflow/SKILL.md`](skills/public/claude-to-deerflow/SKILL.md)。

### 子 Agent

复杂任务很少能一次性完成。DeerFlow 会将它们分解。

主导 Agent 可以动态生成子 Agent —— 每个子 Agent 都有自己的作用域上下文、工具和终止条件。子 Agent 在可能的情况下并行运行，报告结构化结果，主导 Agent 将所有内容综合成连贯的输出。

这就是 DeerFlow 处理需要数分钟到数小时任务的方式：一个研究任务可能会分支出十几个子 Agent，每个探索不同的角度，然后汇聚成一份报告 —— 或一个网站 —— 或一个带有生成视觉效果的幻灯片。一个框架，多只手。

### 沙箱与文件系统

DeerFlow 不仅仅是**谈论**做事。它有自己的计算机。

每个任务都在一个独立的 Docker 容器中运行，拥有完整的文件系统 —— 技能、工作区、上传、输出。Agent 读取、写入和编辑文件。它执行 bash 命令和代码。它查看图像。全部沙箱化，全部可审计，会话之间零污染。

这就是一个有工具访问权限的聊天机器人和一个有实际执行环境的 Agent 之间的区别。

```
# 沙箱容器内的路径
/mnt/user-data/
├── uploads/          ← 你的文件
├── workspace/        ← Agent 的工作目录
└── outputs/          ← 最终交付物
```

### 上下文工程

**隔离的子 Agent 上下文**：每个子 Agent 在自己隔离的上下文中运行。这意味着子 Agent 无法看到主导 Agent 或其他子 Agent 的上下文。这对于确保子 Agent 能够专注于手头的任务而不被主导 Agent 或其他子 Agent 的上下文分心非常重要。

**摘要**：在会话内，DeerFlow 积极管理上下文 —— 摘要已完成的子任务、将中间结果卸载到文件系统、压缩不再立即相关的内容。这使它能够在长时间的多步骤任务中保持敏锐，而不会撑爆上下文窗口。

### 长期记忆

大多数 Agent 在对话结束的那一刻就会忘记一切。DeerFlow 会记住。

跨会话，DeerFlow 建立对你个人资料、偏好和积累知识的持久记忆。你使用得越多，它就越了解你 —— 你的写作风格、你的技术栈、你的重复工作流。记忆存储在本地，由你控制。

## 推荐模型

DeerFlow 与模型无关 —— 它适用于任何实现 OpenAI 兼容 API 的 LLM。尽管如此，它在支持以下特性的模型上表现最佳：

- **长上下文窗口**（100k+ tokens）用于深度研究和多步骤任务
- **推理能力**用于自适应规划和复杂分解
- **多模态输入**用于图像理解和视频理解
- **强大的工具使用**用于可靠的函数调用和结构化输出

## 嵌入式 Python 客户端

DeerFlow 可以作为嵌入式 Python 库使用，无需运行完整的 HTTP 服务。`DeerFlowClient` 提供对所有 Agent 和 Gateway 能力的直接进程内访问，返回与 HTTP Gateway API 相同的响应模式：

```python
from deerflow.client import DeerFlowClient

client = DeerFlowClient()

# 聊天
response = client.chat("帮我分析这篇论文", thread_id="my-thread")

# 流式传输（LangGraph SSE 协议：values、messages-tuple、end）
for event in client.stream("你好"):
    if event.type == "messages-tuple" and event.data.get("type") == "ai":
        print(event.data["content"])

# 配置与管理 —— 返回与 Gateway 对齐的字典
models = client.list_models()        # {"models": [...]}
skills = client.list_skills()        # {"skills": [...]}
client.update_skill("web-search", enabled=True)
client.upload_files("thread-1", ["./report.pdf"])  # {"success": True, "files": [...]}
```

所有返回字典的方法都在 CI 中针对 Gateway Pydantic 响应模型进行验证（`TestGatewayConformance`），确保嵌入式客户端与 HTTP API 模式保持同步。完整 API 文档请参阅 `backend/packages/harness/deerflow/client.py`。

## 文档

- [贡献指南](CONTRIBUTING.md) - 开发环境设置和工作流程
- [配置指南](backend/docs/CONFIGURATION.md) - 设置和配置说明
- [架构概览](backend/CLAUDE.md) - 技术架构详情
- [后端架构](backend/README.md) - 后端架构和 API 参考

## 贡献

我们欢迎贡献！请参阅 [CONTRIBUTING.md](CONTRIBUTING.md) 了解开发设置、工作流程和指南。

回归测试覆盖包括 Docker 沙箱模式检测和 provisioner kubeconfig-path 处理测试，位于 `backend/tests/`。

## 许可证

本项目是开源的，采用 [MIT 许可证](./LICENSE)。

## 致谢

DeerFlow 建立在开源社区出色工作之上。我们深深感谢所有项目和贡献者的努力，是他们让 DeerFlow 成为可能。真的，我们站在巨人的肩膀上。

我们要向以下项目表达我们真诚的感谢，感谢他们宝贵的贡献：

- **[LangChain](https://github.com/langchain-ai/langchain)**：他们卓越的框架为我们的 LLM 交互和链提供动力，实现了无缝集成和功能。
- **[LangGraph](https://github.com/langchain-ai/langgraph)**：他们创新的多 Agent 编排方法对于实现 DeerFlow 复杂的工作流程至关重要。

这些项目体现了开源协作的变革力量，我们很自豪能在它们的基础上构建。

### 核心贡献者

衷心感谢 `DeerFlow` 的核心作者，他们的愿景、热情和奉献让这个项目成为现实：

- **[Daniel Walnut](https://github.com/hetaoBackend/)**
- **[Henry Li](https://github.com/magiccube/)**

你们坚定不移的承诺和专业知识一直是 DeerFlow 成功的推动力。我们很荣幸有你们引领这段旅程。

## Star 历史

[![Star History Chart](https://api.star-history.com/svg?repos=bytedance/deer-flow&type=Date)](https://star-history.com/#bytedance/deer-flow&Date)