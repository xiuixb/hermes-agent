# 会话内消息拼接（Session Message Assembly）

> Hermes Agent 在单个会话中如何逐轮组装 API 请求消息，包括角色交替规则、工具调用/结果的注入方式、以及每轮 API 调用前消息列表的构建全过程。

---

## 一、总览

一个会话由多个 **turn**（用户轮次）组成。每个 turn 内部可能包含多轮 **API call**（LLM 调用），因为模型可能会多次调用工具才能完成任务。消息列表 `messages` 是一维的 `List[Dict]`，在持续增长中维护严格的角色交替。

**核心文件：**

| 文件 | 作用 |
|------|------|
| `agent/conversation_loop.py` (371-4250行) | 主对话循环：turn 上下文、API 重试、工具分发、消息追加 |
| `agent/turn_context.py` | 每 turn 启动时的准备逻辑（系统提示词恢复、上下文压缩等） |
| `agent/chat_completion_helpers.py` (555行+) | `build_api_kwargs()` — 根据 api_mode 构建最终 API 请求参数 |
| `agent/prompt_caching.py` | Anthropic 缓存控制断点注入 |
| `run_agent.py` (4671行+) | `_build_api_kwargs()`, `_build_assistant_message()` 等转发方法 |

---

## 二、消息列表的初始状态

```
空 → [user_msg_1, assistant_msg_1, tool_rslt_1, ... , user_msg_N]
```

- **所有角色：** `system`, `user`, `assistant`, `tool`
- **角色交替规则：** 对于 OpenAI Chat Completions API，必须是 `user → assistant → tool → assistant → ...` 的交替。绝不允许连续两个同角色消息。

---

## 三、单 Turn 完整流程

```
用户发消息
    │
    ▼
┌──────────────────────────────────────────────────┐
│ Step 1:  Prologue (build_turn_context)            │
│   · 重置 retry 计数器 / 迭代预算                  │
│   · 恢复或构建系统提示词                          │
│   · Preflight 上下文压缩                          │
│   · pre_llm_call 插件钩子 (上下文注入到用户消息)   │
│   · 外部记忆预取                                  │
│   · messages.append({"role":"user","content":...})│
└──────────────────────┬───────────────────────────┘
                       │
                       ▼
╔══════════════════════════════════════════════════╗
║     Loop: 每个 API 迭代 (直到 done 或上限)        ║
║                                                  ║
║  ┌────────────────────────────────────────────┐  ║
║  │ Step 2: 构建 api_messages (每次 API 调用前)  │  ║
║  │   · 从 messages 浅拷贝                      │  ║
║  │   · 注入临时上下文到当前用户消息              │  ║
║  │   · 清理内部字段 (reasoning, finish_reason) │  ║
║  │   · 规范化 tool_call JSON (sort_keys)       │  ║
║  └──────────────────┬─────────────────────────┘  ║
║                     │                            ║
║  ┌──────────────────▼─────────────────────────┐  ║
║  │ Step 3: 前置系统提示词 + 预填充消息           │  ║
║  │   · api_messages = [{system_prompt}] + msgs │  ║
║  │   · 插入 prefill_messages                  │  ║
║  └──────────────────┬─────────────────────────┘  ║
║                     │                            ║
║  ┌──────────────────▼─────────────────────────┐  ║
║  │ Step 4: 缓存标记 + 安全清理                   │  ║
║  │   · apply_anthropic_cache_control()         │  ║
║  │   · _sanitize_api_messages()               │  ║
║  │   · _drop_thinking_only_and_merge_users()  │  ║
║  └──────────────────┬─────────────────────────┘  ║
║                     │                            ║
║  ┌──────────────────▼─────────────────────────┐  ║
║  │ Step 5: build_api_kwargs → API 调用          │  ║
║  │   · 根据 api_mode 转换消息格式               │  ║
║  │   · anthropic_messages / codex_responses     │  ║
║  │   · 或标准 chat_completions                 │  ║
║  └──────────────────┬─────────────────────────┘  ║
║                     │                            ║
║                     ▼                            ║
║          ┌─────────────────────┐                 ║
║          │  有 tool_calls?     │                 ║
║          └───┬─────────────┬───┘                 ║
║       YES    │             │    NO                ║
║              ▼             ▼                      ║
║  ┌──────────────┐  ┌──────────────┐              ║
║  │ Step 6a:     │  │ Step 6b:     │              ║
║  │ 执行工具      │  │ 返回最终结果  │              ║
║  │ → 追加结果    │  │ → 退出循环   │              ║
║  │ → continue   │  │ → break      │              ║
║  └──────────────┘  └──────────────┘              ║
║                                                  ║
╚══════════════════════════════════════════════════╝
                       │
                       ▼
┌──────────────────────────────────────────────────┐
│ Step 7:  Epilogue (Post-turn)                    │
│   · 记忆审查 nudge                               │
│   · 技能创建 nudge                               │
│   · 会话持久化                                   │
└──────────────────────────────────────────────────┘
```

