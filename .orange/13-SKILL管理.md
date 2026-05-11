# 13 SKILL 管理

## 概览对比

| 维度 | deer-flow | hermes-agent | nanobot |
|------|-----------|--------------|---------|
| **文件格式** | `SKILL.md`（YAML frontmatter + Markdown）| `SKILL.md`（相同格式）| `SKILL.md`（兼容格式）|
| **内置技能数量** | 21 个（public/）| 40+ 个 | 1 个（skill-creator）|
| **技能目录** | `skills/public/` + `skills/custom/` | `~/.hermes/skills/` + external | `~/.nanobot/workspace/skills/` |
| **加载时机** | 启动时扫描，`enabled_only=True` 过滤 | 启动时 + 按需动态加载 | 每轮构建上下文时扫描 |
| **注入方式** | XML 列表（`<skill>` 标签）→ 系统提示 | 系统提示列表 + 内容注入 | 系统提示摘要 + 按需读取 |
| **进化机制** | `skill_evolution.enabled`（LLM 自动创建）| Nudge 提醒（每 15 次工具调用）| Dream Phase 2（定时）|
| **可用性检查** | `extensions_config.json` 启用/禁用状态 | `bins`/`env` 依赖检查 | `bins`/`env` 依赖检查 |
| **子 Agent 技能** | 作为 SystemMessage 注入（非系统提示）| 继承父级可用技能 | 继承父级技能列表 |

---

## SKILL.md 格式（三项目通用核心）

三个项目的 `SKILL.md` 格式高度兼容，核心结构相同：

```markdown
---
name: web-research
description: "系统性网页搜索和信息整合，适用于任何需要从多个来源收集信息的任务"
version: 1.0.0
author: AuthorName
platforms: [macos, linux, windows]

metadata:
  # hermes-agent 特有
  hermes:
    tags: [research, web, information]
    config:
      - key: search.provider
        description: 搜索引擎选择
        default: "duckduckgo"
    fallback_for_tools: ["web_search"]    # 工具不可用时激活
    requires_toolsets: ["web"]            # 需要特定工具集

  # nanobot 特有
  nanobot:
    always: false    # true = 每次对话都加载到系统提示

# deer-flow 使用：
license: MIT
allowed-tools: [web_search, web_fetch, read_file]

requires:
  bins: [curl, python3]
  env: [BRAVE_API_KEY]
---

# Web Research Skill

## 适用场景
...

## 执行步骤
1. 分解搜索主题为 3-5 个独立子查询
...
```

**兼容字段：** `name`, `description`, `requires.bins`, `requires.env`
**hermes 特有：** `metadata.hermes.tags`, `config`, `fallback_for_tools`, `requires_toolsets`
**nanobot 特有：** `metadata.nanobot.always`
**deer-flow 特有：** `allowed-tools`, `license`

---

## deer-flow：21 个内置技能 + 自动进化

### 内置技能目录

```
skills/public/
├── academic-paper-review/    # 学术论文评审
├── bootstrap/                # 项目初始化
├── chart-visualization/      # 数据可视化图表
├── code-documentation/       # 代码文档生成
├── data-analysis/            # 数据分析工作流
├── deep-research/            # 深度网络研究
├── find-skills/              # 元技能：搜索可用技能
├── frontend-design/          # 前端设计（React/TailwindCSS）
├── github-deep-research/     # GitHub 项目深度调研
├── image-generation/         # 图片生成
├── skill-creator/            # 元技能：创建新技能
├── systematic-literature-review/  # 系统文献综述
├── web-design-guidelines/    # 网页设计规范
└── ... (21 个)
```

### 加载流程

```python
# skills/loader.py - load_skills()
def load_skills(
    skills_path: Path | None = None,
    use_config: bool = True,
    enabled_only: bool = False,
) -> list[Skill]:

    all_skills = {}

    for category in ["public", "custom"]:
        category_path = skills_path / category
        for root, dirs, files in os.walk(category_path, followlinks=True):
            if "SKILL.md" not in files:
                continue
            skill = parse_skill_file(Path(root) / "SKILL.md", category=category)
            all_skills[skill.name] = skill

    if enabled_only:
        # 从 extensions_config.json 读取启用状态
        extensions_config = ExtensionsConfig.from_file()
        for skill in all_skills.values():
            skill.enabled = extensions_config.is_skill_enabled(skill.name, skill.category)
        return [s for s in all_skills.values() if s.enabled]

    return list(all_skills.values())
```

### 注入系统提示（懒加载策略）

