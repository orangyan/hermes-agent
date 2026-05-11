# 10 Lead-Agent / Sub-Agent

## 概览对比

| 维度 | deer-flow | hermes-agent | nanobot |
|------|-----------|--------------|---------|
| **调用方式** | `task` 工具（LangChain tool）| `delegate_task` 工具 | `spawn` 工具 |
| **并行度** | 最多 3 个并行（SubagentLimitMiddleware）| ThreadPoolExecutor（按配置）| asyncio.Task 并发 |
| **深度限制** | 不允许子 Agent 再调用 `task`（disallowed_tools）| 深度计数器（默认 max=2）| 无显式层级，通过 session_key 区分 |
| **工具继承** | 父级工具集 - disallowed_tools | 父级交集（子不得超越父权限）| 可配置继承 |
| **内存隔离** | 完全隔离（独立 LangGraph 实例）| memory 工具排除，不写 MEMORY.md | 独立 session_key |
| **结果回传** | `task_running` 事件流 + 最终结果字符串 | 同步返回结果字符串 | `subagent_result_callback()` 注入 pending_queue |
| **超时/取消** | `cancel_event`（协作式）| `ThreadPoolExecutor` 超时 | asyncio.Task.cancel() |
| **注册表** | `_background_tasks` dict | `_active_subagents` dict + lock | `_active_tasks` dict |

---

## deer-flow：task 工具 + SubagentExecutor

### task 工具接口

```python
# tools/builtins/task_tool.py
@tool("task", parse_docstring=True)
async def task_tool(
    runtime,
    description: str,           # 任务描述（展示给用户）
    prompt: str,                 # 子 Agent 的详细指令
    subagent_type: str,          # 子 Agent 类型（"general-purpose" 等）
    tool_call_id: str,           # LangGraph 工具调用 ID，作为 task_id
    max_turns: int | None = None,
) -> str:
```

### 执行流程

```
Lead Agent LLM 调用 task() × N（最多 3 个并行）
   │
   ├── SubagentLimitMiddleware：超过 3 个的调用被静默截断
   │
   ▼
SubagentExecutor.execute_async(prompt, task_id=tool_call_id)
   │   启动后台线程，立即返回 task_id
   ▼
_scheduler_pool（3 workers）→ run_task()
   │   设置 status=RUNNING
   ▼
_execution_pool（3 workers）→ execute()
   │
   ├── 检测当前是否有 asyncio event loop（ASGI 环境）
   │   如有 → 提交到 _isolated_loop_pool（新 event loop）
   │   如无 → asyncio.run() 直接运行
   │
   ▼
_aexecute()（异步）
   │   调用 create_agent() 创建独立 LangGraph 子图
   │   agent.astream(state, stream_mode="values")
   │
   ├── 每次 astream 迭代
   │   ├── 检查 cancel_event.is_set() → 协作式取消
   │   └── 收集 AIMessage → result.ai_messages
   │
   ▼
父 task_tool() 协程轮询
   │   while True:
   │       result = get_background_task_result(task_id)
   │       if result.status == COMPLETED:
   │           return f"Task Succeeded. Result: {result.result}"
   │       writer({type: "task_running", ai_messages: [...]})
   │       await asyncio.sleep(5)  # 每 5 秒轮询
```

### 子 Agent 工具集配置

```python
# subagents/builtins/general_purpose.py
GENERAL_PURPOSE_CONFIG = SubagentConfig(
    name="general-purpose",
    tools=None,                  # None = 继承父级所有工具
    disallowed_tools=[
        "task",                  # 禁止子 Agent 再派生子 Agent（防止无限嵌套）
        "ask_clarification",     # 子 Agent 不能向用户提问
        "present_files",         # 子 Agent 不直接呈现文件给用户
    ],
    model="inherit",             # 与父 Agent 使用相同模型
    max_turns=100,
)
```

### 并发控制（SubagentLimitMiddleware）

