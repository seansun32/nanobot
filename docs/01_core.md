# Nanobot 核心抽象与运行机制定位

> 针对五个关键问题，精确定位到具体文件、类、方法，并附源码片段佐证。

---

## 一、哪些文件定义了 Agent 的核心抽象？

Nanobot 的核心抽象由 **6 个文件** 构成，可分为三层：

### 1.1 Agent 行为抽象

| 文件 | 核心抽象 | 说明 |
|------|---------|------|
| `nanobot/agent/tools/base.py` | `Tool` (ABC) | 所有工具的抽象基类，定义了 Agent "能做什么" |
| `nanobot/providers/base.py` | `LLMProvider` (ABC) | LLM 调用的抽象接口，定义了 Agent "如何思考" |
| `nanobot/channels/base.py` | `BaseChannel` (ABC) | 消息渠道的抽象接口，定义了 Agent "如何通信" |

### 1.2 数据流抽象

| 文件 | 核心抽象 | 说明 |
|------|---------|------|
| `nanobot/bus/events.py` | `InboundMessage` / `OutboundMessage` | Agent 世界中所有消息的标准信封格式 |
| `nanobot/providers/base.py` | `LLMResponse` / `ToolCallRequest` | LLM 返回的标准响应格式与工具调用请求 |

### 1.3 状态抽象

| 文件 | 核心抽象 | 说明 |
|------|---------|------|
| `nanobot/session/manager.py` | `Session` | 单次对话的状态容器（消息列表 + 元数据 + 压缩游标） |

### 各抽象的接口契约

**`Tool`** — Agent 能力的原子单元：

```python
# nanobot/agent/tools/base.py
class Tool(ABC):
    @property
    @abstractmethod
    def name(self) -> str: ...           # 工具标识（如 "read_file"）

    @property
    @abstractmethod
    def description(self) -> str: ...    # 功能描述（注入 LLM 上下文）

    @property
    @abstractmethod
    def parameters(self) -> dict: ...    # JSON Schema（LLM 生成调用参数的约束）

    @abstractmethod
    async def execute(self, **kwargs) -> str: ...  # 执行并返回文本结果

    def validate_params(self, params) -> list[str]: ...  # 参数校验（内置）
    def to_schema(self) -> dict: ...     # 转 OpenAI function calling 格式（内置）
```

**`LLMProvider`** — LLM 交互的统一接口：

```python
# nanobot/providers/base.py
class LLMProvider(ABC):
    @abstractmethod
    async def chat(
        self,
        messages: list[dict],       # 完整消息列表
        tools: list[dict] | None,   # 工具 Schema 列表
        model: str | None,
        max_tokens: int,
        temperature: float,
    ) -> LLMResponse: ...           # 返回内容 + 工具调用

    @abstractmethod
    def get_default_model(self) -> str: ...
```

**`BaseChannel`** — 消息渠道的适配器接口：

```python
# nanobot/channels/base.py
class BaseChannel(ABC):
    @abstractmethod
    async def start(self) -> None: ...   # 启动监听（长运行）
    @abstractmethod
    async def stop(self) -> None: ...    # 停止并清理
    @abstractmethod
    async def send(self, msg: OutboundMessage) -> None: ...  # 发送消息

    # 内置：权限校验 + 消息转发
    def is_allowed(self, sender_id: str) -> bool: ...
    async def _handle_message(self, sender_id, chat_id, content, ...) -> None: ...
```

---

## 二、哪个文件负责控制 Agent 的执行循环？

**核心文件：`nanobot/agent/loop.py`** — `AgentLoop` 类

这是整个系统的 **调度中枢**，包含两层循环：

### 2.1 外层循环：消息消费循环

```python
# nanobot/agent/loop.py:191-214
async def run(self) -> None:
    self._running = True
    while self._running:
        msg = await asyncio.wait_for(self.bus.consume_inbound(), timeout=1.0)
        response = await self._process_message(msg)
        if response:
            await self.bus.publish_outbound(response)
```

职责：从 `MessageBus.inbound` 队列持续拉取消息，逐条交给 `_process_message()` 处理，将结果投递到 `outbound` 队列。

### 2.2 内层循环：ReAct 工具调用循环

