# LLM Prompt 构建与输出解析全解析

> 追踪每一段 Prompt 的来源、组装过程、状态注入方式，
> 以及 LLM 输出的解析链路与潜在断裂点。

---

## 一、Prompt 的三种角色分别定义在哪里？

### 1.1 总览

| 角色 | 构造位置 | 动态内容来源 | 出现次数 |
|------|---------|-------------|---------|
| `system` | `ContextBuilder.build_system_prompt()` | 身份信息、bootstrap 文件、长期记忆、技能摘要 | 每次请求 1 条 |
| `user` | `ContextBuilder.build_messages()` + ReAct 循环 | 历史消息、当前用户消息、反思提示 | N 条 |
| `assistant` | `ContextBuilder.add_assistant_message()` | LLM 的回复 + 工具调用声明 + 思维链 | N 条 |
| `tool` | `ContextBuilder.add_tool_result()` | 工具执行结果 | N 条 |

另有两个独立的 prompt 构造点：

| 场景 | 构造位置 | 特点 |
|------|---------|------|
| 子 Agent | `SubagentManager._build_subagent_prompt()` | 独立的简化 system prompt，不含记忆/技能 |
| 记忆压缩 | `AgentLoop._consolidate_memory()` | 硬编码的 system + user prompt，要求返回 JSON |

### 1.2 system prompt 的完整构成

```
文件: nanobot/agent/context.py:28-71
方法: ContextBuilder.build_system_prompt()
```

`build_system_prompt()` 将多个片段用 `"\n\n---\n\n"` 拼接：

```python
def build_system_prompt(self, skill_names=None) -> str:
    parts = []
    parts.append(self._get_identity())              # ① 核心身份
    bootstrap = self._load_bootstrap_files()         # ② Bootstrap 文件
    if bootstrap: parts.append(bootstrap)
    memory = self.memory.get_memory_context()         # ③ 长期记忆
    if memory: parts.append(f"# Memory\n\n{memory}")
    always_skills = self.skills.get_always_skills()   # ④ 常驻技能（完整内容）
    if always_skills: ...
    skills_summary = self.skills.build_skills_summary()  # ⑤ 技能摘要（XML 索引）
    if skills_summary: ...
    return "\n\n---\n\n".join(parts)
```

#### ① 核心身份（`_get_identity()`，`context.py:73-110`）

硬编码的模板字符串，包含动态变量：

```python
return f"""# nanobot 🐈

You are nanobot, a helpful AI assistant. You have access to tools that allow you to:
- Read, write, and edit files
- Execute shell commands
- Search the web and fetch web pages
- Send messages to users on chat channels
- Spawn subagents for complex background tasks

## Current Time
{now} ({tz})                          ← datetime.now() 动态生成

## Runtime
{runtime}                              ← platform.system() + python_version()

## Workspace
Your workspace is at: {workspace_path}  ← self.workspace 路径
- Long-term memory: {workspace_path}/memory/MEMORY.md
- History log: {workspace_path}/memory/HISTORY.md (grep-searchable)
- Custom skills: {workspace_path}/skills/{{skill-name}}/SKILL.md

IMPORTANT: When responding to direct questions or conversations, reply directly ...
Only use the 'message' tool when you need to send a message to a specific chat channel ...

Always be helpful, accurate, and concise. ...
When remembering something important, write to {workspace_path}/memory/MEMORY.md
To recall past events, grep {workspace_path}/memory/HISTORY.md"""
```

#### ② Bootstrap 文件（`_load_bootstrap_files()`，`context.py:112-122`）

从 workspace 根目录按固定顺序加载 Markdown 文件：

```python
BOOTSTRAP_FILES = ["AGENTS.md", "SOUL.md", "USER.md", "TOOLS.md", "IDENTITY.md"]

def _load_bootstrap_files(self):
    parts = []
    for filename in self.BOOTSTRAP_FILES:
        file_path = self.workspace / filename
        if file_path.exists():
            content = file_path.read_text(encoding="utf-8")
            parts.append(f"## {filename}\n\n{content}")
    return "\n\n".join(parts) if parts else ""
```

这些文件的内容直接拼接为 system prompt 的一部分。

#### ③ 长期记忆（`MemoryStore.get_memory_context()`，`memory.py:28-30`）

