# Hermes Agent 记忆系统架构

> 完整的三层记忆设计：内置文件记忆（MEMORY.md / USER.md）、外部记忆插件（Honcho / Holographic / RetainDB 等）、会话回溯（session_search FTS5 全文搜索）。

---

## 一、系统概览

Hermes Agent 的记忆系统不是单一存储，而是**三层分层架构**：

```
┌─────────────────────────────────────────────────────────────┐
│                     System Prompt (Volatile 层)               │
│  ┌──────────────────┐  ┌──────────────┐  ┌───────────────┐ │
│  │  MEMORY.md 快照   │  │ USER.md 快照  │  │ 外部记忆块     │ │
│  │  (agent个人笔记)  │  │  (用户画像)   │  │ (Honcho等)    │ │
│  └──────┬───────────┘  └──────┬───────┘  └───────┬───────┘ │
│         │                     │                   │          │
└─────────┼─────────────────────┼───────────────────┼──────────┘
          │                     │                   │
    会话开始一次注入      会话开始一次注入      每轮API调用前注入
    (冻结快照，不随轮次变)  (冻结快照)          (prefetch结果)
          │                     │                   │
╔═════════╧═════════════════════╧═══════════════════╧══════════╗
║                    核心运行时                                   ║
║  ┌─────────────────────┐  ┌───────────────────────────────┐  ║
║  │  MemoryStore        │  │  MemoryManager                │  ║
║  │  内置文件存储        │  │  外部 provider 管理器           │  ║
║  │  (tools/memory_tool)│  │  (agent/memory_manager)       │  ║
║  └─────────┬───────────┘  └────────────┬──────────────────┘  ║
║            │                           │                      ║
╚════════════╪═══════════════════════════╪══════════════════════╝
             │                           │
    ┌────────▼────────┐     ┌────────────▼──────────────┐
    │  ~/.hermes/memories/   │  plugins/memory/<provider>/  │
    │  MEMORY.md           │  honcho/ holographic/       │
    │  USER.md             │  retaindb/ mem0/ hindsight/ │
    │  §-delimited entries │  openviking/ byterover/     │
    └─────────────────────┘  └───────────────────────────┘

┌──────────────────────────────────────────────────────────────┐
│  Session Search (跨会话回溯)                                    │
│  SQLite FTS5 全文索引 → 搜索历史会话中的所有消息                  │
│  tools/session_search_tool.py                                 │
└──────────────────────────────────────────────────────────────┘
```

**三层各司其职：**

| 层级 | 存什么 | 什么时候用 | 例子 |
|------|--------|------------|------|
| **MEMORY.md** | Agent 学到的稳定事实 | 每个会话的系统提示词 | "项目用 pytest -n auto"、"用户偏好中文回复" |
| **USER.md** | 用户身份与偏好 | 每个会话的系统提示词 | "用户是后端工程师，叫张工"、"代码风格偏好 snake_case" |
| **外部记忆插件** | 基于模型的泛化记忆 | 每轮 API 调用前的 prefetch | 话题回想、用户多维度画像、情感分析 |
| **session_search** | 过去会话的原始消息 | Agent 主动查询时 | "上次那个数据库迁移我们怎么改的？" |

**关键区分：** memory 是"从过去提炼出的知识"，而 session_search 是"回到过去看原文"。

---

## 二、第一层：内置文件记忆（MemoryStore）

### 2.1 存储位置

```
~/.hermes/memories/
├── MEMORY.md      ← Agent 的个人笔记：环境事实、项目约定、工具特性、经验教训
└── USER.md        ← Agent 对用户的了解：偏好、沟通风格、习惯、角色
```

每个 Hermes profile 有独立的 `memories/` 目录（通过 `HERMES_HOME` 环境变量实现 profile 隔离）。

### 2.2 存储格式

条目用 `§`（section sign）分隔：

```
用户偏好中文回复，代码注释用英文§
用户的时区是 UTC+8§
项目位于 /home/user/app，Python 3.12 环境
```

