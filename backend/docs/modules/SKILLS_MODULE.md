---

# DeerFlow Skills 模块技术文档

## 1. 模块概述

### 1.1 模块职责和功能定位

Skills（技能）模块是 DeerFlow 框架中的知识扩展系统，负责：

- **技能发现与加载**：扫描技能目录，解析 SKILL.md 文件提取元数据
- **技能状态管理**：根据配置文件控制技能的启用/禁用状态
- **系统提示词注入**：将启用的技能信息动态注入到 Agent 系统提示词中
- **技能安装**：支持从 `.skill` 文件（ZIP 压缩包）安装自定义技能
- **容器路径映射**：为沙箱环境提供技能文件路径映射

### 1.2 技能的概念

**Skill（技能）** 是一套预定义的知识和工作流程，用于指导 Agent 更好地完成特定类型的任务。每个技能包含：

- **元数据**：名称、描述、许可证等
- **工作流程**：详细的执行步骤和最佳实践
- **参考资源**：相关的脚本、模板、示例等

```
┌─────────────────────────────────────────────────────────────────┐
│                      技能目录结构示例                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  skills/public/frontend-design/                                │
│  ├── SKILL.md              # 技能主文件（必需）                  │
│  ├── LICENSE.txt           # 许可证文件（可选）                  │
│  ├── reference/            # 参考资源目录（可选）                │
│  │   └── design-patterns.md                                     │
│  └── templates/            # 模板文件（可选）                    │
│      └── component.html                                         │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 1.3 与其他模块的关系

```
┌─────────────────────────────────────────────────────────────────┐
│                        DeerFlow 系统架构                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌──────────────┐    ┌──────────────┐    ┌──────────────┐      │
│  │ Gateway API  │───►│ Extensions   │◄───│ Frontend     │      │
│  │ (skills.py)  │    │ Config       │    │ (skills UI)  │      │
│  └──────┬───────┘    └──────┬───────┘    └──────────────┘      │
│         │                   │                                   │
│         ▼                   ▼                                   │
│  ┌──────────────┐    ┌──────────────┐                         │
│  │ Skills API   │◄───│ Skills Module│                         │
│  │ (CRUD)       │    │ (本模块)      │                         │
│  └──────────────┘    └──────┬───────┘                         │
│                             │                                   │
│                             ▼                                   │
│                      ┌──────────────┐                          │
│                      │ Lead Agent   │                          │
│                      │ Prompt       │                          │
│                      │ (注入技能)    │                          │
│                      └──────────────┘                          │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**依赖关系**：

| 模块 | 关系说明 |
|------|----------|
| `src.config.extensions_config` | 配置管理，提供技能启用状态配置 |
| `src.config.skills_config` | 技能路径配置，提供容器路径映射 |
| `src.agents.lead_agent.prompt` | 提示词构建，消费技能列表注入系统提示词 |
| `src.gateway.routers.skills` | API 路由，提供技能管理 REST 接口 |
| `src.client` | 客户端工具，提供技能读取和路径获取方法 |

---

## 2. 目录结构

### 2.1 模块源码结构

```
backend/src/skills/
├── __init__.py      # 模块入口，导出公共 API
├── types.py         # 数据类型定义（Skill 数据类）
├── parser.py        # SKILL.md 文件解析器
└── loader.py        # 技能加载器（目录扫描、状态管理）
```

**文件职责说明**：

| 文件 | 行数 | 职责 |
|------|------|------|
| `__init__.py` | 4 | 模块公共 API 导出 |
| `types.py` | 53 | Skill 数据类定义、路径计算方法 |
| `parser.py` | 65 | YAML front matter 解析、元数据提取 |
| `loader.py` | 98 | 技能目录扫描、启用状态判断、结果排序 |

### 2.2 技能文件系统结构

```
deer-flow/skills/
├── public/                      # 公共技能目录（内置）
│   ├── frontend-design/
│   │   └── SKILL.md
│   ├── deep-research/
│   │   └── SKILL.md
│   ├── ppt-generation/
│   │   └── SKILL.md
│   └── ...                      # 其他内置技能
│
└── custom/                      # 自定义技能目录（用户安装）
    └── my-skill/
        └── SKILL.md
```

**目录分类**：

| 类别 | 说明 | 路径 |
|------|------|------|
| `public` | 内置公共技能，随项目分发 | `skills/public/` |
| `custom` | 用户自定义技能，通过 API 安装 | `skills/custom/` |

---

## 3. 技能数据结构

### 3.1 Skill 数据类