```python
def get_memory_context(self) -> str:
    long_term = self.read_long_term()          # 读取 MEMORY.md 全文
    return f"## Long-term Memory\n{long_term}" if long_term else ""
```

#### ④ 常驻技能（`get_always_skills()` + `load_skills_for_context()`）

标记了 `always=true` 的技能，其 SKILL.md **完整内容** 被嵌入 system prompt：

```python
# context.py:55-59
always_skills = self.skills.get_always_skills()
if always_skills:
    always_content = self.skills.load_skills_for_context(always_skills)
    parts.append(f"# Active Skills\n\n{always_content}")
```

#### ⑤ 技能摘要（`build_skills_summary()`，`skills.py:101-140`）

非常驻技能以 **XML 索引** 形式提供，Agent 按需用 `read_file` 加载完整内容：

```xml
<skills>
  <skill available="true">
    <name>image-gen</name>
    <description>Generate images using DALL-E</description>
    <location>/home/user/.nanobot/workspace/skills/image-gen/SKILL.md</location>
  </skill>
  <skill available="false">
    <name>pdf-reader</name>
    <description>Extract text from PDF files</description>
    <location>/home/user/.nanobot/workspace/skills/pdf-reader/SKILL.md</location>
    <requires>CLI: pdftotext</requires>
  </skill>
</skills>
```

#### ⑥ 会话元数据追加（`build_messages()` 中，`context.py:151-152`）

```python
if channel and chat_id:
    system_prompt += f"\n\n## Current Session\nChannel: {channel}\nChat ID: {chat_id}"
```

### 1.3 user prompt 的来源

| 来源 | 位置 | 内容 |
|------|------|------|
| 历史消息 | `session.get_history()` → `build_messages()` | 之前的 user/assistant 轮次 |
| 当前用户消息 | `build_messages()` 的 `current_message` 参数 | 本次用户输入（可含 base64 图片） |
| 反思提示 | `_run_agent_loop()` L184 | 固定文本 `"Reflect on the results and decide next steps."` |

### 1.4 tool prompt 的来源

| 来源 | 位置 | 内容 |
|------|------|------|
| 工具 Schema | `ToolRegistry.get_definitions()` | 每个工具的 `to_schema()` 输出，作为 `chat(tools=...)` 参数 |
| 工具结果 | `ContextBuilder.add_tool_result()` | `role="tool"` 消息，含 `tool_call_id` + `name` + `content` |

### 1.5 子 Agent 的 system prompt

```
文件: nanobot/agent/subagent.py:218-253
方法: SubagentManager._build_subagent_prompt()
```

独立的硬编码模板，仅包含时间、规则约束、能力说明和 workspace 路径。不含记忆、不含 bootstrap 文件、不含技能。

### 1.6 记忆压缩的 prompt

```
文件: nanobot/agent/loop.py:375-387
方法: AgentLoop._consolidate_memory()
```

两条消息，硬编码：

```python
messages=[
    {"role": "system", "content": "You are a memory consolidation agent. Respond only with valid JSON."},
    {"role": "user", "content": prompt},   # prompt 包含旧对话 + 当前 MEMORY.md + JSON 格式要求
]
```

---

## 二、是否使用了 Prompt 模板化机制？

**没有使用任何模板引擎。** 所有 prompt 都是 **Python f-string 拼接 + 字符串条件组装**。

### 2.1 当前的模板化方式

| 机制 | 使用场景 | 示例 |
|------|---------|------|
| **f-string** | 身份信息、子 Agent prompt、记忆压缩 prompt | `f"Your workspace is at: {workspace_path}"` |
| **条件拼接** | system prompt 各段落按存在性组装 | `if bootstrap: parts.append(bootstrap)` |
| **`"\n\n---\n\n".join()`** | system prompt 顶层片段之间的分隔符 | `return "\n\n---\n\n".join(parts)` |
| **`"\n\n".join()`** | bootstrap 文件之间 | `return "\n\n".join(parts)` |
| **`"\n\n---\n\n".join()`** | 技能内容之间 | `return "\n\n---\n\n".join(parts)` |

### 2.2 模板化程度评估

