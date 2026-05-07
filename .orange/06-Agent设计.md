# 06 Agent 设计

## 核心 Agent 循环（ReAct 模式）

Hermes 的 Agent 核心是**同步 ReAct 循环**，简洁直接，没有 LangGraph 等框架的复杂性：

```python
# run_agent.py 核心循环（简化）
class AIAgent:
    def run_conversation(self, user_message: str) -> str:
        messages = [
            {"role": "system", "content": system_prompt},
            *conversation_history,
            {"role": "user", "content": user_message},
        ]
        tools = model_tools.get_tool_definitions(self.enabled_toolsets)

        while (self.iteration_budget.remaining > 0
               and api_call_count < self.max_iterations):

            response = self.llm_client.chat.completions.create(
                model=self.model,
                messages=messages,
                tools=tools,
                stream=True,
            )

            if not response.tool_calls:
                # 无工具调用 → 最终答案
                return response.content

            # 有工具调用 → 执行工具
            for tool_call in response.tool_calls:
                result = self.handle_function_call(
                    tool_call.name,
                    tool_call.args,
                    task_id=self._current_task_id,
                )
                messages.append(tool_result_message(result))

            api_call_count += 1
```

**关键参数：**
- `max_iterations`：默认 90（通过 `IterationBudget` 线程安全控制）
- `execute_code` 工具调用**不消耗迭代次数**（RPC 执行不需要 LLM）
- `IterationBudget` 是线程安全的，子 Agent 有独立的 budget

## 工具系统设计

### 注册表模式（Registry Pattern）

每个工具在模块导入时自注册（无需显式配置）：

```python
# tools/web_tools.py
from tools.registry import registry

registry.register(
    name="web_search",
    toolset="web",
    schema={
        "type": "function",
        "function": {
            "name": "web_search",
            "description": "Search the web for information",
            "parameters": {
                "type": "object",
                "properties": {
                    "query": {"type": "string", "description": "Search query"},
                    "max_results": {"type": "integer", "default": 5},
                },
                "required": ["query"],
            },
        },
    },
    handler=lambda args, **kw: web_search_impl(args["query"], args.get("max_results", 5)),
    check_fn=lambda: True,          # 检查工具是否可用（如检查 API Key）
    emoji="🔍",
    is_async=True,
    max_result_size_chars=50_000,
)
```

`ToolEntry` 包含的元数据：
- `name, toolset, schema, handler`
- `check_fn`：运行时可用性检查（环境变量是否存在等）
- `requires_env`：依赖的环境变量列表
- `is_async`：是否是异步 handler
- `emoji`：显示在进度条中的图标
- `max_result_size_chars`：结果截断限制

### 工具分层：Agent 级 vs 注册表级

```
工具调用
    ↓
handle_function_call(name, args, task_id)
    ↓
if name in AGENT_LEVEL_TOOLS:
    # Agent 级工具：直接由 AIAgent 处理
    # 原因：需要访问 AIAgent 的内部状态
    todo           → self._todos
    memory         → self.memory_manager
    session_search → self.session_db
    delegate_task  → 创建子 AIAgent
else:
    # 注册表工具：委托给 ToolRegistry
    registry.dispatch(name, args, **context)
```

### 工具集（Toolset）系统

```python
# toolsets.py
_HERMES_CORE_TOOLS = [
    "web_search", "web_extract",
    "read_file", "write_file", "patch", "search_files",
    "memory", "session_search",
    "skills_list", "skill_view", "skill_manage",
    "todo", "clarify",
    "delegate_task",
]

# 平台预设
PLATFORM_TOOLSETS = {
    "hermes-cli":      _HERMES_CORE_TOOLS + ["terminal", "process"],
    "hermes-telegram": _HERMES_CORE_TOOLS + ["send_message", "text_to_speech"],
    "hermes-discord":  _HERMES_CORE_TOOLS + ["send_message"],
}

# 工具集可组合（递归解析）
resolve_toolset(["hermes-cli", "browser", "image"])
→ 展开为具体工具名列表
```

### 并行执行策略

```python
_PARALLEL_SAFE_TOOLS = frozenset({
    "web_search", "web_extract",     # 只读网络
    "read_file", "search_files",     # 只读文件（路径不冲突时）
    "vision_analyze",                # 只读图片
    "session_search",                # 只读 DB
    "skill_view", "skills_list",    # 只读技能
})

_NEVER_PARALLEL_TOOLS = frozenset({
    "clarify",  # 需要等待用户输入（阻塞性）
})

# 路径作用域工具：检查路径是否冲突
_PATH_SCOPED_TOOLS = frozenset({
    "read_file", "write_file", "patch",
})

def _execute_tool_call_batch(tool_calls):
    parallel = []
    serial = []
    for tc in tool_calls:
        if can_parallelize(tc, already_running=parallel):
            parallel.append(tc)
        else:
            serial.append(tc)
    # 并行：ThreadPoolExecutor
    # 串行：顺序 await
```

## 记忆管理系统

### MemoryManager 设计

```
MemoryManager（协调器）
    │
    ├── builtin provider（内置，始终存在）
    │   ├── MEMORY.md 读写
    │   ├── USER.md 读写
    │   └── nudge 提醒机制
    │
    └── external provider（可选，最多 1 个）
        └── Honcho（典型集成）
            → 结构化用户建模 API
            → 召回相关用户上下文
```

