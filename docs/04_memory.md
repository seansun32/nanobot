# Memory 系统全解析

> 从短期会话到长期记忆，追踪每一条信息的存储、检索、压缩与淘汰机制。

---

## 一、Memory 系统总览

Nanobot 的记忆系统由三个独立层次组成，各自解决不同时间跨度的信息持久化问题：

```
┌─────────────────────────────────────────────────────────────────┐
│                    Memory Architecture                          │
│                                                                 │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │ Layer 1: Session (短期)                                  │   │
│  │ 存储: ~/.nanobot/sessions/{key}.jsonl                    │   │
│  │ 容量: memory_window 条消息 (默认 50)                      │   │
│  │ 粒度: 完整的 user/assistant 消息                          │   │
│  │ 生命周期: 跨请求持久化, /new 清除                          │   │
│  └────────────────────────┬────────────────────────────────┘   │
│                           │ 超出 memory_window 时触发压缩        │
│                           ▼                                     │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │ Layer 2: HISTORY.md (中期)                               │   │
│  │ 存储: {workspace}/memory/HISTORY.md                      │   │
│  │ 容量: 无限 (append-only)                                  │   │
│  │ 粒度: 每次压缩生成一段摘要 (2-5 句)                        │   │
│  │ 生命周期: 永久, 手动清理                                   │   │
│  │ 检索方式: Agent 通过 grep 工具搜索                          │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │ Layer 3: MEMORY.md (长期)                                │   │
│  │ 存储: {workspace}/memory/MEMORY.md                       │   │
│  │ 容量: 单文件, 每次压缩时被 LLM 重写                        │   │
│  │ 粒度: 结构化的事实/偏好/笔记                               │   │
│  │ 生命周期: 永久, LLM 自主更新                               │   │
│  │ 检索方式: 每次请求自动注入 system prompt                    │   │
│  └─────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────┘
```

| 层次 | 文件位置 | 格式 | 写入方 | 读取方 | 容量控制 |
|------|---------|------|--------|--------|---------|
| Session | `~/.nanobot/sessions/{key}.jsonl` | JSONL | AgentLoop | ContextBuilder | `memory_window` 滑动窗口 |
| HISTORY.md | `{workspace}/memory/HISTORY.md` | Markdown | 压缩 Agent | Agent 通过 grep 工具 | 无限追加 |
| MEMORY.md | `{workspace}/memory/MEMORY.md` | Markdown | 压缩 Agent + Agent 工具 | ContextBuilder(自动) | LLM 每次重写 |

---

## 二、Layer 1: Session — 短期会话记忆

### 2.1 数据结构

```
文件: nanobot/session/manager.py:14-52
类: Session
```

```python
@dataclass
class Session:
    key: str                              # "channel:chat_id", 如 "telegram:12345"
    messages: list[dict[str, Any]]        # 完整消息列表
    created_at: datetime                  # 会话创建时间
    updated_at: datetime                  # 最后更新时间
    metadata: dict[str, Any]             # 扩展元数据
    last_consolidated: int = 0            # 已压缩到文件的消息数量
```

每条消息的结构：

```python
{
    "role": "user",                       # 或 "assistant"
    "content": "消息文本内容",
    "timestamp": "2025-01-15T10:30:00",   # ISO 格式时间戳
    "tools_used": ["read_file", "exec"]   # 仅 assistant 消息, 可选
}
```

**关键设计决策**：Session 的 messages 列表是 **append-only** 的。压缩过程不会删除或修改已有消息，只更新 `last_consolidated` 指针来标记哪些消息已经被压缩过。

### 2.2 持久化格式 — JSONL

```
文件: nanobot/session/manager.py:131-145
方法: SessionManager.save()
```

Session 以 JSONL（JSON Lines）格式存储，每行一个 JSON 对象：

```jsonl
{"_type": "metadata", "created_at": "2025-01-15T10:00:00", "updated_at": "2025-01-15T10:30:00", "metadata": {}, "last_consolidated": 10}
{"role": "user", "content": "你好", "timestamp": "2025-01-15T10:00:01"}
{"role": "assistant", "content": "你好！有什么可以帮您的？", "timestamp": "2025-01-15T10:00:03"}
{"role": "user", "content": "帮我读取 README.md", "timestamp": "2025-01-15T10:01:00"}
{"role": "assistant", "content": "README.md 的内容是...", "timestamp": "2025-01-15T10:01:05", "tools_used": ["read_file"]}
```