- 分隔符：`\n§\n`（节号前后各一个换行）
- 可多行：条目内部可以包含完整段落
- 容量上限：MEMORY.md 默认 2,200 字符，USER.md 默认 1,375 字符

### 2.3 MemoryStore 核心机制

**位置：** `tools/memory_tool.py:113-607`

```
class MemoryStore
├── memory_entries: List[str]     ← 实时列表（工具写入会修改）
├── user_entries: List[str]       ← 实时列表
├── _system_prompt_snapshot       ← 冻结快照（系统提示词注入用）
│   {memory: "渲染后的 markdown 块", user: "渲染后的 markdown 块"}
│
├── load_from_disk()              ← 从磁盘加载 + 创建快照
├── add(target, content)          ← 追加条目（去重、容量检查）
├── replace(target, old, new)     ← 子串匹配替换（最旧的匹配项）
├── remove(target, old)           ← 子串匹配删除
├── format_for_system_prompt(t)   ← 返回冻结快照（不是实时状态）
├── save_to_disk(target)          ← 原子写入磁盘（tempfile + os.replace）
│
└── 安全守卫：
    ├── _scan_memory_content()    ← 写入前扫描注入/越狱模式
    ├── _detect_external_drift()  ← 检测外部修改（shell append 等）
    ├── _sanitize_entries_for_snapshot() ← 快照中屏蔽有问题条目
    └── _file_lock()              ← 文件锁（Unix fcntl / Windows msvcrt）
```

### 2.4 冻结快照模式（核心设计决策）

这是理解内置记忆的关键：

```
                   会话开始时                    会话进行中（写 memory）
                   ──────────                    ──────────────────────
MEMORY.md 磁盘 → load_from_disk() → snapshot    memory(action=add) → 写磁盘 ✓
                                        ↓           ↓              ↓
                              注入系统提示词      内存中的 entries 更新  snapshot 不变 ✗
                              （每次都相同）
```

**为什么这么设计？**
1. 系统提示词必须在整个会话生命周期中位级稳定（byte-stable），以保证 LLM 提供商的前缀缓存命中
2. 如果每次写记忆都更新系统提示词，前缀缓存就会被破坏 → 每个后续 turn 的输入 token 翻倍 → 用户成本暴增
3. 新写的记忆在**下一次新会话**中才会进入系统提示词

### 2.5 工具 schema（模型视角）

`memory_tool` 函数接受四个参数：

```python
{
    "action": "add" | "replace" | "remove",    # 操作类型
    "target": "memory" | "user",               # 存储目标
    "content": "条目内容",                       # add/replace 需要
    "old_text": "要匹配的子串"                   # replace/remove 需要
}
```

- 没有 `read` action——read 是 `replace` / `remove` 的副作用：匹配失败时返回所有条目列表帮助模型定位
- 子串匹配（不是精确匹配或 ID）：`old_text` 只需是目标条目的唯一子串
- 多匹配时拒绝并返回所有匹配预览

### 2.6 写入门控（Write Gate）

`_apply_write_gate()`（第 609 行）在执行实际写入前可以：
- **Block（拦截）** — 拒绝写入并返回理由
- **Stage（暂存）** — 将写入搁置等待用户审批（`/memory approve` 命令）
- **Pass-through（放行）** — 直接写入（默认模式）

---

## 三、第二层：外部记忆插件（MemoryProvider）

### 3.1 架构

外部记忆通过 **Provider 模式** 接入。`MemoryManager` 管理多个 provider，但**同一时间只允许一个外部 provider**。