**生命周期 Hooks：**

| Hook | 触发时机 | 用途 |
|------|----------|------|
| `on_turn_start` | 每次 LLM 调用前 | prefetch 相关记忆 |
| `on_session_end` | 会话结束 | 触发记忆更新 |
| `on_pre_compress` | 上下文压缩前 | 保存重要信息 |
| `on_memory_write` | MEMORY.md 写入后 | 通知外部 provider 同步 |
| `on_delegation` | 子 Agent 完成后 | 记录子任务结果 |

### Nudge 机制（自我提升核心）

```python
# 每 N 轮对话提醒保存记忆
if api_call_count % nudge_interval == 0:  # nudge_interval=10
    messages.append({
        "role": "user",
        "content": "[SYSTEM NUDGE] Please consider updating your MEMORY.md "
                   "and USER.md with important information from this session."
    })

# 每 M 次工具调用提醒创建技能
if total_tool_calls % creation_nudge_interval == 0:  # creation_nudge_interval=15
    messages.append({
        "role": "user",
        "content": "[SYSTEM NUDGE] You have completed many tool operations. "
                   "Consider creating a skill to capture this workflow."
    })
```

这是 Hermes 的"自我提升"核心机制——Agent 被周期性提醒主动学习。

## 技能系统设计

### 技能索引缓存（两级缓存）

```
Level 1: 内存 LRU 缓存
    → 技能目录列表（name + description）
    → 技能全文（按需缓存）
    → TTL: 5 分钟

Level 2: 磁盘快照
    → ~/.hermes/skills/.index_snapshot.json
    → 记录文件 mtime，文件未变化时跳过重新扫描
```

### 技能条件激活（frontmatter）

```yaml
---
name: telegram-message-formatting
platforms:
  - telegram    # 只在 Telegram 渠道显示此技能
requires_tools:
  - send_message
fallback_for:
  - message-formatting   # 当 message-formatting 技能不可用时候选
---
```

系统提示中的 `<available_skills>` 块只显示满足条件的技能。

## 上下文压缩设计

```
触发条件: prompt_tokens >= 0.50 × context_window

压缩算法：
    messages = [system, ...middle..., ...tail(20 msgs)]
                                ↓
    1. 保护：system 消息（不压缩）
    2. 保护：尾部 20 条消息（最新上下文）
    3. 预处理：工具输出预裁剪（超长 tool_result → 截断）
    4. LLM 摘要：
       auxiliary_client.summarize(middle_messages)
       → 生成结构化摘要（保留关键决策、结论、数据）
    5. 插入：
       compressed = {
           "role": "assistant",
           "content": "[CONTEXT COMPACTION]\n\n{summary}"
       }
    6. 新消息列表 = [system, compressed, *tail]

辅助模型选择（速度/成本优先）：
    compression.summary_model = "google/gemini-flash-2.5"
```

## 子 Agent 委托设计

```python
# delegate_tool.py
def delegate_task(
    task: str,
    tools: list[str] = ["terminal", "file", "web"],
    max_iterations: int = 50,
    batch: bool = False,        # True = 立即返回 task_id，不等待
) -> str:

    sub_agent = AIAgent(
        session_id=f"{parent_session_id}:child:{uuid4()}",
        model=parent_model,
        enabled_toolsets=tools,
        max_iterations=max_iterations,
        parent_session_id=parent_session_id,
    )

    if batch:
        # 非阻塞：立即返回 task_id，后台执行
        future = thread_pool.submit(sub_agent.run_conversation, task)
        return {"status": "queued", "task_id": task_id}
    else:
        # 阻塞：等待完成
        result = sub_agent.run_conversation(task)
        return result
```

**并行子 Agent 模式：**
```
# Lead Agent 连续发起 3 个 batch 委托
t1 = delegate_task("分析 AWS", batch=True)   → task_id_1
t2 = delegate_task("分析 Azure", batch=True) → task_id_2
t3 = delegate_task("分析 GCP", batch=True)   → task_id_3

# 然后等待所有结果
r1 = wait_for_task(task_id_1)
r2 = wait_for_task(task_id_2)
r3 = wait_for_task(task_id_3)

# 汇总
summarize([r1, r2, r3])
```

## execute_code 工具设计（RPC 沙盒）

这是 Hermes 独特的工具之一：

```
Agent 调用 execute_code(code="...", globals={...})
                ↓
code_execution_tool.py：
    → 创建独立 Python RPC 进程（subprocess）
    → code 中可以调用 hermes 工具（read_file/web_search 等）
    → 工具调用通过 RPC 管道传回主进程执行
    → 结果通过 RPC 管道返回
                ↓
关键特性：
    → 执行任意 Python 代码（pandas/numpy/matplotlib 等）
    → 可以在 code 中调用 hermes 工具（不是 subprocess 内调用 LLM）
    → 不消耗 LLM 迭代次数（纯计算，不需要 LLM 决策）
    → 结果包含 stdout + return_value
```

这使得 Agent 可以高效执行：数据分析、图表生成、批量文件处理等纯计算任务。
