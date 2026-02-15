# 工具（Tool）系统全解析

> 从定义、决策、校验、回注到新增工具的完整指南。

---

## 一、工具是如何定义的？

Nanobot 采用 **ABC 抽象基类 + JSON Schema** 的方式定义工具，不使用装饰器。

### 1.1 抽象基类：`Tool`

```
文件: nanobot/agent/tools/base.py
```

每个工具必须继承 `Tool` 并实现 4 个抽象成员：

```python
class Tool(ABC):

    @property
    @abstractmethod
    def name(self) -> str:
        """工具唯一标识，如 "read_file"。LLM 在 function calling 时使用此名称。"""

    @property
    @abstractmethod
    def description(self) -> str:
        """功能描述，注入 LLM 上下文，决定 LLM 何时选择该工具。"""

    @property
    @abstractmethod
    def parameters(self) -> dict[str, Any]:
        """JSON Schema（OpenAI function calling 格式），约束 LLM 生成的参数。"""

    @abstractmethod
    async def execute(self, **kwargs: Any) -> str:
        """异步执行，接收经校验的参数，返回纯文本结果。"""
```

基类还提供两个内置方法（无需重写）：

| 方法 | 作用 |
|------|------|
| `validate_params(params)` | 递归校验参数是否符合 JSON Schema，返回错误列表 |
| `to_schema()` | 将工具转为 OpenAI function calling 格式的 JSON 字典 |

### 1.2 两种定义风格

代码库中存在两种等价风格：

**风格 A：property 方法**（filesystem.py、shell.py、spawn.py、message.py、cron.py）

```python
class ReadFileTool(Tool):
    @property
    def name(self) -> str:
        return "read_file"

    @property
    def description(self) -> str:
        return "Read the contents of a file at the given path."

    @property
    def parameters(self) -> dict[str, Any]:
        return { "type": "object", "properties": { ... }, "required": [...] }
```

**风格 B：类属性直接赋值**（web.py）

```python
class WebSearchTool(Tool):
    name = "web_search"
    description = "Search the web. Returns titles, URLs, and snippets."
    parameters = {
        "type": "object",
        "properties": { ... },
        "required": ["query"]
    }
```

风格 B 更简洁。两者均合法，因为 Python 的 `@property` 和类属性在 `ABC` 子类化时都能通过 `@abstractmethod` 检查。

### 1.3 Schema 到 LLM 的转换

`to_schema()` 生成的字典直接用于 OpenAI/LiteLLM 的 `tools` 参数：

```python
# base.py:93-102
def to_schema(self) -> dict[str, Any]:
    return {
        "type": "function",
        "function": {
            "name": self.name,              # → LLM 调用的函数名
            "description": self.description, # → LLM 理解工具用途
            "parameters": self.parameters,   # → LLM 生成参数的约束
        }
    }
```

### 1.4 现有工具清单

| 文件 | 类名 | `name` | 构造器依赖 | 上下文感知 |
|------|------|--------|-----------|-----------|
| `filesystem.py` | `ReadFileTool` | `read_file` | `allowed_dir` | 否 |
| `filesystem.py` | `WriteFileTool` | `write_file` | `allowed_dir` | 否 |
| `filesystem.py` | `EditFileTool` | `edit_file` | `allowed_dir` | 否 |
| `filesystem.py` | `ListDirTool` | `list_dir` | `allowed_dir` | 否 |
| `shell.py` | `ExecTool` | `exec` | `timeout`, `working_dir`, `deny_patterns` | 否 |
| `web.py` | `WebSearchTool` | `web_search` | `api_key` | 否 |
| `web.py` | `WebFetchTool` | `web_fetch` | `max_chars` | 否 |
| `message.py` | `MessageTool` | `message` | `send_callback` | **是** (`set_context`) |
| `spawn.py` | `SpawnTool` | `spawn` | `SubagentManager` | **是** (`set_context`) |
| `cron.py` | `CronTool` | `cron` | `CronService` | **是** (`set_context`) |

