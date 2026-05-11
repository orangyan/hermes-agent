# 12 Thinking（推理链）模式

## 概览对比

| 维度 | deer-flow | hermes-agent | nanobot |
|------|-----------|--------------|---------|
| **支持的 Thinking 类型** | Anthropic extended/adaptive + vLLM + OpenAI reasoning | Anthropic adaptive + manual budget + OpenAI reasoning + Gemini | Anthropic adaptive thinking |
| **配置粒度** | 模型级（factory 统一处理）| effort 级（none/low/medium/high/xhigh）| thinking_budget（token 数）|
| **默认开启** | `thinking_enabled=True`（除非 `supports_thinking: false`）| `reasoning_effort: medium`（可配置）| 按模型自动检测 |
| **子 Agent 独立配置** | 继承父 Agent | `delegation.reasoning_effort` 独立配置 | 继承父 Agent |
| **Thinking 内容保留** | `display: "summarized"`（可见摘要）| `reasoning_per_turn` 记录到 session | 可选保留 |
| **Budget 分配策略** | 80% of max_tokens（自动计算）| 映射表（xhigh=32000 tokens）| 手动配置 |

---

## Thinking 技术背景

### 什么是 Extended Thinking

Anthropic Claude 的 Extended Thinking 允许模型在生成最终回复前，先进行一段**隐式推理过程**（thinking blocks）。这些 thinking 内容：
- 不计入用户的对话历史（不是 assistant message 的一部分）
- 消耗额外 token（需要 `budget_tokens` 预算）
- 可以包含：思路探索、问题分解、方案对比、自我纠错
- 显著提升复杂任务（数学、代码、规划）的准确性

### Anthropic Thinking 模式演进

| 模式 | 适用模型 | 机制 |
|------|----------|------|
| **Manual thinking** | Claude 3.5 Sonnet 等旧版 | 显式设置 `budget_tokens`，必须 `temperature=1` |
| **Adaptive thinking** | Claude claude-sonnet-4-6（claude-sonnet-4-6）+ | 框架自动管理 budget，通过 `effort` 控制强度 |
| **output_config.effort** | Claude claude-opus-4-5（claude-opus-4-5）+ | 替代 adaptive thinking，支持 `xhigh` 级别 |

---

## deer-flow：模型工厂统一处理

### thinking_enabled 开关

```python
# models/factory.py - create_chat_model()
def create_chat_model(
    name: str | None = None,
    thinking_enabled: bool = False,
    *,
    app_config=None,
    **kwargs,
) -> BaseChatModel:

    model_config = app_config.models.get_config(name)

    # 如果模型声明不支持 thinking，强制关闭
    if not model_config.get("supports_thinking", True):
        thinking_enabled = False

    # thinking 开启时，合并 when_thinking_enabled 配置块
    if thinking_enabled and "when_thinking_enabled" in model_config:
        effective_config = {**model_config, **model_config["when_thinking_enabled"]}
    else:
        effective_config = model_config

    provider = model_config.get("provider", "openai_compat")
    return _create_model_for_provider(provider, effective_config, thinking_enabled, **kwargs)
```

**config.yaml 模型配置示例：**

```yaml
models:
  default:
    name: claude-sonnet-4-6-20251119
    provider: anthropic
    supports_thinking: true
    when_thinking_enabled:
      max_tokens: 16000
      thinking:
        type: "enabled"
        budget_tokens: 10000    # 若不设，代码自动算 80%

  fast:
    name: claude-haiku-3-5
    provider: anthropic
    supports_thinking: false    # 明确禁用

  qwen-thinking:
    name: qwen3-32b
    provider: openai_compat
    api_base: "http://localhost:8000/v1"
    supports_thinking: true
    when_thinking_enabled:
      extra_body:
        chat_template_kwargs:
          enable_thinking: true     # vLLM/Qwen 的开关
    when_thinking_disabled:
      extra_body:
        chat_template_kwargs:
          enable_thinking: false
```

### Anthropic thinking budget 自动计算

```python
# models/claude_provider.py
THINKING_BUDGET_RATIO = 0.8

class ClaudeProvider(ChatAnthropic):
    def _apply_thinking_budget(self, payload: dict) -> None:
        thinking = payload.get("thinking", {})
        if thinking.get("type") != "enabled":
            return
        if thinking.get("budget_tokens"):
            return    # 已显式设置，不覆盖

        max_tokens = payload.get("max_tokens", 8192)
        # 自动分配 80% 给 thinking
        thinking["budget_tokens"] = int(max_tokens * THINKING_BUDGET_RATIO)
        # 例：max_tokens=16000 → budget_tokens=12800
```