**第一行**是特殊的元数据行（`_type: "metadata"`），包含会话级别的信息。**后续行**是按时间顺序的消息。

### 2.3 Session 的生命周期

```
                       ┌──────────────────────────────────────────────────────┐
                       │              SessionManager                          │
                       │                                                      │
 用户发消息 ──────────▶│  get_or_create(key)                                  │
                       │    │                                                 │
                       │    ├─ 内存缓存命中? ──yes──▶ 返回缓存的 Session       │
                       │    │                                                 │
                       │    └─ _load(key) ─▶ 读 JSONL ─▶ 解析 ─▶ 缓存       │
                       │                       │                              │
                       │                       └─ 不存在 ─▶ 新建空 Session    │
                       │                                                      │
 处理完成 ──────────▶ │  save(session)                                        │
                       │    └─ 全量写入 JSONL（元数据行 + 所有消息）             │
                       │                                                      │
 /new 命令 ──────────▶│  session.clear() + save() + invalidate()             │
                       │    └─ 清空消息 + 重写文件 + 清除缓存                   │
                       │    └─ 异步触发 archive_all 压缩                       │
                       └──────────────────────────────────────────────────────┘
```

### 2.4 Session 到 Prompt 的映射

```
文件: nanobot/agent/loop.py:267-268
```

```python
# 从 Session 取最近 N 条消息作为历史
history = session.get_history(max_messages=self.memory_window)
```

`get_history()` 的实现（`manager.py:44-46`）：

```python
def get_history(self, max_messages: int = 500) -> list[dict[str, Any]]:
    return [{"role": m["role"], "content": m["content"]} for m in self.messages[-max_messages:]]
```

注意：
- 只保留 `role` 和 `content` 两个字段（去掉 `timestamp`、`tools_used` 等元数据）
- 只取最后 `max_messages` 条（默认 50，由 `memory_window` 配置）
- **不包含** ReAct 循环中间的工具调用过程 — Session 只存储每轮的首尾（用户问题 + 最终回复）

### 2.5 Session Key 的命名规则

```python
# InboundMessage 的 session_key 属性
key = f"{channel}:{chat_id}"
# 例如:
# "telegram:12345"        — Telegram 用户对话
# "whatsapp:8613800138000" — WhatsApp 用户对话
# "cli:direct"            — CLI 直接调用
# "feishu:ou_xxxx"        — 飞书用户对话
```

Session key 被 `safe_filename()` 转换为安全文件名，冒号变下划线：

```python
# manager.py:68-70
def _get_session_path(self, key: str) -> Path:
    safe_key = safe_filename(key.replace(":", "_"))
    return self.sessions_dir / f"{safe_key}.jsonl"
# "telegram:12345" → "telegram_12345.jsonl"
```

---

## 三、Layer 2: HISTORY.md — 中期事件日志

### 3.1 文件格式

HISTORY.md 是一个 append-only 的 Markdown 文件，每次压缩追加一段摘要：

```markdown
[2025-01-15 10:30] User asked about Python packaging. Discussed pyproject.toml vs setup.py,
recommended using pyproject.toml with hatchling backend. User decided to migrate their project.

[2025-01-15 14:00] Helped user debug a failing test in test_auth.py. The issue was a missing
mock for the database connection. Fixed by adding @patch decorator. User also mentioned they
prefer pytest over unittest.

[2025-01-16 09:15] User requested a new API endpoint for user preferences. Created
/api/v1/preferences with GET/PUT methods. Used SQLAlchemy ORM for database access.
Discussed pagination but deferred to a future task.
```

### 3.2 写入机制

```
文件: nanobot/agent/memory.py:24-26
方法: MemoryStore.append_history()
```

```python
def append_history(self, entry: str) -> None:
    with open(self.history_file, "a", encoding="utf-8") as f:
        f.write(entry.rstrip() + "\n\n")
```

特点：
- **Append-only**：只追加，不修改，不删除
- 每个 entry 之间用空行分隔
- entry 内容由 LLM 在压缩时生成（2-5 句话）

### 3.3 检索机制

HISTORY.md **不会** 自动注入 system prompt。Agent 通过以下方式检索：

1. **Agent 使用 grep/exec 工具主动搜索**：system prompt 中明确告知 Agent 可以 grep 搜索 HISTORY.md
2. **Agent 使用 read_file 工具读取全文**：适合需要完整上下文的场景

