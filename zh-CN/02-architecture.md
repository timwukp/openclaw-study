> 本文为中文翻译版本。英文原版为权威版本，如有差异以英文版为准。
> [English Version](../02-architecture.md)

# OpenClaw 技术架构

## 技术栈

| 组件             | 技术                               |
|-----------------|------------------------------------|
| 编程语言        | TypeScript                         |
| 运行时          | Node.js                            |
| 包管理器        | pnpm（Monorepo 架构）              |
| 构建工具        | tsdown                             |
| 测试框架        | Vitest（单元测试、端到端测试、实时测试、网关测试）|
| 代码检查        | oxlint + oxfmt（基于 Rust）        |
| 原生应用        | Swift（macOS/iOS）、Android        |
| 容器化          | Docker + Podman                    |

## 架构图

```
+=====================================================================+
|                        FRONTEND / CLIENTS                            |
+=====================================================================+
|                                                                      |
|  +----------+ +----------+ +----------+ +----------+ +----------+   |
|  | WhatsApp | | Telegram | | Discord  |  |  Slack   | |  Signal  |  |
|  +----+-----+ +----+-----+ +----+-----+ +----+-----+ +----+-----+  |
|       |             |            |             |            |        |
|  +----+-----+ +----+-----+ +----+-----+ +----+-----+               |
|  | iMessage | |   LINE   | |  Twitch  | |Mattermost|   ... more    |
|  +----+-----+ +----+-----+ +----+-----+ +----+-----+               |
|       |             |            |             |                     |
|  +----+----------------------------------------------+-----+        |
|  |                   Web UI (src/web/)                      |       |
|  +----+-----------------------------------------------------+       |
|       |                                                              |
|  +----+-----------------------------------------------------+       |
|  |              Native Companion Apps (apps/)                |       |
|  |   +---------+    +---------+    +---------+               |      |
|  |   |  macOS  |    |   iOS   |    | Android |               |      |
|  |   | (Swift) |    | (Swift) |    |         |               |      |
|  |   +---------+    +---------+    +---------+               |      |
|  +----+-----------------------------------------------------+       |
|       |                                                              |
|  +----+-----------------------------------------------------+       |
|  |                CLI / TUI (src/cli/, src/tui/)             |       |
|  +----+-----------------------------------------------------+       |
|       |                                                              |
+=====================================================================+
                            |
                  +---------v---------+
                  |   Bridge :18790   |  (companion app connections)
                  +---------+---------+
                            |
+===========================v==========================================+
|                    OPENCLAW GATEWAY (:18789)                         |
|                    (src/gateway/)                                    |
+======================================================================+
|                                                                      |
|  +----------------------------------------------------------------+  |
|  |                    CHANNEL LAYER (src/channels/)                |  |
|  |                                                                |  |
|  |  +-------------+ +--------------+ +--------------------+      |  |
|  |  |  Allowlists  | |Session Mgmt  | | Command Gating     |     |  |
|  |  |  & Auth      | |& Threading   | | & Mention Gating   |     |  |
|  |  +-------------+ +--------------+ +--------------------+      |  |
|  |  +-------------+ +--------------+ +--------------------+      |  |
|  |  |  Dock       | | Typing       | | Model Overrides    |      |  |
|  |  |  (Router)   | | Indicators   | | (per channel)      |      |  |
|  |  +------+------+ +--------------+ +--------------------+      |  |
|  +---------+------------------------------------------------------+  |
|            |                                                         |
|  +---------v------------------------------------------------------+  |
|  |                   ROUTING LAYER (src/routing/)                 |  |
|  |                                                                |  |
|  |  resolve-route.ts -> picks provider + model based on:         |  |
|  |    - channel config / model overrides                          |  |
|  |    - account bindings / API key rotation                       |  |
|  |    - session continuity                                        |  |
|  +----------+-----------------------------------------------------+  |
|             |                                                        |
|  +----------v-----------------------------------------------------+  |
|  |                    AGENT LAYER (src/agents/)                   |  |
|  |                                                                |  |
|  |  Orchestrates: prompt assembly, tool calls, response           |  |
|  |  Flat architecture (no nested planner trees)                   |  |
|  +--+----------+----------+----------+----------------------------+  |
|     |          |          |          |                               |
|  +--v---+  +--v---+  +--v---+  +--v--------------------------+     |
|  |Skills|  |Memory|  |Plugins|  | Built-in Tools              |    |
|  |      |  |      |  |      |  |                              |    |
|  |skill/|  |src/  |  |src/  |  | +--------+ +-------------+  |    |
|  |  +   |  |memo- |  |plug- |  | |Browser | |Link & Media |  |    |
|  |Claw- |  |ry/   |  |ins/  |  | |Sandbox | |Understanding|  |    |
|  |Hub   |  |      |  |  +   |  | +--------+ +-------------+  |    |
|  |      |  |(1    |  |plugin|  | +--------+ +-------------+  |    |
|  |      |  |active|  |-sdk/)|  | |  TTS   | |  Cron Jobs  |  |    |
|  |      |  |at a  |  |      |  | |(Voice) | | (Scheduled) |  |    |
|  |      |  |time) |  |      |  | +--------+ +-------------+  |    |
|  +------+  +------+  +------+  +-----------------------------+     |
|                                                                      |
|  +----------------------------------------------------------------+  |
|  |                 SECURITY & SANDBOXING                          |  |
|  |  src/security/  src/secrets/                                   |  |
|  |  Dockerfile.sandbox  Dockerfile.sandbox-browser                |  |
|  +----------------------------------------------------------------+  |
|                                                                      |
|  +----------------------------------------------------------------+  |
|  |                    MCP BRIDGE (via mcporter - external)        |  |
|  |  Decoupled from core; add/remove MCP servers without restart   |  |
|  +----------------------------------------------------------------+  |
|                                                                      |
+============================+=========================================+
                             |
+============================v=========================================+
|                    LLM PROVIDERS (src/providers/)                     |
+======================================================================+
|                                                                      |
|  +-----------------+  +-----------------+  +-----------------+       |
|  |    Anthropic    |  |     OpenAI      |  |  Google Gemini  |       |
|  |  Claude 4/3.5/3 |  |  GPT-4o/4/3.5   |  |  Gemini 2.x    |      |
|  +-----------------+  +-----------------+  +-----------------+       |
|                                                                      |
|  +-----------------+  +-----------------+  +-----------------+       |
|  |   OpenRouter    |  | GitHub Copilot  |  |   Qwen Portal   |      |
|  |  (100+ models)  |  |  (OAuth token)  |  |  (China market) |      |
|  +-----------------+  +-----------------+  +-----------------+       |
|                                                                      |
|  +-----------------+  +-----------------+  +-----------------+       |
|  |    MiniMax      |  |      ZAI        |  |   AI Gateway    |      |
|  +-----------------+  +-----------------+  +-----------------+       |
|                                                                      |
|  +-----------------+  +--------------------------------------+       |
|  |   Kilocode      |  |  Any OpenAI-compatible endpoint      |      |
|  +-----------------+  |  (Ollama, LM Studio, vLLM, etc.)    |       |
|                        +--------------------------------------+       |
|                                                                      |
|  Features: multi-key rotation, per-channel model overrides,          |
|            session continuity, account bindings                      |
+======================================================================+
```

