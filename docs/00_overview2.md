# Nanobot Agent 架构深度分析

> 从 AI Agent 框架工程视角，对 nanobot 进行架构级解读。

---

## 一、Agent 架构类型判定

### 1.1 主架构：ReAct（Reasoning + Acting）

Nanobot 的核心实现是一个 **经典的 ReAct 循环**。证据链如下：

| 特征 | Nanobot 中的体现 |
|------|-----------------|
| **观察-思考-行动循环** | `AgentLoop._run_agent_loop()` 中 `while iteration < max_iterations` 的迭代循环 |
| **LLM 作为推理引擎** | 每轮迭代调用 `provider.chat(messages, tools)` 让 LLM 决定是否使用工具 |
| **工具调用作为行动** | LLM 返回 `tool_calls` 时执行工具，将结果追加到上下文中 |
| **反思提示** | 工具执行后注入 `"Reflect on the results and decide next steps."` 引导 LLM 反思 |
| **自然终止** | 当 LLM 不再请求工具调用时，循环终止并输出最终回复 |

### 1.2 辅助架构特征

除 ReAct 主循环外，nanobot 还融合了以下架构模式：

- **Tool-Augmented**：通过 `ToolRegistry` 动态注册 10 种工具，LLM 以 OpenAI function calling 协议调用
- **Event-Driven**：`MessageBus` 实现异步消息驱动，Channel 与 Agent 通过事件队列解耦
- **Multi-Agent（轻量）**：`SubagentManager` 支持主 Agent 派生后台子 Agent，但采用扁平结构（无分层调度），子 Agent 不能再派生子 Agent
- **Proactive Agent**：`HeartbeatService` + `CronService` 让 Agent 可以主动唤醒执行任务，而非仅被动响应

### 1.3 架构定性总结

```
主架构：ReAct Loop（推理-行动循环）
增强层：Tool-Augmented（工具增强）+ Event-Driven（消息驱动）
扩展层：Lightweight Multi-Agent（轻量多 Agent）+ Proactive Scheduling（主动调度）
```

---

## 二、核心运行时组件及职责

### 2.1 组件总览

| 组件 | 类名 | 所在模块 | 核心职责 |
|------|------|---------|---------|
| **Agent 主循环** | `AgentLoop` | `agent/loop.py` | ReAct 循环的调度中枢：消费消息 → 构建上下文 → 调用 LLM → 执行工具 → 返回响应 |
| **上下文构建器** | `ContextBuilder` | `agent/context.py` | 组装系统提示词：拼接身份信息、bootstrap 文件、记忆、技能摘要、会话历史 |
| **记忆存储** | `MemoryStore` | `agent/memory.py` | 双层持久化记忆：`MEMORY.md`（长期事实）+ `HISTORY.md`（事件日志） |
| **工具注册中心** | `ToolRegistry` | `agent/tools/registry.py` | 工具的注册、查找、Schema 生成、参数校验与执行分派 |
| **子 Agent 管理器** | `SubagentManager` | `agent/subagent.py` | 派生后台子 Agent：独立上下文、受限工具集、完成后通过系统消息汇报 |
| **技能加载器** | `SkillsLoader` | `agent/skills.py` | 扫描 Markdown 技能文件，解析元数据，按需注入上下文 |
| **消息总线** | `MessageBus` | `bus/queue.py` | 双向异步队列：解耦 Channel 层与 Agent 核心 |
| **渠道管理器** | `ChannelManager` | `channels/manager.py` | 统一管理多平台接入：初始化、启停、出站消息路由 |
| **LLM 提供商** | `LiteLLMProvider` | `providers/litellm_provider.py` | 统一 LLM 调用接口：模型解析、环境变量配置、多供应商适配 |
| **会话管理器** | `SessionManager` | `session/manager.py` | 对话历史持久化：按 `channel:chat_id` 键管理，JSONL 存储 |
| **定时任务服务** | `CronService` | `cron/service.py` | 定时/周期/一次性任务调度，触发 Agent 执行 |
| **心跳服务** | `HeartbeatService` | `heartbeat/service.py` | 周期性唤醒 Agent 检查待办（`HEARTBEAT.md`） |
| **配置系统** | `Config` + `load_config` | `config/` | Pydantic 模型校验 + JSON 加载 + camelCase 转换 |