System prompt 中的相关指引（`context.py:101,110`）：

```
- History log: {workspace_path}/memory/HISTORY.md (grep-searchable)
...
To recall past events, grep {workspace_path}/memory/HISTORY.md
```

### 3.4 容量控制

**没有容量控制。** HISTORY.md 会无限增长。这在长期运行的 Agent 中可能成为问题。

---

## 四、Layer 3: MEMORY.md — 长期事实记忆

### 4.1 初始化模板

```
文件: nanobot/cli/commands.py:253-268
触发: nanobot init
```

```markdown
# Long-term Memory

This file stores important information that should persist across sessions.

## User Information

(Important facts about the user)

## Preferences

(User preferences learned over time)

## Important Notes

(Things to remember)
```

### 4.2 读取机制

```
文件: nanobot/agent/memory.py:16-19
方法: MemoryStore.read_long_term()
```

```python
def read_long_term(self) -> str:
    if self.memory_file.exists():
        return self.memory_file.read_text(encoding="utf-8")
    return ""
```

### 4.3 注入 System Prompt

MEMORY.md 的内容在 **每次请求** 时自动注入 system prompt：

```
文件: nanobot/agent/context.py:48-51
```

```python
# 构建 system prompt 时:
memory = self.memory.get_memory_context()
if memory:
    parts.append(f"# Memory\n\n{memory}")
```

`get_memory_context()` 返回格式化后的内容（`memory.py:28-30`）：

```python
def get_memory_context(self) -> str:
    long_term = self.read_long_term()
    return f"## Long-term Memory\n{long_term}" if long_term else ""
```

最终在 system prompt 中的样子：

```
# Memory

## Long-term Memory
# Long-term Memory

This file stores important information that should persist across sessions.

## User Information
- User is a Python developer based in Shanghai
- Prefers Chinese for conversation

## Preferences
- Uses pytest for testing
- Prefers pyproject.toml over setup.py
```

### 4.4 写入机制 — 两种路径

#### 路径 1：LLM 压缩时自动写入

压缩 Agent 生成 `memory_update` 字段，覆写整个文件：

```python
# loop.py:404-406
if update := result.get("memory_update"):
    if update != current_memory:          # 只在内容有变化时写入
        memory.write_long_term(update)
```

这意味着 **每次压缩都可能重写整个 MEMORY.md**。LLM 需要在 prompt 中收到当前 MEMORY.md 的内容，在此基础上新增/修改/删除信息。

#### 路径 2：Agent 通过工具直接写入

System prompt 告知 Agent 可以直接写入 MEMORY.md（`context.py:109`）：

```
When remembering something important, write to {workspace_path}/memory/MEMORY.md
```

Agent 可以使用 `write_file` 或 `edit_file` 工具直接修改 MEMORY.md。这种方式是 **Agent 主动行为**，不受压缩流程控制。

### 4.5 潜在冲突

两种写入路径可能产生竞态条件：

```
时间线:
  T1: 用户发送消息
  T2: 消息数 > memory_window, 触发异步压缩任务
  T3: Agent 处理用户请求, 通过 write_file 修改 MEMORY.md
  T4: 压缩任务完成, 用旧版 MEMORY.md + 旧对话生成 memory_update, 覆写 MEMORY.md
  → T3 的修改被 T4 覆盖!
```

当前代码没有对此加锁或做冲突检测。

---

## 五、Memory 压缩流程详解

### 5.1 触发条件

```
文件: nanobot/agent/loop.py:263-264
```

```python
if len(session.messages) > self.memory_window:
    asyncio.create_task(self._consolidate_memory(session))
```

**触发条件**：`session.messages` 数量超过 `memory_window`（默认 50）。
**执行方式**：`asyncio.create_task()` — **异步后台执行**，不阻塞当前请求处理。

另一个触发点是 `/new` 命令（`loop.py:246-256`）：

```python
if cmd == "/new":
    messages_to_archive = session.messages.copy()
    session.clear()
    self.sessions.save(session)
    self.sessions.invalidate(session.key)

    async def _consolidate_and_cleanup():
        temp_session = Session(key=session.key)
        temp_session.messages = messages_to_archive
        await self._consolidate_memory(temp_session, archive_all=True)

    asyncio.create_task(_consolidate_and_cleanup())
```

