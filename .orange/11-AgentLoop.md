# 11 AgentLoop

## 概览对比

| 维度 | deer-flow | hermes-agent | nanobot |
|------|-----------|--------------|---------|
| **Loop 形态** | LangGraph StateGraph（中间件链驱动）| 手写 while 循环（`run_conversation`）| 异步事件总线消费循环 |
| **框架** | LangChain + LangGraph | 无框架（纯手写）| 无框架（纯 asyncio）|
| **消息来源** | HTTP 请求 → `agent.astream()` | 渠道回调直接调用 | MessageBus 异步队列 |
| **并发模型** | asyncio（每个请求独立图执行）| 线程 + async（eval 环境）| asyncio + Semaphore + Lock |
| **中间件/钩子** | 18 个中间件（严格有序）| 无中间件（单体循环内处理）| AgentRunner + Hook 回调 |
| **工具执行** | LangGraph 并行 tool dispatcher | `handle_function_call()` 逐个执行 | `asyncio.gather()` 并行执行 |
| **循环终止条件** | `stop_reason == "end_turn"` 或到达 END 节点 | 无 tool_calls 返回 | `stop_reason == "end_turn"` |
| **最大迭代次数** | `recursion_limit=100` | `max_turns=90`（可配置）| `max_iterations=50`（可配置）|
| **流式输出** | SSE via StreamBridge | SSE / delta callback | 流式 callback |

---

## deer-flow：LangGraph 中间件链

### 架构设计

deer-flow 没有手写 `StateGraph.add_node/add_edge`，而是使用 LangChain 的 `create_agent()` 编译图，所有自定义逻辑通过**中间件链**注入：

```
[START]
  ↓ middleware.before_agent()
  ↓
[ReAct Loop]:
  ↓ middleware.before_model()
  → LLM.invoke(messages)          # 模型调用
  ↓ middleware.after_model()
  ↓
  [decision]:
    no tool_calls → [END]
    has tool_calls → dispatch_tools()
  ↓ middleware.before_tools()
  → execute tool_calls in parallel  # LangGraph 并行执行
  ↓ middleware.after_tools()
  → (loop back to before_model)
  ↓
middleware.after_agent()
[END]
```

### 18 个中间件（严格执行顺序）

```python
# agents/lead_agent/agent.py - _build_middlewares()
def _build_middlewares(config, app_config) -> list[Middleware]:
    middlewares = []

    # ── 基础设施层 ────────────────────────────────────────────
    middlewares.append(ThreadDataMiddleware(config))       # per-thread 工作目录
    middlewares.append(UploadsMiddleware())                # 上传文件注入
    middlewares.append(SandboxMiddleware(config))          # Docker/Modal 沙箱
    middlewares.append(DanglingToolCallMiddleware())       # 修复悬挂工具调用

    # ── 错误处理层 ────────────────────────────────────────────
    middlewares.append(LLMErrorHandlingMiddleware())       # rate limit/timeout 规范化
    middlewares.append(GuardrailMiddleware(config))        # 工具调用前授权检查
    middlewares.append(SandboxAuditMiddleware())           # Shell/文件操作审计
    middlewares.append(ToolErrorHandlingMiddleware())      # 工具异常 → ToolMessage

    # ── 上下文管理层 ──────────────────────────────────────────
    middlewares.append(DeerFlowSummarizationMiddleware(config))  # token 超限压缩
    middlewares.append(TodoMiddleware(config))             # Plan 模式 write_todos
    middlewares.append(TokenUsageMiddleware())             # token 消耗追踪

    # ── 功能增强层 ────────────────────────────────────────────
    middlewares.append(TitleMiddleware())                  # 自动生成 thread 标题
    middlewares.append(MemoryMiddleware(config))           # 异步触发记忆更新
    middlewares.append(ViewImageMiddleware())              # 图片 URL → base64

    # ── 子 Agent 控制层 ───────────────────────────────────────
    middlewares.append(DeferredToolFilterMiddleware(config))   # 延迟工具注册
    middlewares.append(SubagentLimitMiddleware(config))        # 截断超额 task 调用

    # ── 安全层 ────────────────────────────────────────────────
    middlewares.append(LoopDetectionMiddleware())          # 检测工具调用循环
    middlewares.append(ClarificationMiddleware())          # ask_clarification 拦截

    return middlewares
```

### 中间件接口

