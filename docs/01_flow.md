# Agent 执行主循环：逐步解析

> 本文聚焦 `nanobot/agent/loop.py` 中的 `AgentLoop` 类，
> 从源码级别拆解两层循环的每一步执行过程。

---

## 1. 循环定位

Nanobot 的执行引擎包含 **两层循环** 和一个 **衔接函数**：

| 层次 | 方法 | 行号 | 角色 |
|------|------|------|------|
| 外层 | `AgentLoop.run()` | L191-214 | 消息消费循环 — 从队列逐条取出消息 |
| 衔接 | `AgentLoop._process_message()` | L221-292 | 单消息处理流水线 — 上下文装配 + 调用内层循环 + 持久化 |
| 内层 | `AgentLoop._run_agent_loop()` | L133-189 | **ReAct 循环** — LLM 推理 → 工具调用 → 反思，反复迭代 |

另外，`SubagentManager._run_subagent()`（L92-184, `subagent.py`）包含一个结构相似但独立的内层循环，用于子 Agent 后台任务。

---

## 2. 逐步执行流程说明

### 阶段 A：外层消息消费循环（`run()`）

```
文件: nanobot/agent/loop.py:191-214
```

**Step A1** — 设置运行标志

```python
self._running = True    # L193
```

**Step A2** — 阻塞等待入站消息（1 秒超时轮询）

```python
msg = await asyncio.wait_for(self.bus.consume_inbound(), timeout=1.0)  # L198-201
```

- **输入**：`MessageBus.inbound` 队列中的 `InboundMessage`
- 若 1 秒内无消息，抛出 `TimeoutError` → `continue` 回到 A2
- 这使得循环可以被 `stop()` 优雅中断（每秒检查一次 `_running`）

**Step A3** — 处理消息，获取响应

```python
response = await self._process_message(msg)  # L203
```

**Step A4** — 发布出站消息

```python
if response:
    await self.bus.publish_outbound(response)  # L204-205
```

**Step A5** — 异常兜底：遇到错误时返回错误消息给用户

```python
except Exception as e:
    await self.bus.publish_outbound(OutboundMessage(
        channel=msg.channel, chat_id=msg.chat_id,
        content=f"Sorry, I encountered an error: {str(e)}"
    ))  # L206-212
```

→ 回到 **Step A2**，继续等待下一条消息。

---

### 阶段 B：单消息处理流水线（`_process_message()`）

```
文件: nanobot/agent/loop.py:221-292
```

**Step B1** — 系统消息路由

```python
if msg.channel == "system":
    return await self._process_system_message(msg)  # L233-234
```

子 Agent 完成后通过 `channel="system"` 将结果注入消息总线，在这里被截获并路由到正确的会话。

**Step B2** — 获取或创建 Session

```python
key = session_key or msg.session_key          # L239，session_key = "channel:chat_id"
session = self.sessions.get_or_create(key)    # L240，内存缓存优先，未命中则从 JSONL 加载
```

**Step B3** — 斜杠命令处理

```python
if cmd == "/new":     # L244-258：清空会话 + 异步记忆压缩
if cmd == "/help":    # L259-261：返回帮助文本
```

`/new` 的特殊处理：
1. 复制当前消息列表 `messages_to_archive = session.messages.copy()`
2. 立即清空会话 `session.clear()` 并保存
3. 异步启动 `_consolidate_memory(temp_session, archive_all=True)` — 不阻塞用户

**Step B4** — 触发记忆压缩（异步，不阻塞）

```python
if len(session.messages) > self.memory_window:     # L263，默认阈值 50
    asyncio.create_task(self._consolidate_memory(session))  # L264
```

**Step B5** — 设置工具上下文（路由信息）

```python
self._set_tool_context(msg.channel, msg.chat_id)  # L266
```

将当前 `channel` 和 `chat_id` 注入 `MessageTool`、`SpawnTool`、`CronTool`，使工具知道"回复给谁"。

**Step B6** — 构建初始消息列表

```python
initial_messages = self.context.build_messages(
    history=session.get_history(max_messages=self.memory_window),
    current_message=msg.content,
    media=msg.media if msg.media else None,
    channel=msg.channel, chat_id=msg.chat_id,
)  # L267-273
```