### 2.2 组件分层

```
┌─────────────────────────────────────────────┐
│              接入层 (Interface)               │
│  CLI · Telegram · Discord · WhatsApp · ...  │
├─────────────────────────────────────────────┤
│           消息总线 (Message Bus)              │
│         InboundQueue ↔ OutboundQueue         │
├─────────────────────────────────────────────┤
│          Agent 核心 (Agent Core)             │
│  AgentLoop · ContextBuilder · ToolRegistry  │
├─────────────────────────────────────────────┤
│          能力层 (Capabilities)               │
│  Tools · Skills · SubagentManager · Memory  │
├─────────────────────────────────────────────┤
│         基础设施 (Infrastructure)             │
│  LLMProvider · SessionManager · Config      │
├─────────────────────────────────────────────┤
│          调度层 (Scheduling)                  │
│       CronService · HeartbeatService         │
└─────────────────────────────────────────────┘
```

---

## 三、Agent 完整生命周期

### 3.1 从输入到输出的完整流程

```
阶段 1 — 消息接入
  用户在聊天平台发送消息
  → Channel 适配器接收（如 TelegramChannel）
  → 权限校验（allowFrom 白名单）
  → 封装为 InboundMessage(channel, sender_id, chat_id, content)
  → 投递到 MessageBus.inbound 队列

阶段 2 — 消息消费
  AgentLoop.run() 从 inbound 队列消费消息
  → 识别消息类型（普通消息 / 系统消息 / 斜杠命令）
  → 通过 session_key 获取或创建 Session
  → 检查是否需要记忆压缩（消息数 > memory_window）

阶段 3 — 上下文构建
  ContextBuilder.build_messages() 组装完整的 LLM 输入：
  → 系统提示词 = 身份信息 + bootstrap 文件 + 长期记忆 + 技能摘要
  → 历史消息 = Session 中最近 N 条对话
  → 当前消息 = 用户输入（可能包含图片附件）

阶段 4 — ReAct 循环（核心）
  _run_agent_loop() 进入迭代：
  ┌─→ 调用 LLM: provider.chat(messages, tools, model)
  │   ├─ LLM 返回 tool_calls → 执行工具 → 追加结果到 messages → 注入反思提示 → 继续循环
  │   └─ LLM 返回纯文本 → 跳出循环，得到 final_content
  └─── 重复，直到无工具调用 或 达到 max_iterations

阶段 5 — 响应与持久化
  → 将 user/assistant 消息对存入 Session
  → Session 持久化为 JSONL 文件
  → 封装为 OutboundMessage 投递到 outbound 队列

阶段 6 — 消息分发
  ChannelManager 从 outbound 队列消费
  → 根据 channel 字段路由到对应 Channel 适配器
  → Channel 调用平台 API 将回复发送给用户
```

### 3.2 特殊生命周期

| 场景 | 流程差异 |
|------|---------|
| **CLI 直接调用** | 跳过 Channel/Bus，直接调用 `AgentLoop.process_direct()` |
| **子 Agent 任务** | 主 Agent 通过 Spawn 工具触发 → 子 Agent 独立运行 ReAct 循环 → 完成后通过系统消息汇报 → 主 Agent 再次进入循环处理结果 |
| **定时任务触发** | CronService 定时器到期 → 注入 InboundMessage → 走正常 Agent 循环 |
| **心跳唤醒** | HeartbeatService 每 30 分钟唤醒 → 读取 HEARTBEAT.md → 有任务则注入消息 |
| **记忆压缩** | 消息数超阈值 → 异步调用 LLM 总结旧消息 → 写入 MEMORY.md + HISTORY.md |

---

## 四、LLM / 记忆 / 工具 / 控制循环的协作机制

### 4.1 四大组件的角色定位