```python
# 中间件生命周期钩子（并非全部都需要实现）
class Middleware:
    async def before_agent(self, state, config) -> state
    async def after_agent(self, state, config) -> state
    async def before_model(self, state, config) -> state
    async def after_model(self, state, config) -> state
    async def before_tools(self, state, config) -> state
    async def after_tools(self, state, config) -> state
```

### 循环检测（LoopDetectionMiddleware）

```python
# agents/middlewares/loop_detection_middleware.py

class LoopDetectionMiddleware:
    # 哈希机制：检测完全相同的工具调用组合
    def _compute_call_hash(self, tool_calls: list) -> str:
        key = tuple(sorted(
            (tc["name"], json.dumps(tc["args"], sort_keys=True))
            for tc in tool_calls
        ))
        return hashlib.md5(str(key).encode()).hexdigest()

    async def after_model(self, state, config):
        call_hash = self._compute_call_hash(last.tool_calls)
        self._hash_counts[call_hash] = self._hash_counts.get(call_hash, 0) + 1

        # 频次机制：同一工具被调用过多次
        for tc in last.tool_calls:
            self._tool_counts[tc["name"]] = self._tool_counts.get(tc["name"], 0) + 1

        hash_count = self._hash_counts[call_hash]
        max_tool_count = max(self._tool_counts.values())

        if hash_count >= 5 or max_tool_count >= 50:
            # 强制停止：移除 tool_calls，图进入 END 路径
            last.tool_calls = []

        elif hash_count == 3 or max_tool_count == 30:
            # 警告：注入用户提示
            warn_msg = HumanMessage(content="[System: Potential loop detected...]")
            state["messages"].append(warn_msg)

        return state
```

---

## hermes-agent：手写 ReAct 循环

### 核心循环结构

```python
# run_agent.py - AIAgent.run_conversation()
async def run_conversation(self, user_message, conversation_history, ...) -> tuple:

    messages = list(conversation_history) + [{"role": "user", "content": user_message}]

    # ── 系统提示（会话内缓存）──────────────────────────────
    if self._cached_system_prompt is None:
        self._cached_system_prompt = self._build_system_prompt(system_message)

    # ── 记忆召回 ────────────────────────────────────────────
    memory_context = self._memory_manager.prefetch_all(user_message)

    # ── 主 ReAct 循环 ────────────────────────────────────────
    for iteration in range(self.max_turns):
        response = await self._call_llm(messages=messages, system=self._cached_system_prompt)
        assistant_message = response.choices[0].message

        # 空响应处理
        if not assistant_message.content and not assistant_message.tool_calls:
            messages.append({"role": "user", "content": EMPTY_RESPONSE_NUDGE})
            continue

        if not assistant_message.tool_calls:
            final_response = assistant_message.content
            break

        # 执行工具调用（串行）
        for tool_call in assistant_message.tool_calls:
            result = handle_function_call(
                tool_call.function.name,
                json.loads(tool_call.function.arguments),
            )
            messages.append({"role": "tool", "tool_call_id": tool_call.id, "content": result})

    # ── 后处理 ──────────────────────────────────────────────
    self._memory_manager.sync_all(user_message, final_response)
    if _should_review_memory or _should_review_skills:
        self._spawn_background_review(...)

    return final_response, messages
```

### eval 环境的 HermesAgentLoop

```python
# environments/agent_loop.py
class HermesAgentLoop:
    """专为 eval/RL 环境设计的简化 async Loop"""

    # 工具线程池：128 workers 支持高并发 eval
    _tool_executor = concurrent.futures.ThreadPoolExecutor(max_workers=128)

    async def run(self, messages: list[dict]) -> AgentResult:
        reasoning_per_turn = []

        for turn in range(self.max_turns):
            response = await self.server.chat_completion(...)
            msg = response.choices[0].message
            reasoning_per_turn.append(extract_reasoning(msg))

            if msg.tool_calls:
                # 并发执行所有工具调用
                tool_results = await asyncio.gather(*[
                    asyncio.get_event_loop().run_in_executor(
                        self._tool_executor,
                        lambda tc=tc: handle_function_call(...)
                    )
                    for tc in msg.tool_calls
                ])
            else:
                return AgentResult(
                    messages=messages,
                    turns_used=turn + 1,
                    finished_naturally=True,
                    reasoning_per_turn=reasoning_per_turn,
                )
```

---

## nanobot：异步事件总线驱动