```
MemoryManager
├── _providers: List[MemoryProvider]       ← 所有注册 provider
│   ├── [0] builtin (内置 MEMORY.md/USER.md)  ← 始终存在
│   └── [1] honcho (或 holographic / retaindb 等) ← 最多一个外部
│
├── _tool_to_provider: Dict[str, Provider]  ← 工具名→provider 路由
├── _sync_executor: ThreadPoolExecutor     ← 后台同步线程（单个 worker）
│
├── add_provider(provider)                ← 注册 provider（防多外部）
├── build_system_prompt() → str           ← system prompt 块
├── prefetch_all(query) → str             ← 轮前预取（阻塞）
├── queue_prefetch_all(query)             ← 轮后排队预取（非阻塞，后台）
├── sync_all(user, asst, messages)        ← 轮后同步（后台线程）
├── handle_tool_call(name, args)          ← 分发到正确 provider
│
├── on_turn_start()                       ← 轮开始钩子
├── on_session_end()                      ← 会话结束钩子
├── on_session_switch()                   ← 会话切换钩子
├── on_pre_compress() → str               ← 压缩前提取
├── on_memory_write()                     ← 内置 memory 写入时镜像
└── on_delegation()                       ← 子代理完成通知
```

### 3.2 MemoryProvider 接口（ABC）

**位置：** `agent/memory_provider.py:42-297`

每个外部插件必须实现此抽象类。关键方法是：

| 方法 | 调用时机 | 作用 |
|------|----------|------|
| `initialize(session_id, **kwargs)` | Agent 启动时 | 连接后端、创建资源、预热 |
| `system_prompt_block() → str` | 系统提示词构建时 | 提供说明文档或状态信息给模型 |
| `prefetch(query) → str` | 每轮 API 调用前 | 查询相关上下文（作为用户消息后缀注入） |
| `queue_prefetch(query)` | 每轮结束后 | 后台预取，为下一轮的 `prefetch()` 准备 |
| `sync_turn(user, asst, messages)` | 每轮结束后（后台） | 将该轮对话写入后端 |
| `get_tool_schemas() → List[dict]` | Agent 初始化时 | 声明此 provider 暴露的工具 |
| `handle_tool_call(name, args)` | 模型调用时 | 实际执行工具 |
| `shutdown()` | Agent 销毁时 | 清理资源 |

### 3.3 现有外部 Provider

| Provider | 目录 | 核心思路 |
|----------|------|----------|
| **Honcho** | `plugins/memory/honcho/` | **辩证式用户建模**（源自柏拉图辩证对话）。建一个慢变化的用户表示（identity/ephemera/off-topic/physical），每轮对话后增量更新 |
| **Holographic** | `plugins/memory/holographic/` | **全息折合表示（HRR）**。用向量符号架构（Vector Symbolic Architecture）编码语义组合性：将 token/词组映射为 1024 维相位向量，通过绑定（bind=binding）、解绑（unbind）、叠加（bundle）三个代数操作实现复合语义编码 |
| **Hindsight** | `plugins/memory/hindsight/` | 时序记忆后端 |
| **RetainDB** | `plugins/memory/retaindb/` | 关系型记忆存储 |
| **Mem0** | `plugins/memory/mem0/` | 云端记忆服务 |
| **OpenViking** | `plugins/memory/openviking/` | 开源记忆后端 |
| **ByteRover** | `plugins/memory/byterover/` | 本地嵌入式记忆后端 |
| **SuperMemory** | `plugins/memory/supermemory/` | 增强记忆后端 |

### 3.4 外部记忆的注入路径

外部记忆通过**两条路径**影响模型行为：

**路径一：系统提示词块**
```
MemoryManager.build_system_prompt()         ← agent/system_prompt.py:323
  → provider.system_prompt_block()           ← 说明如何使用 provider 工具
  → 注入到 Volatile 层（随时间变化，不进缓存控制）
```

**路径二：每轮上下文注入**
```
pre_llm_call 钩子                ← turn_context.py:318
  → MemoryManager.prefetch_all(query)       ← agent/turn_context.py:372
    → provider.prefetch(query)              ← 返回当前轮相关的记忆
      → 包裹在 <memory-context> 标签中
        → 注入到用户消息的末尾（不是 system prompt！）
```

注入格式（`agent/memory_manager.py:235-249`）：
```
<memory-context>
[System note: The following is recalled memory context, NOT new user input.
Treat as authoritative reference data — this is the agent's persistent memory
and should inform all responses.]

<provider 返回的纯文本上下文>

</memory-context>
```