---

## 四、Step 2 详解：构建 api_messages

**位置：** `agent/conversation_loop.py:606-649`

这是每轮 API 调用前最关键的消息处理步骤：

```python
api_messages = []
for idx, msg in enumerate(messages):
    api_msg = msg.copy()  # 浅拷贝，不修改原始 messages

    # 1) 注入临时上下文到当前 turn 的用户消息
    if idx == current_turn_user_idx and msg.get("role") == "user":
        _injections = []
        if _ext_prefetch_cache:                    # 外部记忆预取结果
            _injections.append(build_memory_context_block(_ext_prefetch_cache))
        if _plugin_user_context:                   # pre_llm_call 插件输出
            _injections.append(_plugin_user_context)
        if _injections:
            api_msg["content"] = _base + "\n\n" + "\n\n".join(_injections)

    # 2) 传递推理内容 (reasoning_content / codex_reasoning_items)
    agent._copy_reasoning_content_for_api(msg, api_msg)

    # 3) 清理内部字段 — API 不接受这些
    api_msg.pop("reasoning", None)      # 轨迹存储用，API 不需要
    api_msg.pop("finish_reason", None)  # 某些严格 API (Mistral) 拒绝
    api_msg.pop("_thinking_prefill", None) # 内部标记

    # 4) 严格 API 清洗 tool_calls 格式
    if provider 要求严格格式:
        agent._sanitize_tool_calls_for_strict_api(api_msg)

    api_messages.append(api_msg)
```

**关键点：** `api_messages` 此时还不包含系统提示词——它是纯对话历史的副本。

---

## 五、Step 3 详解：前置系统提示词

**位置：** `agent/conversation_loop.py:666-678`

```python
# 拼接缓存的系统提示词 + 临时系统提示词
effective_system = active_system_prompt or ""      # _cached_system_prompt
if agent.ephemeral_system_prompt:                   # 按轮次的临时追加
    effective_system = (effective_system + "\n\n" + agent.ephemeral_system_prompt).strip()

# 前置到消息列表最前面
if effective_system:
    api_messages = [{"role": "system", "content": effective_system}] + api_messages

# 在系统提示词之后、对话历史之前插入预填充消息
if agent.prefill_messages:
    sys_offset = 1
    for pfm in agent.prefill_messages:
        api_messages.insert(sys_offset, pfm.copy())
```

**此时 api_messages 的结构：**

```
[0] {role: "system",    content: "完整的三层系统提示词 + 临时追加"}
[1] {role: PRE_ROLE,    ...}  ← 可选的 prefill 消息（仅在 reasoning 场景）
[2] {role: "user",      content: "用户消息 + 外部记忆 + 插件上下文"}
[3] {role: "assistant", content: "reply", tool_calls: [...]}   ← 历史回放
[4] {role: "tool",      content: "result", tool_call_id: "..."} ← 历史回放
[5] {role: "assistant", content: "reply", tool_calls: [...]}   ← 历史回放
[6] {role: "tool",      content: "result", tool_call_id: "..."}
...
[N] {role: "user",      content: "当前轮用户消息"}    ← current_turn_user_idx
```

---

## 六、Step 4 详解：缓存标记与安全清理

```python
# 1) 注入 Anthropic 缓存断点
if agent._use_prompt_caching:
    api_messages = apply_anthropic_cache_control(
        api_messages,
        cache_ttl=agent._cache_ttl,
        native_anthropic=agent._use_native_cache_layout,
    )
```

`apply_anthropic_cache_control` 在以下位置设置 `cache_control` 断点：
- 系统提示词（第一条消息） → 缓存所有 token
- 最后 3 条消息 → 缓存最近上下文