```python
# nanobot/agent/loop.py:133-189
async def _run_agent_loop(self, initial_messages) -> tuple[str | None, list[str]]:
    messages = initial_messages
    iteration = 0
    while iteration < self.max_iterations:        # 安全阀：默认 20 次
        iteration += 1
        response = await self.provider.chat(       # ① 调用 LLM
            messages=messages,
            tools=self.tools.get_definitions(),
        )
        if response.has_tool_calls:                # ② LLM 请求使用工具
            # 追加 assistant 消息（含 tool_calls）
            messages = self.context.add_assistant_message(messages, ...)
            for tool_call in response.tool_calls:
                result = await self.tools.execute(tool_call.name, tool_call.arguments)  # ③ 执行工具
                messages = self.context.add_tool_result(messages, ...)  # ④ 追加工具结果
            messages.append({"role": "user",
                             "content": "Reflect on the results and decide next steps."})  # ⑤ 反思提示
        else:
            final_content = response.content       # ⑥ 无工具调用，终止
            break
    return final_content, tools_used
```

### 2.3 消息处理流水线

`_process_message()` 是连接两层循环的桥梁：

```python
# nanobot/agent/loop.py:221-292 (简化)
async def _process_message(self, msg, session_key=None):
    # 1. 识别系统消息 / 斜杠命令
    if msg.channel == "system":
        return await self._process_system_message(msg)
    if cmd == "/new": ...  # 新会话 + 异步记忆压缩
    if cmd == "/help": ...

    # 2. 触发记忆压缩（若超阈值）
    if len(session.messages) > self.memory_window:
        asyncio.create_task(self._consolidate_memory(session))

    # 3. 设置工具上下文（channel + chat_id）
    self._set_tool_context(msg.channel, msg.chat_id)

    # 4. 构建完整的 LLM 消息列表
    initial_messages = self.context.build_messages(
        history=session.get_history(max_messages=self.memory_window),
        current_message=msg.content,
        media=msg.media, channel=msg.channel, chat_id=msg.chat_id,
    )

    # 5. 进入 ReAct 循环
    final_content, tools_used = await self._run_agent_loop(initial_messages)

    # 6. 持久化会话
    session.add_message("user", msg.content)
    session.add_message("assistant", final_content, tools_used=tools_used)
    self.sessions.save(session)

    return OutboundMessage(channel=msg.channel, chat_id=msg.chat_id, content=final_content)
```

### 2.4 执行循环的控制流全景

```
AgentLoop.run()          [外层：消息消费]
  └─ _process_message()  [流水线：上下文准备 → 循环 → 持久化]
       └─ _run_agent_loop()  [内层：ReAct 循环]
            ├─ provider.chat()     → LLM 推理
            ├─ tools.execute()     → 工具执行
            ├─ context.add_*()     → 消息追加
            └─ 反思提示注入         → 下一轮迭代
```

---

## 三、LLM 调用逻辑集中在哪个位置？

LLM 调用逻辑分布在 **抽象层** 和 **实现层** 两个位置：

### 3.1 抽象层：`nanobot/providers/base.py`

定义了 LLM 交互的三个核心数据结构：

```python
@dataclass
class ToolCallRequest:        # LLM 请求调用工具
    id: str                   # 调用 ID（用于匹配结果）
    name: str                 # 工具名
    arguments: dict           # 工具参数

@dataclass
class LLMResponse:            # LLM 响应
    content: str | None       # 文本回复
    tool_calls: list[ToolCallRequest]  # 工具调用列表
    finish_reason: str        # 终止原因
    usage: dict[str, int]     # token 用量
    reasoning_content: str | None      # 思维链（Kimi/DeepSeek-R1 等）

    @property
    def has_tool_calls(self) -> bool:  # 是否包含工具调用
        return len(self.tool_calls) > 0
```

### 3.2 实现层：`nanobot/providers/litellm_provider.py`

唯一的 `LLMProvider` 实现，通过 LiteLLM 库适配 14+ 家 LLM 供应商：

| 方法 | 职责 |
|------|------|
| `__init__()` | 接收配置，调用 `_setup_env()` 设置环境变量 |
| `_setup_env()` | 根据 `ProviderRegistry` 将 API Key 映射到对应环境变量 |
| `_resolve_model()` | 自动添加供应商前缀（如 `claude-3` → `anthropic/claude-3`） |
| `_apply_model_overrides()` | 应用模型特定参数覆盖（如 Kimi 的 temperature=1.0） |
| `chat()` | **核心调用入口** — 调用 `litellm.acompletion()` |
| `_parse_response()` | 将 LiteLLM 响应转换为 `LLMResponse` |

### 3.3 调用链路

LLM 调用在系统中出现在 **三个场景**：