"上下文感知" 指工具在每次消息处理前通过 `_set_tool_context(channel, chat_id)` 注入路由信息。

---

## 二、Agent 是如何决定是否调用某个工具的？

**核心结论：Agent 代码本身 _不做_ 任何工具调用决策。决策权完全交给 LLM。**

### 2.1 决策链路

```
┌──────────────────────────────────────────────────────────────────────────────┐
│ 1. Schema 注入                                                              │
│    tools.get_definitions()                                                  │
│    → 遍历所有注册工具，调用 tool.to_schema()                                  │
│    → 生成 JSON Schema 列表                                                   │
│    → 传入 provider.chat(tools=schemas)                                       │
│                                                                             │
│ 2. LLM 内部决策                                                             │
│    LLM 基于以下信息决定是否调用工具、调用哪个、传什么参数：                       │
│    ├── system prompt（身份 + 记忆 + 技能）                                    │
│    ├── 历史消息                                                              │
│    ├── 当前用户消息                                                           │
│    └── tools 参数中的 name + description + parameters                        │
│                                                                             │
│ 3. 响应解析                                                                  │
│    LLMResponse.has_tool_calls                                                │
│    ├── true  → 进入工具执行分支（循环继续）                                     │
│    └── false → LLM 直接返回文本（循环终止）                                     │
└──────────────────────────────────────────────────────────────────────────────┘
```

### 2.2 关键代码

```python
# loop.py:148-187 — _run_agent_loop()
while iteration < self.max_iterations:
    response = await self.provider.chat(
        messages=messages,
        tools=self.tools.get_definitions(),   # ← 所有工具 Schema 传给 LLM
        ...
    )
    if response.has_tool_calls:               # ← LLM 决定了
        # 执行工具...
    else:
        final_content = response.content      # ← LLM 决定不调用工具
        break
```

### 2.3 影响 LLM 决策的因素

| 因素 | 如何影响 | 代码位置 |
|------|---------|---------|
| `description` | 描述越精准，LLM 越能正确选择工具 | 每个工具的 `description` 属性 |
| `parameters` | JSON Schema 约束 LLM 生成的参数格式 | 每个工具的 `parameters` 属性 |
| system prompt | 全局行为指导（如 "Only use the 'message' tool when you need to send a message to a specific chat channel"） | `context.py:104-106` |
| 反思提示 | 工具执行后注入 "Reflect on the results and decide next steps."，引导 LLM 继续或停止 | `loop.py:184` |

### 2.4 LLM 不调用工具的情况

- LLM 认为可以直接回答（不需要外部信息或操作）
- LLM 在反思后认为任务已完成
- `tools` 参数为空列表（理论上不会发生，因为总有注册工具）

---

## 三、工具输入是如何被校验的？

### 3.1 校验流程

```python
# registry.py:38-62 — ToolRegistry.execute()
async def execute(self, name: str, params: dict[str, Any]) -> str:
    tool = self._tools.get(name)
    if not tool:
        return f"Error: Tool '{name}' not found"          # ① 工具存在性检查

    try:
        errors = tool.validate_params(params)              # ② JSON Schema 校验
        if errors:
            return f"Error: Invalid parameters for tool '{name}': " + "; ".join(errors)
        return await tool.execute(**params)                 # ③ 执行
    except Exception as e:
        return f"Error executing {name}: {str(e)}"         # ④ 运行时异常兜底
```

### 3.2 校验器实现

`validate_params()` 是 `Tool` 基类的内置方法，实现了一个轻量级的递归 JSON Schema 验证器：

```python
# base.py:55-91
def validate_params(self, params: dict) -> list[str]:
    schema = self.parameters or {}
    return self._validate(params, {**schema, "type": "object"}, "")

def _validate(self, val, schema, path) -> list[str]:
    # 支持的校验规则：
    # ├── type 检查：string, integer, number, boolean, array, object
    # ├── enum 约束
    # ├── 数值范围：minimum, maximum
    # ├── 字符串长度：minLength, maxLength
    # ├── 对象属性：properties + required
    # └── 数组元素：items（递归校验每个元素）
```