```
         无模板 ──────────────── 轻量模板 ──────────────── 重型模板
         (f-string)              (Jinja2)                 (LangChain)
              ^
              │
        Nanobot 在这里
```

**优点**：简单、无依赖、代码即模板、IDE 可直接跳转。

**缺点**：

| 问题 | 表现 |
|------|------|
| prompt 与逻辑耦合 | 修改 prompt 措辞需要修改 Python 代码 |
| 无变量验证 | f-string 中引用不存在的变量会抛 `NameError` |
| 难以 A/B 测试 | 无法在运行时切换不同版本的 prompt |
| 无 escape 机制 | 用户内容中的 `{` `}` 不会被转义（但因为不在 f-string 内使用，实际无影响） |

### 2.3 外部模板的替代方案：Bootstrap 文件

虽然不使用模板引擎，但 Nanobot 通过 **Bootstrap 文件** 实现了一定程度的"外部化"：

```
~/.nanobot/workspace/
  ├── AGENTS.md       ← 自定义 Agent 行为指令
  ├── SOUL.md         ← 自定义人格特征
  ├── USER.md         ← 用户画像
  ├── TOOLS.md        ← 工具使用指南
  └── IDENTITY.md     ← 身份覆写
```

用户可以通过编辑这些 Markdown 文件来修改 system prompt 的内容，无需修改代码。这在本质上是一种 **文件驱动的模板机制**。

---

## 三、Agent 的状态是如何影响 Prompt 内容的？

### 3.1 状态到 Prompt 的映射关系

```
┌─────────────────────────────────────────────────────────────────────┐
│                         system prompt                              │
│                                                                    │
│  ┌───────────────┐   ┌──────────────────┐   ┌──────────────────┐  │
│  │ 核心身份      │   │ Bootstrap 文件    │   │ 长期记忆          │  │
│  │               │   │                  │   │                  │  │
│  │ 动态变量:     │   │ 静态内容:         │   │ 动态内容:         │  │
│  │ • 当前时间    │   │ • AGENTS.md      │   │ • MEMORY.md 全文  │  │
│  │ • OS 平台     │   │ • SOUL.md        │   │                  │  │
│  │ • Python 版本 │   │ • USER.md        │   │ 更新时机:         │  │
│  │ • workspace   │   │ • TOOLS.md       │   │ • 每次压缩后      │  │
│  │   路径        │   │ • IDENTITY.md    │   │ • LLM 主动写入    │  │
│  └───────────────┘   └──────────────────┘   └──────────────────┘  │
│                                                                    │
│  ┌──────────────────┐   ┌──────────────────┐                      │
│  │ 常驻技能          │   │ 技能摘要          │                      │
│  │                  │   │                  │                      │
│  │ always=true 的   │   │ XML 格式索引     │                      │
│  │ SKILL.md 完整内容 │   │ (懒加载指引)      │                      │
│  └──────────────────┘   └──────────────────┘                      │
│                                                                    │
│  ┌──────────────────┐                                              │
│  │ 会话元数据        │                                              │
│  │ Channel + ChatID │                                              │
│  └──────────────────┘                                              │
└─────────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────────┐
│                    历史消息 (user/assistant)                         │
│                                                                    │
│  来源: session.get_history(max_messages=memory_window)             │
│  格式: [{"role": "user", "content": ...},                         │
│         {"role": "assistant", "content": ...}, ...]               │
│  特点:                                                             │
│  • 只含首尾（不含中间的工具调用过程）                                  │
│  • 最多 memory_window 条（默认 50）                                 │
│  • 超出部分异步压缩到 MEMORY.md/HISTORY.md                          │
└─────────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────────┐
│                    当前用户消息                                      │
│                                                                    │
│  来源: InboundMessage.content + media                              │
│  处理: _build_user_content() 将图片 base64 编码为 multimodal 格式   │
└─────────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────────┐
│              ReAct 循环中动态追加的消息                               │
│                                                                    │
│  每次迭代追加:                                                      │
│  ① assistant 消息（含 tool_calls + reasoning_content）              │
│  ② tool 消息（每个工具调用一条）                                     │
│  ③ user 消息（反思提示）                                            │
│                                                                    │
│  这些消息只存在于当前请求的内存中，不持久化到 Session                    │
└─────────────────────────────────────────────────────────────────────┘
```