### 三种 Provider 的 thinking 关闭方式

```python
# models/factory.py
def _disable_thinking_for_provider(provider: str, model_config: dict) -> dict:
    if provider == "anthropic":
        return {"thinking": {"type": "disabled"}}

    elif provider == "openai_compat":
        return {
            "extra_body": {
                "thinking": {"type": "disabled"}
            }
        }

    elif provider == "vllm_qwen":
        return {
            "extra_body": {
                "chat_template_kwargs": {"enable_thinking": False}
            }
        }
```

---

## hermes-agent：多 Provider 统一 effort 接口

### 统一 effort 级别

```python
# hermes_constants.py
VALID_REASONING_EFFORTS = ("minimal", "low", "medium", "high", "xhigh")

def parse_reasoning_effort(effort: str) -> dict | None:
    if effort == "none":
        return {"enabled": False}
    if effort in VALID_REASONING_EFFORTS:
        return {"enabled": True, "effort": effort}
    return None
```

**config.yaml：**

```yaml
agent:
  reasoning_effort: medium   # Agent 自身的 thinking 强度

delegation:
  reasoning_effort: medium   # 子 Agent 独立控制（可以比主 Agent 低以节省 token）
```

### Anthropic Adaptive Thinking

```python
# agent/anthropic_adapter.py

# Budget 映射表（manual thinking 模式）
THINKING_BUDGET = {
    "xhigh": 32000,
    "high":  16000,
    "medium": 8000,
    "low":    4000,
}

# Adaptive thinking effort 映射
ADAPTIVE_EFFORT_MAP = {
    "max":    "max",
    "xhigh":  "xhigh",
    "high":   "high",
    "medium": "medium",
    "low":    "low",
    "minimal":"low",
}

def build_anthropic_kwargs(model: str, effort: str, max_tokens: int) -> dict:
    kwargs = {}

    if _supports_adaptive_thinking(model):
        # 新版：adaptive thinking + output_config.effort
        kwargs["thinking"] = {
            "type": "adaptive",
            "display": "summarized",   # CLI 中显示思考摘要
        }
        adaptive_effort = ADAPTIVE_EFFORT_MAP.get(effort, "medium")
        if adaptive_effort == "xhigh" and not _supports_xhigh_effort(model):
            adaptive_effort = "max"    # claude-sonnet-4-6 降级为 max
        kwargs["output_config"] = {"effort": adaptive_effort}

    else:
        # 旧版：manual thinking with budget_tokens
        budget = THINKING_BUDGET.get(effort, 8000)
        kwargs["thinking"] = {"type": "enabled", "budget_tokens": budget}
        kwargs["temperature"] = 1      # manual thinking 必须 temperature=1
        kwargs["max_tokens"] = max(max_tokens, budget + 4096)

    if _no_sampling_params(model):
        # claude-opus-4-5 不接受 temperature/top_p/top_k
        kwargs.pop("temperature", None)
        kwargs.pop("top_p", None)
        kwargs.pop("top_k", None)

    return kwargs
```

### OpenAI Reasoning（o-series / Codex）

```python
# agent/transports/codex.py
class CodexTransport:
    async def chat_completion(self, messages, tools, effort="medium", **kwargs):
        reasoning_config = {"effort": effort, "summary": "auto"}
        return await client.chat.completions.create(
            model=self.model,
            messages=messages,
            tools=tools,
            reasoning=reasoning_config,   # OpenAI o-series 专有参数
        )
```

### Gemini Thinking

```python
# agent/transports/chat_completions.py
def _build_gemini_thinking_config(model: str, reasoning_config: dict | None) -> dict | None:
    if not model.startswith("google/") and not model.startswith("gemini"):
        return None

    thinking_config: dict = {"includeThoughts": True}
    effort = (reasoning_config or {}).get("effort", "medium")

    if effort == "low":
        thinking_config["thinkingLevel"] = "low"
    elif effort in ("high", "xhigh", "max"):
        thinking_config["thinkingLevel"] = "high"
    else:
        thinking_config["thinkingLevel"] = "medium"

    return thinking_config
```