`/new` 命令会：
1. 复制当前消息列表
2. 立即清空 Session 并保存
3. 在后台用复制的消息执行 `archive_all=True` 压缩

### 5.2 压缩算法

```
文件: nanobot/agent/loop.py:337-414
方法: AgentLoop._consolidate_memory()
```

```
_consolidate_memory(session, archive_all=False)
│
├─ archive_all = True (来自 /new):
│   └─ old_messages = session.messages  (全部消息)
│
├─ archive_all = False (自动触发):
│   │
│   ├─ keep_count = memory_window // 2    (默认 25)
│   │
│   ├─ 检查: messages 总数 <= keep_count?
│   │   └─ 是 → return (无需压缩)
│   │
│   ├─ 检查: 有新消息未压缩?
│   │   └─ messages_to_process = len(messages) - last_consolidated
│   │   └─ <= 0 → return (已全部压缩)
│   │
│   └─ old_messages = messages[last_consolidated : -keep_count]
│       (从上次压缩位置到保留窗口之前的消息)
│
├─ 格式化对话为文本:
│   for m in old_messages:
│       "[2025-01-15T10:30] USER: 你好"
│       "[2025-01-15T10:31] ASSISTANT [tools: read_file]: README 内容是..."
│
├─ 读取当前 MEMORY.md 内容
│
├─ 构建压缩 prompt (要求返回 JSON):
│   ┌───────────────────────────────────────────────────────────┐
│   │ system: "You are a memory consolidation agent.            │
│   │          Respond only with valid JSON."                   │
│   │                                                           │
│   │ user:   "Process this conversation and return JSON:       │
│   │          1. history_entry: 2-5 句摘要, 带时间戳            │
│   │          2. memory_update: 更新后的长期记忆                 │
│   │                                                           │
│   │          ## Current Long-term Memory                      │
│   │          {当前 MEMORY.md 内容}                              │
│   │                                                           │
│   │          ## Conversation to Process                       │
│   │          {格式化的对话文本}"                                 │
│   └───────────────────────────────────────────────────────────┘
│
├─ 调用 LLM (使用与主循环相同的 model)
│
├─ 解析 LLM 返回的 JSON:
│   ├─ 去除可能的 Markdown 代码块包裹
│   ├─ json.loads(text)
│   │
│   ├─ result["history_entry"]:
│   │   └─ memory.append_history(entry)  → 追加到 HISTORY.md
│   │
│   └─ result["memory_update"]:
│       └─ if 内容有变化:
│           └─ memory.write_long_term(update)  → 覆写 MEMORY.md
│
└─ 更新 last_consolidated 指针:
    ├─ archive_all: last_consolidated = 0
    └─ 正常:       last_consolidated = len(messages) - keep_count
```

### 5.3 压缩窗口示意

```
Session messages (共 60 条, memory_window=50, keep_count=25):

  ┌─────────┬──────────────────────────────────┬─────────────────────┐
  │ 已压缩  │          本次要压缩的              │     保留窗口         │
  │ (10条)  │          (25条)                   │     (25条)          │
  │         │                                   │                     │
  │ index   │ index                             │ index               │
  │ 0..9    │ 10..34                            │ 35..59              │
  │         │                                   │                     │
  │ last_   │ old_messages =                    │ -keep_count:        │
  │ consoli │ messages[10:35]                   │ 最后 25 条          │
  │ dated=10│                                   │                     │
  └─────────┴──────────────────────────────────┴─────────────────────┘
                       │
                       ▼
               LLM 压缩为:
               ┌─────────────────────────────────────────┐
               │ history_entry: "[2025-01-15 14:00] ..."  │ → HISTORY.md
               │ memory_update: "# Long-term Memory..."  │ → MEMORY.md
               └─────────────────────────────────────────┘
                       │
                       ▼
               更新: last_consolidated = 35
```

### 5.4 压缩是增量的

关键细节：Session 的 messages 列表 **不会因为压缩而缩短**。

```python
# Session docstring (manager.py:21-23):
# Important: Messages are append-only for LLM cache efficiency.
# The consolidation process writes summaries to MEMORY.md/HISTORY.md
# but does NOT modify the messages list or get_history() output.
```

这意味着：
- `session.messages` 永远只增不减（除非 `/new` 清空）
- `last_consolidated` 是一个"已处理"水位线
- `get_history(max_messages=50)` 始终取最后 50 条，与压缩无关
- 压缩的作用是：将旧消息的 **精华** 写入持久文件，而不是从 Session 中删除它们