```python
# backend/src/skills/types.py

from dataclasses import dataclass
from pathlib import Path


@dataclass
class Skill:
    """表示一个技能及其元数据和文件路径。"""

    name: str                    # 技能名称（来自 SKILL.md front matter）
    description: str             # 技能描述（来自 SKILL.md front matter）
    license: str | None          # 许可证信息（可选）
    skill_dir: Path              # 技能目录的绝对路径
    skill_file: Path             # SKILL.md 文件的绝对路径
    relative_path: Path          # 相对于类别根目录的相对路径
    category: str                # 技能类别：'public' 或 'custom'
    enabled: bool = False        # 是否启用（来自配置文件）
```

### 3.2 Skill 类方法详解

#### 3.2.1 `skill_path` 属性

```python
@property
def skill_path(self) -> str:
    """返回从类别根目录（skills/{category}）到技能目录的相对路径。

    示例：
        - skills/public/frontend-design → ""
        - skills/public/subdir/my-skill → "subdir"

    Returns:
        相对路径字符串，如果是技能目录本身则返回空字符串
    """
    path = self.relative_path.as_posix()
    return "" if path == "." else path
```

#### 3.2.2 `get_container_path()` 方法

```python
def get_container_path(self, container_base_path: str = "/mnt/skills") -> str:
    """获取技能在沙箱容器中的完整路径。

    沙箱环境中技能文件被挂载到容器的 /mnt/skills 目录下，
    此方法用于计算正确的容器内路径。

    Args:
        container_base_path: 容器中技能的挂载基础路径

    Returns:
        容器内技能目录的完整路径

    示例：
        >>> skill = Skill(name="frontend-design", category="public", ...)
        >>> skill.get_container_path()
        '/mnt/skills/public'
        >>> skill = Skill(name="my-skill", category="custom", relative_path=Path("tools"), ...)
        >>> skill.get_container_path()
        '/mnt/skills/custom/tools'
    """
    category_base = f"{container_base_path}/{self.category}"
    skill_path = self.skill_path
    if skill_path:
        return f"{category_base}/{skill_path}"
    return category_base
```

#### 3.2.3 `get_container_file_path()` 方法

```python
def get_container_file_path(self, container_base_path: str = "/mnt/skills") -> str:
    """获取技能主文件（SKILL.md）在容器中的完整路径。

    用于在系统提示词中提供技能文件路径，Agent 可以通过
    read_file 工具读取该文件来加载技能内容。

    Args:
        container_base_path: 容器中技能的挂载基础路径

    Returns:
        容器内 SKILL.md 文件的完整路径

    示例：
        >>> skill.get_container_file_path()
        '/mnt/skills/public/SKILL.md'
    """
    return f"{self.get_container_path(container_base_path)}/SKILL.md"
```

### 3.3 技能元数据格式

技能元数据通过 YAML front matter 定义在 SKILL.md 文件头部：

```markdown
---
name: frontend-design
description: Create distinctive, production-grade frontend interfaces with high design quality.
license: Complete terms in LICENSE.txt
---

# 技能内容...

## 工作流程

...
```

**元数据字段说明**：

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `name` | string | 是 | 技能名称，必须是 hyphen-case（小写字母、数字、连字符） |
| `description` | string | 是 | 技能描述，最多 1024 字符，不能包含 `<` 或 `>` |
| `license` | string | 否 | 许可证信息 |

---

## 4. 技能加载机制

### 4.1 核心函数：`load_skills()`

```python
# backend/src/skills/loader.py

def load_skills(
    skills_path: Path | None = None,
    use_config: bool = True,
    enabled_only: bool = False
) -> list[Skill]:
    """从技能目录加载所有技能。

    扫描 public 和 custom 两个技能目录，解析 SKILL.md 文件提取元数据。
    技能的启用状态由 extensions_config.json 文件决定。

    Args:
        skills_path: 自定义技能目录路径。如果未提供且 use_config=True，
                     则从配置文件读取路径，否则使用默认路径。
        use_config: 是否从配置文件读取技能路径（默认 True）
        enabled_only: 是否只返回启用的技能（默认 False）

    Returns:
        按名称排序的技能列表
    """
```

### 4.2 加载流程图