`ContextBuilder.build_messages()` 内部装配过程：

```
messages = [
  ① { role: "system",    content: 身份 + bootstrap文件 + 长期记忆 + 技能摘要 + 会话元数据 }
  ② { role: "user/assistant", ... }  ×N  (历史消息，最多 memory_window 条)
  ③ { role: "user",      content: 当前用户消息（可含 base64 图片）}
]
```

**Step B7** — 进入 ReAct 循环

```python
final_content, tools_used = await self._run_agent_loop(initial_messages)  # L274
```

**Step B8** — 空响应兜底

```python
if final_content is None:
    final_content = "I've completed processing but have no response to give."  # L276-277
```

**Step B9** — 持久化会话

```python
session.add_message("user", msg.content)                           # L282
session.add_message("assistant", final_content, tools_used=...)    # L283-284
self.sessions.save(session)                                        # L285
```

注意：这里保存的是 **原始的用户消息** 和 **最终的 Agent 回复**，中间的工具调用过程不写入 Session（仅存在于 ReAct 循环的临时 `messages` 列表中）。

**Step B10** — 构造出站消息返回

```python
return OutboundMessage(channel=msg.channel, chat_id=msg.chat_id,
                       content=final_content, metadata=msg.metadata)  # L287-292
```

---

### 阶段 C：ReAct 内层循环（`_run_agent_loop()`） — **核心**

```
文件: nanobot/agent/loop.py:133-189
```

这是整个系统最关键的函数。每次迭代完成一轮 "思考→行动→观察" 的 ReAct 步骤。

**输入**：

| 参数 | 类型 | 来源 |
|------|------|------|
| `initial_messages` | `list[dict]` | `_process_message()` 中由 `ContextBuilder.build_messages()` 构建 |

包含 system prompt + 历史消息 + 当前用户消息。

**Step C0** — 初始化循环变量

```python
messages = initial_messages       # 可变消息列表（循环中不断追加）
iteration = 0                     # 迭代计数器
final_content = None              # 最终回复（循环结束后返回）
tools_used: list[str] = []        # 已使用工具的名称列表
```

**Step C1** — 迭代计数 + 安全阀检查

```python
while iteration < self.max_iterations:   # 默认 max_iterations=20
    iteration += 1
```

**Step C2** — 调用 LLM

```python
response = await self.provider.chat(
    messages=messages,                        # 完整消息列表
    tools=self.tools.get_definitions(),       # 所有注册工具的 JSON Schema
    model=self.model,                         # 模型标识（如 "anthropic/claude-opus-4-5"）
    temperature=self.temperature,             # 默认 0.7
    max_tokens=self.max_tokens,               # 默认 4096
)
```

返回 `LLMResponse`：
- `content`: 文本回复（可能为 None）
- `tool_calls`: `list[ToolCallRequest]`（可能为空）
- `reasoning_content`: 思维链内容（Kimi/DeepSeek-R1 等）

**Step C3** — 分支决策：工具调用 vs 最终回复

```python
if response.has_tool_calls:      # len(tool_calls) > 0
    → 进入 Step C4（工具执行分支）
else:
    → 进入 Step C7（终止分支）
```

**决策者是 LLM 本身**。Agent 代码不做任何工具调用决策 — 它完全信任 LLM 的 function calling 输出。

**Step C4** — 追加 assistant 消息（含工具调用声明）

```python
tool_call_dicts = [
    {"id": tc.id, "type": "function",
     "function": {"name": tc.name, "arguments": json.dumps(tc.arguments)}}
    for tc in response.tool_calls
]
messages = self.context.add_assistant_message(
    messages, response.content, tool_call_dicts,
    reasoning_content=response.reasoning_content,
)
```

此时 messages 新增：
```json
{ "role": "assistant", "content": "...", "tool_calls": [...], "reasoning_content": "..." }
```

**Step C5** — 逐个执行工具调用

```python
for tool_call in response.tool_calls:
    tools_used.append(tool_call.name)                                          # 记录
    result = await self.tools.execute(tool_call.name, tool_call.arguments)     # 执行
    messages = self.context.add_tool_result(messages, tool_call.id, tool_call.name, result)  # 追加结果
```