这个设计的优点是：
1. 避免修改消息列表导致的 LLM KV-cache 失效
2. 简化并发安全（无需锁定消息列表）
3. Session 文件可以作为完整审计日志

缺点是：
1. Session 文件会持续增长（但通常由 `/new` 重置）
2. `get_history(50)` 取的是尾部 50 条，前面的消息无法通过历史回忆

---

## 六、Memory 的读写时序图

### 6.1 正常对话流程

```
用户消息 ──▶ _process_message()
               │
               ├─ 1. sessions.get_or_create(key)
               │      └─ 从缓存或 JSONL 文件加载 Session
               │
               ├─ 2. len(messages) > memory_window?
               │      └─ 是: asyncio.create_task(_consolidate_memory())  ←── 后台
               │
               ├─ 3. context.build_messages()
               │      ├─ build_system_prompt()
               │      │    └─ memory.get_memory_context()
               │      │         └─ 读取 MEMORY.md ──▶ 注入 system prompt
               │      │
               │      ├─ session.get_history(50)
               │      │    └─ 取最后 50 条消息 ──▶ 作为历史
               │      │
               │      └─ 当前用户消息 ──▶ 追加
               │
               ├─ 4. _run_agent_loop(messages)
               │      └─ 可能通过工具读写 MEMORY.md / HISTORY.md
               │
               ├─ 5. session.add_message("user", content)
               ├─    session.add_message("assistant", response)
               └─ 6. sessions.save(session) ──▶ 写 JSONL 文件
```

### 6.2 /new 命令流程

```
/new ──▶ _process_message()
            │
            ├─ 1. messages_to_archive = session.messages.copy()
            │
            ├─ 2. session.clear()           ← 清空消息, last_consolidated=0
            ├─    sessions.save(session)     ← 写空 JSONL
            ├─    sessions.invalidate(key)   ← 清除缓存
            │
            ├─ 3. asyncio.create_task:       ← 后台任务
            │      temp_session.messages = messages_to_archive
            │      _consolidate_memory(temp_session, archive_all=True)
            │        ├─ 全部消息 → LLM 压缩
            │        ├─ history_entry → HISTORY.md (追加)
            │        └─ memory_update → MEMORY.md (覆写)
            │
            └─ 4. 立即返回 "New session started. Memory consolidation in progress."
```

### 6.3 Memory 数据流全景

```
┌──────────────────────────────────────────────────────────────────────────┐
│                          数据流全景                                      │
│                                                                         │
│  ┌─────────┐   add_message()   ┌──────────┐  save()  ┌──────────────┐ │
│  │ 用户消息 │ ────────────────▶ │ Session  │ ───────▶ │ .jsonl 文件   │ │
│  └─────────┘                   │ (内存)    │          └──────────────┘ │
│                                └──────────┘                            │
│                                     │                                  │
│                                     │ get_history(50)                  │
│                                     ▼                                  │
│                             ┌──────────────┐                           │
│                             │  LLM 请求     │                           │
│                             │  messages[]   │                           │
│                             └──────────────┘                           │
│                                     ▲                                  │
│               自动注入               │                                  │
│           ┌──────────────────────────┘                                  │
│           │                                                            │
│  ┌────────┴─────┐         压缩写入          ┌─────────────────────┐   │
│  │  MEMORY.md   │ ◀───────────────────────── │ _consolidate_       │   │
│  │  (长期事实)   │         覆写               │ memory()            │   │
│  └──────────────┘                            │                     │   │
│                                              │ LLM 生成 JSON:     │   │
│  ┌──────────────┐         压缩写入           │ {history_entry,     │   │
│  │ HISTORY.md   │ ◀───────────────────────── │  memory_update}     │   │
│  │ (事件日志)    │         追加               └─────────────────────┘   │
│  └──────────────┘                                    ▲                 │
│        │                                             │                 │
│        │ grep 搜索                        messages > window            │
│        ▼                                             │                 │
│  ┌──────────────┐                            ┌──────────────┐         │
│  │  Agent 工具   │ ◀─ read_file/grep ──────▶ │   Session     │         │
│  │  (按需检索)   │                            │   messages    │         │
│  └──────────────┘                            └──────────────┘         │
│        │                                                               │
│        │ write_file / edit_file                                        │
│        ▼                                                               │
│  ┌──────────────┐                                                      │
│  │  MEMORY.md   │  ← Agent 也可以直接写入                               │
│  └──────────────┘                                                      │
└──────────────────────────────────────────────────────────────────────────┘
```