类型映射表：

```python
_TYPE_MAP = {
    "string": str,
    "integer": int,
    "number": (int, float),
    "boolean": bool,
    "array": list,
    "object": dict,
}
```

### 3.3 校验覆盖范围

| 校验能力 | 支持 | 示例 |
|---------|------|------|
| 必选字段 | ✅ | `"required": ["path"]` → 缺少 `path` 报错 |
| 类型检查 | ✅ | `"type": "string"` → 传入 `int` 报错 |
| 枚举约束 | ✅ | `"enum": ["add", "list", "remove"]` → 非法值报错 |
| 数值范围 | ✅ | `"minimum": 1, "maximum": 10` |
| 字符串长度 | ✅ | `"minLength": 1, "maxLength": 100` |
| 嵌套对象 | ✅ | 递归校验 `properties` |
| 数组元素 | ✅ | 递归校验 `items` |
| pattern (正则) | ❌ | 不支持 |
| oneOf / anyOf | ❌ | 不支持 |
| additionalProperties | ❌ | 不支持（多余字段不会报错） |

### 3.4 校验失败的结果

校验失败 **不会抛异常**，而是返回错误字符串给 LLM：

```
"Error: Invalid parameters for tool 'cron': action must be one of ['add', 'list', 'remove']"
```

LLM 会看到这个错误信息，并在下一次迭代中尝试修正参数。

---

## 四、工具的执行结果是如何回注到 Agent 上下文中的？

### 4.1 完整回注路径

```python
# loop.py:159-184 — _run_agent_loop() 中的工具执行分支

# Step 1：LLM 返回工具调用请求
response = await self.provider.chat(messages=messages, tools=...)
# response.tool_calls = [ToolCallRequest(id="call_abc", name="read_file", arguments={"path": "/foo"})]

# Step 2：追加 assistant 消息（声明"我要调用这些工具"）
messages = self.context.add_assistant_message(
    messages,
    content=response.content,           # LLM 的思考文本（可能为 None）
    tool_calls=tool_call_dicts,          # 工具调用声明列表
    reasoning_content=response.reasoning_content,  # 思维链（Kimi/DeepSeek-R1）
)
# messages 新增：
# { role: "assistant", content: "...", tool_calls: [{id, type, function: {name, arguments}}] }

# Step 3：逐个执行工具并追加结果
for tool_call in response.tool_calls:
    result = await self.tools.execute(tool_call.name, tool_call.arguments)
    messages = self.context.add_tool_result(
        messages,
        tool_call_id=tool_call.id,       # 与 Step 2 中的 id 对应
        tool_name=tool_call.name,
        result=result,                   # 工具执行结果（纯文本字符串）
    )
# messages 新增（每个工具调用一条）：
# { role: "tool", tool_call_id: "call_abc", name: "read_file", content: "文件内容..." }

# Step 4：注入反思提示
messages.append({"role": "user", "content": "Reflect on the results and decide next steps."})
```

### 4.2 消息格式详解

回注遵循 OpenAI function calling 协议，完整的消息序列如下：

```json
[
  {"role": "system", "content": "身份 + 记忆 + 技能..."},
  {"role": "user", "content": "历史消息1"},
  {"role": "assistant", "content": "历史回复1"},
  {"role": "user", "content": "当前用户消息"},

  // ─── 迭代 1 ───
  {
    "role": "assistant",
    "content": "让我读取文件看看...",
    "tool_calls": [
      {
        "id": "call_abc",
        "type": "function",
        "function": {
          "name": "read_file",
          "arguments": "{\"path\": \"/home/user/nanobot/README.md\"}"
        }
      }
    ]
  },
  {
    "role": "tool",
    "tool_call_id": "call_abc",
    "name": "read_file",
    "content": "# Nanobot\n\nA lightweight AI agent framework..."
  },
  {"role": "user", "content": "Reflect on the results and decide next steps."},

  // ─── 迭代 2 ───
  {
    "role": "assistant",
    "content": "文件已读取，现在我可以直接回答你的问题了。\n\nNanobot 是一个轻量级的..."
    // 无 tool_calls → 循环终止
  }
]
```