```
┌─────────────────────────────────────────────────────────────────┐
│                        技能加载流程                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  load_skills()                                                  │
│       │                                                         │
│       ▼                                                         │
│  ┌─────────────────────────────────────┐                       │
│  │ 1. 确定技能目录路径                  │                       │
│  │    - skills_path 参数               │                       │
│  │    - 配置文件中的路径                │                       │
│  │    - 默认路径 deer-flow/skills      │                       │
│  └──────────────────┬──────────────────┘                       │
│                     │                                           │
│                     ▼                                           │
│  ┌─────────────────────────────────────┐                       │
│  │ 2. 扫描技能目录                      │                       │
│  │    for category in ["public", "custom"]:                   │
│  │      os.walk() 遍历子目录            │                       │
│  │      查找 SKILL.md 文件              │                       │
│  └──────────────────┬──────────────────┘                       │
│                     │                                           │
│                     ▼                                           │
│  ┌─────────────────────────────────────┐                       │
│  │ 3. 解析 SKILL.md 文件                │                       │
│  │    parse_skill_file()               │                       │
│  │    提取 YAML front matter            │                       │
│  │    创建 Skill 对象                   │                       │
│  └──────────────────┬──────────────────┘                       │
│                     │                                           │
│                     ▼                                           │
│  ┌─────────────────────────────────────┐                       │
│  │ 4. 加载启用状态配置                  │                       │
│  │    ExtensionsConfig.from_file()     │                       │
│  │    is_skill_enabled()               │                       │
│  │    更新 skill.enabled               │                       │
│  └──────────────────┬──────────────────┘                       │
│                     │                                           │
│                     ▼                                           │
│  ┌─────────────────────────────────────┐                       │
│  │ 5. 过滤和排序                        │                       │
│  │    enabled_only 过滤                 │                       │
│  │    按 name 排序                      │                       │
│  └──────────────────┬──────────────────┘                       │
│                     │                                           │
│                     ▼                                           │
│              返回 Skill 列表                                    │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 4.3 目录扫描实现

```python
def load_skills(...) -> list[Skill]:
    # ... 路径确定逻辑 ...

    skills = []

    # 扫描 public 和 custom 目录
    for category in ["public", "custom"]:
        category_path = skills_path / category
        if not category_path.exists() or not category_path.is_dir():
            continue

        # 使用 os.walk 遍历目录树
        for current_root, dir_names, file_names in os.walk(category_path):
            # 保持遍历确定性，跳过隐藏目录
            dir_names[:] = sorted(name for name in dir_names if not name.startswith("."))

            # 跳过没有 SKILL.md 的目录
            if "SKILL.md" not in file_names:
                continue

            skill_file = Path(current_root) / "SKILL.md"
            relative_path = skill_file.parent.relative_to(category_path)

            skill = parse_skill_file(skill_file, category=category, relative_path=relative_path)
            if skill:
                skills.append(skill)

    # ... 启用状态加载和排序 ...
```

### 4.4 启用状态判断

```python
# 从配置文件加载启用状态
# 注意：使用 ExtensionsConfig.from_file() 而非 get_extensions_config()
# 确保总是从磁盘读取最新配置，反映 Gateway API 的修改
try:
    from src.config.extensions_config import ExtensionsConfig

    extensions_config = ExtensionsConfig.from_file()
    for skill in skills:
        skill.enabled = extensions_config.is_skill_enabled(skill.name, skill.category)
except Exception as e:
    # 配置加载失败时，默认全部启用
    print(f"Warning: Failed to load extensions config: {e}")

# 按启用状态过滤
if enabled_only:
    skills = [skill for skill in skills if skill.enabled]
```

### 4.5 技能根路径获取

```python
def get_skills_root_path() -> Path:
    """获取技能目录的根路径。

    路径计算逻辑：
    - 当前文件：backend/src/skills/loader.py
    - backend 目录：loader.py 的父目录的父目录的父目录
    - 技能目录：backend 目录的同级目录 skills

    Returns:
        技能目录路径（deer-flow/skills）
    """
    # backend 目录是当前文件的父目录的父目录的父目录
    backend_dir = Path(__file__).resolve().parent.parent.parent
    # skills 目录是 backend 目录的同级目录
    skills_dir = backend_dir.parent / "skills"
    return skills_dir
```

---

## 5. 技能解析

### 5.1 核心函数：`parse_skill_file()`

```python
# backend/src/skills/parser.py

def parse_skill_file(
    skill_file: Path,
    category: str,
    relative_path: Path | None = None
) -> Skill | None:
    """解析 SKILL.md 文件并提取元数据。

    使用正则表达式解析 YAML front matter，提取 name、description、license 字段。

    Args:
        skill_file: SKILL.md 文件的路径
        category: 技能类别（'public' 或 'custom'）
        relative_path: 技能目录相对于类别根目录的路径

    Returns:
        解析成功返回 Skill 对象，失败返回 None

    Note:
        此函数使用简单的正则解析 YAML front matter，不支持嵌套结构。
        如需复杂 YAML 解析，应使用 PyYAML 库。
    """