---

## 七、Memory 配置项

### 7.1 memory_window

```
文件: nanobot/config/schema.py:165
```

```python
memory_window: int = 50
```

控制：
- `get_history(max_messages=memory_window)` — 发送给 LLM 的历史消息数量
- `len(session.messages) > memory_window` — 触发压缩的阈值
- `keep_count = memory_window // 2` — 压缩后保留的消息数量

调整建议：
- **增大**（如 100）：更多上下文，但 token 成本更高
- **减小**（如 20）：更频繁压缩，更低 token 成本，但上下文更短
- **memory_window ÷ 2** 的设计使得压缩不会太频繁（要积累到 window 的一半才触发新压缩）

### 7.2 workspace 路径

```
文件: nanobot/config/schema.py
默认值: ~/.nanobot/workspace
```

所有 memory 文件都存储在 `{workspace}/memory/` 目录下：

```
~/.nanobot/workspace/
  └── memory/
      ├── MEMORY.md       ← 长期记忆
      └── HISTORY.md      ← 事件日志
```

Session 文件单独存储在 `~/.nanobot/sessions/` 目录下。

---

## 八、MemoryStore 类详解

```
文件: nanobot/agent/memory.py
```

```python
class MemoryStore:
    """Two-layer memory: MEMORY.md (long-term facts) + HISTORY.md (grep-searchable log)."""

    def __init__(self, workspace: Path):
        self.memory_dir = ensure_dir(workspace / "memory")   # 自动创建目录
        self.memory_file = self.memory_dir / "MEMORY.md"
        self.history_file = self.memory_dir / "HISTORY.md"

    def read_long_term(self) -> str:
        """读取 MEMORY.md 全文, 不存在返回空字符串"""
        if self.memory_file.exists():
            return self.memory_file.read_text(encoding="utf-8")
        return ""

    def write_long_term(self, content: str) -> None:
        """覆写 MEMORY.md (注意: 不是追加, 是全量替换)"""
        self.memory_file.write_text(content, encoding="utf-8")

    def append_history(self, entry: str) -> None:
        """追加一条记录到 HISTORY.md, 末尾加两个换行"""
        with open(self.history_file, "a", encoding="utf-8") as f:
            f.write(entry.rstrip() + "\n\n")

    def get_memory_context(self) -> str:
        """格式化为可注入 system prompt 的文本"""
        long_term = self.read_long_term()
        return f"## Long-term Memory\n{long_term}" if long_term else ""
```

整个类只有 **31 行代码**，极其轻量。没有缓存、没有锁、没有索引。

### 8.1 MemoryStore 的实例化位置

| 位置 | 用途 |
|------|------|
| `ContextBuilder.__init__()` | 每次构建 system prompt 时读取 MEMORY.md |
| `AgentLoop._consolidate_memory()` | 压缩时读写 MEMORY.md 和 HISTORY.md |
| Agent 工具调用 | Agent 通过 read_file/write_file 直接操作文件（不经过 MemoryStore） |

注意：ContextBuilder 和压缩过程各自创建独立的 MemoryStore 实例，它们不共享状态（因为 MemoryStore 没有内存缓存，每次都直接读文件）。

---

## 九、压缩 Prompt 的详细分析

### 9.1 完整 Prompt

```python
# loop.py:375-387

prompt = f"""You are a memory consolidation agent. Process this conversation and return a JSON object with exactly two keys:

1. "history_entry": A paragraph (2-5 sentences) summarizing the key events/decisions/topics. Start with a timestamp like [YYYY-MM-DD HH:MM]. Include enough detail to be useful when found by grep search later.

2. "memory_update": The updated long-term memory content. Add any new facts: user location, preferences, personal info, habits, project context, technical decisions, tools/services used. If nothing new, return the existing content unchanged.

## Current Long-term Memory
{current_memory or "(empty)"}

## Conversation to Process
{conversation}

Respond with ONLY valid JSON, no markdown fences."""
```

### 9.2 对话格式化

```python
# loop.py:366-372
lines = []
for m in old_messages:
    if not m.get("content"):
        continue
    tools = f" [tools: {', '.join(m['tools_used'])}]" if m.get("tools_used") else ""
    lines.append(f"[{m.get('timestamp', '?')[:16]}] {m['role'].upper()}{tools}: {m['content']}")
conversation = "\n".join(lines)
```