### 3.2 哪些状态变化会影响 Prompt？

| 状态变化 | 影响的 Prompt 区域 | 时机 |
|---------|-------------------|------|
| 时间流逝 | 身份区的 `Current Time` | 每次请求重新生成 |
| 用户编辑 Bootstrap 文件 | system prompt 中的 Bootstrap 段 | 每次请求重新加载 |
| 长期记忆被更新 | system prompt 中的 Memory 段 | 压缩后下次请求生效 |
| 新消息产生 | 历史消息区 | `session.get_history()` 取最新 N 条 |
| 技能安装/卸载 | 技能摘要区 | `list_skills()` 每次重新扫描 |
| 工具注册/注销 | `tools` 参数 | `get_definitions()` 每次重新生成 |
| 切换 channel/chat_id | 会话元数据 | `_set_tool_context()` 每次请求设置 |

### 3.3 状态不影响的部分

| 内容 | 原因 |
|------|------|
| 核心身份文本 | 硬编码在 `_get_identity()` 中 |
| 反思提示文本 | 硬编码 `"Reflect on the results and decide next steps."` |
| 子 Agent 的 system prompt | 硬编码在 `_build_subagent_prompt()` 中 |
| 记忆压缩的 prompt 模板 | 硬编码在 `_consolidate_memory()` 中 |

---

## 四、LLM 的输出是如何被解析的？

### 4.1 解析入口

```
文件: nanobot/providers/litellm_provider.py:161-199
方法: LiteLLMProvider._parse_response()
```

### 4.2 解析流程

```python
def _parse_response(self, response: Any) -> LLMResponse:
    choice = response.choices[0]           # ① 取第一个 choice
    message = choice.message               # ② 取 message 对象

    # ③ 解析工具调用
    tool_calls = []
    if hasattr(message, "tool_calls") and message.tool_calls:
        for tc in message.tool_calls:
            args = tc.function.arguments
            if isinstance(args, str):         # 参数可能是 JSON 字符串
                try:
                    args = json.loads(args)    # 尝试解析
                except json.JSONDecodeError:
                    args = {"raw": args}       # 解析失败：包装为 {"raw": "原始文本"}
            tool_calls.append(ToolCallRequest(
                id=tc.id,
                name=tc.function.name,
                arguments=args,
            ))

    # ④ 解析 token 用量
    usage = {}
    if hasattr(response, "usage") and response.usage:
        usage = {
            "prompt_tokens": response.usage.prompt_tokens,
            "completion_tokens": response.usage.completion_tokens,
            "total_tokens": response.usage.total_tokens,
        }

    # ⑤ 解析思维链（Kimi/DeepSeek-R1 等）
    reasoning_content = getattr(message, "reasoning_content", None)

    return LLMResponse(
        content=message.content,            # 文本回复（可能为 None）
        tool_calls=tool_calls,
        finish_reason=choice.finish_reason or "stop",
        usage=usage,
        reasoning_content=reasoning_content,
    )
```

### 4.3 三种输出格式的处理路径

```
LLM 输出
  │
  ├─ 纯文本回复 ─────────────────────────────────────────────────┐
  │   message.content = "你好，这是回答"                          │
  │   message.tool_calls = None                                 │
  │   → LLMResponse(content="你好...", tool_calls=[])            │
  │   → has_tool_calls = False                                  │
  │   → 循环终止，返回 content 作为最终回复                        │
  │                                                             │
  ├─ Function Calling ──────────────────────────────────────────┤
  │   message.content = "让我查一下..." (或 None)                 │
  │   message.tool_calls = [                                    │
  │     FunctionCall(id="call_abc",                             │
  │       function=Function(name="read_file",                   │
  │         arguments='{"path": "/foo/bar"}'))                  │
  │   ]                                                         │
  │   → LLMResponse(                                            │
  │       content="让我查一下...",                                │
  │       tool_calls=[ToolCallRequest(                           │
  │         id="call_abc", name="read_file",                    │
  │         arguments={"path": "/foo/bar"})])                   │
  │   → has_tool_calls = True                                   │
  │   → 进入工具执行分支                                         │
  │                                                             │
  └─ 思维链 + 文本/工具调用 ────────────────────────────────────┤
      message.reasoning_content = "让我想想..."                  │
      message.content = "答案是..."                              │
      → LLMResponse(                                            │
          content="答案是...",                                   │
          reasoning_content="让我想想...")                        │
      → reasoning_content 在回注时保留，                          │
        传入 add_assistant_message() 确保后续轮次兼容             │
      （部分模型会拒绝不含 reasoning_content 的历史消息）           │
└───────────────────────────────────────────────────────────────┘
```