```

### 5.2 YAML Front Matter 解析流程

```
┌─────────────────────────────────────────────────────────────────┐
│                    YAML Front Matter 解析流程                    │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  输入：SKILL.md 文件内容                                         │
│  ────────────────────────────────────────────                   │
│  ---                                                            │
│  name: frontend-design                                          │
│  description: Create distinctive interfaces...                  │
│  license: MIT                                                   │
│  ---                                                            │
│  # 技能内容...                                                   │
│                                                                 │
│  步骤 1：正则匹配 front matter                                   │
│  ────────────────────────────────────────                       │
│  pattern: r"^---\s*\n(.*?)\n---\s*\n"                           │
│  匹配结果：group(1) = YAML 内容块                                │
│                                                                 │
│  步骤 2：逐行解析键值对                                          │
│  ────────────────────────────────────────                       │
│  for line in front_matter.split("\n"):                         │
│      if ":" in line:                                            │
│          key, value = line.split(":", 1)                        │
│          metadata[key.strip()] = value.strip()                  │
│                                                                 │
│  步骤 3：提取必需字段                                            │
│  ────────────────────────────────────────                       │
│  name = metadata.get("name")        # 必需                      │
│  description = metadata.get("description")  # 必需              │
│  license = metadata.get("license")  # 可选                      │
│                                                                 │
│  步骤 4：验证必需字段                                            │
│  ────────────────────────────────────────                       │
│  if not name or not description:                               │
│      return None  # 解析失败                                    │
│                                                                 │
│  步骤 5：创建 Skill 对象                                         │
│  ────────────────────────────────────────                       │
│  return Skill(                                                  │
│      name=name,                                                 │
│      description=description,                                   │
│      license=license_text,                                      │
│      skill_dir=skill_file.parent,                              │
│      skill_file=skill_file,                                     │
│      relative_path=relative_path or Path(skill_file.parent.name),│
│      category=category,                                         │
│      enabled=True,  # 默认启用，实际状态来自配置文件             │
│  )                                                              │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 5.3 解析代码详解

```python
def parse_skill_file(skill_file: Path, category: str, relative_path: Path | None = None) -> Skill | None:
    # 验证文件存在且名称正确
    if not skill_file.exists() or skill_file.name != "SKILL.md":
        return None

    try:
        content = skill_file.read_text(encoding="utf-8")

        # 提取 YAML front matter
        # Pattern: ---\nkey: value\n---
        front_matter_match = re.match(r"^---\s*\n(.*?)\n---\s*\n", content, re.DOTALL)

        if not front_matter_match:
            return None

        front_matter = front_matter_match.group(1)

        # 解析 YAML front matter（简单键值对解析）
        metadata = {}
        for line in front_matter.split("\n"):
            line = line.strip()
            if not line:
                continue
            if ":" in line:
                key, value = line.split(":", 1)
                metadata[key.strip()] = value.strip()

        # 提取必需字段
        name = metadata.get("name")
        description = metadata.get("description")

        if not name or not description:
            return None

        license_text = metadata.get("license")

        return Skill(
            name=name,
            description=description,
            license=license_text,
            skill_dir=skill_file.parent,
            skill_file=skill_file,
            relative_path=relative_path or Path(skill_file.parent.name),
            category=category,
            enabled=True,  # 默认启用，实际状态来自配置文件
        )

    except Exception as e:
        print(f"Error parsing skill file {skill_file}: {e}")
        return None
```

---

## 6. 系统提示词注入

### 6.1 技能注入函数：`get_skills_prompt_section()`