| 组件 | 角色 | 类比 |
|------|------|------|
| **LLM** | 决策大脑 | 负责理解意图、选择工具、生成回复 |
| **Memory** | 长期与短期记忆 | Session = 短期工作记忆；MEMORY.md = 长期事实记忆；HISTORY.md = 事件日志 |
| **Tools** | 执行器 | Agent 与外部世界交互的手脚（文件、Shell、网络、消息） |
| **Control Loop** | 调度框架 | 协调 LLM 与 Tools 的交互节奏，控制终止条件 |

### 4.2 协作流程详解

```
                    ┌──────────────────┐
                    │  ContextBuilder  │
                    │  组装 Prompt     │
                    └──────┬───────────┘
                           │ 注入
         ┌─────────────────▼────────────────┐
         │         System Prompt            │
         │  = Identity + Bootstrap Files    │
         │  + MEMORY.md (长期记忆)           │
         │  + Skills Summary (技能摘要)      │
         ├──────────────────────────────────┤
         │     History (短期会话记忆)         │
         │  = Session 最近 N 条消息          │
         ├──────────────────────────────────┤
         │     Current Message (用户输入)    │
         └─────────────────┬────────────────┘
                           │
                    ┌──────▼──────┐
              ┌────►│     LLM     │◄────┐
              │     └──────┬──────┘     │
              │            │            │
              │     ┌──────▼──────┐     │
              │     │  决策分支    │     │
              │     └──┬──────┬───┘     │
              │        │      │         │
              │  tool_calls  纯文本     │
              │        │      │         │
              │   ┌────▼───┐  │         │
              │   │ Tools  │  │         │
              │   │ 执行   │  │         │
              │   └────┬───┘  │         │
              │        │      │         │
              │  工具结果      │         │
              │  + 反思提示    │         │
              │        │      │         │
              └────────┘      ▼         │
                         最终回复       │
                           │            │
                    ┌──────▼──────┐     │
                    │  Session    │     │
                    │  记录对话   │─────┘ (下次对话时作为 History)
                    └──────┬──────┘
                           │ 超阈值时触发
                    ┌──────▼──────────┐
                    │  记忆压缩       │
                    │  旧消息 → LLM   │
                    │  → MEMORY.md    │
                    │  → HISTORY.md   │
                    └─────────────────┘
```

### 4.3 关键协作细节

**LLM ↔ Control Loop：**
- Control Loop 向 LLM 提交带 tools schema 的消息列表
- LLM 通过 `finish_reason` 和 `tool_calls` 字段隐式告知 Control Loop 下一步动作
- Control Loop 在每次工具执行后注入反思提示 `"Reflect on the results and decide next steps."`，引导 LLM 进行 ReAct 式推理
- `max_iterations`（默认 20）作为安全阀防止无限循环

**LLM ↔ Memory：**
- 读取方向：`ContextBuilder` 在构建系统提示词时读取 `MEMORY.md`，注入 LLM 的长期上下文
- 写入方向：LLM 可通过 `write_file` 工具直接写 `MEMORY.md`；也可由 `_consolidate_memory()` 异步调用 LLM 压缩旧消息
- Session 历史作为短期工作记忆，每轮对话保留最近 `memory_window`（默认 50）条

**LLM ↔ Tools：**
- LLM 通过 OpenAI function calling 协议声明要调用的工具和参数
- `ToolRegistry` 负责参数校验（JSON Schema）和执行分派
- 工具结果以 `role: tool` 消息回传 LLM，触发下一轮推理

**Memory ↔ Control Loop：**
- Control Loop 在处理消息前检查 Session 长度，超阈值则异步触发记忆压缩
- 压缩过程本身调用 LLM（作为"记忆压缩 Agent"），生成结构化 JSON 更新 MEMORY.md 和 HISTORY.md
- `/new` 命令触发全量归档：清空 Session，将所有消息压缩入长期记忆

---

## 五、架构总览图（Mermaid）