### 4.4 JSON 解析的特殊场景

#### 场景 1：工具参数解析

```python
# litellm_provider.py:170-175
args = tc.function.arguments
if isinstance(args, str):
    try:
        args = json.loads(args)       # 正常路径：JSON 字符串 → dict
    except json.JSONDecodeError:
        args = {"raw": args}          # 异常路径：保留原始文本
```

这个兜底设计意味着：即使 LLM 生成了不合法的 JSON 参数，系统也不会崩溃。但 `{"raw": "..."}` 会导致工具的 `validate_params()` 报错（缺少 required 字段），错误字符串回传给 LLM 重试。

#### 场景 2：记忆压缩的 JSON 解析

```python
# loop.py:397-406
text = (response.content or "").strip()
if text.startswith("```"):                              # 去除 Markdown 代码块
    text = text.split("\n", 1)[-1].rsplit("```", 1)[0].strip()
result = json.loads(text)                                # 直接 json.loads

if entry := result.get("history_entry"):
    memory.append_history(entry)
if update := result.get("memory_update"):
    if update != current_memory:
        memory.write_long_term(update)
```

这是系统中 **唯一要求 LLM 返回 JSON 格式** 的地方。处理了 Markdown 代码块包裹的情况，但如果 LLM 返回的不是合法 JSON，整个压缩操作会在 `try/except` 中静默失败。

### 4.5 LLM 错误的处理

```python
# litellm_provider.py:151-159
try:
    response = await acompletion(**kwargs)
    return self._parse_response(response)
except Exception as e:
    return LLMResponse(
        content=f"Error calling LLM: {str(e)}",    # 错误作为 content 返回
        finish_reason="error",
    )
```

LLM API 调用失败时不会抛异常到上层，而是返回一个 `finish_reason="error"` 的 `LLMResponse`。此时 `has_tool_calls = False`，ReAct 循环会终止，错误消息会作为"最终回复"发送给用户。

---

## 五、模型输出格式变化时的断裂点分析

### 5.1 高风险断裂点

#### 断裂点 1：`response.choices[0].message` 结构变化

```python
# litellm_provider.py:163-164
choice = response.choices[0]
message = choice.message
```

**风险**：如果新模型/API 版本改变了 `choices` 的结构（如嵌套方式不同、字段名变化），会抛 `AttributeError` 或 `IndexError`。

**影响范围**：所有 LLM 调用（主循环 + 子 Agent + 记忆压缩）。

**缓解**：LiteLLM 作为中间层负责标准化，但 LiteLLM 本身的版本升级也可能引入不兼容变化。

#### 断裂点 2：`tool_calls` 格式变化

```python
# litellm_provider.py:167-181
if hasattr(message, "tool_calls") and message.tool_calls:
    for tc in message.tool_calls:
        args = tc.function.arguments      # ← 依赖 .function.arguments 属性路径
        tool_calls.append(ToolCallRequest(
            id=tc.id,                      # ← 依赖 .id
            name=tc.function.name,         # ← 依赖 .function.name
            arguments=args,
        ))
```

**风险**：
- 如果 API 将 `function` 改为 `tool` 或扁平化结构
- 如果 `arguments` 从 JSON 字符串变为原生 dict（已处理）或变为其他格式
- 如果 `id` 字段名变化或格式变化

**影响范围**：所有依赖工具调用的流程。

#### 断裂点 3：工具调用 ID 的匹配

```python
# 回注时：
messages.append({
    "role": "tool",
    "tool_call_id": tool_call.id,    # ← 必须与 assistant 消息中的 id 匹配
    ...
})
```

**风险**：如果模型生成的 `id` 格式不被 API 接受（如长度、字符集变化），后续的 LLM 调用可能拒绝该消息序列。

**影响范围**：ReAct 循环第二次及之后的迭代。

#### 断裂点 4：`reasoning_content` 字段

```python
# litellm_provider.py:191
reasoning_content = getattr(message, "reasoning_content", None)