```python
# agents/lead_agent/prompt.py - get_skills_prompt_section()
@lru_cache(maxsize=32)    # 缓存相同技能集的渲染结果
def get_skills_prompt_section(skills: tuple[str, ...]) -> str:
    lines = []
    for skill in loaded_skills:
        lines.append(f"""<skill>
    <name>{skill.name}</name>
    <description>{skill.description}</description>
    <location>{skill.container_path}/SKILL.md</location>
</skill>""")

    return """<available-skills>
The following skills are available. When relevant, use read_file(<location>) to load skill instructions.
""" + "\n".join(lines) + "\n</available-skills>"
```

**关键设计：** 系统提示只包含技能的名称和描述，**不包含技能全文**。Agent 在判断需要使用某技能时，主动调用 `read_file("{location}/SKILL.md")` 获取完整内容。这是**渐进式加载**模式，避免所有技能的全文撑爆上下文窗口。

### 技能进化（自动创建新技能）

```python
# config.yaml
skill_evolution:
  enabled: true
  min_tool_calls: 5           # 至少使用 5 次工具才触发评估
  create_on_corrections: true # 用户纠错时触发技能创建
```

当启用后，系统提示中会包含指导：
```xml
<skill-evolution-guidance>
After completing complex tasks (5+ tool calls), consider whether this workflow
could benefit others. If so, use the skill-creator skill to document it.

Triggers for skill creation:
- Complex multi-step workflows you just executed
- User corrections that reveal a better approach
- Repeated patterns across multiple tasks
</skill-evolution-guidance>
```

### 子 Agent 的技能注入方式

```python
# subagents/executor.py - _load_skill_messages()
def _load_skill_messages(self, config: RunnableConfig) -> list[BaseMessage]:
    skills = load_skills(enabled_only=True)
    messages = []
    for skill in skills:
        content = skill.load_content()
        # 注入为 SystemMessage（对话历史），而非修改系统提示
        messages.append(SystemMessage(
            content=f'<skill name="{skill.name}">\n{content}\n</skill>'
        ))
    return messages
```

**与主 Agent 的区别：** 主 Agent 的技能在系统提示里只有摘要，需要 read_file 加载全文；子 Agent 的技能直接注入全文到对话历史。原因：子 Agent 任务时间短，预注入减少一次工具调用往返。

---

## hermes-agent：40+ 技能 + Nudge 进化

### 技能目录结构

```
skills/                              # 内置技能（随代码发布）
├── autonomous-ai-agents/
│   ├── hermes-agent/SKILL.md       # 关于 hermes-agent 自身的元技能
│   └── multi-agent-frameworks/SKILL.md
├── coding/
│   ├── test-driven-development/SKILL.md
│   ├── code-review/SKILL.md
│   └── ...
├── research/
├── writing/
└── ...（40+ 技能）

~/.hermes/skills/                    # 用户自定义技能（Agent 自动创建）
```

### 技能解析（parse_frontmatter）

```python
# agent/skill_utils.py
def parse_frontmatter(content: str) -> tuple[dict, str]:
    front_matter_match = re.match(r"^---\s*\n(.*?)\n---\s*\n", content, re.DOTALL)
    if not front_matter_match:
        return {}, content
    frontmatter = yaml.safe_load(front_matter_match.group(1))
    body = content[front_matter_match.end():]
    return frontmatter, body

def extract_skill_conditions(frontmatter: dict) -> dict:
    hermes = frontmatter.get("metadata", {}).get("hermes", {})
    return {
        "fallback_for_toolsets": hermes.get("fallback_for_toolsets", []),
        "requires_toolsets":     hermes.get("requires_toolsets", []),
        "fallback_for_tools":    hermes.get("fallback_for_tools", []),
        "requires_tools":        hermes.get("requires_tools", []),
    }
```

### 条件激活（工具/工具集依赖）

```python
# skill_commands.py
def get_applicable_skills(available_tools: set[str], toolsets: set[str]) -> list[dict]:
    result = []
    for skill in all_skills:
        conditions = extract_skill_conditions(skill.frontmatter)

        # fallback_for_tools：仅当工具不可用时激活
        if conditions["fallback_for_tools"]:
            tool_name = conditions["fallback_for_tools"][0]
            if tool_name in available_tools:
                continue  # 工具可用 → 不需要这个回退技能

        # requires_toolsets：需要特定工具集
        if conditions["requires_toolsets"]:
            if not any(ts in toolsets for ts in conditions["requires_toolsets"]):
                continue

        result.append(skill)
    return result
```

### 斜杠命令激活

```python
# agent/skill_commands.py
def handle_skill_slash_command(user_input: str, agent) -> str | None:
    if not user_input.startswith("/"):
        return None

    skill_name = user_input[1:].split()[0]
    args = user_input[len(skill_name)+2:]

    loaded_skill, skill_dir, name = _load_skill_payload(skill_name)
    parts = [loaded_skill["content"]]
    _inject_skill_config(loaded_skill, parts)
    if args:
        parts.append(f"\nUser provided additional context: {args}")

    return "\n".join(parts)
```

