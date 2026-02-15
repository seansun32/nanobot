# Nanobot 项目架构概览

> 面向新开发者的快速上手指南，基于项目目录结构与源码分析。

## 一、项目整体结构与分层逻辑

Nanobot 是一个超轻量级（约 3,500 行核心代码）的个人 AI 助手框架，采用 **Python + TypeScript** 双语言栈：

- **Python（主体）**：agent 核心、消息总线、多渠道接入、定时任务、配置管理
- **TypeScript（bridge/）**：WhatsApp 接入桥，通过 WebSocket 与 Python 后端通信

整体采用 **消息驱动 + 分层解耦** 的架构风格，核心数据流为：

```
用户消息 → Channel → MessageBus(inbound) → AgentLoop → LLM + Tools → MessageBus(outbound) → Channel → 用户
```

项目顶层目录结构：

```
nanobot/                  # 项目根目录
├── nanobot/              # Python 主包（核心代码）
│   ├── agent/            # 🧠 Agent 核心引擎
│   ├── bus/              # 🚌 消息总线
│   ├── channels/         # 📱 聊天渠道接入层
│   ├── cli/              # 🖥️ CLI 命令行入口
│   ├── config/           # ⚙️ 配置加载与校验
│   ├── cron/             # ⏰ 定时任务调度
│   ├── heartbeat/        # 💓 周期性唤醒服务
│   ├── providers/        # 🤖 LLM 提供商抽象
│   ├── session/          # 💬 会话持久化
│   ├── skills/           # 🎯 内置技能（Markdown 指令文件）
│   └── utils/            # 🔧 工具函数
├── bridge/               # 🌉 WhatsApp Node.js 桥接服务
├── workspace/            # 📁 Agent 运行时工作区模板
├── tests/                # 🧪 测试用例
├── case/                 # 📸 演示 GIF 素材
├── docs/                 # 📖 项目文档
├── pyproject.toml        # 📦 项目元数据与依赖
├── Dockerfile            # 🐳 容器化部署
└── README.md             # 📝 项目说明
```

## 二、各主要目录 / 模块的功能定位

### 2.1 `nanobot/agent/` — Agent 核心引擎

整个项目的 **大脑**，负责接收消息、构建上下文、调用 LLM、执行工具、管理记忆。

| 文件 | 功能 |
|------|------|
| `loop.py` | **Agent 主循环**：从 MessageBus 消费消息 → 构建上下文 → 调用 LLM → 执行工具调用 → 循环直到完成。是整个系统的调度中枢。 |
| `context.py` | **上下文构建器**：组装系统提示词，加载 bootstrap 文件（AGENTS.md、SOUL.md 等）、记忆、技能摘要，构建完整的 LLM 消息列表。 |
| `memory.py` | **记忆系统**：双层持久化设计 — `MEMORY.md`（长期事实记忆）+ `HISTORY.md`（追加式事件日志）。 |
| `skills.py` | **技能加载器**：扫描内置和用户自定义技能目录，解析 SKILL.md 的 YAML 前置元数据，按需加载技能指令到上下文中。 |
| `subagent.py` | **子 Agent 管理器**：创建隔离的后台任务执行实例，拥有独立上下文，完成后通过系统消息将结果汇报给主 Agent。 |
| `tools/` | **内置工具集**：Agent 可调用的工具实现。 |

#### `nanobot/agent/tools/` — 工具子系统

| 文件 | 工具 | 功能 |
|------|------|------|
| `base.py` | `Tool` (ABC) | 工具抽象基类，定义 `name`、`description`、`parameters`、`execute()` 接口 |
| `registry.py` | `ToolRegistry` | 工具注册中心，动态注册/查找/执行工具，生成 OpenAI 格式的工具 schema |
| `filesystem.py` | `ReadFile`, `WriteFile`, `EditFile`, `ListDir` | 文件系统操作，支持工作区路径限制 |
| `shell.py` | `Exec` | Shell 命令执行，内置危险命令拦截（如 `rm -rf /`），支持超时与输出截断 |
| `web.py` | `WebSearch`, `WebFetch` | 网页搜索（Brave Search API）与网页内容抓取（Readability 提取） |
| `message.py` | `Message` | 向聊天渠道发送消息 |
| `spawn.py` | `Spawn` | 创建后台子 Agent 执行异步任务 |
| `cron.py` | `Cron` | 创建/管理定时任务（支持 cron 表达式、固定间隔、一次性定时） |

### 2.2 `nanobot/bus/` — 消息总线

Agent 与 Channel 之间的 **解耦层**，基于 `asyncio.Queue` 实现异步消息传递。

| 文件 | 功能 |
|------|------|
| `events.py` | 定义消息数据结构：`InboundMessage`（渠道 → Agent）和 `OutboundMessage`（Agent → 渠道），会话键为 `{channel}:{chat_id}` |
| `queue.py` | `MessageBus` 实现：双向队列（inbound/outbound），支持发布/消费/订阅模式，后台分发出站消息到对应渠道 |

### 2.3 `nanobot/channels/` — 聊天渠道接入层