# context.py:234-235
if reasoning_content:
    msg["reasoning_content"] = reasoning_content
```

**风险**：
- 不同模型使用不同的字段名（如 `thinking`、`reasoning`、`chain_of_thought`）
- 模型要求在历史消息中包含此字段，但 Nanobot 只在有值时添加 — 如果模型要求即使为空也必须存在，会导致拒绝

**影响范围**：思维链模型（Kimi、DeepSeek-R1 等）的历史消息兼容性。

#### 断裂点 5：记忆压缩的 JSON 输出

```python
# loop.py:400-401
result = json.loads(text)
if entry := result.get("history_entry"):
```

**风险**：
- 不同模型对"只返回 JSON"指令的遵从度不同
- 模型可能在 JSON 前后添加解释性文字
- 模型可能使用不同的 key 名（如 `history` 而非 `history_entry`）

**影响范围**：长期记忆系统 — 压缩失败会导致记忆不更新（静默退化，不崩溃）。

### 5.2 中风险断裂点

#### 断裂点 6：`finish_reason` 值域

```python
# base.py:22
finish_reason: str = "stop"

# litellm_provider.py:196
finish_reason=choice.finish_reason or "stop"
```

当前代码不基于 `finish_reason` 做任何分支决策（只靠 `has_tool_calls`）。但如果新模型使用特殊的 `finish_reason`（如 `"content_filter"`、`"max_tokens"`），可能需要额外处理。

#### 断裂点 7：`content` 为 None 的处理

```python
# context.py:228
msg: dict[str, Any] = {"role": "assistant", "content": content or ""}
```

当 LLM 只返回工具调用时，`content` 可能为 `None`。代码已处理（转为空字符串）。但如果新 API 版本改为不返回 `content` 字段本身（而非返回 `None`），`message.content` 可能抛 `AttributeError`。

### 5.3 断裂点影响矩阵

| 断裂点 | 触发条件 | 崩溃 vs 退化 | 影响范围 |
|--------|---------|-------------|---------|
| choices 结构 | API 大版本更新 | **崩溃** | 全局 |
| tool_calls 格式 | 新模型/新 API | **崩溃** | 工具调用 |
| tool_call_id 匹配 | 新模型 ID 格式 | 退化（LLM 报错） | 多轮工具调用 |
| reasoning_content | 模型字段名变化 | 退化（思维链丢失） | 思维链模型 |
| JSON 压缩输出 | 模型不遵守格式 | 退化（静默跳过） | 记忆压缩 |
| finish_reason | 新终止类型 | 退化（被忽略） | 安全/限制场景 |
| content = None | API 不返回字段 | **崩溃** | 全局 |

---

## 六、Prompt 组合流程图

### 6.1 完整组装时序

```
_process_message(msg)
│
├─① session = sessions.get_or_create(msg.session_key)
│
├─② context.build_messages(history, current_message, media, channel, chat_id)
│   │
│   ├─ build_system_prompt()
│   │   ├─ _get_identity()
│   │   │   └─ f-string 插入: now, tz, runtime, workspace_path
│   │   │
│   │   ├─ _load_bootstrap_files()
│   │   │   └─ for f in [AGENTS.md, SOUL.md, USER.md, TOOLS.md, IDENTITY.md]:
│   │   │       if exists: parts.append(f"## {f}\n\n{content}")
│   │   │
│   │   ├─ memory.get_memory_context()
│   │   │   └─ read MEMORY.md → f"## Long-term Memory\n{content}"
│   │   │
│   │   ├─ skills.get_always_skills() → load full SKILL.md content
│   │   │
│   │   ├─ skills.build_skills_summary() → XML index
│   │   │
│   │   └─ += f"\n\n## Current Session\nChannel: {channel}\nChat ID: {chat_id}"
│   │
│   ├─ messages = [{"role": "system", "content": system_prompt}]
│   │
│   ├─ messages.extend(history)    ← session.get_history(max=50)
│   │
│   └─ messages.append({"role": "user", "content": _build_user_content(text, media)})
│       └─ if media: base64 encode images → multimodal content
│
├─③ _run_agent_loop(initial_messages)
│   │
│   │  ╔══════════════════════════════════════╗
│   │  ║  迭代 N                              ║
│   │  ╚══════════════════════════════════════╝
│   │
│   ├─ provider.chat(messages, tools=get_definitions())
│   │   │
│   │   └─ LiteLLMProvider.chat()
│   │       ├─ _resolve_model()           ← 供应商前缀
│   │       ├─ _apply_model_overrides()   ← 模型特定参数
│   │       ├─ acompletion(**kwargs)       ← LiteLLM 调用
│   │       └─ _parse_response()          ← 解析为 LLMResponse
│   │
│   ├─ if has_tool_calls:
│   │   ├─ add_assistant_message(content, tool_calls, reasoning_content)
│   │   ├─ for tc: execute → add_tool_result(id, name, result)
│   │   └─ append({"role": "user", "content": "Reflect on..."})
│   │   → 继续迭代
│   │
│   └─ else:
│       └─ final_content = response.content → break
│
├─④ session.add_message("user", msg.content)
├─  session.add_message("assistant", final_content, tools_used=...)
├─  sessions.save(session)
│
└─⑤ return OutboundMessage(channel, chat_id, final_content)
```

### 6.2 三种场景的消息列表对比

#### 场景 A：简单问答（无工具调用）

```json
[
  {"role": "system", "content": "[身份 + bootstrap + 记忆 + 技能 + 会话元数据]"},
  {"role": "user", "content": "你好"},
  {"role": "assistant", "content": "之前的回复"},
  {"role": "user", "content": "今天天气怎么样？"}
]