### 4.3 `ContextBuilder` 中的回注方法

```python
# context.py:182-238

def add_tool_result(self, messages, tool_call_id, tool_name, result) -> list:
    """追加一条 role="tool" 消息"""
    messages.append({
        "role": "tool",
        "tool_call_id": tool_call_id,   # 必须与 assistant.tool_calls[].id 匹配
        "name": tool_name,
        "content": result,              # 纯文本字符串
    })
    return messages

def add_assistant_message(self, messages, content, tool_calls=None,
                          reasoning_content=None) -> list:
    """追加一条 role="assistant" 消息（可含 tool_calls 和 reasoning_content）"""
    msg = {"role": "assistant", "content": content or ""}
    if tool_calls:
        msg["tool_calls"] = tool_calls
    if reasoning_content:
        msg["reasoning_content"] = reasoning_content  # 思维链模型需要此字段
    messages.append(msg)
    return messages
```

### 4.4 回注生命周期

```
工具结果的生命周期：

  execute() 返回 str
       │
       ▼
  追加到 messages 列表（内存，ReAct 循环的局部变量）
       │
       ▼
  LLM 在下一次迭代中读取所有 tool 消息
       │
       ▼
  循环结束后，messages 列表被丢弃
       │
       ▼
  只有 final_content 被写入 Session
  （工具调用细节不持久化到会话历史）
```

这意味着工具执行结果是 **短暂的** — 仅存在于当前请求的 ReAct 循环中。如果需要跨会话保留，必须通过工具本身的副作用（如 `write_file` 写磁盘、`memory` 写 MEMORY.md）来实现。

---

## 五、新增 `search_docs` 工具指南

### 5.1 最小可运行示例

创建文件 `nanobot/agent/tools/docs.py`：

```python
"""Documentation search tool."""

from pathlib import Path
from typing import Any

from nanobot.agent.tools.base import Tool


class SearchDocsTool(Tool):
    """Tool to search documentation files by keyword."""

    def __init__(self, docs_dir: str | Path = "docs"):
        self._docs_dir = Path(docs_dir).resolve()

    @property
    def name(self) -> str:
        return "search_docs"

    @property
    def description(self) -> str:
        return (
            "Search documentation files (.md, .txt, .rst) by keyword. "
            "Returns matching file paths and the lines containing the keyword."
        )

    @property
    def parameters(self) -> dict[str, Any]:
        return {
            "type": "object",
            "properties": {
                "query": {
                    "type": "string",
                    "description": "Keyword or phrase to search for",
                    "minLength": 1,
                },
                "max_results": {
                    "type": "integer",
                    "description": "Maximum number of matching files to return (default: 5)",
                    "minimum": 1,
                    "maximum": 20,
                },
            },
            "required": ["query"],
        }

    async def execute(self, query: str, max_results: int = 5, **kwargs: Any) -> str:
        if not self._docs_dir.exists():
            return f"Error: docs directory not found: {self._docs_dir}"

        matches: list[str] = []
        extensions = {".md", ".txt", ".rst"}
        query_lower = query.lower()

        for file_path in sorted(self._docs_dir.rglob("*")):
            if file_path.suffix not in extensions or not file_path.is_file():
                continue

            try:
                content = file_path.read_text(encoding="utf-8")
            except Exception:
                continue

            matched_lines = []
            for i, line in enumerate(content.splitlines(), 1):
                if query_lower in line.lower():
                    matched_lines.append(f"  L{i}: {line.strip()}")

            if matched_lines:
                rel_path = file_path.relative_to(self._docs_dir)
                header = f"📄 {rel_path} ({len(matched_lines)} matches)"
                # Show at most 5 lines per file
                preview = "\n".join(matched_lines[:5])
                if len(matched_lines) > 5:
                    preview += f"\n  ... and {len(matched_lines) - 5} more"
                matches.append(f"{header}\n{preview}")

            if len(matches) >= max_results:
                break

        if not matches:
            return f"No matches found for '{query}' in {self._docs_dir}"

        return f"Found {len(matches)} file(s) matching '{query}':\n\n" + "\n\n".join(matches)
```