```python
# backend/src/agents/lead_agent/prompt.py

def get_skills_prompt_section(available_skills: set[str] | None = None) -> str:
    """生成技能提示词部分，包含可用技能列表。

    返回 <skill_system>...</skill_system> 块，列出所有启用的技能，
    适合注入到任何 Agent 的系统提示词中。

    Args:
        available_skills: 可选的技能名称集合，用于过滤可用技能

    Returns:
        格式化的技能提示词部分字符串
    """
    # 加载启用的技能
    skills = load_skills(enabled_only=True)

    # 获取容器基础路径
    try:
        from src.config import get_app_config
        config = get_app_config()
        container_base_path = config.skills.container_path
    except Exception:
        container_base_path = "/mnt/skills"

    if not skills:
        return ""

    # 按可用技能集合过滤
    if available_skills is not None:
        skills = [skill for skill in skills if skill.name in available_skills]

    # 生成技能列表 XML
    skill_items = "\n".join(
        f"    <skill>\n"
        f"        <name>{skill.name}</name>\n"
        f"        <description>{skill.description}</description>\n"
        f"        <location>{skill.get_container_file_path(container_base_path)}</location>\n"
        f"    </skill>"
        for skill in skills
    )
    skills_list = f"<available_skills>\n{skill_items}\n</available_skills>"

    return f"""<skill_system>
You have access to skills that provide optimized workflows for specific tasks. Each skill contains best practices, frameworks, and references to additional resources.

**Progressive Loading Pattern:**
1. When a user query matches a skill's use case, immediately call `read_file` on the skill's main file using the path attribute provided in the skill tag below
2. Read and understand the skill's workflow and instructions
3. The skill file contains references to external resources under the same folder
4. Load referenced resources only when needed during execution
5. Follow the skill's instructions precisely

**Skills are located at:** {container_base_path}

{skills_list}

</skill_system>"""
```

### 6.2 注入时机

技能提示词在 `apply_prompt_template()` 函数中被注入：

```python
def apply_prompt_template(
    subagent_enabled: bool = False,
    max_concurrent_subagents: int = 3,
    *,
    agent_name: str | None = None,
    available_skills: set[str] | None = None
) -> str:
    """应用提示词模板，生成完整的系统提示词。"""

    # ... 其他部分 ...

    # 获取技能部分
    skills_section = get_skills_prompt_section(available_skills)

    # 格式化提示词
    prompt = SYSTEM_PROMPT_TEMPLATE.format(
        agent_name=agent_name or "DeerFlow 2.0",
        soul=get_agent_soul(agent_name),
        skills_section=skills_section,  # 技能部分
        memory_context=memory_context,
        subagent_section=subagent_section,
        subagent_reminder=subagent_reminder,
        subagent_thinking=subagent_thinking,
    )

    return prompt + f"\n<current_date>{datetime.now().strftime('%Y-%m-%d, %A')}</current_date>"
```

### 6.3 注入后的系统提示词示例

```xml
<skill_system>
You have access to skills that provide optimized workflows for specific tasks. Each skill contains best practices, frameworks, and references to additional resources.

**Progressive Loading Pattern:**
1. When a user query matches a skill's use case, immediately call `read_file` on the skill's main file using the path attribute provided in the skill tag below
2. Read and understand the skill's workflow and instructions
3. The skill file contains references to external resources under the same folder
4. Load referenced resources only when needed during execution
5. Follow the skill's instructions precisely

**Skills are located at:** /mnt/skills

<available_skills>
    <skill>
        <name>frontend-design</name>
        <description>Create distinctive, production-grade frontend interfaces with high design quality.</description>
        <location>/mnt/skills/public/SKILL.md</location>
    </skill>
    <skill>
        <name>deep-research</name>
        <description>Conduct comprehensive research on any topic using web search and analysis.</description>
        <location>/mnt/skills/public/SKILL.md</location>
    </skill>
</available_skills>

</skill_system>
```

### 6.4 Agent 技能使用流程

```
┌─────────────────────────────────────────────────────────────────┐
│                    Agent 技能使用流程                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  1. 系统提示词包含可用技能列表                                    │
│     └── <skill><name>frontend-design</name>...</skill>         │
│                                                                 │
│  2. 用户请求匹配技能描述                                         │
│     └── "帮我设计一个网页"                                       │
│                                                                 │
│  3. Agent 调用 read_file 加载技能                                │
│     └── read_file("/mnt/skills/public/SKILL.md")               │
│                                                                 │
│  4. 技能内容返回给 Agent                                         │
│     └── 包含工作流程、最佳实践、参考资源                         │
│                                                                 │
│  5. Agent 按技能指导执行任务                                     │
│     └── 遵循技能中定义的流程                                     │
│     └── 按需加载额外资源                                         │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 7. 配置项

### 7.1 SkillsConfig 配置类

```python
# backend/src/config/skills_config.py

from pathlib import Path
from pydantic import BaseModel, Field