→ LLM 返回: content="我无法实时获取天气信息...", tool_calls=[]
→ 循环终止，1 次迭代
```

#### 场景 B：单工具调用

```json
[
  {"role": "system", "content": "[身份 + bootstrap + 记忆 + 技能 + 会话元数据]"},
  {"role": "user", "content": "读取 README.md"}
]

→ 迭代 1:
  LLM 返回: tool_calls=[read_file(path="README.md")]
  追加:
  {"role": "assistant", "content": "", "tool_calls": [...]},
  {"role": "tool", "tool_call_id": "call_1", "name": "read_file", "content": "# Nanobot..."},
  {"role": "user", "content": "Reflect on the results and decide next steps."}

→ 迭代 2:
  LLM 返回: content="README.md 的内容是...", tool_calls=[]
  循环终止
```

#### 场景 C：多轮工具链

```json
[
  {"role": "system", "content": "[身份 + bootstrap + 记忆 + 技能 + 会话元数据]"},
  {"role": "user", "content": "找到所有 Python 文件并统计总行数"}
]

→ 迭代 1:
  LLM: tool_calls=[exec(command="find . -name '*.py'")]
  追加: assistant → tool(文件列表) → user(Reflect)

→ 迭代 2:
  LLM: tool_calls=[exec(command="wc -l file1.py file2.py ...")]
  追加: assistant → tool(行数统计) → user(Reflect)

→ 迭代 3:
  LLM: content="共有 15 个 Python 文件，总计 2,847 行代码"
  循环终止
```

---

## 七、修改前 / 修改后的示例对比

### 7.1 修改 system prompt 的身份描述

**目标**：将 Agent 的身份从通用助手改为专注于代码审查的助手。

**修改前**（`context.py:83-110`）：

```python
return f"""# nanobot 🐈

You are nanobot, a helpful AI assistant. You have access to tools that allow you to:
- Read, write, and edit files
- Execute shell commands
- Search the web and fetch web pages
- Send messages to users on chat channels
- Spawn subagents for complex background tasks

...
Always be helpful, accurate, and concise. When using tools, think step by step: ...
"""
```

**修改后**（方案 A — 修改代码）：

```python
return f"""# nanobot 🐈 — Code Review Specialist

You are nanobot, a code review assistant. Your primary focus is:
- Reviewing code changes for bugs, security issues, and style problems
- Suggesting improvements with concrete code examples
- Explaining complex code patterns

You have access to tools that allow you to:
- Read, write, and edit files
- Execute shell commands (git diff, linting, tests)
- Search the web for best practices

...
When reviewing code, always:
1. Read the full file before commenting
2. Check for OWASP Top 10 vulnerabilities
3. Suggest specific fixes, not just point out problems
"""
```

**修改后**（方案 B — 使用 Bootstrap 文件，无需改代码）：

创建 `~/.nanobot/workspace/IDENTITY.md`：

```markdown
# Code Review Specialist