### 核心消费循环

```python
# agent/loop.py - AgentLoop
class AgentLoop:
    async def run(self) -> None:
        self._running = True

        while self._running:
            try:
                msg = await asyncio.wait_for(
                    self.bus.consume_inbound(),
                    timeout=1.0,
                )
            except asyncio.TimeoutError:
                # 超时：检查是否需要触发 Dream
                await self._maybe_start_dream()
                continue

            effective_key = self._effective_session_key(msg)

            if effective_key in self._pending_queues:
                # 该会话的 Agent 正在运行 → 进入注入队列
                self._pending_queues[effective_key].put_nowait(msg)
            else:
                # 创建新任务
                task = asyncio.create_task(self._dispatch(msg))
                self._active_tasks.setdefault(effective_key, []).append(task)

    async def _dispatch(self, msg: InboundMessage) -> None:
        key = self._effective_session_key(msg)
        self._pending_queues[key] = asyncio.Queue(maxsize=20)

        try:
            async with self._session_locks[key]:        # 会话串行
                async with self._concurrency_gate:      # 全局并发门控
                    await self._run_agent(msg)
        finally:
            pending = self._pending_queues.pop(key, None)
            if pending and not pending.empty():
                next_msg = pending.get_nowait()
                asyncio.create_task(self._dispatch(next_msg))
```

### AgentRunner 内部的 LLM 迭代

```python
# agent/runner.py - AgentRunner
class AgentRunner:
    async def run(self, spec: AgentRunSpec) -> AgentRunResult:
        messages = list(spec.initial_messages)

        for iteration in range(spec.max_iterations):
            response = await self.provider.chat_with_retry(
                model=spec.model,
                messages=messages,
                tools=[t.schema for t in spec.tools],
            )

            if response.stop_reason == "end_turn":
                return AgentRunResult(messages=messages, ...)

            if response.stop_reason == "tool_use":
                # 并行执行所有工具调用
                tool_results = await asyncio.gather(*[
                    self._execute_tool(tc, spec.tools)
                    for tc in response.tool_calls
                ])
                messages.append({"role": "tool", "content": tool_results})

        return AgentRunResult(..., truncated=True)
```

### Dream 调度集成

```python
# agent/loop.py
async def _maybe_start_dream(self) -> None:
    """在 consume_inbound() 超时时检查是否需要运行 Dream"""
    if not self._dream or self._dream_running:
        return

    if await self._dream.maybe_run():
        self._dream_running = True
        asyncio.create_task(self._run_dream_task())  # 后台运行，不阻塞主 Loop
```

---

## 横向对比：AgentLoop 设计总结

### 架构范式对比

```
deer-flow：框架抽象派
  LangGraph 编译图 → 中间件链注入 → 关注点分离清晰
  优势：中间件可独立测试、热插拔、有序保证
  代价：抽象层厚，调试需要熟悉 LangGraph 原语

hermes-agent：手写单体派
  一个大函数（run_conversation）包含全部逻辑
  优势：没有框架依赖，逻辑清晰可见，易于 debug
  代价：关注点混合，扩展新功能需要修改核心循环

nanobot：事件总线派
  MessageBus 解耦来源与处理，asyncio 原生并发
  优势：多渠道天然支持，并发控制精细，子 Agent 异步回调
  代价：异步思维门槛高，追踪消息流需要理解总线
```

### 并发控制对比

| 项目 | 并发粒度 | 控制机制 | 最大并发 |
|------|----------|----------|----------|
| deer-flow | 每个 HTTP 请求独立 | asyncio（ASGI 框架）| 无上限（反向代理限流）|
| hermes-agent | 每个 Agent 实例 | ThreadPoolExecutor | 128（eval）/ 配置 |
| nanobot | 会话级 + 全局 | Session Lock + Semaphore | 3（可调）|

### 工具执行并行化

| 项目 | 工具执行 | 并行度 |
|------|----------|--------|
| deer-flow | LangGraph 并行 dispatcher | 完全并行（所有 tool_calls 同时执行）|
| hermes-agent（主循环）| 顺序 for loop | 串行 |
| hermes-agent（eval）| `asyncio.gather()` | 完全并行 |
| nanobot | `asyncio.gather()` | 完全并行 |

**注意：** hermes-agent 的主对话循环是串行工具执行（安全优先），而 eval 环境为提升吞吐量改为并行。