执行细节（`ToolRegistry.execute()`）：
1. 按名称查找工具 → 未找到返回错误字符串
2. `tool.validate_params(params)` → JSON Schema 校验
3. `await tool.execute(**params)` → 异步执行，返回字符串结果
4. 任何异常被捕获并转为 `"Error executing {name}: {str(e)}"`

每次执行后 messages 新增：
```json
{ "role": "tool", "tool_call_id": "call_xxx", "name": "read_file", "content": "文件内容..." }
```

**注意**：工具调用是 **串行** 的。即使 LLM 一次返回多个 `tool_calls`，也逐个顺序执行。

**Step C6** — 注入反思提示

```python
messages.append({"role": "user", "content": "Reflect on the results and decide next steps."})
```

这是一个关键设计：在工具结果之后插入一条 "用户" 消息，引导 LLM 在下一次迭代中反思工具执行结果并决定是否需要继续使用工具。

→ 回到 **Step C1**，开始下一次迭代。

**Step C7** — 无工具调用，循环终止

```python
else:
    final_content = response.content   # LLM 的最终文本回复
    break
```

**输出**：

```python
return final_content, tools_used    # (最终回复文本, 使用过的工具名列表)
```

---

### 阶段 C 中的状态更新时序

以一个 3 次迭代的场景为例，追踪 `messages` 列表的增长：

```
迭代 0（初始状态）:
  messages = [
    { system: 身份+记忆+技能 },
    { user: 历史消息1 },
    { assistant: 历史回复1 },
    ...
    { user: 当前用户消息 }        ← build_messages() 的输出
  ]

迭代 1（LLM 决定调用 read_file + exec）:
  messages += [
    { assistant: "让我先看一下文件...", tool_calls: [read_file, exec] },   ← C4
    { tool: read_file 结果 },                                             ← C5
    { tool: exec 结果 },                                                  ← C5
    { user: "Reflect on the results and decide next steps." }             ← C6
  ]

迭代 2（LLM 决定调用 write_file）:
  messages += [
    { assistant: "基于分析结果，我需要修改...", tool_calls: [write_file] },
    { tool: write_file 结果 },
    { user: "Reflect on the results and decide next steps." }
  ]

迭代 3（LLM 认为任务完成，不再调用工具）:
  final_content = "我已经完成了..."
  break                                                                   ← C7
```

---

## 3. 循环终止条件

| 条件 | 触发位置 | 含义 |
|------|---------|------|
| **LLM 未返回工具调用** | `_run_agent_loop()` L185-187 | LLM 认为任务完成，返回最终文本回复 → `break` |
| **达到最大迭代次数** | `_run_agent_loop()` L148 | `iteration >= max_iterations`（默认 20）→ 循环自然结束，`final_content` 可能为 `None` |
| **外层循环停止** | `run()` L196 | `self._running = False`（由 `stop()` 设置）→ 外层 `while` 退出 |

子 Agent 的终止条件相同，但最大迭代次数为 15（`subagent.py:126`）。

### 终止后的兜底处理

```python
# 主 Agent — _process_message() L276-277
if final_content is None:
    final_content = "I've completed processing but have no response to give."

# 子 Agent — _run_subagent() L175-176
if final_result is None:
    final_result = "Task completed but no final response was generated."
```

---

## 4. 同步 / 异步 / 事件驱动？

**答：异步（async/await），并在特定节点表现出事件驱动特征。**

### 4.1 异步本体

整个 Agent 运行在 Python `asyncio` 事件循环上，所有关键操作都是 `async`：

```python
async def run(self) -> None:                    # 外层循环
async def _process_message(self, msg) -> ...:   # 消息处理
async def _run_agent_loop(self, ...) -> ...:    # ReAct 循环
async def chat(self, ...) -> LLMResponse:       # LLM 调用（网络 I/O）
async def execute(self, ...) -> str:            # 工具执行
```

### 4.2 事件驱动节点