示例输出：

```
[2025-01-15T10:30] USER: 帮我看看 README.md
[2025-01-15T10:31] ASSISTANT [tools: read_file]: README.md 内容如下...
[2025-01-15T10:32] USER: 把标题改成 "Nanobot"
[2025-01-15T10:33] ASSISTANT [tools: edit_file]: 已将标题修改为 "Nanobot"
```

### 9.3 JSON 解析容错

```python
# loop.py:397-406
text = (response.content or "").strip()

# 处理 LLM 用 Markdown 代码块包裹 JSON 的情况
if text.startswith("```"):
    text = text.split("\n", 1)[-1].rsplit("```", 1)[0].strip()

result = json.loads(text)

if entry := result.get("history_entry"):
    memory.append_history(entry)
if update := result.get("memory_update"):
    if update != current_memory:
        memory.write_long_term(update)
```

### 9.4 错误处理

整个压缩流程包裹在 `try/except` 中：

```python
try:
    response = await self.provider.chat(...)
    # ... 解析和写入 ...
except Exception as e:
    logger.error(f"Memory consolidation failed: {e}")
```

**失败不会影响用户**：压缩是后台异步任务，失败只记日志。最坏情况是记忆不更新。

---

## 十、Memory 系统的优势与局限

### 10.1 优势

| 优势 | 说明 |
|------|------|
| **极简实现** | 整个 MemoryStore 只有 31 行代码，无外部依赖 |
| **纯文件存储** | 无需数据库，用户可直接查看和编辑 Markdown 文件 |
| **Append-only Session** | 不修改历史消息，对 LLM KV-cache 友好 |
| **异步压缩** | 不阻塞用户请求处理 |
| **双层持久化** | HISTORY.md（可搜索日志）+ MEMORY.md（结构化事实）各有侧重 |
| **Agent 可自主写入** | Agent 不仅能读记忆，还能主动通过工具更新 MEMORY.md |
| **LLM 驱动的压缩** | 压缩质量取决于 LLM 能力，不是硬编码规则 |

### 10.2 局限

| 局限 | 影响 | 可能的改进方向 |
|------|------|--------------|
| **无并发控制** | Agent 工具写入和异步压缩可能竞争 MEMORY.md | 文件锁或压缩队列 |
| **HISTORY.md 无限增长** | 长期运行后文件可能变得很大 | 按日期轮转或摘要合并 |
| **全量覆写 MEMORY.md** | 每次压缩都重写整个文件，LLM 可能遗漏已有信息 | 增量更新机制 |
| **无语义检索** | 只能靠 grep 文本匹配搜索历史 | 向量数据库或嵌入索引 |
| **压缩使用主模型** | 使用与对话相同的（可能很贵的）模型做压缩 | 使用更便宜的模型 |
| **JSON 输出不稳定** | 不同模型对 "只返回 JSON" 的遵从度不同 | 结构化输出 / 重试机制 |
| **Session 文件线性增长** | messages 不从列表中删除 | 定期清理已压缩的消息 |
| **无跨 Session 记忆共享** | 不同 channel:chat_id 的 Session 独立 | 共享记忆层（MEMORY.md 已实现） |

### 10.3 跨 Session 共享分析

虽然 Session（Layer 1）是 per-key 隔离的，但 MEMORY.md 和 HISTORY.md（Layer 2 & 3）是 **全局共享** 的：

```
Telegram 用户 A ──▶ Session "telegram:A" ──┐
                                            ├──▶ 共享 MEMORY.md
WhatsApp 用户 B ──▶ Session "whatsapp:B" ──┤    共享 HISTORY.md
                                            │