class SkillsConfig(BaseModel):
    """技能系统配置。"""

    path: str | None = Field(
        default=None,
        description="技能目录路径。未指定时默认为 backend 目录的 ../skills",
    )
    container_path: str = Field(
        default="/mnt/skills",
        description="技能在沙箱容器中的挂载路径",
    )

    def get_skills_path(self) -> Path:
        """获取解析后的技能目录路径。

        优先使用配置的路径（支持绝对路径和相对路径），
        未配置时使用默认路径。
        """
        if self.path:
            path = Path(self.path)
            if not path.is_absolute():
                path = Path.cwd() / path
            return path.resolve()
        else:
            from src.skills.loader import get_skills_root_path
            return get_skills_root_path()

    def get_skill_container_path(self, skill_name: str, category: str = "public") -> str:
        """获取特定技能在容器中的完整路径。

        Args:
            skill_name: 技能名称（目录名）
            category: 技能类别（public 或 custom）

        Returns:
            容器内技能目录的完整路径
        """
        return f"{self.container_path}/{category}/{skill_name}"
```

### 7.2 extensions_config.json 中的技能状态

```json
{
  "mcpServers": {
    // MCP 服务器配置...
  },
  "skills": {
    "frontend-design": {
      "enabled": true
    },
    "deep-research": {
      "enabled": true
    },
    "ppt-generation": {
      "enabled": false
    }
  }
}
```

### 7.3 SkillStateConfig 配置类

```python
# backend/src/config/extensions_config.py

class SkillStateConfig(BaseModel):
    """单个技能的状态配置。"""

    enabled: bool = Field(default=True, description="是否启用此技能")
```

### 7.4 启用状态判断逻辑

```python
# backend/src/config/extensions_config.py

class ExtensionsConfig(BaseModel):
    """MCP 服务器和技能的统一配置。"""

    # ...

    def is_skill_enabled(self, skill_name: str, skill_category: str) -> bool:
        """检查技能是否启用。

        判断逻辑：
        1. 如果配置文件中有该技能的配置，返回配置的 enabled 值
        2. 如果没有配置，对于 public 和 custom 类别默认启用

        Args:
            skill_name: 技能名称
            skill_category: 技能类别

        Returns:
            True 表示启用，False 表示禁用
        """
        skill_config = self.skills.get(skill_name)
        if skill_config is None:
            # public 和 custom 技能默认启用
            return skill_category in ("public", "custom")
        return skill_config.enabled
```

### 7.5 配置文件查找优先级

```python
@classmethod
def resolve_config_path(cls, config_path: str | None = None) -> Path | None:
    """解析扩展配置文件路径。

    优先级：
    1. 参数指定的 config_path
    2. 环境变量 DEER_FLOW_EXTENSIONS_CONFIG_PATH
    3. 当前目录的 extensions_config.json
    4. 父目录的 extensions_config.json
    5. 当前目录的 mcp_config.json（向后兼容）
    6. 父目录的 mcp_config.json（向后兼容）
    7. 未找到返回 None（扩展是可选的）
    """
```

---

## 8. Gateway API

### 8.1 技能管理 API 端点

```python
# backend/src/gateway/routers/skills.py

router = APIRouter(prefix="/api", tags=["skills"])

# GET /api/skills - 列出所有技能
@router.get("/skills", response_model=SkillsListResponse)
async def list_skills() -> SkillsListResponse:
    """获取所有可用技能列表。"""
    skills = load_skills(enabled_only=False)
    return SkillsListResponse(skills=[_skill_to_response(skill) for skill in skills])

# GET /api/skills/{skill_name} - 获取单个技能
@router.get("/skills/{skill_name}", response_model=SkillResponse)
async def get_skill(skill_name: str) -> SkillResponse:
    """获取指定技能的详细信息。"""
    skills = load_skills(enabled_only=False)
    skill = next((s for s in skills if s.name == skill_name), None)
    if skill is None:
        raise HTTPException(status_code=404, detail=f"Skill '{skill_name}' not found")
    return _skill_to_response(skill)

# PUT /api/skills/{skill_name} - 更新技能状态
@router.put("/skills/{skill_name}", response_model=SkillResponse)
async def update_skill(skill_name: str, request: SkillUpdateRequest) -> SkillResponse:
    """更新技能的启用状态。"""
    # 验证技能存在
    # 更新 extensions_config.json
    # 重新加载配置
    # 返回更新后的技能信息

# POST /api/skills/install - 安装技能
@router.post("/skills/install", response_model=SkillInstallResponse)
async def install_skill(request: SkillInstallRequest) -> SkillInstallResponse:
    """从 .skill 文件安装技能。"""
    # 解压 ZIP 文件
    # 验证 SKILL.md 格式
    # 复制到 custom 目录