```
场景 1：主 Agent ReAct 循环
  AgentLoop._run_agent_loop()
    → self.provider.chat(messages, tools, model)
    → LiteLLMProvider.chat()
    → litellm.acompletion()

场景 2：子 Agent 执行
  SubagentManager._run_subagent()
    → self.provider.chat(messages, tools, model)
    → LiteLLMProvider.chat()
    → litellm.acompletion()

场景 3：记忆压缩
  AgentLoop._consolidate_memory()
    → self.provider.chat(messages=[system, user], model)  # 无 tools
    → LiteLLMProvider.chat()
    → litellm.acompletion()
```

### 3.4 供应商路由机制

`nanobot/providers/registry.py` 中的 `ProviderSpec` 声明式注册表决定了模型名称如何映射到具体供应商：

```
用户配置 model="claude-opus-4-5"
  → registry.find_by_model("claude-opus-4-5")  匹配 keywords 含 "claude" 的 ProviderSpec
  → 命中 anthropic
  → _resolve_model() 添加前缀 → "anthropic/claude-opus-4-5"
  → _setup_env() 设置 ANTHROPIC_API_KEY
  → litellm.acompletion(model="anthropic/claude-opus-4-5")
```

---

## 四、工具（Tools）是在哪里注册和调用的？

### 4.1 工具定义

每个工具是 `Tool` 基类的子类，分布在 `nanobot/agent/tools/` 目录下：

| 文件 | 工具类 | 工具名 |
|------|--------|--------|
| `filesystem.py` | `ReadFileTool` | `read_file` |
| `filesystem.py` | `WriteFileTool` | `write_file` |
| `filesystem.py` | `EditFileTool` | `edit_file` |
| `filesystem.py` | `ListDirTool` | `list_directory` |
| `shell.py` | `ExecTool` | `exec` |
| `web.py` | `WebSearchTool` | `web_search` |
| `web.py` | `WebFetchTool` | `web_fetch` |
| `message.py` | `MessageTool` | `message` |
| `spawn.py` | `SpawnTool` | `spawn` |
| `cron.py` | `CronTool` | `cron` |

### 4.2 工具注册

注册发生在 `AgentLoop.__init__()` 中，调用 `_register_default_tools()`：

```python
# nanobot/agent/loop.py:87-117
def _register_default_tools(self) -> None:
    allowed_dir = self.workspace if self.restrict_to_workspace else None

    # 文件系统工具（4 个）
    self.tools.register(ReadFileTool(allowed_dir=allowed_dir))
    self.tools.register(WriteFileTool(allowed_dir=allowed_dir))
    self.tools.register(EditFileTool(allowed_dir=allowed_dir))
    self.tools.register(ListDirTool(allowed_dir=allowed_dir))

    # Shell 工具
    self.tools.register(ExecTool(working_dir=str(self.workspace), ...))

    # Web 工具（2 个）
    self.tools.register(WebSearchTool(api_key=self.brave_api_key))
    self.tools.register(WebFetchTool())

    # 消息工具
    self.tools.register(MessageTool(send_callback=self.bus.publish_outbound))

    # 子 Agent 工具
    self.tools.register(SpawnTool(manager=self.subagents))

    # 定时任务工具（可选）
    if self.cron_service:
        self.tools.register(CronTool(self.cron_service))
```

子 Agent 有 **独立的工具注册**（受限集合，无 Message/Spawn/Cron）：

```python
# nanobot/agent/subagent.py:103-116
tools = ToolRegistry()
tools.register(ReadFileTool(allowed_dir=allowed_dir))
tools.register(WriteFileTool(allowed_dir=allowed_dir))
tools.register(EditFileTool(allowed_dir=allowed_dir))
tools.register(ListDirTool(allowed_dir=allowed_dir))
tools.register(ExecTool(...))
tools.register(WebSearchTool(api_key=self.brave_api_key))
tools.register(WebFetchTool())
# 注意：无 MessageTool、SpawnTool、CronTool
```

### 4.3 工具调用

调用链路贯穿 ReAct 循环的三个步骤：