CLI 用户 ─────────▶ Session "cli:direct" ──┘
```

这意味着：
- 用户 A 在 Telegram 上说的偏好，用户 B 在 WhatsApp 上也能看到（通过 MEMORY.md 注入）
- 这在单用户多终端场景下是特性（跨渠道记忆同步）
- 在多用户场景下是问题（隐私泄露）

---

## 十一、Memory 系统与其他模块的交互

```
┌──────────────────────────────────────────────────────────────────┐
│                    模块交互关系                                   │
│                                                                  │
│  nanobot init                                                    │
│  (cli/commands.py)                                               │
│       │                                                          │
│       └─ 创建 MEMORY.md (模板) + HISTORY.md (空文件)              │
│                                                                  │
│  用户消息到达                                                     │
│  (loop.py: _process_message)                                     │
│       │                                                          │
│       ├─▶ SessionManager.get_or_create()                         │
│       │     └─ 加载/创建 Session                                  │
│       │                                                          │
│       ├─▶ 检查 memory_window → 异步 _consolidate_memory()        │
│       │     ├─ MemoryStore.read_long_term()     ← 读 MEMORY.md   │
│       │     ├─ LLM 生成 JSON                                     │
│       │     ├─ MemoryStore.append_history()      → 写 HISTORY.md │
│       │     └─ MemoryStore.write_long_term()     → 写 MEMORY.md  │
│       │                                                          │
│       ├─▶ ContextBuilder.build_messages()                        │
│       │     └─ MemoryStore.get_memory_context()  ← 读 MEMORY.md  │
│       │         └─ 注入 system prompt                             │
│       │                                                          │
│       ├─▶ _run_agent_loop()                                      │
│       │     └─ Agent 可能通过工具:                                 │
│       │         ├─ read_file(MEMORY.md)           ← 读            │
│       │         ├─ write_file(MEMORY.md, ...)     → 写            │
│       │         ├─ exec("grep ... HISTORY.md")    ← 搜索          │
│       │         └─ read_file(HISTORY.md)          ← 读            │
│       │                                                          │
│       ├─▶ session.add_message("user", ...)                       │
│       ├─▶ session.add_message("assistant", ...)                  │
│       └─▶ sessions.save(session)                                 │
│             └─ 写 JSONL 文件                                      │
│                                                                  │
│  SubagentManager._build_subagent_prompt()                        │
│       └─ ⚠ 不包含 MEMORY.md (子 Agent 无法访问长期记忆)            │
│                                                                  │
│  /new 命令                                                       │
│       ├─ session.clear()                                         │
│       └─ asyncio.create_task(_consolidate_memory(archive_all))   │
└──────────────────────────────────────────────────────────────────┘
```

---

## 十二、完整的 Memory 读写路径追踪

### 12.1 MEMORY.md 的一生

```
1. nanobot init
   └─ 创建模板文件:
      "# Long-term Memory\n\n## User Information\n..."

2. 第 1 次对话
   └─ build_system_prompt() → 读取模板内容 → 注入 system prompt
   └─ Agent 可能通过 write_file 添加用户信息

3. 第 51 条消息 (超过 memory_window=50)
   └─ _consolidate_memory() 触发
   └─ 读取当前 MEMORY.md
   └─ LLM 分析对话 → 生成 memory_update
   └─ 如果有新信息 → 覆写 MEMORY.md

4. 后续每次对话
   └─ build_system_prompt() 读取最新 MEMORY.md
   └─ LLM 看到更新后的长期记忆

5. /new 命令
   └─ archive_all 压缩 → 最后一次更新 MEMORY.md
   └─ Session 清空，但 MEMORY.md 保留
   └─ 新 Session 开始时，MEMORY.md 的内容仍在 system prompt 中
```

### 12.2 HISTORY.md 的一生

```
1. nanobot init
   └─ 创建空文件

2. 第 51 条消息 (首次压缩)
   └─ LLM 生成 history_entry
   └─ append_history(entry) → 追加到文件末尾

3. 后续每次压缩
   └─ 新的 entry 追加

4. Agent 需要回忆时
   └─ exec("grep '关键词' HISTORY.md") 或 read_file("HISTORY.md")

5. 文件持续增长，没有自动清理机制
```

### 12.3 Session JSONL 的一生

```
1. 首次收到消息
   └─ SessionManager.get_or_create() → 创建空 Session

2. 每次处理完消息
   └─ session.add_message("user", ...)
   └─ session.add_message("assistant", ...)
   └─ sessions.save(session) → 全量写入 JSONL

3. 压缩触发
   └─ 只更新 last_consolidated 指针
   └─ 消息列表不变

4. /new 命令
   └─ session.clear() → messages=[], last_consolidated=0
   └─ sessions.save() → 写只含 metadata 行的 JSONL
   └─ 旧消息已在后台被压缩到 MEMORY.md + HISTORY.md

5. 进程重启
   └─ SessionManager._load() → 读取 JSONL → 重建 Session
```