### 5.2 注册工具

在 `nanobot/agent/loop.py` 中注册：

```python
# 1. 添加 import（文件顶部）
from nanobot.agent.tools.docs import SearchDocsTool

# 2. 在 _register_default_tools() 中添加注册代码
def _register_default_tools(self) -> None:
    # ... 现有工具注册 ...

    # Docs search tool
    self.tools.register(SearchDocsTool(docs_dir=self.workspace / "docs"))
```

如果子 Agent 也需要此工具，还需在 `nanobot/agent/subagent.py` 的 `_run_subagent()` 中添加：

```python
# subagent.py:_run_subagent() 中的工具注册块
tools.register(SearchDocsTool(docs_dir=self.workspace / "docs"))
```

### 5.3 需要修改的文件列表

| 文件 | 操作 | 说明 |
|------|------|------|
| `nanobot/agent/tools/docs.py` | **新建** | 工具实现 |
| `nanobot/agent/loop.py` | **修改** | 添加 import + 注册（2 行） |
| `nanobot/agent/subagent.py` | **可选修改** | 若子 Agent 需要此工具，添加 import + 注册（2 行） |

不需要修改的文件：
- `base.py` — 基类无需变更
- `registry.py` — 注册中心无需变更
- `context.py` — 上下文构建无需变更
- `__init__.py` — 除非需要从 `nanobot.agent.tools` 包直接导出

### 5.4 从注册到被 LLM 调用的完整链路

```
启动时：
  AgentLoop.__init__()
    → _register_default_tools()
      → self.tools.register(SearchDocsTool(docs_dir=...))
        → ToolRegistry._tools["search_docs"] = SearchDocsTool 实例

用户发送消息后：
  _run_agent_loop()
    → self.tools.get_definitions()
      → 包含 SearchDocsTool.to_schema()：
        {
          "type": "function",
          "function": {
            "name": "search_docs",
            "description": "Search documentation files...",
            "parameters": { ... }
          }
        }
    → provider.chat(messages=..., tools=[...包含 search_docs...])

LLM 决定调用：
    → response.tool_calls = [ToolCallRequest(
        id="call_xyz",
        name="search_docs",
        arguments={"query": "authentication", "max_results": 3}
      )]

执行：
    → self.tools.execute("search_docs", {"query": "authentication", "max_results": 3})
      → ToolRegistry.get("search_docs") → SearchDocsTool 实例
      → validate_params({"query": "authentication", "max_results": 3}) → []（无错误）
      → SearchDocsTool.execute(query="authentication", max_results=3)
      → "Found 2 file(s) matching 'authentication':..."

回注：
    → context.add_tool_result(messages, "call_xyz", "search_docs", "Found 2 file(s)...")
    → LLM 下一轮看到搜索结果并决定下一步
```

---

## 六、常见的坑与注意事项

### 坑 1：`name` 必须全局唯一

`ToolRegistry` 使用 `dict[str, Tool]`，`name` 是唯一键。重复注册会 **静默覆盖** 前一个工具：

```python
# registry.py:18-20
def register(self, tool: Tool) -> None:
    self._tools[tool.name] = tool    # 直接覆盖，不报错
```

**建议**：在注册前检查 `if not self.tools.has("search_docs")`，或在命名时加前缀避免冲突。

### 坑 2：`execute()` 必须返回字符串

所有工具的 `execute()` 返回值类型是 `str`。如果返回 `None`、`dict` 或其他类型，LLM 会收到类型转换后的结果（通常是 `"None"` 或 `repr()`），可能导致解析混乱。