`StreamingContextScrubber`（`memory_manager.py:70-233`）确保这些 `<memory-context>` 块在流式输出到 UI 时被过滤掉——用户永远不会看到原始记忆数据污染屏幕。

### 3.5 Honcho 的辩证式用户建模（重点案例）

Honcho 是最具代表性的外部 provider。源自 [Honcho](https://honcho.dev) 项目，核心思想如下：

```
User Message → Honcho 分析 → 更新用户状态：
  {
    "identity":   "用户的后端工程师身份描述",      ← 缓慢变化
    "ephemera":   "今天想重构数据库，心情不错",      ← 快速变化
    "off_topic":  "昨晚看了什么电影",               ← 与工作无关
    "physical":   "使用 MacBook, 咖啡店工作"        ← 环境状态
  }

→ 持久化为 Honcho 文档
→ 下次 prefetch() 时，根据用户消息语义检索最相关的历史状态
```

**Initialize 钩子**：为每个 Hermes profile 创建独立的 Honcho 用户/会话
**Sync 钩子**：每轮对话后向 Honcho 写入新的用户消息
**Prefetch 钩子**：根据当前查询返回相关的历史状态
**Memory write 镜像**：内置 memory 工具写入时，同步到 Honcho 后端

---

## 四、第三层：会话搜索（Session Search）

### 4.1 定位

`session_search` 不是"记忆"——它是**对过去会话原始消息的全文搜索**。

```
memory        → "我是怎么知道这个的？"（Agent 提炼的知识）
session_search → "那次我们具体讨论了什么？"（原始对话记录）
```

### 4.2 实现方案

**位置：** `tools/session_search_tool.py`

**底层存储：** SQLite FTS5（全文搜索）索引，建立在 `hermes_state.py` 中的 `session_db` 之上。

**三种调用模式（无需显式 mode 参数，从参数中推断）：**

| 模式 | 参数 | 行为 | LLM 成本 |
|------|------|------|----------|
| **Discovery** | `query` | FTS5 搜索 → 按会话族系统重 → 返回 top N 结果（含 ±5 消息上下文窗口 + 会话首尾书签） | 零 |
| **Scroll** | `session_id` + `around_message_id` | 返回以锚点为中心的 ±N 消息窗口；要滚动时重新锚定到窗口的首/末消息 | 零 |
| **Browse** | 无参数 | 按时间倒序列出最近会话（标题、摘要、时间戳） | 零 |

**关键设计：零 LLM 成本。** 三种模式全部直接从 SQLite DB 返回原始消息——不调用任何 LLM 做摘要。与旧版不同，没有 "summary mode"（用 LLM 概括结果），因为那会增加额外的推理开销且可能丢失细节。

### 4.3 工具 schema 描述中的行为指导

`session_search` 的 schema description 会告诉模型：
- **何时用 session_search**：用户提到之前的事、或怀疑存在跨会话上下文
- **何时用 memory**：需要长期记住的稳定事实（偏好、约定、环境细节）
- **不要混淆两者**：不要把 session_search 当成 memory 用

---

## 五、完整生命周期

以一个完整的"用户发消息 → Agent 回复"turn 为例，记忆系统的参与点：

```
Turn 开始 (turn_context.py)
  ├── [1] 如果 _cached_system_prompt 为空，调用 build_system_prompt_parts()
  │        ├── Stable 层构建
  │        ├── Context 层构建
  │        └── Volatile 层构建
  │            ├── _memory_store.format_for_system_prompt("memory")  ← 冻结快照
  │            ├── _memory_store.format_for_system_prompt("user")    ← 冻结快照
  │            └── _memory_manager.build_system_prompt()             ← provider 块
  │
  ├── [2] pre_llm_call 钩子触发             ← turn_context.py:320
  │
  ├── [3] MemoryManager.prefetch_all(query) ← turn_context.py:372
  │        └── provider.prefetch(query)     ← 包裹 <memory-context>
  │
  ├── [4] 用户消息 + prefetch 上下文 → 注入到 api_messages
  │
API 调用 ──→ 模型回复 ──→ tool_calls (如有)
  │
  ├── [5] 如果模型调用了 memory 工具：
  │        ├── MemoryTool.add/replace/remove → 写 MEMORY.md 磁盘
  │        ├── _apply_write_gate() → 可能的审批拦截
  │        └── MemoryManager.on_memory_write() → 镜像到外部 provider
  │
  ├── [6] 如果模型调用了 session_search 工具：
  │        └── FTS5 搜索 → 返回历史消息窗口
  │
  └── [7] Turn 完成 (turn_finalizer.py)
           ├── MemoryManager.sync_all(user, asst, messages)  ← 后台线程写入
           │    └── provider.sync_turn(user_content, assistant_content, messages)
           ├── MemoryManager.queue_prefetch_all(query)       ← 后台线程预取
           │    └── provider.queue_prefetch(query)
           └── Nudge 逻辑：每 N 轮提醒模型回顾 memory(action=read)
```

---

## 六、安全设计

### 6.1 注入防护

记忆内容会进入系统提示词，所以必须防止恶意内容通过记忆注入绕过安全护栏：

1. **写入时扫描**：`memory(action=add)` 写入前调用 `_scan_memory_content()` 匹配 `tools/threat_patterns.py` 中的 strict 模式
2. **加载时二次扫描**：`load_from_disk()` 对每个条目调用 `_sanitize_entries_for_snapshot()`，匹配的条目在快照中被替换为 `[BLOCKED: ...]` 占位符——但实时列表保留原文，让用户能看到并删除
3. **外部漂移检测**：`_detect_external_drift()` 检测非 memory 工具（patch_tool、shell append、手动编辑）写入的内容，发现漂移时创建 `.bak` 备份并拒绝后续 mutations

### 6.2 流式输出清理

外部记忆的 `<memory-context>` 块通过 `StreamingContextScrubber` 从流式输出中过滤——即使 provider 的连接断开导致标签不完整，状态机也能正确处理部分标签。

---

## 七、配置项

在 `config.yaml` 中：

```yaml
agent:
  memory_management:
    review_interval: 5       # 每 N 个 turn 提醒模型回顾记忆
    memory_char_limit: 2200  # MEMORY.md 字符上限
    user_char_limit: 1375   # USER.md 字符上限

memory:
  provider: honcho           # 外部记忆后端 (honcho/holographic/retaindb/mem0/...)
```

---

## 八、相关文件

| 文件 | 角色 |
|------|------|
| `tools/memory_tool.py` | 内置 MemoryStore + memory 工具注册 |
| `agent/memory_manager.py` | MemoryManager — 外部 provider 管理器 |
| `agent/memory_provider.py` | ABC 抽象基类 |
| `tools/session_search_tool.py` | 会话搜索工具（FTS5） |
| `agent/system_prompt.py:307-323` | 记忆注入到 Volatile 层 |
| `agent/turn_context.py:359-374` | 每轮 prefetch + 插件上下文注入 |
| `agent/prompt_builder.py:143-179` | MEMORY_GUIDANCE + SESSION_SEARCH_GUIDANCE |
| `agent/turn_finalizer.py` | 轮后 sync + queue_prefetch |
| `tools/threat_patterns.py` | 安全扫描模式库（与 memory 共用） |
| `plugins/memory/honcho/` | Honcho 辩证用户建模 |
| `plugins/memory/holographic/` | 全息折合表示（HRR） |
| `plugins/memory/retaindb/` | RetainDB 关系型记忆 |
| `plugins/memory/mem0/` | Mem0 云端记忆 |
| `plugins/memory/hindsight/` | Hindsight 时序记忆 |
| `plugins/memory/openviking/` | OpenViking 开源后端 |
| `plugins/memory/byterover/` | ByteRover 嵌入式后端 |
| `plugins/memory/supermemory/` | SuperMemory 增强后端 |
| `hermes_state.py` | session_db — FTS5 索引建立与维护 |