多平台消息接入的 **适配器层**，每个渠道实现统一的 `BaseChannel` 接口。

| 文件 | 功能 |
|------|------|
| `base.py` | 渠道抽象基类：定义 `start()`、`stop()`、`send()` 接口，提供发送者白名单校验 |
| `manager.py` | 渠道管理器：按配置懒加载启用的渠道，统一启动/停止，路由出站消息到对应渠道实例 |
| `telegram.py` | Telegram Bot 接入 |
| `discord.py` | Discord Bot 接入 |
| `whatsapp.py` | WhatsApp 接入（通过 bridge/ WebSocket 桥接） |
| `feishu.py` | 飞书接入（WebSocket 长连接） |
| `dingtalk.py` | 钉钉接入（Stream 模式） |
| `slack.py` | Slack 接入（Socket 模式） |
| `email.py` | Email 接入（IMAP 轮询 + SMTP 回复） |
| `mochat.py` | Mochat / Claw IM 接入（Socket.IO） |
| `qq.py` | QQ 接入（botpy SDK） |

### 2.4 `nanobot/providers/` — LLM 提供商抽象

通过 **Provider Registry + LiteLLM** 实现多模型供应商的统一接入。

| 文件 | 功能 |
|------|------|
| `base.py` | 抽象接口：定义 `LLMProvider`、`LLMResponse`、`ToolCallRequest` 数据结构 |
| `registry.py` | **Provider 注册表**：声明式定义所有供应商元数据（`ProviderSpec`），包括模型前缀、环境变量映射、网关检测策略等。添加新供应商只需在此注册 |
| `litellm_provider.py` | **LiteLLM 实现**：实际的 LLM 调用层，处理环境变量设置、模型名称解析、供应商特定参数覆盖 |
| `transcription.py` | 语音转文字（通过 Groq Whisper） |

### 2.5 `nanobot/config/` — 配置管理

| 文件 | 功能 |
|------|------|
| `schema.py` | Pydantic 模型定义：`Config`（根）、`AgentsConfig`、`ChannelsConfig`、`ProvidersConfig`、`ToolsConfig` 及各渠道独立配置 |
| `loader.py` | 配置加载/保存：支持 camelCase ↔ snake_case 自动转换、配置迁移、JSON 格式 |

配置文件路径：`~/.nanobot/config.json`

### 2.6 `nanobot/session/` — 会话管理

| 文件 | 功能 |
|------|------|
| `manager.py` | `SessionManager`：按会话键（`channel:chat_id`）管理对话历史，持久化为 JSONL 文件，存储于 `~/.nanobot/sessions/` |

### 2.7 `nanobot/cron/` — 定时任务

| 文件 | 功能 |
|------|------|
| `service.py` | `CronService`：支持三种调度方式 — cron 表达式、固定间隔、一次性定时。任务持久化为 JSON 文件 |
| `types.py` | 定时任务数据类型定义 |

### 2.8 `nanobot/heartbeat/` — 心跳服务

| 文件 | 功能 |
|------|------|
| `service.py` | `HeartbeatService`：每 30 分钟唤醒 Agent 检查 `HEARTBEAT.md`，如有待办任务则触发 Agent 处理 |

### 2.9 `nanobot/cli/` — 命令行入口

| 文件 | 功能 |
|------|------|
| `commands.py` | 基于 Typer 的 CLI 应用，提供 `onboard`、`agent`、`gateway`、`status`、`channels`、`cron` 等命令 |

入口点定义于 `pyproject.toml`：`nanobot = "nanobot.cli.commands:app"`

### 2.10 `nanobot/skills/` — 内置技能

以 Markdown 文件（`SKILL.md`）形式定义的 Agent 技能指令，每个子目录为一个技能：

| 技能 | 功能 |
|------|------|
| `cron/` | 定时任务管理指令 |
| `github/` | GitHub 操作指令 |
| `memory/` | 记忆管理指令 |
| `skill-creator/` | 技能创建指令 |
| `summarize/` | 内容总结指令 |
| `tmux/` | tmux 会话管理（含 shell 脚本） |
| `weather/` | 天气查询指令 |

### 2.11 `bridge/` — WhatsApp 桥接服务

独立的 Node.js/TypeScript 服务，通过 Baileys 库连接 WhatsApp：

| 文件 | 功能 |
|------|------|
| `src/index.ts` | 入口，启动桥接服务器 |
| `src/server.ts` | WebSocket 服务器（默认端口 3001），接收 Python 端指令、广播 WhatsApp 消息事件 |
| `src/whatsapp.ts` | WhatsApp 客户端封装 |
| `src/types.d.ts` | TypeScript 类型定义 |

### 2.12 `workspace/` — 运行时工作区模板

Agent 初始化时复制到 `~/.nanobot/workspace/` 的模板文件：

| 文件 | 功能 |
|------|------|
| `AGENTS.md` | Agent 行为指令：工具使用规范、记忆管理、心跳任务处理 |
| `SOUL.md` | Agent 人格定义：性格特征、价值观、沟通风格 |
| `USER.md` | 用户画像模板：姓名、时区、偏好、兴趣 |
| `TOOLS.md` | 工具使用文档与示例 |
| `HEARTBEAT.md` | 周期性待办任务列表 |
| `memory/MEMORY.md` | 长期记忆存储 |