**建议**：始终显式返回字符串。对于结构化数据，使用 `json.dumps()`。

### 坑 3：异常不会中断循环

`ToolRegistry.execute()` 会捕获所有异常并转为错误字符串：

```python
except Exception as e:
    return f"Error executing {name}: {str(e)}"
```

这意味着工具的 bug **不会让 Agent 崩溃**，但 LLM 会看到错误信息并可能反复重试。如果工具有严重 bug，可能会耗尽 `max_iterations`。

**建议**：在开发阶段手动测试工具的 `execute()` 方法，确保常见路径不会抛异常。

### 坑 4：`execute()` 的参数签名必须与 Schema 一致

`ToolRegistry.execute()` 使用 `**params` 展开参数传给 `tool.execute()`：

```python
return await tool.execute(**params)    # registry.py:60
```

如果 Schema 中定义了参数 `"query"` 但 `execute()` 方法没有 `query` 形参，会抛 `TypeError`。
反之，如果 `execute()` 有 `query` 形参但 Schema 中没有且 LLM 没有传递，也会缺少参数。

**建议**：
1. `execute()` 的必选参数 = Schema 中 `required` 的字段
2. `execute()` 的可选参数（有默认值）= Schema 中非 `required` 的字段
3. 始终添加 `**kwargs` 兜底，防止 LLM 传入 Schema 中未定义的额外参数

### 坑 5：description 质量直接影响调用准确性

LLM 通过 `description` 决定何时使用工具。模糊或过于宽泛的描述会导致：
- **误调用**：LLM 在不需要时调用工具
- **漏调用**：LLM 在需要时选择其他工具

**建议**：
- 明确说明工具 **做什么**、**适用场景** 和 **限制**
- 参考现有工具的描述风格：简洁、直接、含可操作的上下文

反例与正例：

```python
# ❌ 描述太模糊
description = "Search for stuff"

# ✅ 描述明确
description = (
    "Search documentation files (.md, .txt, .rst) by keyword. "
    "Returns matching file paths and the lines containing the keyword."
)
```

### 坑 6：不支持流式结果

`execute()` 必须一次性返回完整结果字符串。对于耗时操作（大文件搜索、网络请求），工具会阻塞当前迭代直到完成。

**建议**：
- 对可能产生大量输出的工具，在实现中截断结果（参考 `ExecTool` 的 10000 字符截断）
- 对耗时操作，设置超时（参考 `ExecTool` 的 `asyncio.wait_for(timeout=...)`)

### 坑 7：上下文感知工具需要 `set_context()`

如果新工具需要知道当前消息来自哪个 channel/chat_id（如发送通知、记录日志），需要：

1. 在工具类中添加 `set_context(channel, chat_id)` 方法
2. 在 `AgentLoop._set_tool_context()` 中添加调用

```python
# loop.py:119-131
def _set_tool_context(self, channel: str, chat_id: str) -> None:
    if message_tool := self.tools.get("message"):
        if isinstance(message_tool, MessageTool):
            message_tool.set_context(channel, chat_id)
    # ... 需要在此添加新工具的上下文注入
```

`search_docs` 通常不需要此机制。

### 坑 8：子 Agent 工具集是独立的

子 Agent（`subagent.py`）有自己的 `ToolRegistry` 实例，只注册了 7 个基础工具。如果新工具需要在子 Agent 中可用，必须在 `_run_subagent()` 中显式注册。

### 坑 9：多次匹配的多余字段不报错

`validate_params()` 不检查 `additionalProperties`。LLM 传入 Schema 中未定义的字段时不会报错，这些字段会通过 `**kwargs` 传入 `execute()` 但被忽略。通常无害，但可能导致调试困惑。

### 坑 10：工具是串行执行的

即使 LLM 一次返回多个 `tool_calls`，它们也按顺序串行执行。如果你的工具涉及 I/O（网络请求、文件读取），可能成为性能瓶颈。当前架构不支持并行工具执行。