```python
# 2) 安全清理：移除孤立的 tool 结果、补全缺失的结果
api_messages = agent._sanitize_api_messages(api_messages)

# 3) 删除仅含 thinking 无实际内容的 assistant 轮次，合并连续 user 消息
api_messages = agent._drop_thinking_only_and_merge_users(api_messages)

# 4) 规范化：strip 所有 content 字符串，tool_call arguments JSON 排序
for am in api_messages:
    if isinstance(am.get("content"), str):
        am["content"] = am["content"].strip()
for am in api_messages:
    for tc in am.get("tool_calls", []):
        tc["function"]["arguments"] = json.dumps(
            json.loads(args), separators=(",", ":"), sort_keys=True
        )
```

---

## 七、Step 5 详解：构建最终 API 请求

**位置：** `agent/chat_completion_helpers.py:555-661`

根据 `agent.api_mode` 选择不同的消息格式：

| api_mode | 消息处理方法 | 最终请求格式 |
|----------|-------------|-------------|
| `chat_completions` | 直接使用 api_messages（标准 OpenAI 格式） | `{model, messages, tools, ...}` |
| `anthropic_messages` | `_prepare_anthropic_messages_for_api()` — 转换图片 URL、system→顶层字段 | `{model, system, messages=[{role,content:[...]}], tools, ...}` |
| `codex_responses` | `_prepare_messages_for_non_vision_model()` — 剥离图片 + Responses API 转换 | `{model, input:[...], tools, ...}` |
| `bedrock_converse` | Bedrock Adapter 自行转换 | AWS Bedrock 原生格式 |

**Anthropic 特化 — Developer Role 交换：**

```python
# prompt_caching.py / anthropic_adapter.py
# 当模型名包含 "gpt-5" 或 "codex" 时，
# 系统消息的角色从 "system" 改为 "developer"
# （OpenAI 新模型的 developer role 给我更强的指令遵循权重）
DEVELOPER_ROLE_MODELS = ("gpt-5", "codex")
```

---

## 八、Step 6a 详解：工具执行与结果追加

**位置：** `agent/conversation_loop.py:3475-3804`

当 API 返回 `tool_calls` 时：

### 8.1 消息追加顺序

```
messages 末尾的当前状态:
  [..., {"role": "user", "content": "当前用户消息"}, {"role": "assistant", "content": "...", "tool_calls": [...]}]

↓ assistant_msg 被构建并追加

messages.append(assistant_msg)   ← 包含 tool_calls 的 assistant 消息
  [..., {"role": "user", "content": "..."}, {"role": "assistant", "tool_calls": [...], "content": "..."}]

↓ 每个 tool_call 执行后追加 tool 角色消息

for tool_result in executed_results:
    messages.append({
        "role": "tool",
        "name": tc.function.name,
        "tool_call_id": tc.id,
        "content": tool_result_content,  # 工具执行的返回值
    })
  [..., {"role": "assistant", "tool_calls": [...]},
       {"role": "tool", "tool_call_id": "call_1", "content": "result1"},
       {"role": "tool", "tool_call_id": "call_2", "content": "result2"}]

↓ continue — 循环回到顶部，模型下一轮看到这些结果

continue  # 等效于 "goto Step 2"，用扩展后的 messages 重新构建 api_messages
```

### 8.2 角色交替维护

工具执行后 `messages` 的尾部是：

```
... → assistant(tool_calls) → tool → tool → tool → ...
```

下一次 API 调用时：
- 模型看到 `tool` 消息作为输入
- 模型返回新的 `assistant` 消息（可能含 tool_calls 或最终文本）
- 这保持了 `assistant ↔ tool ↔ assistant` 的正确交替

### 8.3 模型的视角

```
Turn N, 迭代1:  system, user, [history...]
                → assistant: tool_calls [read_file, terminal]
                执行 read_file → tool: "文件内容..."
                执行 terminal  → tool: "命令输出..."

Turn N, 迭代2:  system, user, [history...], assistant(tool_calls), tool, tool
                → assistant: "根据文件内容，我建议..."    ← 最终回复
```

**关键点：** system 提示词在每次迭代中都被重新前置到 api_messages 前面（不走 messages 列表），但在 LLM 看来它始终是上下文的第一条消息。模型看到的完整序列是：

```
[system_prompt] [user_turn_1] [assistant_turn_1_tools] [tool_results] [assistant_final]
```

---

## 九、Step 6b 详解：无工具调用的最终回复

**位置：** `agent/conversation_loop.py:3806-3900`