```
步骤 1 — Schema 传递给 LLM：
  tools.get_definitions()
  → 遍历所有注册工具，调用 tool.to_schema()
  → 生成 OpenAI function calling 格式的 JSON Schema 列表
  → 作为 provider.chat(tools=...) 的参数传入

步骤 2 — LLM 决策调用：
  LLM 返回 response.tool_calls = [ToolCallRequest(id, name, arguments)]
  → has_tool_calls 为 True，进入工具执行分支

步骤 3 — 工具执行：
  tools.execute(name, arguments)                    # registry.py:38
  → tool = self._tools.get(name)                    # 按名称查找
  → errors = tool.validate_params(params)           # JSON Schema 校验
  → result = await tool.execute(**params)            # 异步执行
  → 返回字符串结果，追加到消息列表
```

### 4.4 工具注册中心

`ToolRegistry`（`nanobot/agent/tools/registry.py`）是一个轻量的字典封装：

```python
class ToolRegistry:
    _tools: dict[str, Tool] = {}

    def register(tool)          # 注册：_tools[tool.name] = tool
    def get(name) -> Tool       # 查找
    def get_definitions()       # 批量生成 Schema（传给 LLM）
    async def execute(name, params)  # 校验 + 执行
```

---

## 五、Agent 的状态存储和更新在什么地方？

Agent 的状态分为 **四个层次**，分布在不同的存储介质中：

### 5.1 状态层次总览

```
┌──────────────────────────────────────────────────────────────────────┐
│  层次 1：即时状态（内存）                                              │
│  ─────────────────────                                              │
│  位置：AgentLoop._run_agent_loop() 局部变量 messages                 │
│  内容：当前 ReAct 循环的完整消息列表（system + history + user + tools）│
│  生命周期：单次请求处理期间                                           │
│  更新时机：每次 LLM 调用 / 工具执行后追加消息                          │
└──────────────────────────────────────────────────────────────────────┘
         │ 处理完成后写入
         ▼
┌──────────────────────────────────────────────────────────────────────┐
│  层次 2：会话状态（JSONL 文件）                                       │
│  ─────────────────────────                                          │
│  管理者：SessionManager (nanobot/session/manager.py)                │
│  存储位置：~/.nanobot/sessions/{channel}_{chat_id}.jsonl            │
│  内容：Session 对象 — messages[] + metadata + last_consolidated      │
│  生命周期：跨请求持久化，/new 命令清空                                 │
│  更新时机：每次请求处理完成后 session.save()                          │
└──────────────────────────────────────────────────────────────────────┘
         │ 消息数超阈值时异步压缩
         ▼
┌──────────────────────────────────────────────────────────────────────┐
│  层次 3：长期记忆（Markdown 文件）                                    │
│  ──────────────────────────                                         │
│  管理者：MemoryStore (nanobot/agent/memory.py)                      │
│  存储位置：~/.nanobot/workspace/memory/MEMORY.md  — 长期事实          │
│           ~/.nanobot/workspace/memory/HISTORY.md — 事件日志          │
│  生命周期：永久，跨所有会话                                           │
│  更新时机：_consolidate_memory() 异步压缩 / LLM 主动调用 write_file  │
└──────────────────────────────────────────────────────────────────────┘
         │ 每次构建上下文时读取
         ▼
┌──────────────────────────────────────────────────────────────────────┐
│  层次 4：身份与行为配置（Markdown + JSON）                             │
│  ────────────────────────────────────                                │
│  存储位置：~/.nanobot/workspace/AGENTS.md, SOUL.md, USER.md, ...    │
│           ~/.nanobot/config.json                                    │
│  内容：Agent 人格、工具指令、用户画像、系统配置                         │
│  生命周期：永久，极少变更                                             │
│  更新时机：onboard 初始化 / 用户手动编辑 / LLM 通过 write_file 修改   │
└──────────────────────────────────────────────────────────────────────┘
```

### 5.2 各层次的关键代码位置

**层次 1 — 即时状态**

```python
# nanobot/agent/loop.py:133-189
# messages 列表在 ReAct 循环中被不断追加
messages = initial_messages                           # 初始化
messages = self.context.add_assistant_message(...)    # LLM 响应
messages = self.context.add_tool_result(...)          # 工具结果
messages.append({"role": "user", "content": "Reflect..."})  # 反思提示
```

**层次 2 — 会话状态**

```python
# nanobot/session/manager.py
@dataclass
class Session:
    key: str                          # "channel:chat_id"
    messages: list[dict]              # 追加式消息列表
    last_consolidated: int = 0        # 已压缩到的消息偏移量
    metadata: dict = field(...)

    def add_message(self, role, content, **kwargs):  # 追加消息
    def get_history(self, max_messages=500):          # 读取最近 N 条
    def clear(self):                                  # /new 命令重置

class SessionManager:
    def get_or_create(self, key) -> Session   # 内存缓存 + JSONL 加载
    def save(self, session)                   # 写回 JSONL
    def invalidate(self, key)                 # 清除缓存
```