### 2.13 `tests/` — 测试

| 文件 | 功能 |
|------|------|
| `test_cli_input.py` | CLI 输入解析测试 |
| `test_commands.py` | CLI 命令测试 |
| `test_consolidate_offset.py` | 记忆压缩偏移测试 |
| `test_docker.sh` | Docker 构建测试脚本 |
| `test_email_channel.py` | Email 渠道测试 |
| `test_tool_validation.py` | 工具参数校验测试 |

## 三、模块间依赖与调用关系

### 3.1 核心数据流

```
┌─────────────────────────────────────────────────────────────────────┐
│                          CLI (commands.py)                          │
│            初始化所有组件，启动 Agent 循环或 Gateway                    │
└──────┬──────────┬──────────┬──────────┬──────────┬──────────────────┘
       │          │          │          │          │
       ▼          ▼          ▼          ▼          ▼
  AgentLoop  ChannelMgr  CronService  Heartbeat  SessionMgr
       │          │          │          │          │
       │          │          │          │          │
       ▼          ▼          ▼          ▼          │
  ┌────────── MessageBus ──────────────────┐      │
  │  inbound queue    outbound queue       │      │
  └──────┬─────────────────┬───────────────┘      │
         │                 │                      │
         ▼                 ▼                      │
    AgentLoop         ChannelMgr                  │
         │                                        │
    ┌────┴────────────────────────────────┐       │
    │          AgentLoop 内部              │       │
    │  ContextBuilder ← MemoryStore       │       │
    │  ContextBuilder ← SkillsLoader      │       │
    │  ContextBuilder ← SessionManager ───┼───────┘
    │  AgentLoop → LLMProvider            │
    │  AgentLoop → ToolRegistry → Tools   │
    │  AgentLoop → SubagentManager        │
    └─────────────────────────────────────┘
```

### 3.2 模块依赖矩阵

| 模块 | 依赖的模块 |
|------|-----------|
| `cli/commands` | config, bus, agent/loop, channels/manager, session/manager, cron/service, heartbeat/service, providers |
| `agent/loop` | bus, providers, agent/context, agent/tools, agent/memory, agent/subagent, session/manager |
| `agent/context` | agent/memory, agent/skills |
| `agent/subagent` | bus, providers, agent/tools |
| `channels/manager` | config, bus, channels/* |
| `channels/*` | bus, channels/base |
| `providers/litellm` | providers/registry, providers/base |
| `config/loader` | config/schema |
| `config/schema` | providers/registry |
| `cron/service` | cron/types |
| `heartbeat/service` | (无外部依赖，通过回调与 AgentLoop 通信) |
| `bus/` | (无外部依赖，纯基础设施) |
| `session/manager` | (无外部依赖) |

### 3.3 关键设计模式

1. **消息总线解耦**：Channel 和 Agent 通过 `MessageBus` 的双向队列通信，彼此不直接引用，实现了渠道与核心逻辑的完全解耦。

2. **注册表模式**：`ToolRegistry` 和 `ProviderRegistry` 均采用声明式注册，新增工具或 LLM 供应商无需修改调用链。

3. **上下文组装**：`ContextBuilder` 将 bootstrap 文件、记忆、技能、会话历史等组装成完整的 LLM 提示词，是 Agent 行为的核心控制点。

4. **适配器模式**：所有 Channel 实现统一的 `BaseChannel` 接口，所有 Provider 实现统一的 `LLMProvider` 接口。

5. **回调通信**：`CronService` 和 `HeartbeatService` 通过回调函数与 `AgentLoop` 交互，避免循环依赖。

6. **子 Agent 隔离**：`SubagentManager` 创建拥有独立上下文但受限工具集的子任务，完成后通过系统消息汇报结果。

### 3.4 新开发者快速定位指南

| 你想做什么 | 从哪里开始 |
|-----------|-----------|
| 理解 Agent 如何处理消息 | `agent/loop.py` → `_process_message()` → `_run_agent_loop()` |
| 添加新的内置工具 | 在 `agent/tools/` 下创建文件，继承 `Tool` 基类，在 `agent/loop.py` 中注册 |
| 接入新的聊天平台 | 在 `channels/` 下创建文件，继承 `BaseChannel`，在 `channels/manager.py` 中注册 |
| 添加新的 LLM 供应商 | 在 `providers/registry.py` 添加 `ProviderSpec`，在 `config/schema.py` 添加配置字段 |
| 修改 Agent 人格/行为 | 编辑 `workspace/SOUL.md`（人格）或 `workspace/AGENTS.md`（行为指令） |
| 添加新技能 | 在 `skills/` 下创建目录和 `SKILL.md` 文件 |
| 理解配置如何加载 | `config/loader.py` → `config/schema.py` |
| 理解消息如何路由 | `bus/queue.py`（MessageBus）→ `channels/manager.py`（分发） |