Your primary focus is code review. When asked to review code:
1. Read the full file before commenting
2. Check for OWASP Top 10 vulnerabilities
3. Suggest specific fixes, not just point out problems
4. Consider performance implications
```

方案 B 更灵活，因为 `IDENTITY.md` 在 `BOOTSTRAP_FILES` 列表中，会自动被加载到 system prompt。

### 7.2 修改反思提示

**目标**：让 Agent 在工具执行后生成更结构化的反思。

**修改前**（`loop.py:184`）：

```python
messages.append({"role": "user", "content": "Reflect on the results and decide next steps."})
```

**修改后**：

```python
messages.append({"role": "user", "content": (
    "Review the tool results above. Then:\n"
    "1. Summarize what you learned\n"
    "2. Identify if more information is needed\n"
    "3. Either use another tool or provide your final answer"
)})
```

**影响**：
- 改善 LLM 的推理质量（更明确的指引）
- 可能增加 token 消耗（更长的提示）
- 对所有模型生效（无法按模型差异化）

### 7.3 修改记忆压缩 prompt 的输出格式

**目标**：让记忆压缩同时提取"待办事项"。

**修改前**（`loop.py:375-387`）：

```python
prompt = f"""You are a memory consolidation agent. Process this conversation and return a JSON object with exactly two keys:

1. "history_entry": A paragraph (2-5 sentences) summarizing ...
2. "memory_update": The updated long-term memory content. ...

...
Respond with ONLY valid JSON, no markdown fences."""
```

**修改后**：

```python
prompt = f"""You are a memory consolidation agent. Process this conversation and return a JSON object with exactly three keys:

1. "history_entry": A paragraph (2-5 sentences) summarizing ...
2. "memory_update": The updated long-term memory content. ...
3. "pending_tasks": A list of strings, each being an unfinished task or follow-up mentioned in the conversation. Empty list if none.

...
Respond with ONLY valid JSON, no markdown fences."""
```

**同时需要修改解析代码**（`loop.py:402-406`）：

```python
# 修改前：
if entry := result.get("history_entry"):
    memory.append_history(entry)
if update := result.get("memory_update"):
    if update != current_memory:
        memory.write_long_term(update)

# 修改后：
if entry := result.get("history_entry"):
    memory.append_history(entry)
if update := result.get("memory_update"):
    if update != current_memory:
        memory.write_long_term(update)
if tasks := result.get("pending_tasks"):
    # 将待办事项追加到 MEMORY.md 的专用段落
    current = memory.read_long_term()
    tasks_section = "\n## Pending Tasks\n" + "\n".join(f"- {t}" for t in tasks)
    memory.write_long_term(current + tasks_section)
```

**风险**：模型可能不稳定地生成第三个字段，需要兜底 `result.get("pending_tasks", [])`。

### 7.4 添加子 Agent 的记忆上下文

**目标**：让子 Agent 也能看到长期记忆。

**修改前**（`subagent.py:218-253`）：

```python
def _build_subagent_prompt(self, task: str) -> str:
    return f"""# Subagent
    ...
    ## Workspace
    Your workspace is at: {self.workspace}
    Skills are available at: {self.workspace}/skills/ (read SKILL.md files as needed)
    ..."""
```

**修改后**：

```python
def _build_subagent_prompt(self, task: str) -> str:
    # 注入长期记忆
    memory = MemoryStore(self.workspace)
    memory_context = memory.get_memory_context()
    memory_section = f"\n\n## Memory\n{memory_context}" if memory_context else ""

    return f"""# Subagent
    ...
    ## Workspace
    Your workspace is at: {self.workspace}
    Skills are available at: {self.workspace}/skills/ (read SKILL.md files as needed)
    {memory_section}
    ..."""
```

**影响**：
- 子 Agent 获得长期记忆上下文，能更好地理解项目背景
- 增加 token 消耗（MEMORY.md 可能较大）
- 需要在 `subagent.py` 顶部添加 `from nanobot.agent.memory import MemoryStore`