```

### 8.2 技能安装流程

```
┌─────────────────────────────────────────────────────────────────┐
│                      技能安装流程                                │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  1. 接收安装请求                                                 │
│     └── thread_id + path (虚拟路径)                             │
│                                                                 │
│  2. 解析虚拟路径                                                 │
│     └── resolve_thread_virtual_path()                          │
│     └── 验证文件存在且为 .skill 扩展名                          │
│                                                                 │
│  3. 解压到临时目录                                               │
│     └── zipfile.extractall()                                   │
│                                                                 │
│  4. 验证技能格式                                                 │
│     └── _validate_skill_frontmatter()                          │
│     └── 检查 SKILL.md 存在                                       │
│     └── 检查 YAML front matter 格式                             │
│     └── 验证 name 和 description 字段                           │
│     └── 检查命名规范（hyphen-case）                              │
│                                                                 │
│  5. 检查技能是否已存在                                           │
│     └── skills/custom/{skill_name} 是否存在                     │
│                                                                 │
│  6. 复制到 custom 目录                                           │
│     └── shutil.copytree()                                       │
│                                                                 │
│  7. 返回安装结果                                                 │
│     └── { success: true, skill_name: "...", message: "..." }   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 8.3 技能验证规则

```python
# 允许的 front matter 属性
ALLOWED_FRONTMATTER_PROPERTIES = {"name", "description", "license", "allowed-tools", "metadata"}

def _validate_skill_frontmatter(skill_dir: Path) -> tuple[bool, str, str | None]:
    """验证技能目录的 SKILL.md front matter。

    验证规则：
    1. SKILL.md 文件必须存在
    2. 必须有有效的 YAML front matter
    3. 不允许未知的属性
    4. name 和 description 必需
    5. name 必须是 hyphen-case（小写字母、数字、连字符）
    6. name 长度不超过 64 字符
    7. description 不能包含 < 或 >
    8. description 长度不超过 1024 字符

    Returns:
        (is_valid, message, skill_name)
    """
```

---

## 9. Client 工具集成

### 9.1 技能读取方法

```python
# backend/src/client.py

class DeerFlowClient:
    # ...

    def get_skills(self, enabled_only: bool = False) -> list[dict]:
        """获取技能列表。

        Args:
            enabled_only: 是否只返回启用的技能

        Returns:
            技能信息字典列表
        """
        from src.skills.loader import load_skills

        return [
            {
                "name": s.name,
                "description": s.description,
                "license": s.license,
                "category": s.category,
                "enabled": s.enabled,
            }
            for s in load_skills(enabled_only=enabled_only)
        ]

    def get_skill(self, name: str) -> dict | None:
        """获取单个技能信息。"""
        from src.skills.loader import load_skills

        skill = next((s for s in load_skills(enabled_only=False) if s.name == name), None)
        if skill is None:
            return None
        return {
            "name": skill.name,
            "description": skill.description,
            "license": skill.license,
            "category": skill.category,
            "enabled": skill.enabled,
        }
```

### 9.2 技能路径获取

```python
def get_skill_container_path(self, skill_name: str, category: str = "public") -> str:
    """获取技能在容器中的路径。"""
    from src.config import get_app_config

    config = get_app_config()
    return config.skills.get_skill_container_path(skill_name, category)
```

---

## 10. 执行流程

### 10.1 完整技能加载流程