```mermaid
graph TB
    subgraph Interface["接入层"]
        CLI["CLI<br/>commands.py"]
        TG["Telegram"]
        DC["Discord"]
        WA["WhatsApp"]
        FS["Feishu"]
        DT["DingTalk"]
        SK["Slack"]
        EM["Email"]
        MC["Mochat"]
        QQ["QQ"]
    end

    subgraph Bus["消息总线"]
        IQ["Inbound Queue"]
        OQ["Outbound Queue"]
    end

    subgraph Core["Agent 核心"]
        AL["AgentLoop<br/>ReAct 主循环"]
        CB["ContextBuilder<br/>上下文组装"]
        TR["ToolRegistry<br/>工具注册中心"]
    end

    subgraph Capabilities["能力层"]
        subgraph Tools["内置工具"]
            FS_T["ReadFile / WriteFile<br/>EditFile / ListDir"]
            SH["Exec (Shell)"]
            WEB["WebSearch / WebFetch"]
            MSG["Message"]
            SP["Spawn"]
            CR_T["Cron"]
        end
        SK_L["SkillsLoader<br/>技能加载"]
        SAM["SubagentManager<br/>子 Agent"]
        MEM["MemoryStore<br/>MEMORY.md + HISTORY.md"]
    end

    subgraph Infra["基础设施"]
        LLM["LiteLLMProvider<br/>多供应商 LLM"]
        PR["ProviderRegistry<br/>供应商元数据"]
        SM["SessionManager<br/>JSONL 会话持久化"]
        CFG["Config<br/>Pydantic 配置"]
    end

    subgraph Scheduling["调度层"]
        CRON["CronService<br/>定时任务"]
        HB["HeartbeatService<br/>周期唤醒"]
    end

    subgraph Bridge["WhatsApp 桥接"]
        BS["BridgeServer<br/>Node.js WebSocket"]
    end

    %% Interface → Bus
    TG & DC & FS & DT & SK & EM & MC & QQ -->|InboundMessage| IQ
    OQ -->|OutboundMessage| TG & DC & FS & DT & SK & EM & MC & QQ
    WA <-->|WebSocket| BS
    BS -->|InboundMessage| IQ
    CLI -->|process_direct| AL

    %% Bus → Core
    IQ --> AL
    AL --> OQ

    %% Core 内部
    AL --> CB
    AL --> TR
    CB --> MEM
    CB --> SK_L

    %% Core → Capabilities
    TR --> FS_T & SH & WEB & MSG & SP & CR_T
    SP --> SAM
    SAM -->|系统消息| IQ

    %% Core → Infra
    AL -->|chat()| LLM
    LLM --> PR
    AL --> SM
    AL --> CFG

    %% Scheduling → Bus
    CRON -->|InboundMessage| IQ
    HB -->|InboundMessage| IQ

    %% ReAct 循环标注
    AL -.->|"迭代: LLM→工具→反思→LLM"| AL

    %% 记忆压缩
    AL -.->|"异步压缩"| MEM
```

---

## 六、关键设计决策总结

| 决策 | 选择 | 理由 |
|------|------|------|
| Agent 范式 | ReAct 循环 | 简单可靠，单文件即可实现核心逻辑 |
| LLM 工具调用协议 | OpenAI function calling | 业界标准，LiteLLM 统一适配 |
| 消息传递 | asyncio.Queue 双向总线 | 轻量级，无需外部消息中间件 |
| 记忆方案 | Markdown 文件（MEMORY.md + HISTORY.md） | 零依赖、人类可读、支持 grep 检索 |
| 会话持久化 | JSONL 追加写 | 简单高效，适合单用户场景 |
| 多供应商支持 | LiteLLM + 声明式 ProviderRegistry | 新增供应商只需两步注册，无 if-elif 链 |
| 技能系统 | Markdown 文件 + YAML 前置元数据 | Agent 自身可读取和创建技能，实现自我扩展 |
| 子 Agent | 共享 LLM Provider、独立上下文、受限工具集 | 平衡能力与安全，避免无限派生 |
| 安全边界 | allowFrom 白名单 + 工作区沙箱 + Shell 命令拦截 | 多层防护，适合个人部署 |