| 节点 | 机制 | 解释 |
|------|------|------|
| 消息到达 | `asyncio.Queue` + `wait_for` 超时轮询 | 外层循环通过队列实现生产者-消费者模式 |
| 出站消息分发 | `MessageBus.dispatch_outbound()` + 订阅回调 | 发布-订阅模式，回调异步执行 |
| 子 Agent 完成 | `asyncio.create_task()` + `bus.publish_inbound()` | 后台任务通过消息总线通知主循环 |
| 记忆压缩 | `asyncio.create_task()` | 异步后台任务，不阻塞主流程 |

### 4.3 关键并发特性

```
asyncio 事件循环
├── AgentLoop.run()              ← 主循环协程（消费入站消息）
├── MessageBus.dispatch_outbound() ← 出站消息分发协程
├── Channel.start()              ← 各渠道监听协程（Telegram/Discord/...）
├── SubagentManager._run_subagent() ← 0~N 个子 Agent 后台任务
└── _consolidate_memory()         ← 0~1 个记忆压缩后台任务
```

所有协程共享同一个事件循环，通过 `await` 交替执行（协作式并发）。

---

## 5. 伪代码

```
# ═══════════════════════════════════════════════════
# 外层循环：消息消费
# ═══════════════════════════════════════════════════

function AgentLoop.run():
    running ← true

    while running:
        msg ← bus.consume_inbound(timeout=1s)
        if timeout:
            continue

        try:
            response ← process_message(msg)
            if response ≠ null:
                bus.publish_outbound(response)
        catch error:
            bus.publish_outbound(ErrorMessage(msg, error))


# ═══════════════════════════════════════════════════
# 衔接层：单消息处理流水线
# ═══════════════════════════════════════════════════

function process_message(msg):
    # ─── 路由 ───
    if msg.channel == "system":
        return process_system_message(msg)

    # ─── 会话管理 ───
    session ← sessions.get_or_create(msg.session_key)

    if msg.content == "/new":
        archive ← copy(session.messages)
        session.clear()
        sessions.save(session)
        background: consolidate_memory(archive, archive_all=true)
        return "New session started."

    if msg.content == "/help":
        return help_text

    # ─── 记忆压缩（异步，不阻塞）───
    if len(session.messages) > memory_window:
        background: consolidate_memory(session)

    # ─── 上下文装配 ───
    set_tool_context(msg.channel, msg.chat_id)
    initial_messages ← context.build_messages(
        system_prompt  = identity + bootstrap_files + long_term_memory + skills_summary,
        history        = session.get_history(max=memory_window),
        current_message = msg.content + optional_media,
        session_meta   = {channel, chat_id}
    )

    # ─── ReAct 循环 ───
    final_content, tools_used ← run_agent_loop(initial_messages)

    # ─── 兜底 ───
    if final_content is null:
        final_content ← "I've completed processing but have no response to give."

    # ─── 持久化 ───
    session.add_message("user", msg.content)
    session.add_message("assistant", final_content, tools_used)
    sessions.save(session)

    return OutboundMessage(msg.channel, msg.chat_id, final_content)


# ═══════════════════════════════════════════════════
# 内层循环：ReAct 工具调用循环
# ═══════════════════════════════════════════════════

function run_agent_loop(initial_messages):
    messages ← initial_messages
    iteration ← 0
    final_content ← null
    tools_used ← []

    while iteration < MAX_ITERATIONS (20):    # ── 安全阀
        iteration ← iteration + 1

        # ──── 调用 LLM ────
        response ← provider.chat(
            messages  = messages,
            tools     = registry.get_definitions(),   # 所有工具的 JSON Schema
            model     = model,
            temperature = temperature,
            max_tokens = max_tokens,
        )

        # ──── 分支决策 ────
        if response.has_tool_calls:

            # 1) 追加 assistant 消息（含工具调用声明 + 可选思维链）
            messages ← append(messages, {
                role: "assistant",
                content: response.content,
                tool_calls: response.tool_calls,
                reasoning_content: response.reasoning_content
            })

            # 2) 串行执行每个工具调用
            for tc in response.tool_calls:
                tools_used ← append(tools_used, tc.name)
                result ← registry.execute(tc.name, tc.arguments)
                    # → registry.get(name)
                    # → tool.validate_params(arguments)
                    # → tool.execute(**arguments)
                messages ← append(messages, {
                    role: "tool",
                    tool_call_id: tc.id,
                    name: tc.name,
                    content: result
                })

            # 3) 注入反思提示
            messages ← append(messages, {
                role: "user",
                content: "Reflect on the results and decide next steps."
            })

            # → 继续下一次迭代

        else:
            # ──── 终止：LLM 返回最终文本 ────
            final_content ← response.content
            break

    return (final_content, tools_used)
```