```
┌─────────────────────────────────────────────────────────────────────────┐
│                          完整技能加载流程                                │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  应用启动                                                                │
│       │                                                                 │
│       ▼                                                                 │
│  ┌─────────────────────────────────────┐                               │
│  │ apply_prompt_template()             │                               │
│  │ (构建系统提示词)                     │                               │
│  └──────────────────┬──────────────────┘                               │
│                     │                                                   │
│                     ▼                                                   │
│  ┌─────────────────────────────────────┐                               │
│  │ get_skills_prompt_section()         │                               │
│  │ 1. load_skills(enabled_only=True)   │                               │
│  │ 2. 获取容器路径配置                  │                               │
│  │ 3. 生成 XML 格式技能列表             │                               │
│  └──────────────────┬──────────────────┘                               │
│                     │                                                   │
│                     ▼                                                   │
│  ┌─────────────────────────────────────┐                               │
│  │ load_skills()                       │                               │
│  │ 1. 确定技能目录路径                  │                               │
│  │ 2. 扫描 public/custom 目录          │                               │
│  │ 3. 解析 SKILL.md 文件                │                               │
│  │ 4. 加载启用状态配置                  │                               │
│  │ 5. 过滤并排序                        │                               │
│  └──────────────────┬──────────────────┘                               │
│                     │                                                   │
│                     ▼                                                   │
│  ┌─────────────────────────────────────┐                               │
│  │ parse_skill_file()                  │                               │
│  │ 1. 读取文件内容                      │                               │
│  │ 2. 正则匹配 YAML front matter       │                               │
│  │ 3. 提取元数据字段                    │                               │
│  │ 4. 创建 Skill 对象                   │                               │
│  └──────────────────┬──────────────────┘                               │
│                     │                                                   │
│                     ▼                                                   │
│  ┌─────────────────────────────────────┐                               │
│  │ ExtensionsConfig.from_file()        │                               │
│  │ 1. 读取 extensions_config.json      │                               │
│  │ 2. 解析 JSON                        │                               │
│  │ 3. is_skill_enabled() 判断状态      │                               │
│  └──────────────────┬──────────────────┘                               │
│                     │                                                   │
│                     ▼                                                   │
│  ┌─────────────────────────────────────┐                               │
│  │ 系统提示词生成完成                   │                               │
│  │ 包含 <skill_system> 块               │                               │
│  └─────────────────────────────────────┘                               │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### 10.2 跨进程配置同步

由于 Gateway API 和 LangGraph Server 运行在不同进程：

```
┌─────────────────┐         修改配置          ┌─────────────────┐
│   Gateway API   │ ─────────────────────►   │ extensions_     │
│  (进程 1)       │                          │ config.json     │
│                 │                          └────────┬────────┘
│  PUT /api/skills/{name}                            │
│  更新 enabled 状态                                 │
└─────────────────┘                                   │
                                                      │
                                                      ▼
                                             ┌─────────────────┐
                                             │  LangGraph      │
                                             │  Server         │
                                             │  (进程 2)       │
                                             │                 │
                                             │ ExtensionsConfig│
                                             │ .from_file()    │
                                             │ 总是从磁盘读取  │
                                             └─────────────────┘
```

**关键设计**：使用 `ExtensionsConfig.from_file()` 而非 `get_extensions_config()` 确保总是从磁盘读取最新配置。

---

## 11. 示例

### 11.1 技能文件示例

```markdown
---
name: frontend-design
description: Create distinctive, production-grade frontend interfaces with high design quality. Use this skill when the user asks to build web components, pages, artifacts, posters, or applications.
license: Complete terms in LICENSE.txt
---

This skill guides creation of distinctive, production-grade frontend interfaces that avoid generic "AI slop" aesthetics.

## Design Thinking

Before coding, understand the context and commit to a BOLD aesthetic direction:
- **Purpose**: What problem does this interface solve? Who uses it?
- **Tone**: Pick an extreme: brutally minimal, maximalist chaos, retro-futuristic...
- **Constraints**: Technical requirements (framework, performance, accessibility).
- **Differentiation**: What makes this UNFORGETTABLE?

## Frontend Aesthetics Guidelines

Focus on:
- **Typography**: Choose fonts that are beautiful, unique, and interesting.
- **Color & Theme**: Commit to a cohesive aesthetic. Use CSS variables for consistency.
- **Motion**: Use animations for effects and micro-interactions.
- **Spatial Composition**: Unexpected layouts. Asymmetry. Overlap. Diagonal flow.

## Output Requirements

**MANDATORY**: The entry HTML file MUST be named `index.html`.
```

### 11.2 使用示例

```python
# 加载所有技能
from src.skills import load_skills

skills = load_skills()
for skill in skills:
    print(f"  - {skill.name}: {skill.description}")

# 只加载启用的技能
enabled_skills = load_skills(enabled_only=True)

# 获取技能容器路径
skill = enabled_skills[0]
container_path = skill.get_container_file_path()
print(f"技能文件路径: {container_path}")
# 输出: 技能文件路径: /mnt/skills/public/SKILL.md
```

---

## 12. 最佳实践

### 12.1 技能设计原则

1. **单一职责**：每个技能专注于一类特定任务
2. **清晰描述**：描述应准确说明技能的用途和触发场景
3. **渐进式加载**：主文件包含核心流程，额外资源按需加载
4. **命名规范**：使用 hyphen-case，名称应简洁且具有描述性

### 12.2 技能文件组织

```
skills/public/my-skill/
├── SKILL.md              # 主文件（必需）
├── LICENSE.txt           # 许可证（可选）
├── reference/            # 参考文档
│   ├── patterns.md
│   └── examples.md
├── templates/            # 模板文件
│   └── component.html
└── assets/               # 静态资源
    └── diagram.svg
```

### 12.3 配置管理

- 使用 `extensions_config.json` 管理技能启用状态
- 配置修改后无需重启服务（自动检测）
- 自定义技能安装在 `custom` 目录