## 数据流（消息生命周期）

```
1. User sends message on WhatsApp (or any channel)
       |
       v
2. Channel adapter receives message
   - Authenticates sender (allowlist check)
   - Applies command/mention gating
   - Creates or resumes session
       |
       v
3. Dock (message router) in Channel Layer
   - Determines conversation context
   - Manages threading
       |
       v
4. Routing Layer (resolve-route.ts)
   - Selects LLM provider based on config
   - Picks specific model
   - Handles API key rotation
       |
       v
5. Agent Layer
   - Assembles prompt (system + memory + context + user message)
   - Calls LLM provider API
   - If LLM requests tool use -> executes tools (browser, search, etc.)
   - May loop: LLM -> tool -> LLM -> tool -> ... -> final response
       |
       v
6. Response flows back through Channel Layer
   - Formats for target platform (markdown, media, etc.)
   - Sends typing indicators
   - Delivers response to user
```

数据流说明：

1. 用户在 WhatsApp（或任何渠道）上发送消息
2. 渠道适配器接收消息
   - 验证发送者身份（白名单检查）
   - 应用命令/提及过滤规则
   - 创建或恢复会话
3. Dock（消息路由器）在渠道层中
   - 确定对话上下文
   - 管理消息线程
4. 路由层（resolve-route.ts）
   - 根据配置选择 LLM 提供商
   - 选择具体模型
   - 处理 API 密钥轮换
5. 智能体层
   - 组装提示词（系统提示 + 记忆 + 上下文 + 用户消息）
   - 调用 LLM 提供商 API
   - 如果 LLM 请求使用工具 -> 执行工具（浏览器、搜索等）
   - 可能循环：LLM -> 工具 -> LLM -> 工具 -> ... -> 最终响应
6. 响应通过渠道层返回
   - 格式化为目标平台格式（Markdown、媒体等）
   - 发送输入指示器
   - 将响应传递给用户

## 双进程 Docker 模型

```yaml
# Gateway (always-on server)
openclaw-gateway:
  ports: 18789 (API), 18790 (bridge for companion apps)
  command: node dist/index.js gateway --bind lan --port 18789
  volumes: ~/.openclaw (config + workspace)

# CLI (interactive terminal client)
openclaw-cli:
  entrypoint: node dist/index.js
  stdin_open: true, tty: true
```