### Thinking 内容持久化

hermes-agent 的 SQLite schema 专门为 thinking 内容设计了字段：

```sql
CREATE TABLE messages (
    ...
    reasoning TEXT,             -- 简要 thinking 摘要
    reasoning_content TEXT,     -- 完整 thinking 文本
    reasoning_details TEXT,     -- JSON，含 thinking blocks 数组
    codex_reasoning_items TEXT, -- OpenAI Codex 的推理 items
    codex_message_items TEXT    -- OpenAI Codex 的消息 items
);
```

`AgentResult` 也包含 `reasoning_per_turn` 字段，供 eval 框架分析推理质量。

---

## nanobot：Anthropic Provider 集成

### Anthropic Provider 实现

```python
# providers/anthropic_provider.py
class AnthropicProvider(BaseProvider):

    async def chat_with_retry(
        self,
        model: str,
        messages: list[dict],
        tools: list | None = None,
        thinking_budget: int | None = None,
        **kwargs,
    ) -> ProviderResponse:

        params = {
            "model": model,
            "messages": messages,
            "max_tokens": self.config.max_tokens,
        }

        # Thinking 配置
        if thinking_budget:
            params["thinking"] = {
                "type": "enabled",
                "budget_tokens": thinking_budget,
            }
            params["temperature"] = 1   # thinking 要求 temperature=1

        # Prompt caching（系统提示）
        if messages and messages[0].get("role") == "system":
            params["system"] = [
                {
                    "type": "text",
                    "text": messages[0]["content"],
                    "cache_control": {"type": "ephemeral"},   # 缓存系统提示
                }
            ]

        # 重试逻辑
        for attempt in range(self.max_retries):
            try:
                return await self._client.messages.create(**params)
            except anthropic.RateLimitError:
                await asyncio.sleep(2 ** attempt)   # 指数退避
```

---

## 横向对比：Thinking 配置对比

### 各项目的 Effort 到 Token Budget 映射

```
hermes-agent 的显式映射：
  xhigh  → 32,000 tokens
  high   → 16,000 tokens
  medium →  8,000 tokens
  low    →  4,000 tokens

deer-flow 的自动计算：
  budget_tokens = int(max_tokens × 0.8)
  （max_tokens=16000 → budget=12,800）

nanobot：
  手动配置 thinking_budget（token 数）
```

### Adaptive Thinking vs Manual Thinking

| 特性 | Manual thinking（旧）| Adaptive thinking（新）|
|------|---------------------|----------------------|
| 控制方式 | 显式设置 `budget_tokens` | 设置 `effort` 级别 |
| Budget 管理 | 应用层控制 | 模型框架自动管理 |
| Temperature | 必须 = 1 | 不受限制 |
| 适用模型 | Claude 3.5 系列 | Claude claude-sonnet-4-6 / claude-opus-4-5 系列 |
| Token 效率 | 固定分配，可能浪费 | 按需分配，更经济 |

### 跨 Provider 的 Thinking 支持矩阵

| Provider | deer-flow | hermes-agent | nanobot |
|----------|-----------|--------------|---------||
| Anthropic Manual | ✓ | ✓ | ✓ |
| Anthropic Adaptive | ✓ | ✓ | 部分 |
| OpenAI o-series | ✓（via extra_body）| ✓（via Codex transport）| 需手动配 |
| Gemini Thinking | ✗ | ✓ | ✗ |
| vLLM/Qwen Thinking | ✓（chat_template_kwargs）| ✗ | ✗ |
| DeepSeek R1 | ✓（OpenAI compat）| ✓ | ✓ |

### 关键设计差异

**hermes-agent 最完整：**
- 支持最多 Provider（Anthropic + OpenAI + Gemini）
- `reasoning_per_turn` 记录每轮推理内容（eval 友好）
- SQLite 持久化 thinking 内容（`reasoning_details` 字段）
- 子 Agent 可独立配置 reasoning_effort

**deer-flow 最自动化：**
- `when_thinking_enabled` config 块自动处理 thinking 开关
- 80% auto-budget 省去手动计算
- 三种 Provider 的关闭方式统一处理（anthropic/openai_compat/vllm_qwen）

**nanobot 最精简：**
- Anthropic-first（主打 Anthropic 深度特性）
- Prompt caching + thinking 同时使用（成本优化）
- 配置简单（`thinking_budget: 8000` 一行）
