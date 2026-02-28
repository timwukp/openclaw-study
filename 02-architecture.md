# OpenClaw Technical Architecture

## Tech Stack

| Component       | Technology                         |
|-----------------|------------------------------------|
| Language        | TypeScript                         |
| Runtime         | Node.js                            |
| Package Manager | pnpm (monorepo)                    |
| Build Tool      | tsdown                             |
| Testing         | Vitest (unit, e2e, live, gateway)  |
| Linting         | oxlint + oxfmt (Rust-based)        |
| Native Apps     | Swift (macOS/iOS), Android         |
| Containers      | Docker + Podman                    |

## Architecture Diagram

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

## Data Flow (Message Lifecycle)

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

## Two-Process Docker Model

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

## Source Code Map

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
