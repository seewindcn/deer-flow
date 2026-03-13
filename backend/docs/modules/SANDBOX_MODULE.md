# Sandbox 模块技术文档

> 本文档详细介绍 DeerFlow 的沙箱系统，包括沙箱抽象、本地沙箱实现、工具集成以及中间件机制。

## 目录

- [模块概述](#模块概述)
- [目录结构](#目录结构)
- [核心类详解](#核心类详解)
- [核心功能](#核心功能)
- [工具函数](#工具函数)
- [懒加载机制](#懒加载机制)
- [中间件](#中间件)
- [配置项](#配置项)
- [异常处理](#异常处理)
- [设计模式总结](#设计模式总结)

---

## 模块概述

### 职责和功能定位

`src.sandbox` 模块是 DeerFlow Agent 框架的核心隔离执行环境，负责：

1. **安全隔离**：为 Agent 提供隔离的命令执行和文件操作环境
2. **虚拟路径映射**：将沙箱内的虚拟路径映射到实际文件系统路径
3. **工具抽象**：提供统一的沙箱操作接口，支持多种后端实现
4. **生命周期管理**：管理沙箱的创建、获取、释放流程
5. **目录自动创建**：按需创建线程相关的数据目录

### 与其他模块的关系

```
┌─────────────────────────────────────────────────────────────────────────┐
│                           sandbox 模块                                   │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  ┌──────────────────┐    ┌──────────────────┐    ┌──────────────────┐  │
│  │    sandbox.py    │    │sandbox_provider.py│    │   exceptions.py  │  │
│  │   (抽象基类)      │◄───│   (提供者基类)     │◄───│    (异常定义)    │  │
│  └──────────────────┘    └──────────────────┘    └──────────────────┘  │
│           │                       │                                     │
│           │                       │                                     │
│           ▼                       ▼                                     │
│  ┌──────────────────┐    ┌──────────────────┐    ┌──────────────────┐  │
│  │    local/        │    │    tools.py      │    │   middleware.py  │  │
│  │ (本地沙箱实现)    │    │   (工具函数)      │    │    (中间件)      │  │
│  └──────────────────┘    └──────────────────┘    └──────────────────┘  │
│           │                       │                                     │
│           │                       │                                     │
│           ▼                       ▼                                     │
│  ┌──────────────────┐    ┌──────────────────┐                          │
│  │ local_sandbox.py │    │   config/        │                          │
│  │local_sandbox_    │    │ (路径、配置管理)  │                          │
│  │   provider.py    │    └──────────────────┘                          │
│  └──────────────────┘                                                   │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
                              │
                              │ 提供沙箱环境给
                              ▼
        ┌─────────────────────────────────────────────────┐
        │                  agents/ 模块                    │
        │       (Lead Agent, Mobile Agent 等)              │
        └─────────────────────────────────────────────────┘
                              │
                              │ 提供工具给
                              ▼
        ┌─────────────────────────────────────────────────┐
        │                  tools/ 模块                     │
        │      (bash_tool, read_file_tool 等)              │
        └─────────────────────────────────────────────────┘
```

**依赖关系**：

| 依赖模块 | 用途 |
|---------|------|
| `src.config` | 获取沙箱配置、路径配置 |
| `src.config.paths` | 虚拟路径前缀、路径解析 |
| `src.agents.thread_state` | 访问线程状态和沙箱状态 |
| `src.reflection` | 动态解析沙箱提供者类 |

---

## 目录结构

```
backend/src/sandbox/
├── __init__.py                    # 模块入口，导出 Sandbox, SandboxProvider, get_sandbox_provider
├── sandbox.py                     # Sandbox 抽象基类定义
├── sandbox_provider.py            # SandboxProvider 抽象基类和单例管理
├── exceptions.py                  # 自定义异常类
├── tools.py                       # 沙箱工具函数实现
├── middleware.py                  # SandboxMiddleware 中间件
└── local/                         # 本地沙箱实现
    ├── __init__.py                # 导出 LocalSandboxProvider
    ├── local_sandbox.py           # LocalSandbox 实现
    ├── local_sandbox_provider.py  # LocalSandboxProvider 实现
    └── list_dir.py                # 目录列表工具函数
```

### 文件说明

| 文件 | 职责 |
|------|------|
| `__init__.py` | 模块公共 API 导出 |
| `sandbox.py` | 定义 `Sandbox` 抽象基类，规定沙箱操作接口 |
| `sandbox_provider.py` | 定义 `SandboxProvider` 抽象基类和全局单例管理函数 |
| `exceptions.py` | 定义沙箱相关的自定义异常类 |
| `tools.py` | 实现 `bash_tool`, `read_file_tool`, `write_file_tool` 等工具函数 |
| `middleware.py` | 实现 `SandboxMiddleware`，管理沙箱生命周期 |
| `local/__init__.py` | 本地沙箱子模块导出 |
| `local/local_sandbox.py` | `LocalSandbox` 实现，提供本地沙箱功能 |
| `local/local_sandbox_provider.py` | `LocalSandboxProvider` 实现，单例模式管理本地沙箱 |
| `local/list_dir.py` | 目录遍历工具函数，支持忽略模式和深度限制 |

---

## 核心类详解

### Sandbox 抽象基类

`Sandbox` 是所有沙箱实现的抽象基类，定义了沙箱必须提供的操作接口。

**文件位置**：`src/sandbox/sandbox.py`

```python
from abc import ABC, abstractmethod


class Sandbox(ABC):
    """Abstract base class for sandbox environments"""

    _id: str

    def __init__(self, id: str):
        self._id = id

    @property
    def id(self) -> str:
        return self._id

    @abstractmethod
    def execute_command(self, command: str) -> str:
        """Execute bash command in sandbox.

        Args:
            command: The command to execute.

        Returns:
            The standard or error output of the command.
        """
        pass

    @abstractmethod
    def read_file(self, path: str) -> str:
        """Read the content of a file.

        Args:
            path: The absolute path of the file to read.

        Returns:
            The content of the file.
        """
        pass

    @abstractmethod
    def list_dir(self, path: str, max_depth=2) -> list[str]:
        """List the contents of a directory.

        Args:
            path: The absolute path of the directory to list.
            max_depth: The maximum depth to traverse. Default is 2.

        Returns:
            The contents of the directory.
        """
        pass

    @abstractmethod
    def write_file(self, path: str, content: str, append: bool = False) -> None:
        """Write content to a file.

        Args:
            path: The absolute path of the file to write to.
            content: The text content to write to the file.
            append: Whether to append the content to the file. If False, the file will be created or overwritten.
        """
        pass

    @abstractmethod
    def update_file(self, path: str, content: bytes) -> None:
        """Update a file with binary content.

        Args:
            path: The absolute path of the file to update.
            content: The binary content to write to the file.
        """
        pass
```

**核心方法说明**：

| 方法 | 功能 | 参数说明 |
|------|------|---------|
| `execute_command` | 执行 shell 命令 | `command`: 要执行的命令字符串 |
| `read_file` | 读取文本文件内容 | `path`: 文件绝对路径 |
| `write_file` | 写入文本文件 | `path`: 文件路径, `content`: 内容, `append`: 是否追加 |
| `update_file` | 写入二进制文件 | `path`: 文件路径, `content`: 二进制内容 |
| `list_dir` | 列出目录内容 | `path`: 目录路径, `max_depth`: 最大遍历深度 |

---

### SandboxProvider 抽象基类

`SandboxProvider` 是所有沙箱提供者的抽象基类，负责沙箱的生命周期管理。

**文件位置**：`src/sandbox/sandbox_provider.py`

```python
from abc import ABC, abstractmethod

from src.config import get_app_config
from src.reflection import resolve_class
from src.sandbox.sandbox import Sandbox


class SandboxProvider(ABC):
    """Abstract base class for sandbox providers"""

    @abstractmethod
    def acquire(self, thread_id: str | None = None) -> str:
        """Acquire a sandbox environment and return its ID.

        Returns:
            The ID of the acquired sandbox environment.
        """
        pass

    @abstractmethod
    def get(self, sandbox_id: str) -> Sandbox | None:
        """Get a sandbox environment by ID.

        Args:
            sandbox_id: The ID of the sandbox environment to retain.
        """
        pass

    @abstractmethod
    def release(self, sandbox_id: str) -> None:
        """Release a sandbox environment.

        Args:
            sandbox_id: The ID of the sandbox environment to destroy.
        """
        pass
```

**核心方法说明**：

| 方法 | 功能 | 返回值 |
|------|------|--------|
| `acquire` | 获取一个沙箱实例 | 沙箱 ID 字符串 |
| `get` | 根据 ID 获取沙箱实例 | `Sandbox` 实例或 `None` |
| `release` | 释放沙箱实例 | 无 |

**单例管理函数**：

```python
_default_sandbox_provider: SandboxProvider | None = None


def get_sandbox_provider(**kwargs) -> SandboxProvider:
    """Get the sandbox provider singleton.

    Returns a cached singleton instance. Use `reset_sandbox_provider()` to clear
    the cache, or `shutdown_sandbox_provider()` to properly shutdown and clear.

    Returns:
        A sandbox provider instance.
    """
    global _default_sandbox_provider
    if _default_sandbox_provider is None:
        config = get_app_config()
        cls = resolve_class(config.sandbox.use, SandboxProvider)
        _default_sandbox_provider = cls(**kwargs)
    return _default_sandbox_provider


def reset_sandbox_provider() -> None:
    """Reset the sandbox provider singleton.

    This clears the cached instance without calling shutdown.
    The next call to `get_sandbox_provider()` will create a new instance.
    Useful for testing or when switching configurations.

    Note: If the provider has active sandboxes, they will be orphaned.
    Use `shutdown_sandbox_provider()` for proper cleanup.
    """
    global _default_sandbox_provider
    _default_sandbox_provider = None


def shutdown_sandbox_provider() -> None:
    """Shutdown and reset the sandbox provider.

    This properly shuts down the provider (releasing all sandboxes)
    before clearing the singleton. Call this when the application
    is shutting down or when you need to completely reset the sandbox system.
    """
    global _default_sandbox_provider
    if _default_sandbox_provider is not None:
        if hasattr(_default_sandbox_provider, "shutdown"):
            _default_sandbox_provider.shutdown()
        _default_sandbox_provider = None


def set_sandbox_provider(provider: SandboxProvider) -> None:
    """Set a custom sandbox provider instance.

    This allows injecting a custom or mock provider for testing purposes.

    Args:
        provider: The SandboxProvider instance to use.
    """
    global _default_sandbox_provider
    _default_sandbox_provider = provider
```

---

### LocalSandbox 本地沙箱实现

`LocalSandbox` 是 `Sandbox` 的本地实现，直接在主机上执行命令和文件操作，支持路径映射。

**文件位置**：`src/sandbox/local/local_sandbox.py`

#### 初始化与路径映射

```python
class LocalSandbox(Sandbox):
    def __init__(self, id: str, path_mappings: dict[str, str] | None = None):
        """
        Initialize local sandbox with optional path mappings.

        Args:
            id: Sandbox identifier
            path_mappings: Dictionary mapping container paths to local paths
                          Example: {"/mnt/skills": "/absolute/path/to/skills"}
        """
        super().__init__(id)
        self.path_mappings = path_mappings or {}
```

#### 路径解析方法

```python
    def _resolve_path(self, path: str) -> str:
        """
        Resolve container path to actual local path using mappings.

        Args:
            path: Path that might be a container path

        Returns:
            Resolved local path
        """
        path_str = str(path)

        # Try each mapping (longest prefix first for more specific matches)
        for container_path, local_path in sorted(self.path_mappings.items(), key=lambda x: len(x[0]), reverse=True):
            if path_str.startswith(container_path):
                # Replace the container path prefix with local path
                relative = path_str[len(container_path) :].lstrip("/")
                resolved = str(Path(local_path) / relative) if relative else local_path
                return resolved

        # No mapping found, return original path
        return path_str

    def _reverse_resolve_path(self, path: str) -> str:
        """
        Reverse resolve local path back to container path using mappings.

        Args:
            path: Local path that might need to be mapped to container path

        Returns:
            Container path if mapping exists, otherwise original path
        """
        path_str = str(Path(path).resolve())

        # Try each mapping (longest local path first for more specific matches)
        for container_path, local_path in sorted(self.path_mappings.items(), key=lambda x: len(x[1]), reverse=True):
            local_path_resolved = str(Path(local_path).resolve())
            if path_str.startswith(local_path_resolved):
                # Replace the local path prefix with container path
                relative = path_str[len(local_path_resolved) :].lstrip("/")
                resolved = f"{container_path}/{relative}" if relative else container_path
                return resolved

        # No mapping found, return original path
        return path_str
```

#### Shell 检测与命令执行

```python
    @staticmethod
    def _get_shell() -> str:
        """Detect available shell executable with fallback.

        Returns the first available shell in order of preference:
        /bin/zsh → /bin/bash → /bin/sh → first `sh` found on PATH.
        Raises a RuntimeError if no suitable shell is found.
        """
        for shell in ("/bin/zsh", "/bin/bash", "/bin/sh"):
            if os.path.isfile(shell) and os.access(shell, os.X_OK):
                return shell
        shell_from_path = shutil.which("sh")
        if shell_from_path is not None:
            return shell_from_path
        raise RuntimeError("No suitable shell executable found. Tried /bin/zsh, /bin/bash, /bin/sh, and `sh` on PATH.")

    def execute_command(self, command: str) -> str:
        # Resolve container paths in command before execution
        resolved_command = self._resolve_paths_in_command(command)

        result = subprocess.run(
            resolved_command,
            executable=self._get_shell(),
            shell=True,
            capture_output=True,
            text=True,
            timeout=600,
        )
        output = result.stdout
        if result.stderr:
            output += f"\nStd Error:\n{result.stderr}" if output else result.stderr
        if result.returncode != 0:
            output += f"\nExit Code: {result.returncode}"

        final_output = output if output else "(no output)"
        # Reverse resolve local paths back to container paths in output
        return self._reverse_resolve_paths_in_output(final_output)
```

#### 文件操作方法

```python
    def read_file(self, path: str) -> str:
        resolved_path = self._resolve_path(path)
        try:
            with open(resolved_path) as f:
                return f.read()
        except OSError as e:
            # Re-raise with the original path for clearer error messages
            raise type(e)(e.errno, e.strerror, path) from None

    def write_file(self, path: str, content: str, append: bool = False) -> None:
        resolved_path = self._resolve_path(path)
        try:
            dir_path = os.path.dirname(resolved_path)
            if dir_path:
                os.makedirs(dir_path, exist_ok=True)
            mode = "a" if append else "w"
            with open(resolved_path, mode) as f:
                f.write(content)
        except OSError as e:
            # Re-raise with the original path for clearer error messages
            raise type(e)(e.errno, e.strerror, path) from None

    def update_file(self, path: str, content: bytes) -> None:
        resolved_path = self._resolve_path(path)
        try:
            dir_path = os.path.dirname(resolved_path)
            if dir_path:
                os.makedirs(dir_path, exist_ok=True)
            with open(resolved_path, "wb") as f:
                f.write(content)
        except OSError as e:
            # Re-raise with the original path for clearer error messages
            raise type(e)(e.errno, e.strerror, path) from None
```

---

### LocalSandboxProvider 本地沙箱提供者

`LocalSandboxProvider` 实现 `SandboxProvider`，使用单例模式管理唯一的 `LocalSandbox` 实例。

**文件位置**：`src/sandbox/local/local_sandbox_provider.py`

```python
from src.sandbox.local.local_sandbox import LocalSandbox
from src.sandbox.sandbox import Sandbox
from src.sandbox.sandbox_provider import SandboxProvider

_singleton: LocalSandbox | None = None


class LocalSandboxProvider(SandboxProvider):
    def __init__(self):
        """Initialize the local sandbox provider with path mappings."""
        self._path_mappings = self._setup_path_mappings()

    def _setup_path_mappings(self) -> dict[str, str]:
        """
        Setup path mappings for local sandbox.

        Maps container paths to actual local paths, including skills directory.

        Returns:
            Dictionary of path mappings
        """
        mappings = {}

        # Map skills container path to local skills directory
        try:
            from src.config import get_app_config

            config = get_app_config()
            skills_path = config.skills.get_skills_path()
            container_path = config.skills.container_path

            # Only add mapping if skills directory exists
            if skills_path.exists():
                mappings[container_path] = str(skills_path)
        except Exception as e:
            # Log but don't fail if config loading fails
            print(f"Warning: Could not setup skills path mapping: {e}")

        return mappings

    def acquire(self, thread_id: str | None = None) -> str:
        global _singleton
        if _singleton is None:
            _singleton = LocalSandbox("local", path_mappings=self._path_mappings)
        return _singleton.id

    def get(self, sandbox_id: str) -> Sandbox | None:
        if sandbox_id == "local":
            if _singleton is None:
                self.acquire()
            return _singleton
        return None

    def release(self, sandbox_id: str) -> None:
        # LocalSandbox uses singleton pattern - no cleanup needed.
        # Note: This method is intentionally not called by SandboxMiddleware
        # to allow sandbox reuse across multiple turns in a thread.
        # For Docker-based providers (e.g., AioSandboxProvider), cleanup
        # happens at application shutdown via the shutdown() method.
        pass
```

**设计特点**：

1. **单例模式**：全局只有一个 `LocalSandbox` 实例
2. **延迟初始化**：在首次调用 `acquire` 时创建实例
3. **路径映射**：自动配置 skills 目录的路径映射
4. **无需释放**：`release` 方法为空操作，支持跨会话重用

---

## 核心功能

### 虚拟路径映射机制

DeerFlow 使用虚拟路径机制，让 Agent 在沙箱内看到统一的路径，而实际映射到不同的文件系统位置。

#### 虚拟路径前缀

**定义位置**：`src/config/paths.py`

```python
# Virtual path prefix seen by agents inside the sandbox
VIRTUAL_PATH_PREFIX = "/mnt/user-data"
```

#### 路径映射规则

```
虚拟路径                              实际主机路径
─────────────────────────────────────────────────────────────────
/mnt/user-data/workspace/*    →    {base_dir}/threads/{thread_id}/user-data/workspace/*
/mnt/user-data/uploads/*      →    {base_dir}/threads/{thread_id}/user-data/uploads/*
/mnt/user-data/outputs/*      →    {base_dir}/threads/{thread_id}/user-data/outputs/*
```

#### 虚拟路径替换函数

**文件位置**：`src/sandbox/tools.py`

```python
def replace_virtual_path(path: str, thread_data: ThreadDataState | None) -> str:
    """Replace virtual /mnt/user-data paths with actual thread data paths.

    Mapping:
        /mnt/user-data/workspace/* -> thread_data['workspace_path']/*
        /mnt/user-data/uploads/* -> thread_data['uploads_path']*
        /mnt/user-data/outputs/* -> thread_data['outputs_path']*

    Args:
        path: The path that may contain virtual path prefix.
        thread_data: The thread data containing actual paths.

    Returns:
        The path with virtual prefix replaced by actual path.
    """
    if not path.startswith(VIRTUAL_PATH_PREFIX):
        return path

    if thread_data is None:
        return path

    # Map virtual subdirectories to thread_data keys
    path_mapping = {
        "workspace": thread_data.get("workspace_path"),
        "uploads": thread_data.get("uploads_path"),
        "outputs": thread_data.get("outputs_path"),
    }

    # Extract the subdirectory after /mnt/user-data/
    relative_path = path[len(VIRTUAL_PATH_PREFIX) :].lstrip("/")
    if not relative_path:
        return path

    # Find which subdirectory this path belongs to
    parts = relative_path.split("/", 1)
    subdir = parts[0]
    rest = parts[1] if len(parts) > 1 else ""

    actual_base = path_mapping.get(subdir)
    if actual_base is None:
        return path

    if rest:
        return f"{actual_base}/{rest}"
    return actual_base
```

#### 命令中的路径替换

```python
def replace_virtual_paths_in_command(command: str, thread_data: ThreadDataState | None) -> str:
    """Replace all virtual /mnt/user-data paths in a command string.

    Args:
        command: The command string that may contain virtual paths.
        thread_data: The thread data containing actual paths.

    Returns:
        The command with all virtual paths replaced.
    """
    if VIRTUAL_PATH_PREFIX not in command:
        return command

    if thread_data is None:
        return command

    # Pattern to match /mnt/user-data followed by path characters
    pattern = re.compile(rf"{re.escape(VIRTUAL_PATH_PREFIX)}(/[^\s\"';&|<>()]*)?")

    def replace_match(match: re.Match) -> str:
        full_path = match.group(0)
        return replace_virtual_path(full_path, thread_data)

    return pattern.sub(replace_match, command)
```

---

### 命令执行流程

```
┌─────────────────────────────────────────────────────────────────────┐
│                       命令执行流程                                    │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  1. Agent 调用 bash_tool                                           │
│     └── command: "ls /mnt/user-data/workspace"                     │
│                                                                     │
│  2. ensure_sandbox_initialized()                                   │
│     └── 获取或创建沙箱实例                                           │
│                                                                     │
│  3. ensure_thread_directories_exist()                              │
│     └── 确保线程目录存在                                             │
│                                                                     │
│  4. is_local_sandbox() 检查                                        │
│     └── 本地沙箱需要路径替换                                          │
│                                                                     │
│  5. replace_virtual_paths_in_command()                             │
│     └── "/mnt/user-data/workspace" → "/home/user/.deer-flow/..."   │
│                                                                     │
│  6. sandbox.execute_command()                                      │
│     └── 在本地 shell 中执行命令                                       │
│                                                                     │
│  7. 输出路径反向替换                                                 │
│     └── 将输出中的实际路径替换回虚拟路径                               │
│                                                                     │
│  8. 返回结果给 Agent                                                │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

### 文件操作

#### 读取文件

```python
@tool("read_file", parse_docstring=True)
def read_file_tool(
    runtime: ToolRuntime[ContextT, ThreadState],
    description: str,
    path: str,
    start_line: int | None = None,
    end_line: int | None = None,
) -> str:
    """Read the contents of a text file.

    Args:
        description: Explain why you are reading this file in short words.
        path: The **absolute** path to the file to read.
        start_line: Optional starting line number (1-indexed, inclusive).
        end_line: Optional ending line number (1-indexed, inclusive).
    """
    try:
        sandbox = ensure_sandbox_initialized(runtime)
        ensure_thread_directories_exist(runtime)
        if is_local_sandbox(runtime):
            thread_data = get_thread_data(runtime)
            path = replace_virtual_path(path, thread_data)
        content = sandbox.read_file(path)
        if not content:
            return "(empty)"
        if start_line is not None and end_line is not None:
            content = "\n".join(content.splitlines()[start_line - 1 : end_line])
        return content
    except SandboxError as e:
        return f"Error: {e}"
    # ... 其他异常处理
```

#### 写入文件

```python
@tool("write_file", parse_docstring=True)
def write_file_tool(
    runtime: ToolRuntime[ContextT, ThreadState],
    description: str,
    path: str,
    content: str,
    append: bool = False,
) -> str:
    """Write text content to a file.

    Args:
        description: Explain why you are writing to this file in short words.
        path: The **absolute** path to the file to write to.
        content: The content to write to the file.
    """
    try:
        sandbox = ensure_sandbox_initialized(runtime)
        ensure_thread_directories_exist(runtime)
        if is_local_sandbox(runtime):
            thread_data = get_thread_data(runtime)
            path = replace_virtual_path(path, thread_data)
        sandbox.write_file(path, content, append)
        return "OK"
    except SandboxError as e:
        return f"Error: {e}"
    # ... 其他异常处理
```

#### 字符串替换

```python
@tool("str_replace", parse_docstring=True)
def str_replace_tool(
    runtime: ToolRuntime[ContextT, ThreadState],
    description: str,
    path: str,
    old_str: str,
    new_str: str,
    replace_all: bool = False,
) -> str:
    """Replace a substring in a file with another substring.

    Args:
        description: Explain why you are replacing the substring.
        path: The **absolute** path to the file.
        old_str: The substring to replace.
        new_str: The new substring.
        replace_all: Whether to replace all occurrences. Default is False.
    """
    try:
        sandbox = ensure_sandbox_initialized(runtime)
        ensure_thread_directories_exist(runtime)
        if is_local_sandbox(runtime):
            thread_data = get_thread_data(runtime)
            path = replace_virtual_path(path, thread_data)
        content = sandbox.read_file(path)
        if not content:
            return "OK"
        if old_str not in content:
            return f"Error: String to replace not found in file: {path}"
        if replace_all:
            content = content.replace(old_str, new_str)
        else:
            content = content.replace(old_str, new_str, 1)
        sandbox.write_file(path, content)
        return "OK"
    except SandboxError as e:
        return f"Error: {e}"
    # ... 其他异常处理
```

---

### 目录操作（list_dir）

**文件位置**：`src/sandbox/local/list_dir.py`

```python
IGNORE_PATTERNS = [
    # Version Control
    ".git", ".svn", ".hg", ".bzr",
    # Dependencies
    "node_modules", "__pycache__", ".venv", "venv", ".env", "env",
    ".tox", ".nox", ".eggs", "*.egg-info", "site-packages",
    # Build outputs
    "dist", "build", ".next", ".nuxt", ".output", ".turbo", "target", "out",
    # IDE & Editor
    ".idea", ".vscode", "*.swp", "*.swo", "*~",
    ".project", ".classpath", ".settings",
    # OS generated
    ".DS_Store", "Thumbs.db", "desktop.ini", "*.lnk",
    # Logs & temp files
    "*.log", "*.tmp", "*.temp", "*.bak", "*.cache", ".cache", "logs",
    # Coverage & test artifacts
    ".coverage", "coverage", ".nyc_output", "htmlcov",
    ".pytest_cache", ".mypy_cache", ".ruff_cache",
]


def _should_ignore(name: str) -> bool:
    """Check if a file/directory name matches any ignore pattern."""
    for pattern in IGNORE_PATTERNS:
        if fnmatch.fnmatch(name, pattern):
            return True
    return False


def list_dir(path: str, max_depth: int = 2) -> list[str]:
    """
    List files and directories up to max_depth levels deep.

    Args:
        path: The root directory path to list.
        max_depth: Maximum depth to traverse (default: 2).
                   1 = only direct children, 2 = children + grandchildren, etc.

    Returns:
        A list of absolute paths for files and directories,
        excluding items matching IGNORE_PATTERNS.
    """
    result: list[str] = []
    root_path = Path(path).resolve()

    if not root_path.is_dir():
        return result

    def _traverse(current_path: Path, current_depth: int) -> None:
        """Recursively traverse directories up to max_depth."""
        if current_depth > max_depth:
            return

        try:
            for item in current_path.iterdir():
                if _should_ignore(item.name):
                    continue

                post_fix = "/" if item.is_dir() else ""
                result.append(str(item.resolve()) + post_fix)

                # Recurse into subdirectories if not at max depth
                if item.is_dir() and current_depth < max_depth:
                    _traverse(item, current_depth + 1)
        except PermissionError:
            pass

    _traverse(root_path, 1)

    return sorted(result)
```

**忽略模式说明**：

| 类别 | 示例 |
|------|------|
| 版本控制 | `.git`, `.svn`, `.hg`, `.bzr` |
| 依赖目录 | `node_modules`, `__pycache__`, `.venv`, `venv` |
| 构建输出 | `dist`, `build`, `.next`, `target` |
| IDE 配置 | `.idea`, `.vscode`, `*.swp` |
| 系统文件 | `.DS_Store`, `Thumbs.db` |
| 测试覆盖 | `.coverage`, `.pytest_cache`, `.mypy_cache` |

---

## 工具函数

### bash_tool

执行 bash 命令的核心工具。

```python
@tool("bash", parse_docstring=True)
def bash_tool(runtime: ToolRuntime[ContextT, ThreadState], description: str, command: str) -> str:
    """Execute a bash command in a Linux environment.

    - Use `python` to run Python code.
    - Use `pip install` to install Python packages.

    Args:
        description: Explain why you are running this command in short words.
        command: The bash command to execute. Always use absolute paths.
    """
    try:
        sandbox = ensure_sandbox_initialized(runtime)
        ensure_thread_directories_exist(runtime)
        if is_local_sandbox(runtime):
            thread_data = get_thread_data(runtime)
            command = replace_virtual_paths_in_command(command, thread_data)
        return sandbox.execute_command(command)
    except SandboxError as e:
        return f"Error: {e}"
    except Exception as e:
        return f"Error: Unexpected error executing command: {type(e).__name__}: {e}"
```

### read_file_tool

读取文本文件内容。

```python
@tool("read_file", parse_docstring=True)
def read_file_tool(
    runtime: ToolRuntime[ContextT, ThreadState],
    description: str,
    path: str,
    start_line: int | None = None,
    end_line: int | None = None,
) -> str:
    """Read the contents of a text file."""
    # 实现见上文"文件操作"部分
```

### write_file_tool

写入文本文件。

```python
@tool("write_file", parse_docstring=True)
def write_file_tool(
    runtime: ToolRuntime[ContextT, ThreadState],
    description: str,
    path: str,
    content: str,
    append: bool = False,
) -> str:
    """Write text content to a file."""
    # 实现见上文"文件操作"部分
```

### str_replace_tool

文件内字符串替换。

```python
@tool("str_replace", parse_docstring=True)
def str_replace_tool(
    runtime: ToolRuntime[ContextT, ThreadState],
    description: str,
    path: str,
    old_str: str,
    new_str: str,
    replace_all: bool = False,
) -> str:
    """Replace a substring in a file with another substring."""
    # 实现见上文"文件操作"部分
```

### ls_tool

列出目录内容。

```python
@tool("ls", parse_docstring=True)
def ls_tool(runtime: ToolRuntime[ContextT, ThreadState], description: str, path: str) -> str:
    """List the contents of a directory up to 2 levels deep in tree format.

    Args:
        description: Explain why you are listing this directory in short words.
        path: The **absolute** path to the directory to list.
    """
    try:
        sandbox = ensure_sandbox_initialized(runtime)
        ensure_thread_directories_exist(runtime)
        if is_local_sandbox(runtime):
            thread_data = get_thread_data(runtime)
            path = replace_virtual_path(path, thread_data)
        children = sandbox.list_dir(path)
        if not children:
            return "(empty)"
        return "\n".join(children)
    except SandboxError as e:
        return f"Error: {e}"
    except FileNotFoundError:
        return f"Error: Directory not found: {path}"
    except PermissionError:
        return f"Error: Permission denied: {path}"
    except Exception as e:
        return f"Error: Unexpected error listing directory: {type(e).__name__}: {e}"
```

---

## 懒加载机制

### ensure_sandbox_initialized()

确保沙箱已初始化，支持延迟获取。

```python
def ensure_sandbox_initialized(runtime: ToolRuntime[ContextT, ThreadState] | None = None) -> Sandbox:
    """Ensure sandbox is initialized, acquiring lazily if needed.

    On first call, acquires a sandbox from the provider and stores it in runtime state.
    Subsequent calls return the existing sandbox.

    Thread-safety is guaranteed by the provider's internal locking mechanism.

    Args:
        runtime: Tool runtime containing state and context.

    Returns:
        Initialized sandbox instance.

    Raises:
        SandboxRuntimeError: If runtime is not available or thread_id is missing.
        SandboxNotFoundError: If sandbox acquisition fails.
    """
    if runtime is None:
        raise SandboxRuntimeError("Tool runtime not available")

    if runtime.state is None:
        raise SandboxRuntimeError("Tool runtime state not available")

    # Check if sandbox already exists in state
    sandbox_state = runtime.state.get("sandbox")
    if sandbox_state is not None:
        sandbox_id = sandbox_state.get("sandbox_id")
        if sandbox_id is not None:
            sandbox = get_sandbox_provider().get(sandbox_id)
            if sandbox is not None:
                runtime.context["sandbox_id"] = sandbox_id  # Ensure sandbox_id is in context for releasing
                return sandbox
            # Sandbox was released, fall through to acquire new one

    # Lazy acquisition: get thread_id and acquire sandbox
    thread_id = runtime.context.get("thread_id")
    if thread_id is None:
        raise SandboxRuntimeError("Thread ID not available in runtime context")

    provider = get_sandbox_provider()
    sandbox_id = provider.acquire(thread_id)

    # Update runtime state - this persists across tool calls
    runtime.state["sandbox"] = {"sandbox_id": sandbox_id}

    # Retrieve and return the sandbox
    sandbox = provider.get(sandbox_id)
    if sandbox is None:
        raise SandboxNotFoundError("Sandbox not found after acquisition", sandbox_id=sandbox_id)

    runtime.context["sandbox_id"] = sandbox_id  # Ensure sandbox_id is in context for releasing
    return sandbox
```

**懒加载流程**：

```
┌─────────────────────────────────────────────────────────────────────┐
│                    懒加载初始化流程                                   │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  第一次调用工具                                                      │
│  ────────────────                                                   │
│  1. runtime.state["sandbox"] 不存在                                 │
│  2. 从 runtime.context 获取 thread_id                               │
│  3. 调用 provider.acquire(thread_id) 获取沙箱                        │
│  4. 将 sandbox_id 存入 runtime.state["sandbox"]                     │
│  5. 返回沙箱实例                                                     │
│                                                                     │
│  后续调用工具                                                        │
│  ────────────────                                                   │
│  1. runtime.state["sandbox"] 已存在                                 │
│  2. 从 state 获取 sandbox_id                                        │
│  3. 调用 provider.get(sandbox_id) 获取沙箱                          │
│  4. 直接返回沙箱实例                                                 │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### ensure_thread_directories_exist()

确保线程数据目录存在。

```python
def ensure_thread_directories_exist(runtime: ToolRuntime[ContextT, ThreadState] | None) -> None:
    """Ensure thread data directories (workspace, uploads, outputs) exist.

    This function is called lazily when any sandbox tool is first used.
    For local sandbox, it creates the directories on the filesystem.
    For other sandboxes (like aio), directories are already mounted in the container.

    Args:
        runtime: Tool runtime containing state and context.
    """
    if runtime is None:
        return

    # Only create directories for local sandbox
    if not is_local_sandbox(runtime):
        return

    thread_data = get_thread_data(runtime)
    if thread_data is None:
        return

    # Check if directories have already been created
    if runtime.state.get("thread_directories_created"):
        return

    # Create the three directories
    import os

    for key in ["workspace_path", "uploads_path", "outputs_path"]:
        path = thread_data.get(key)
        if path:
            os.makedirs(path, exist_ok=True)

    # Mark as created to avoid redundant operations
    runtime.state["thread_directories_created"] = True
```

---

## 中间件

### SandboxMiddleware 实现

`SandboxMiddleware` 负责管理沙箱的生命周期，与 Agent 执行流程集成。

**文件位置**：`src/sandbox/middleware.py`

```python
import logging
from typing import NotRequired, override

from langchain.agents import AgentState
from langchain.agents.middleware import AgentMiddleware
from langgraph.runtime import Runtime

from src.agents.thread_state import SandboxState, ThreadDataState
from src.sandbox import get_sandbox_provider

logger = logging.getLogger(__name__)


class SandboxMiddlewareState(AgentState):
    """Compatible with the `ThreadState` schema."""

    sandbox: NotRequired[SandboxState | None]
    thread_data: NotRequired[ThreadDataState | None]


class SandboxMiddleware(AgentMiddleware[SandboxMiddlewareState]):
    """Create a sandbox environment and assign it to an agent.

    Lifecycle Management:
    - With lazy_init=True (default): Sandbox is acquired on first tool call
    - With lazy_init=False: Sandbox is acquired on first agent invocation (before_agent)
    - Sandbox is reused across multiple turns within the same thread
    - Sandbox is NOT released after each agent call to avoid wasteful recreation
    - Cleanup happens at application shutdown via SandboxProvider.shutdown()
    """

    state_schema = SandboxMiddlewareState

    def __init__(self, lazy_init: bool = True):
        """Initialize sandbox middleware.

        Args:
            lazy_init: If True, defer sandbox acquisition until first tool call.
                      If False, acquire sandbox eagerly in before_agent().
                      Default is True for optimal performance.
        """
        super().__init__()
        self._lazy_init = lazy_init

    def _acquire_sandbox(self, thread_id: str) -> str:
        provider = get_sandbox_provider()
        sandbox_id = provider.acquire(thread_id)
        logger.info(f"Acquiring sandbox {sandbox_id}")
        return sandbox_id

    @override
    def before_agent(self, state: SandboxMiddlewareState, runtime: Runtime) -> dict | None:
        # Skip acquisition if lazy_init is enabled
        if self._lazy_init:
            return super().before_agent(state, runtime)

        # Eager initialization (original behavior)
        if "sandbox" not in state or state["sandbox"] is None:
            thread_id = runtime.context["thread_id"]
            sandbox_id = self._acquire_sandbox(thread_id)
            logger.info(f"Assigned sandbox {sandbox_id} to thread {thread_id}")
            return {"sandbox": {"sandbox_id": sandbox_id}}
        return super().before_agent(state, runtime)

    @override
    def after_agent(self, state: SandboxMiddlewareState, runtime: Runtime) -> dict | None:
        sandbox = state.get("sandbox")
        if sandbox is not None:
            sandbox_id = sandbox["sandbox_id"]
            logger.info(f"Releasing sandbox {sandbox_id}")
            get_sandbox_provider().release(sandbox_id)
            return None

        if runtime.context.get("sandbox_id") is not None:
            sandbox_id = runtime.context.get("sandbox_id")
            logger.info(f"Releasing sandbox {sandbox_id} from context")
            get_sandbox_provider().release(sandbox_id)
            return None

        # No sandbox to release
        return super().after_agent(state, runtime)
```

**生命周期图示**：

```
┌─────────────────────────────────────────────────────────────────────┐
│                    SandboxMiddleware 生命周期                        │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  lazy_init=True (默认，延迟初始化)                                    │
│  ─────────────────────────────────                                  │
│                                                                     │
│  before_agent()                                                     │
│  └── 直接返回，不做任何操作                                           │
│                                                                     │
│  Tool 执行时                                                         │
│  └── ensure_sandbox_initialized() 首次获取沙箱                       │
│                                                                     │
│  after_agent()                                                      │
│  └── 释放沙箱（对于 LocalSandboxProvider 为空操作）                    │
│                                                                     │
│  ─────────────────────────────────────────────────────────────────  │
│                                                                     │
│  lazy_init=False (立即初始化)                                        │
│  ─────────────────────────                                          │
│                                                                     │
│  before_agent()                                                     │
│  └── 调用 _acquire_sandbox() 获取沙箱                                │
│  └── 设置 state["sandbox"] = {"sandbox_id": ...}                    │
│                                                                     │
│  Tool 执行时                                                         │
│  └── 使用已存在的沙箱                                                 │
│                                                                     │
│  after_agent()                                                      │
│  └── 释放沙箱                                                        │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 配置项

### SandboxConfig 配置类

**文件位置**：`src/config/sandbox_config.py`

```python
from pydantic import BaseModel, ConfigDict, Field


class VolumeMountConfig(BaseModel):
    """Configuration for a volume mount."""

    host_path: str = Field(..., description="Path on the host machine")
    container_path: str = Field(..., description="Path inside the container")
    read_only: bool = Field(default=False, description="Whether the mount is read-only")


class SandboxConfig(BaseModel):
    """Config section for a sandbox.

    Common options:
        use: Class path of the sandbox provider (required)

    AioSandboxProvider specific options:
        image: Docker image to use
        port: Base port for sandbox containers
        replicas: Maximum number of concurrent sandbox containers
        container_prefix: Prefix for container names
        idle_timeout: Idle timeout in seconds before sandbox is released
        mounts: List of volume mounts to share directories with the container
        environment: Environment variables to inject into the container
    """

    use: str = Field(
        ...,
        description="Class path of the sandbox provider (e.g. src.sandbox.local:LocalSandboxProvider)",
    )
    image: str | None = Field(
        default=None,
        description="Docker image to use for the sandbox container",
    )
    port: int | None = Field(
        default=None,
        description="Base port for sandbox containers",
    )
    replicas: int | None = Field(
        default=None,
        description="Maximum number of concurrent sandbox containers",
    )
    container_prefix: str | None = Field(
        default=None,
        description="Prefix for container names",
    )
    idle_timeout: int | None = Field(
        default=None,
        description="Idle timeout in seconds before sandbox is released",
    )
    mounts: list[VolumeMountConfig] = Field(
        default_factory=list,
        description="List of volume mounts to share directories between host and container",
    )
    environment: dict[str, str] = Field(
        default_factory=dict,
        description="Environment variables to inject into the sandbox container",
    )

    model_config = ConfigDict(extra="allow")
```

### 配置示例

**config.yaml 示例**：

```yaml
# 本地沙箱配置
sandbox:
  use: src.sandbox.local:LocalSandboxProvider

# AioSandboxProvider (Docker) 配置示例
# sandbox:
#   use: src.community.aio_sandbox:LocalSandboxProvider
#   image: enterprise-public-cn-beijing.cr.volces.com/vefaas-public/all-in-one-sandbox:latest
#   port: 8080
#   replicas: 3
#   container_prefix: deer-flow-sandbox
#   idle_timeout: 600
#   mounts:
#     - host_path: /path/to/host/dir
#       container_path: /mnt/data
#       read_only: false
#   environment:
#     CUSTOM_VAR: value
#     FROM_ENV: $HOST_ENV_VAR
```

### 路径配置

**文件位置**：`src/config/paths.py`

```python
class Paths:
    """
    Centralized path configuration for DeerFlow application data.

    Directory layout (host side):
        {base_dir}/
        ├── memory.json
        ├── USER.md          <-- global user profile
        ├── agents/
        │   └── {agent_name}/
        │       ├── config.yaml
        │       ├── SOUL.md
        │       └── memory.json
        └── threads/
            └── {thread_id}/
                └── user-data/         <-- mounted as /mnt/user-data/ inside sandbox
                    ├── workspace/     <-- /mnt/user-data/workspace/
                    ├── uploads/       <-- /mnt/user-data/uploads/
                    └── outputs/       <-- /mnt/user-data/outputs/
    """

    def sandbox_work_dir(self, thread_id: str) -> Path:
        """Host path for the agent's workspace directory."""
        return self.thread_dir(thread_id) / "user-data" / "workspace"

    def sandbox_uploads_dir(self, thread_id: str) -> Path:
        """Host path for user-uploaded files."""
        return self.thread_dir(thread_id) / "user-data" / "uploads"

    def sandbox_outputs_dir(self, thread_id: str) -> Path:
        """Host path for agent-generated artifacts."""
        return self.thread_dir(thread_id) / "user-data" / "outputs"

    def resolve_virtual_path(self, thread_id: str, virtual_path: str) -> Path:
        """Resolve a sandbox virtual path to the actual host filesystem path."""
        # 实现见上文
```

---

## 异常处理

### 自定义异常类

**文件位置**：`src/sandbox/exceptions.py`

```python
"""Sandbox-related exceptions with structured error information."""


class SandboxError(Exception):
    """Base exception for all sandbox-related errors."""

    def __init__(self, message: str, details: dict | None = None):
        super().__init__(message)
        self.message = message
        self.details = details or {}

    def __str__(self) -> str:
        if self.details:
            detail_str = ", ".join(f"{k}={v}" for k, v in self.details.items())
            return f"{self.message} ({detail_str})"
        return self.message


class SandboxNotFoundError(SandboxError):
    """Raised when a sandbox cannot be found or is not available."""

    def __init__(self, message: str = "Sandbox not found", sandbox_id: str | None = None):
        details = {"sandbox_id": sandbox_id} if sandbox_id else None
        super().__init__(message, details)
        self.sandbox_id = sandbox_id


class SandboxRuntimeError(SandboxError):
    """Raised when sandbox runtime is not available or misconfigured."""

    pass


class SandboxCommandError(SandboxError):
    """Raised when a command execution fails in the sandbox."""

    def __init__(self, message: str, command: str | None = None, exit_code: int | None = None):
        details = {}
        if command:
            details["command"] = command[:100] + "..." if len(command) > 100 else command
        if exit_code is not None:
            details["exit_code"] = exit_code
        super().__init__(message, details)
        self.command = command
        self.exit_code = exit_code


class SandboxFileError(SandboxError):
    """Raised when a file operation fails in the sandbox."""

    def __init__(self, message: str, path: str | None = None, operation: str | None = None):
        details = {}
        if path:
            details["path"] = path
        if operation:
            details["operation"] = operation
        super().__init__(message, details)
        self.path = path
        self.operation = operation


class SandboxPermissionError(SandboxFileError):
    """Raised when a permission error occurs during file operations."""

    pass


class SandboxFileNotFoundError(SandboxFileError):
    """Raised when a file or directory is not found."""

    pass
```

### 异常类层次结构

```
SandboxError (基类)
├── SandboxNotFoundError        # 沙箱未找到
├── SandboxRuntimeError         # 运行时错误
├── SandboxCommandError         # 命令执行错误
└── SandboxFileError            # 文件操作错误
    ├── SandboxPermissionError  # 权限错误
    └── SandboxFileNotFoundError # 文件未找到
```

### 异常使用示例

```python
from src.sandbox.exceptions import (
    SandboxError,
    SandboxNotFoundError,
    SandboxRuntimeError,
    SandboxCommandError,
    SandboxFileError,
)

# 沙箱未找到
raise SandboxNotFoundError("Sandbox not found", sandbox_id="local")

# 运行时错误
raise SandboxRuntimeError("Tool runtime not available")

# 命令执行错误
raise SandboxCommandError("Command failed", command="ls -la", exit_code=1)

# 文件操作错误
raise SandboxFileError("Cannot read file", path="/path/to/file", operation="read")
```

---

## 设计模式总结

### 1. 抽象工厂模式（Abstract Factory）

`SandboxProvider` 作为抽象工厂，负责创建 `Sandbox` 产品：

```python
# 抽象工厂
class SandboxProvider(ABC):
    @abstractmethod
    def acquire(self, thread_id: str | None = None) -> str: ...

# 具体工厂
class LocalSandboxProvider(SandboxProvider):
    def acquire(self, thread_id: str | None = None) -> str:
        return LocalSandbox("local", ...).id
```

### 2. 单例模式（Singleton）

全局唯一的沙箱提供者实例：

```python
_default_sandbox_provider: SandboxProvider | None = None

def get_sandbox_provider() -> SandboxProvider:
    global _default_sandbox_provider
    if _default_sandbox_provider is None:
        config = get_app_config()
        cls = resolve_class(config.sandbox.use, SandboxProvider)
        _default_sandbox_provider = cls()
    return _default_sandbox_provider
```

`LocalSandbox` 本身也是单例：

```python
_singleton: LocalSandbox | None = None

class LocalSandboxProvider(SandboxProvider):
    def acquire(self, thread_id: str | None = None) -> str:
        global _singleton
        if _singleton is None:
            _singleton = LocalSandbox("local", ...)
        return _singleton.id
```

### 3. 策略模式（Strategy）

不同的沙箱实现作为可互换的策略：

```python
# 配置决定使用哪种策略
sandbox:
  use: src.sandbox.local:LocalSandboxProvider  # 本地策略
  # use: src.community.aio_sandbox:AioSandboxProvider  # Docker 策略
```

### 4. 模板方法模式（Template Method）

工具函数定义了固定的执行流程：

```python
@tool("bash")
def bash_tool(runtime, description, command):
    # 1. 确保沙箱初始化
    sandbox = ensure_sandbox_initialized(runtime)
    # 2. 确保目录存在
    ensure_thread_directories_exist(runtime)
    # 3. 路径替换（本地沙箱）
    if is_local_sandbox(runtime):
        command = replace_virtual_paths_in_command(command, ...)
    # 4. 执行命令
    return sandbox.execute_command(command)
```

### 5. 代理模式（Proxy）

`LocalSandbox` 通过路径映射代理实际文件系统操作：

```python
def read_file(self, path: str) -> str:
    # 代理：将虚拟路径解析为实际路径
    resolved_path = self._resolve_path(path)
    with open(resolved_path) as f:
        return f.read()
```

### 6. 中间件模式（Middleware）

`SandboxMiddleware` 拦截 Agent 执行流程：

```python
class SandboxMiddleware(AgentMiddleware):
    def before_agent(self, state, runtime):
        # 前置处理：获取沙箱
        ...

    def after_agent(self, state, runtime):
        # 后置处理：释放沙箱
        ...
```

### 7. 懒加载模式（Lazy Loading）

沙箱在首次使用时才初始化：

```python
def ensure_sandbox_initialized(runtime):
    # 检查是否已初始化
    if sandbox_state is not None:
        return existing_sandbox
    # 首次使用时才获取
    return provider.acquire(thread_id)
```

---

## 总结

DeerFlow 的 Sandbox 模块提供了一个灵活、安全的沙箱执行环境：

1. **抽象设计**：通过 `Sandbox` 和 `SandboxProvider` 抽象基类，支持多种沙箱后端实现
2. **虚拟路径**：提供统一的虚拟路径机制，让 Agent 无需关心实际文件系统布局
3. **懒加载**：支持延迟初始化，优化资源使用
4. **单例管理**：全局单例确保资源高效复用
5. **异常体系**：完善的异常类层次，便于错误诊断和处理
6. **中间件集成**：与 Agent 执行流程无缝集成，自动管理沙箱生命周期

通过这些设计，Sandbox 模块为 Agent 提供了安全、隔离、高效的执行环境。