### Nudge 进化机制

```python
# run_agent.py
# 每 15 次工具调用提醒 Agent 考虑创建技能
if (self._iters_since_skill >= self._skill_nudge_interval
        and "skill_manage" in self.valid_tool_names):
    self._spawn_background_review(review_skills=True)
    self._iters_since_skill = 0
```

**后台 review 的系统提示：**
```
You just completed a multi-step task using many tool calls.
Review the recent conversation and consider whether you discovered:
1. A non-obvious workflow worth documenting as a skill
2. Environment-specific commands or configurations
3. Patterns you'll likely need again

If yes, use skill_manage(action="create", ...) to create a new skill.
```

---

## nanobot：轻量技能 + Dream 自动更新

### 技能加载

```python
# agent/skills.py - SkillsLoader
class SkillsLoader:
    skills_dirs: list[Path]   # 内置 skills/ + ~/.nanobot/workspace/skills/

    def list_skills(self) -> list[SkillMeta]:
        skills = []
        for skills_dir in self.skills_dirs:
            for skill_dir in skills_dir.iterdir():
                skill_file = skill_dir / "SKILL.md"
                if not skill_file.exists():
                    continue
                meta = self._parse_meta(skill_file)
                meta.available = self._check_requirements(meta)
                skills.append(meta)
        return skills

    def build_skills_summary(self) -> str:
        lines = ["Available skills:"]
        for meta in self.list_skills():
            if meta.available:
                lines.append(f"  - {meta.name}: {meta.description}")
            else:
                missing = self._get_missing_requirements(meta)
                lines.append(f"  - {meta.name}: (unavailable: {missing})")
        return "\n".join(lines)
```

### always 技能（始终注入全文）

```yaml
# skills/some-skill/SKILL.md frontmatter
metadata:
  nanobot:
    always: true    # 每次对话都将全文注入系统提示
```

**使用场景：** 将核心工作规范（如 "不要修改生产数据库" 的安全约定）设置为 always=true，确保每次对话都强制加载。

### Dream 自动创建/更新技能

Dream Phase 2 执行时，Agent 可以创建新技能文件：

```python
# agent/memory.py - Dream._phase2_execute()
async def _phase2_execute(self, phase1_analysis: str) -> None:
    existing_skills = self.skills_loader.build_skills_summary()

    phase2_prompt = f"""
分析报告：
{phase1_analysis}

现有技能列表：
{existing_skills}

请根据分析报告执行必要的更新。
技能文件路径：~/.nanobot/workspace/skills/{{skill-name}}/SKILL.md
"""
    # AgentRunner 使用 write_file 工具创建技能
    await self._runner.run(AgentRunSpec(
        system=render_template("agent/dream_phase2.md"),
        tools=[read_file_tool, edit_file_tool, write_file_tool],
        initial_messages=[{"role": "user", "content": phase2_prompt}],
    ))
```

---

## 横向对比：SKILL 管理设计总结

### 技能注入策略对比

```
deer-flow：懒加载策略
  系统提示：技能名 + 描述 + 文件路径
  Agent 主动 read_file 加载全文
  优势：上下文最省，技能多时也不撑爆窗口
  代价：每次使用技能需要一次额外 read_file 调用

hermes-agent：按需全文注入
  /skill-name 命令激活 → 技能全文注入对话
  条件激活（工具可用性过滤）
  优势：激活即可用，不需要额外工具调用
  代价：激活后增加对话上下文

nanobot：摘要 + always 双轨
  普通技能：系统提示只有摘要，Agent 按需 read_file
  always 技能：全文始终注入系统提示
  优势：灵活，核心规范可强制加载
```

### 技能进化机制对比

| 项目 | 触发方式 | 执行方式 | 持久化 |
|------|----------|----------|--------|
| deer-flow | 任务完成后 LLM 自评（5+ 工具调用）| skill-creator 元技能指导 | `skills/custom/` |
| hermes-agent | Nudge（每 15 次工具调用）| 后台 review Agent + skill_manage 工具 | `~/.hermes/skills/` |
| nanobot | Dream 定时（每 2 小时）| Dream Phase 2 → write_file | `~/.nanobot/workspace/skills/` |

**hermes-agent 最主动：** 15 次工具调用就触发一次技能创建检查，进化频率最高。
**nanobot 最深度：** Dream 两阶段分析保证技能创建的质量，而非仅靠 LLM 即兴决策。
**deer-flow 最精细：** `skill_evolution.min_tool_calls` + 用户纠错触发，可配置性强。
