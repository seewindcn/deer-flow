# AGENTS.md - DeerFlow 开发指南

# Agent 行为规范（强制）
**重要规则：本项目中的 Agent 交互与对话必须使用中文。**

本文件为在此代码库中运行的 Agent 提供必要的开发指南。

## 项目概述

DeerFlow 是一个全栈"超级 Agent 框架"，包含：
- **后端**：Python 3.12，LangGraph + FastAPI 网关，沙箱/工具系统，记忆模块，MCP 集成
- **前端**：Next.js 16 + React 19 + TypeScript + pnpm
- **统一入口**：`make dev` 启动所有服务，访问地址 `http://localhost:2026`

## 构建/检查/测试命令

### 根目录命令
```bash
make check      # 验证 Node.js 22+、pnpm、uv、nginx 是否已安装
make install    # 安装所有依赖（后端 + 前端）
make dev        # 启动所有服务（LangGraph:2024, Gateway:8001, Frontend:3000, nginx:2026）
make stop       # 停止所有运行中的服务
make config     # 从模板生成 config.yaml（仅首次设置）
```

### 后端命令（在 `backend/` 目录下）
```bash
make install    # 使用 uv 安装 Python 依赖
make dev        # 仅运行 LangGraph 服务器（端口 2024）
make gateway    # 仅运行 Gateway API（端口 8001）
make lint       # 使用 ruff 进行代码检查（ruff check .）
make format     # 使用 ruff 格式化代码（ruff check --fix && ruff format）
make test       # 使用 pytest 运行所有测试

# 运行单个测试文件
PYTHONPATH=. uv run pytest tests/test_model_factory.py -v

# 运行单个测试函数
PYTHONPATH=. uv run pytest tests/test_model_factory.py::test_create_chat_model -v
```

### 前端命令（在 `frontend/` 目录下）
```bash
pnpm install    # 安装依赖
pnpm dev        # 使用 Turbo 启动开发服务器（端口 3000）
pnpm lint       # 使用 ESLint 进行代码检查
pnpm lint:fix   # 代码检查并自动修复
pnpm typecheck  # TypeScript 类型检查
pnpm build      # 生产构建（需要 BETTER_AUTH_SECRET）

# 使用密钥进行构建
BETTER_AUTH_SECRET=local-dev-secret pnpm build
```

## 代码风格指南

### 后端（Python）

**格式化与检查**：
- 使用 `ruff` 进行代码检查和格式化
- 行长度：240 字符
- Python 3.12+ 并使用类型注解
- 双引号，空格缩进（4 个空格）
- 配置文件：`backend/ruff.toml`

**导入顺序**：
```python
# 标准库
from __future__ import annotations
from pathlib import Path
from typing import Any

# 第三方库
import pytest
from langchain.chat_models import BaseChatModel

# 本地导入
from src.config.app_config import AppConfig
```

**命名规范**：
- 文件：`snake_case.py`
- 类：`PascalCase`
- 函数/变量：`snake_case`
- 私有成员：`_leading_underscore`
- 常量：`UPPER_SNAKE_CASE`

**错误处理**：
- 使用明确的异常并提供描述性消息
- 领域错误优先使用自定义异常类
- 使用 `try/except` 捕获特定异常类型

**类型注解**：
- 始终为函数参数和返回值添加类型注解
- 使用 `from __future__ import annotations` 处理前向引用
- 优先使用 `list[T]` 而非 `List[T]`（Python 3.12+）

### 前端（TypeScript/React）

**格式化与检查**：
- ESLint + TypeScript 规则 + Prettier
- 配置文件：`frontend/eslint.config.js`

**导入顺序**（由 `import/order` 规则强制）：
```typescript
// 1. 内置模块
import { useState } from "react";

// 2. 外部包
import { useQuery } from "@tanstack/react-query";

// 3. 内部别名 (@/*)
import { Button } from "@/components/ui/button";

// 4. 父级/同级导入
import { helper } from "./utils";

// 5. CSS/MD 导入
import "./styles.css";
```

**类型导入**（由 `@typescript-eslint/consistent-type-imports` 强制）：
```typescript
// 推荐
import { type SomeType } from "module";
import type { Props } from "./types";

// 避免
import { SomeType } from "module";
```

**命名规范**：
- 文件：组件使用 `kebab-case.tsx`
- 组件：`PascalCase` 函数组件
- Hooks：`useCamelCase`
- 工具函数：`camelCase`
- 类型/接口：`PascalCase`

**React 模式**：
- 使用函数组件，而非类组件
- 使用 `@/*` 路径别名进行内部导入
- 未使用的参数使用下划线前缀：`_unused`

**TypeScript 配置**：
- 启用严格模式
- `noUncheckedIndexedAccess: true`
- `verbatimModuleSyntax: true`
- 目标：ES2022

## 项目结构

```
deer-flow/
├── config.yaml              # 主应用配置（已忽略）
├── extensions_config.json   # MCP 服务器和技能配置
├── backend/
│   ├── src/
│   │   ├── agents/          # LangGraph Agent 系统
│   │   ├── gateway/         # FastAPI Gateway API
│   │   ├── sandbox/         # 沙箱执行系统
│   │   ├── subagents/       # 子 Agent 委派系统
│   │   ├── mcp/             # MCP 集成
│   │   └── models/          # 模型工厂
│   └── tests/               # 单元测试
├── frontend/
│   └── src/
│       ├── app/             # Next.js App Router
│       ├── components/      # React 组件
│       ├── core/            # 业务逻辑
│       └── hooks/           # 自定义 Hooks
└── skills/
    ├── public/              # 内置技能
    └── custom/              # 自定义技能（已忽略）
```

## 提交前检查清单

提交更改前：

1. **后端**：`cd backend && make lint && make test`
2. **前端**：`cd frontend && pnpm lint && pnpm typecheck`
3. **构建测试**（涉及环境/认证变更时）：`BETTER_AUTH_SECRET=... pnpm build`

## 重要说明

- **BETTER_AUTH_SECRET** 是前端生产构建的必需项
- **代理环境变量**可能导致 `pnpm install` 失败，如有问题请取消设置
- **`make config`** 在 `config.yaml` 已存在时会中止（这是设计如此）
- **测试是强制性的** — 每个功能/修复都必须包含测试
- **TDD 工作流** — 新功能优先编写测试