---

## 6. 时序图

```
┌──────┐      ┌──────────┐    ┌──────────────────┐    ┌───────────────┐   ┌─────────┐   ┌──────────┐
│Client│      │MessageBus│    │    AgentLoop      │    │ContextBuilder │   │LLMProvider│  │ToolRegistry│
└──┬───┘      └────┬─────┘    └────────┬──────────┘    └──────┬────────┘   └────┬─────┘  └─────┬──────┘
   │               │                   │                      │                 │              │
   │  用户发送消息  │                   │                      │                 │              │
   │──────────────>│                   │                      │                 │              │
   │               │ publish_inbound() │                      │                 │              │
   │               │                   │                      │                 │              │
   │               │   consume_inbound │                      │                 │              │
   │               │<─────────────────>│                      │                 │              │
   │               │   InboundMessage  │                      │                 │              │
   │               │                   │                      │                 │              │
   │               │                   │  _process_message()  │                 │              │
   │               │                   │──┐                   │                 │              │
   │               │                   │  │                   │                 │              │
   │               │                   │  │ get_or_create()   │                 │              │
   │               │                   │  │ → Session         │                 │              │
   │               │                   │  │                   │                 │              │
   │               │                   │  │ build_messages()  │                 │              │
   │               │                   │  │──────────────────>│                 │              │
   │               │                   │  │                   │─┐               │              │
   │               │                   │  │                   │ │ build_system_prompt()        │
   │               │                   │  │                   │ │ 身份+bootstrap+记忆+技能     │
   │               │                   │  │                   │<┘               │              │
   │               │                   │  │   [system, ...history, user]        │              │
   │               │                   │  │<──────────────────│                 │              │
   │               │                   │  │                   │                 │              │
   │               │                   │  │ _run_agent_loop() │                 │              │
   │               │                   │  │──┐                │                 │              │
   │               │                   │  │  │                │                 │              │
   │               │                   │  │  │                │                 │              │
   │               │                   │  │  │  ╔═══════════════════════════════════════════╗  │
   │               │                   │  │  │  ║  ReAct 循环（迭代 1 / N）                  ║  │
   │               │                   │  │  │  ╚═══════════════════════════════════════════╝  │
   │               │                   │  │  │                │                 │              │
   │               │                   │  │  │  chat(messages, tools)           │              │
   │               │                   │  │  │─────────────────────────────────>│              │
   │               │                   │  │  │                │                 │──┐           │
   │               │                   │  │  │                │                 │  │ litellm   │
   │               │                   │  │  │                │                 │  │.acompletion()
   │               │                   │  │  │                │                 │<─┘           │
   │               │                   │  │  │  LLMResponse(tool_calls=[read_file, exec])     │
   │               │                   │  │  │<─────────────────────────────────│              │
   │               │                   │  │  │                │                 │              │
   │               │                   │  │  │  has_tool_calls → true           │              │
   │               │                   │  │  │                │                 │              │
   │               │                   │  │  │  add_assistant_message()         │              │
   │               │                   │  │  │──────────────────────────────────────────┐      │
   │               │                   │  │  │  messages += {assistant + tool_calls}    │      │
   │               │                   │  │  │<─────────────────────────────────────────┘      │
   │               │                   │  │  │                │                 │              │
   │               │                   │  │  │  tools.execute("read_file", {path: ...})        │
   │               │                   │  │  │────────────────────────────────────────────────>│
   │               │                   │  │  │                │                 │              │─┐
   │               │                   │  │  │                │                 │              │ │validate
   │               │                   │  │  │                │                 │              │ │+ execute
   │               │                   │  │  │                │                 │              │<┘
   │               │                   │  │  │  result = "文件内容..."           │              │
   │               │                   │  │  │<────────────────────────────────────────────────│
   │               │                   │  │  │                │                 │              │
   │               │                   │  │  │  add_tool_result()               │              │
   │               │                   │  │  │  messages += {tool: result}      │              │
   │               │                   │  │  │                │                 │              │
   │               │                   │  │  │  tools.execute("exec", {cmd: ...})              │
   │               │                   │  │  │────────────────────────────────────────────────>│
   │               │                   │  │  │  result = "命令输出..."           │              │
   │               │                   │  │  │<────────────────────────────────────────────────│
   │               │                   │  │  │                │                 │              │
   │               │                   │  │  │  add_tool_result()               │              │
   │               │                   │  │  │  messages += {tool: result}      │              │
   │               │                   │  │  │                │                 │              │
   │               │                   │  │  │  messages += {user: "Reflect on the results..."} │
   │               │                   │  │  │                │                 │              │
   │               │                   │  │  │  ╔═══════════════════════════════════════════╗  │
   │               │                   │  │  │  ║  ReAct 循环（迭代 2 / N）                  ║  │
   │               │                   │  │  │  ╚═══════════════════════════════════════════╝  │
   │               │                   │  │  │                │                 │              │
   │               │                   │  │  │  chat(messages, tools)           │              │
   │               │                   │  │  │─────────────────────────────────>│              │
   │               │                   │  │  │  LLMResponse(content="最终回复", tool_calls=[]) │
   │               │                   │  │  │<─────────────────────────────────│              │
   │               │                   │  │  │                │                 │              │
   │               │                   │  │  │  has_tool_calls → false          │              │
   │               │                   │  │  │  final_content = "最终回复"      │              │
   │               │                   │  │  │  break                           │              │
   │               │                   │  │  │                │                 │              │
   │               │                   │  │<─┘                │                 │              │
   │               │                   │  │                   │                 │              │
   │               │                   │  │ return (final_content, tools_used)  │              │
   │               │                   │  │                   │                 │              │
   │               │                   │  │ session.add_message("user", ...)    │              │
   │               │                   │  │ session.add_message("assistant", ...)│             │
   │               │                   │  │ sessions.save(session)              │              │
   │               │                   │<─┘                   │                 │              │
   │               │                   │                      │                 │              │
   │               │ publish_outbound()│                      │                 │              │
   │               │<──────────────────│                      │                 │              │
   │               │ OutboundMessage   │                      │                 │              │
   │  最终回复     │                   │                      │                 │              │
   │<──────────────│                   │                      │                 │              │
   │               │                   │                      │                 │              │
```