```python
# agents/middlewares/subagent_limit_middleware.py
class SubagentLimitMiddleware:
    max_concurrent_subagents: int = 3  # from config

    def after_model(self, state, config):
        task_calls = [tc for tc in last_msg.tool_calls if tc["name"] == "task"]
        if len(task_calls) <= self.max_concurrent_subagents:
            return state

        # 静默截断超出限制的 task 调用
        truncated_tasks = task_calls[:self.max_concurrent_subagents]
        last_msg.tool_calls = other_calls + truncated_tasks
        return state
```

### Skills 注入到子 Agent

```python
# subagents/executor.py - _load_skill_messages()
def _load_skill_messages(self, config) -> list[BaseMessage]:
    skills = load_skills(enabled_only=True)
    messages = []
    for skill in skills:
        content = skill.load_content()
        # 注入为 SystemMessage，而非系统提示（子 Agent 有自己的系统提示）
        messages.append(
            SystemMessage(content=f'<skill name="{skill.name}">\n{content}\n</skill>')
        )
    return messages
```

---

## hermes-agent：delegate_task 工具

### 接口设计

```python
# tools/delegate_tool.py
def delegate_task(
    goal: str | None = None,           # 任务目标（单任务模式）
    context: str | None = None,        # 额外上下文
    toolsets: list[str] | None = None, # 工具集限制（默认继承父级）
    tasks: list[dict] | None = None,   # 批量任务列表（多任务模式）
    max_iterations: int | None = None,
    role: str | None = None,           # 'leaf'（默认）| 'orchestrator'
    parent_agent = None,               # 父 Agent 对象（自动注入）
) -> str:
```

### 深度限制机制

```python
# tools/delegate_tool.py
depth = getattr(parent_agent, "_delegate_depth", 0)
max_spawn = _get_max_spawn_depth()   # config.delegation.max_spawn_depth，默认 2

if depth >= max_spawn:
    return json.dumps({
        "error": f"Delegation depth limit reached (depth={depth}, max={max_spawn}). "
                 "Cannot spawn further subagents."
    })

# 子 Agent 创建时设置深度
child._delegate_depth = depth + 1
child._parent_subagent_id = getattr(parent_agent, "_subagent_id", None)
```

**深度 = 2 的含义：**
```
Lead Agent (depth=0)
  └── Sub-Agent A (depth=1)          # 允许
        └── Sub-Agent B (depth=2)    # 允许（depth < max=2 时）
              └── Sub-Agent C        # 被拒绝（depth >= max=2）
```

### 工具集继承与限制

```python
# 子 Agent 可用工具 = 父 Agent 工具集 ∩ 请求的工具集
# 子 Agent 永远不能获得父 Agent 没有的工具

parent_tools = set(parent_agent.valid_tool_names)
requested_tools = set(toolsets) if toolsets else parent_tools

# 特殊排除：子 Agent 不能写共享 MEMORY.md
EXCLUDED_FROM_SUBAGENTS = {"memory"}
child_tools = (parent_tools & requested_tools) - EXCLUDED_FROM_SUBAGENTS
```

### Spawn 暂停机制

```python
# tools/delegate_tool.py
_spawn_paused: bool = False

def set_spawn_paused(paused: bool) -> None:
    global _spawn_paused
    _spawn_paused = paused

# 在 delegate_task() 入口处检查
if is_spawn_paused():
    return tool_error("Delegation spawning is paused. Resume via set_spawn_paused(False).")
```

**使用场景：** 系统资源紧张时临时禁用子 Agent 派生，不需要修改配置文件。

### 批量任务模式

```python
# delegate_task(tasks=[...]) 批量模式
if tasks:
    with ThreadPoolExecutor(max_workers=min(len(tasks), 4)) as pool:
        futures = {
            pool.submit(
                delegate_task,
                goal=t["goal"],
                context=t.get("context"),
                toolsets=t.get("toolsets"),
                parent_agent=parent_agent,
            ): i
            for i, t in enumerate(tasks)
        }
        for future in as_completed(futures):
            results.append({"task": futures[future], "result": future.result()})
    return json.dumps(results)
```

---

## nanobot：spawn 工具 + SubagentManager

### spawn 工具接口