说明：
- **网关（Gateway）**：常驻服务器，监听端口 18789（API）和 18790（配套应用桥接），挂载 `~/.openclaw` 作为配置和工作空间卷
- **CLI**：交互式终端客户端，启用标准输入和终端模式

## 源代码目录结构

```
openclaw/
  +-- src/
  |   +-- gateway/          # HTTP server, API endpoints
  |   +-- channels/         # Shared channel infrastructure
  |   +-- routing/          # Model/provider selection
  |   +-- agents/           # Agent orchestration
  |   +-- memory/           # Persistent memory (pluggable)
  |   +-- plugins/          # Plugin loading & lifecycle
  |   +-- plugin-sdk/       # SDK for building plugins
  |   +-- providers/        # LLM API adapters
  |   +-- security/         # Security infrastructure
  |   +-- secrets/          # Secret management
  |   +-- browser/          # Browser automation (computer-use)
  |   +-- cron/             # Scheduled/proactive tasks
  |   +-- tts/              # Text-to-speech
  |   +-- media/            # Media handling
  |   +-- media-understanding/  # Image/audio analysis
  |   +-- link-understanding/   # URL parsing & summarization
  |   +-- auto-reply/       # Automatic reply logic
  |   +-- hooks/            # Lifecycle hooks
  |   +-- sessions/         # Conversation sessions
  |   +-- daemon/           # Background process management
  |   +-- acp/              # Agent Communication Protocol
  |   +-- cli/              # CLI interface
  |   +-- tui/              # Terminal UI
  |   +-- web/              # Web frontend
  |   +-- whatsapp/         # WhatsApp channel
  |   +-- telegram/         # Telegram channel
  |   +-- discord/          # Discord channel
  |   +-- slack/            # Slack channel
  |   +-- signal/           # Signal channel
  |   +-- imessage/         # iMessage channel
  |   +-- line/             # LINE channel
  |   +-- config/           # Configuration management
  |   +-- infra/            # Infrastructure utilities
  |   +-- shared/           # Shared utilities
  |   +-- types/            # TypeScript type definitions
  |   +-- wizard/           # Setup wizard
  |   +-- pairing/          # Device pairing
  |   +-- logging/          # Logging infrastructure
  |   +-- process/          # Process management
  |   +-- commands/         # CLI commands
  |   +-- scripts/          # Build/utility scripts
  |   +-- markdown/         # Markdown processing
  |   +-- canvas-host/      # Canvas rendering
  |   +-- node-host/        # Node.js host utilities
  |   +-- terminal/         # Terminal utilities
  |   +-- compat/           # Compatibility shims
  |   +-- docs/             # Internal documentation
  |   +-- test-helpers/     # Test utilities
  |   +-- test-utils/       # More test utilities
  |
  +-- apps/
  |   +-- macos/            # macOS native app (Swift)
  |   +-- ios/              # iOS native app (Swift)
  |   +-- android/          # Android native app
  |   +-- shared/           # Shared app code
  |
  +-- packages/
  |   +-- clawdbot/         # Legacy package (backward compat)
  |   +-- moltbot/          # Legacy package (backward compat)
  |
  +-- skills/               # Bundled skills
  +-- extensions/           # Extension hosting
  +-- ui/                   # UI assets
  +-- docs/                 # Documentation
  +-- vendor/               # Vendored dependencies
  +-- scripts/              # Build/deploy scripts
  +-- patches/              # Dependency patches
  +-- Swabble/              # Unknown (possibly test fixtures)
```

目录说明：

| 目录 | 说明 |
|------|------|
| `src/gateway/` | HTTP 服务器、API 端点 |
| `src/channels/` | 共享渠道基础设施 |
| `src/routing/` | 模型/提供商选择 |
| `src/agents/` | 智能体编排 |
| `src/memory/` | 持久化记忆（可插拔） |
| `src/plugins/` | 插件加载与生命周期 |
| `src/plugin-sdk/` | 插件开发 SDK |
| `src/providers/` | LLM API 适配器 |
| `src/security/` | 安全基础设施 |
| `src/secrets/` | 密钥管理 |
| `src/browser/` | 浏览器自动化（计算机操控） |
| `src/cron/` | 定时/主动任务 |
| `src/tts/` | 文本转语音 |
| `src/media/` | 媒体处理 |
| `src/media-understanding/` | 图像/音频分析 |
| `src/link-understanding/` | URL 解析与摘要 |
| `src/auto-reply/` | 自动回复逻辑 |
| `src/hooks/` | 生命周期钩子 |
| `src/sessions/` | 对话会话管理 |
| `src/daemon/` | 后台进程管理 |
| `src/acp/` | 智能体通信协议 |
| `apps/` | 原生配套应用（macOS、iOS、Android） |
| `packages/` | 旧版兼容包 |
| `skills/` | 内置技能 |
| `extensions/` | 扩展托管 |