```python
# No tool calls - this is the final response
final_response = assistant_message.content or ""

# 检测空回复处理 (empty-content retry with prefill)
if not agent._has_content_after_think_block(final_response):
    # 1) 尝试从流式传输中恢复部分内容
    # 2) 如果上一轮 housekeeping 工具调用时已给出回复，直接使用
    # 3) 否则用 prefill 消息重试（最多 2 次）
```

最终追加最终的 assistant 消息到 `messages`：

```python
assistant_msg = agent._build_assistant_message(assistant_message, finish_reason)
messages.append(assistant_msg)
break  # 退出 while 循环
```

---

## 十、中途转向（/steer）的消息注入

**位置：** `agent/conversation_loop.py:522-571`

- Steer 消息只在工具执行后才被注入
- 注入位置：**追溯最后一个 tool 角色的消息**，将 steer 文本追加到该 tool 消息的 content 末尾
- 使用特殊标记包裹：

```
[OUT-OF-BAND USER MESSAGE — a direct message from the user, delivered mid-turn; not tool output]
<用户的中途指令>
[/OUT-OF-BAND USER MESSAGE]
```

- 如果当前没有 tool 消息（首轮 API 调用前），steer 保持 pending 状态，等下一批工具执行后注入
- 这保证了角色交替不被破坏——steer 永不作为独立的 user 消息注入

---

## 十一、关键不变式（Invariants）

1. **System prompt 不在 messages 列表中：** 系统提示词缓存在 `agent._cached_system_prompt` 上，每次 API 调用时才动态前置到 `api_messages`。`messages` 列表和会话数据库中不存储系统提示词。

2. **messages 列表是真相来源：** `messages` 列表存储了所有的 `user`, `assistant`, `tool` 角色消息。`api_messages` 是每轮 API 调用时从此列表构建的副本。

3. **api_messages 是本轮本地副本：** 所有规范化、清理和注入都发生在 `api_messages` 上，不影响 `messages` 原列表。会话持久化始终使用 `messages`。

4. **严格的角色交替：** `messages` 中绝不允许连续两个同角色消息。如果出现异常（如会话加载错误），`_repair_message_sequence()` 会在构建 api_messages 前修复。

5. **时间戳不进 API：** 系统提示词中的时间戳精确到天（防止分钟变化破坏缓存），模型需要精确时间时应使用工具查询。

---

## 十二、完整流程示例

以一次用户问"读取 README.md 并总结"为例：

```
┌─ Turn 1, 迭代 1 ─────────────────────────────────────────────┐
│ messages: [user: "读取 README.md 并总结"]                       │
│ api_messages 构建:                                              │
│   [system: "<三层系统提示词>"]                                   │
│   [user:    "读取 README.md 并总结\n\n[外部记忆上下文]"]          │
│ → API 返回: assistant.tool_calls = [read_file("README.md")]    │
│                                                                 │
│ messages.append(assistant_msg)  # {role:"assistant",tool_calls}│
│ 执行 read_file → 结果追加:                                      │
│ messages.append({role:"tool", content:"<文件全文...>"})          │
│ continue                                                        │
└────────────────────────────────────────────────────────────────┘

┌─ Turn 1, 迭代 2 ─────────────────────────────────────────────┐
│ api_messages 构建:                                              │
│   [system: "<三层系统提示词>"]                                   │
│   [user:    "读取 README.md 并总结"]                             │
│   [assistant: {tool_calls: [read_file("README.md")]}]          │
│   [tool:    "<文件全文...>"]                                     │
│                                                                 │
│ → API 返回: assistant.content = "该项目的功能包括..."            │
│                                                                 │
│ messages.append({role:"assistant", content:"该项目的功能包括..."})│
│ break — 无 tool_calls，最终回复                                  │
│                                                                 │
│ 最终 messages:                                                  │
│   [user, assistant(tool_calls), tool, assistant(final)]         │
└────────────────────────────────────────────────────────────────┘
```

---

## 十三、特殊路径

| 路径 | 描述 |
|------|------|
| **truncated 响应重试** | 响应被截断时，追加 `{role:"user", content:"Please continue..."}` 到 messages，重新 API 调用 |
| **role 交替修复** | `_repair_message_sequence()` 合并相邻同角色消息或插入存根 |
| **Subagent 委托** | `delegate_task` 产生独立的子会话，使用 `_subagent_session_id` |
| **Session 恢复** | 从数据库恢复 messages 时，系统提示词重新构建（因为 `_cached_system_prompt = None`）|
| **上下文压缩** | 触发 `_compress_context()` → 创建新 session_id + 重新构建系统提示词 → 截断早期消息 |