```python
# agent/tools/spawn.py
class SpawnTool(BaseTool):
    name = "spawn"
    description = "派生一个子 Agent 执行独立任务"

    async def _run(
        self,
        task: str,
        session_key: str | None = None,   # 默认自动生成
        inherit_tools: bool = True,        # 是否继承父级工具集
        context: str | None = None,
    ) -> str:
```

### 子 Agent session_key 生成

```python
# 子 Agent 的 session_key 格式：{parent_key}:sub:{short_uuid}
# 例：父 = "telegram:123456789"
#     子 = "telegram:123456789:sub:abc8d3f1"

child_key = f"{parent_session_key}:sub:{uuid.uuid4().hex[:8]}"
```

### 子 Agent 结果回传机制

nanobot 的独特设计：子 Agent 完成后，结果**注入父 Agent 的当前处理轮次**：

```python
# agent/subagent.py
class SubagentManager:
    async def spawn(self, spec: SpawnSpec, parent_key: str) -> str:
        child_key = f"{parent_key}:sub:{uuid.uuid4().hex[:8]}"

        # 创建等待事件
        done_event = asyncio.Event()
        self._pending[child_key] = done_event

        # 发布到 MessageBus（子 Agent 走正常的 AgentLoop 路径）
        await self.bus.publish_inbound(InboundMessage(
            session_key=child_key,
            content=spec.task,
            channel="subagent",
            metadata={"parent_key": parent_key, "inherit_tools": spec.inherit_tools},
        ))

        # 等待子 Agent 完成
        await asyncio.wait_for(done_event.wait(), timeout=spec.timeout)
        return self._results.pop(child_key)

# agent/loop.py - subagent_result_callback()
def subagent_result_callback(self, child_key: str, result: str) -> None:
    parent_key = self._get_parent_key(child_key)
    if parent_key in self._pending_queues:
        # 将结果注入父 Agent 的 pending queue
        self._pending_queues[parent_key].put_nowait(
            SubagentResultMessage(child_key=child_key, result=result)
        )
```

---

## 横向对比：核心设计差异

### 子 Agent 架构对比图

```
deer-flow：
Lead Agent
  ├── task_tool("A") ──► SubagentExecutor(thread pool) ──► 独立 LangGraph 图
  ├── task_tool("B") ──► SubagentExecutor(thread pool) ──► 独立 LangGraph 图
  └── task_tool("C") ──► SubagentExecutor(thread pool) ──► 独立 LangGraph 图
  （并行，polling 5s，streaming events）

hermes-agent：
Lead Agent
  └── delegate_task("A") ──► ThreadPoolExecutor ──► 独立 AIAgent 实例
  └── delegate_task(tasks=[A,B,C]) ──► ThreadPoolExecutor(4 workers) ──► 并行
  （深度限制 2，工具集交集，memory 工具排除）

nanobot：
Lead Agent ──► MessageBus ──► AgentLoop
                                ├── spawn("A") ──► publish_inbound(child_key)
                                ├── AgentLoop._dispatch(child_key)
                                └── subagent_result_callback() ──► pending_queue
  （纯异步事件总线，结果注入当前轮次）
```

### 嵌套深度控制

| 项目 | 控制方式 | 默认最大深度 | 超限行为 |
|------|----------|-------------|----------|
| deer-flow | `disallowed_tools=["task"]`（禁用嵌套）| 1（子 Agent 不能再用 task）| 工具不可用 |
| hermes-agent | `_delegate_depth` 计数器 | 2（可配置）| 返回错误 JSON |
| nanobot | session_key 层级（无硬限制）| 无 | 无（依赖设计约定）|

### 共享资源隔离对比

| 资源 | deer-flow | hermes-agent | nanobot |
|------|-----------|--------------|---------|
| 会话/消息历史 | 完全独立的 LangGraph state | 独立 session_id | 独立 session_key + JSON 文件 |
| MEMORY.md | 独立（不注入子 Agent 系统提示）| 排除 memory 工具 | 不写（只读）|
| 工具集 | 父集减去 disallowed | 父集与请求集的交集 | 可配置继承 |
| LLM 模型 | 继承父级（`model="inherit"`）| 可独立配置 | 继承父级配置 |