**层次 3 — 长期记忆**

```python
# nanobot/agent/memory.py
class MemoryStore:
    def read_long_term(self) -> str          # 读取 MEMORY.md
    def write_long_term(self, content)       # 覆写 MEMORY.md
    def append_history(self, entry)          # 追加 HISTORY.md
    def get_memory_context(self) -> str      # 格式化后注入上下文
```

记忆压缩的触发与执行：

```python
# nanobot/agent/loop.py:263-264 — 触发条件
if len(session.messages) > self.memory_window:
    asyncio.create_task(self._consolidate_memory(session))

# nanobot/agent/loop.py:337-414 — 压缩过程
async def _consolidate_memory(self, session, archive_all=False):
    # 1. 取出未压缩的旧消息
    old_messages = session.messages[session.last_consolidated:-keep_count]
    # 2. 格式化为文本
    conversation = "\n".join(lines)
    # 3. 调用 LLM 生成 JSON：{history_entry, memory_update}
    response = await self.provider.chat(messages=[...])
    result = json.loads(response.content)
    # 4. 写入文件
    memory.append_history(result["history_entry"])   # → HISTORY.md
    memory.write_long_term(result["memory_update"])  # → MEMORY.md
    # 5. 更新压缩游标
    session.last_consolidated = len(session.messages) - keep_count
```

**层次 4 — 身份与配置**

```python
# nanobot/agent/context.py:112-122 — bootstrap 文件加载
BOOTSTRAP_FILES = ["AGENTS.md", "SOUL.md", "USER.md", "TOOLS.md", "IDENTITY.md"]

def _load_bootstrap_files(self):
    for filename in self.BOOTSTRAP_FILES:
        file_path = self.workspace / filename
        if file_path.exists():
            content = file_path.read_text(encoding="utf-8")
            parts.append(f"## {filename}\n\n{content}")
```

### 5.3 状态流转图

```
用户消息到达
    │
    ▼
SessionManager.get_or_create(key)     ──→  从 JSONL 加载 Session（层次 2）
    │
    ▼
ContextBuilder.build_messages()
    ├─ 读取 MEMORY.md                 ──→  注入长期记忆（层次 3）
    ├─ 读取 bootstrap files           ──→  注入身份配置（层次 4）
    └─ session.get_history()          ──→  注入会话历史（层次 2）
    │
    ▼
_run_agent_loop(messages)             ──→  messages 在循环中不断追加（层次 1）
    │
    ▼
session.add_message("user", ...)      ──→  用户消息写入 Session（层次 2）
session.add_message("assistant", ...) ──→  助手回复写入 Session（层次 2）
sessions.save(session)                ──→  持久化到 JSONL（层次 2）
    │
    ▼ (异步，若超阈值)
_consolidate_memory()
    ├─ memory.append_history(...)     ──→  追加事件日志（层次 3）
    └─ memory.write_long_term(...)    ──→  更新长期记忆（层次 3）
```

---

## 总结速查表

| 问题 | 答案 | 核心文件 |
|------|------|---------|
| Agent 核心抽象 | `Tool`、`LLMProvider`、`BaseChannel`、`InboundMessage`/`OutboundMessage`、`LLMResponse`/`ToolCallRequest`、`Session` | `tools/base.py`、`providers/base.py`、`channels/base.py`、`bus/events.py`、`session/manager.py` |
| 执行循环控制 | `AgentLoop` — 外层消息消费 + 内层 ReAct 工具循环 | `agent/loop.py` |
| LLM 调用逻辑 | 抽象在 `LLMProvider`，实现在 `LiteLLMProvider`，调用点在 `AgentLoop`/`SubagentManager`/记忆压缩 | `providers/base.py`、`providers/litellm_provider.py` |
| 工具注册与调用 | 注册在 `AgentLoop._register_default_tools()`，执行在 `ToolRegistry.execute()`，Schema 传递在 `get_definitions()` | `agent/loop.py`、`agent/tools/registry.py` |
| 状态存储与更新 | 四层：即时状态（内存 messages）→ 会话状态（JSONL）→ 长期记忆（Markdown）→ 身份配置（Markdown + JSON） | `agent/loop.py`、`session/manager.py`、`agent/memory.py`、`agent/context.py` |