---

## 7. 关键设计观察

### 7.1 反思提示（Reflect Prompt）

```python
messages.append({"role": "user", "content": "Reflect on the results and decide next steps."})
```

这条 **伪用户消息** 是 ReAct 模式的核心驱动力。它在每轮工具执行后强制 LLM 进行反思，决定是继续使用工具还是给出最终回复。没有它，LLM 可能会在没有"被提问"的情况下不产生任何输出。

### 7.2 工具调用串行执行

即使 LLM 一次返回多个 `tool_calls`（如同时调用 `read_file` 和 `exec`），它们也是 **串行** 执行的（`for tool_call in response.tool_calls`）。这简化了错误处理，但可能影响延迟。

### 7.3 会话只记录首尾

`Session` 只保存 **用户原始消息** 和 **最终 Agent 回复**（`_process_message()` L282-284），中间的多轮工具调用过程不被持久化。这意味着：
- 会话历史是压缩的，不包含工具调用细节
- 下次对话时 LLM 看不到之前用过哪些工具（除非通过 `tools_used` 元数据推断）
- 长期记忆压缩（`_consolidate_memory()`）会保留 `tools_used` 标记

### 7.4 主 Agent vs 子 Agent 循环对比

| 维度 | 主 Agent (`_run_agent_loop`) | 子 Agent (`_run_subagent`) |
|------|------|------|
| 最大迭代 | 20 | 15 |
| 工具集 | 10 种（含 Message/Spawn/Cron） | 7 种（纯执行工具） |
| 反思提示 | 有 | 无 |
| 消息构建 | `ContextBuilder`（含记忆/技能） | 简化 system prompt |
| 结果传递 | 直接返回 | 通过 `bus.publish_inbound()` 注入为系统消息 |
| 会话持久化 | 写入 Session JSONL | 不持久化 |
