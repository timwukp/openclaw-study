> 本文为中文翻译版本。英文原版为权威版本，如有差异以英文版为准。
> [English Version](../06-security-sandboxing.md)

# 第二阶段：安全与沙箱 (Sandboxing) 深入分析

## 执行摘要

OpenClaw 拥有**相当完善的安全基础设施**——远超大多数开源 AI 项目。仅 `src/security/` 目录就包含约 300KB 的代码和约 150KB 的测试。然而，分析揭示了若干架构性风险：安全模型依赖于配置的正确性而非结构性保障，且"安全"与"危险"之间的界限可能被配置标志、插件或对 AI 本身的社会工程攻击 (Social Engineering) 所突破。

**总体评估**：安全态势属于"**尽力而为的纵深防御 (Defense-in-Depth)**"，而非"**构造安全 (Secure by Construction)**"。这一点至关重要，因为该系统对用户通信和设备访问拥有 root 级别的能力。

---

## 1. 沙箱架构 (Sandbox Architecture)

### 1.1 基于容器的沙箱

OpenClaw 使用**三个 Docker 沙箱镜像**：

#### Dockerfile.sandbox（基础版）
```
Base: debian:bookworm-slim (通过 SHA 固定——良好实践)
User: 非 root 用户 "sandbox"
Tools: bash, curl, git, jq, python3, ripgrep
CMD: sleep infinity (等待命令)
```

#### Dockerfile.sandbox-browser
```
Base: debian:bookworm-slim (通过 SHA 固定)
User: 非 root 用户 "sandbox"
Tools: chromium, xvfb, x11vnc, novnc, websockify, socat
暴露端口: 9222 (CDP), 5900 (VNC), 6080 (noVNC web)
```

#### Dockerfile.sandbox-common（扩展版）
```
Base: openclaw-sandbox (基于基础版构建)
User: 可配置 (默认: sandbox)
Tools: nodejs, npm, pnpm, bun, python3, golang, rustc, cargo,
       build-essential, homebrew
```

### 1.2 沙箱风险分析

| 发现 | 严重程度 | 详情 |
|------|----------|------|
| **非 root 用户** | 良好 | 沙箱以 `sandbox` 用户运行，而非 root |
| **SHA 固定的基础镜像** | 良好 | 防止基础镜像的供应链攻击 (Supply Chain Attack) |
| **无网络隔离** | 高风险 | Dockerfile 中没有 `--network=none`。沙箱拥有完整的网络访问权限 |
| **无资源限制** | 中风险 | Dockerfile 中没有 CPU/内存/磁盘上限 |
| **CDP 端口 9222 暴露** | 高风险 | Chrome DevTools Protocol 允许完全控制浏览器。如果可从容器外部访问，任何进程都可以控制浏览器 |
| **VNC 端口 5900 暴露** | 中风险 | 可通过 VNC 访问沙箱桌面 |
| **sandbox-common 权限过大** | 高风险 | 包含 go、rust、cargo、brew、build-essential——可编译并运行任意本地代码 |
| **无 seccomp/AppArmor 配置文件** | 中风险 | 默认的 Docker seccomp 可能不够充分 |
| **Sleep infinity 模式** | 低风险 | 容器无限期运行，等待命令 |

### 1.3 关键缺陷：沙箱中的网络访问

沙箱容器拥有**不受限制的网络访问权限**。这意味着：
- 在沙箱中运行的代码可以向任何服务器**回传数据 (Phone Home)**
- 浏览器沙箱可以**访问任何网站**，包括本地网络服务
- 在沙箱中执行的恶意插件可以通过网络**外泄数据 (Exfiltrate Data)**
- 如果在云上运行，沙箱可以访问**云元数据端点** (169.254.169.254)

**建议**：沙箱应使用 `--network=none` 或仅允许访问网关的受限网络。

---

## 2. 安全模块分析 (`src/security/`)

### 2.1 文件清单（按大小排列，表示复杂度）

| 文件 | 大小 | 用途 |
|------|------|------|
| `audit.test.ts` | 89KB | 安全审计测试套件 |
| `audit-extra.sync.ts` | 46KB | 同步安全检查 |
| `audit-extra.async.ts` | 41KB | 异步安全检查 |
| `audit.ts` | 39KB | 核心安全审计引擎 |
| `audit-channel.ts` | 28KB | 频道特定的安全审计 |
| `windows-acl.test.ts` | 17KB | Windows ACL 安全测试 |
| `fix.ts` | 14KB | 安全问题自动修复 |
| `dm-policy-shared.test.ts` | 14KB | 私信访问策略测试 |
| `external-content.test.ts` | 13KB | 外部内容安全测试 |
| `skill-scanner.test.ts` | 12KB | 技能扫描测试 |
| `skill-scanner.ts` | 12KB | **扫描技能中的恶意模式** |
| `dm-policy-shared.ts` | 11KB | 私信/群组访问策略 |
| `external-content.ts` | 10KB | **外部内容净化** |
| `fix.test.ts` | 9KB | 自动修复测试 |
| `windows-acl.ts` | 8KB | Windows ACL 处理 |
| `audit-fs.ts` | 5KB | 文件系统权限审计 |
| `safe-regex.ts` | 3KB | 防 ReDoS 安全正则 |
| `mutable-allowlist-detectors.ts` | 2KB | 可变白名单检测 |
| `channel-metadata.ts` | 1.4KB | 频道安全元数据 |
| `dangerous-tools.ts` | 1.3KB | **危险工具定义** |
| `dangerous-config-flags.ts` | 1.2KB | **危险配置标志检测** |
| `scan-paths.ts` | 1.3KB | 路径遍历 (Path Traversal) 防护 |
| `secret-equal.ts` | 0.4KB | 时序安全比较 (Timing-Safe Comparison) |
| `audit-tool-policy.ts` | 0.07KB | 工具策略（很小——仅为重导出？） |

### 2.2 危险工具注册表

来自 `dangerous-tools.ts`，以下工具在**网关 HTTP 上默认被拒绝**：

```
sessions_spawn    — "远程生成代理即是 RCE"（他们自己的原话！）
sessions_send     — 跨会话消息注入
cron              — 创建/更新/删除定时运行
gateway           — 网关重配置
whatsapp_login    — 需要终端的交互式设置
```

以下 **ACP 工具始终需要显式批准**：

```
exec              — 执行任意命令
spawn             — 生成进程
shell             — Shell 访问
sessions_spawn    — 远程代理生成
sessions_send     — 跨会话注入
gateway           — 网关控制
fs_write          — 写文件
fs_delete         — 删除文件
fs_move           — 移动文件
apply_patch       — 应用代码补丁
```

#### 风险评估：

| 发现 | 严重程度 | 详情 |
|------|----------|------|
| 工具拒绝列表是**显式排除模式** | 高风险 | 仅命名的工具被阻止。新工具默认允许。新增的危险工具可能不会被阻止 |
| "exec" 作为工具存在 | 高风险 | AI 可以执行任意命令。即使有审批，这本质上就是远程代码执行 (RCE) |
| "fs_write/fs_delete/fs_move" 存在 | 高风险 | AI 可以修改进程用户可访问的任何文件 |
| 网关 HTTP 拒绝列表与 ACP 拒绝列表不同 | 中风险 | 不同的安全表面有不同的拒绝列表，造成混淆 |
| 拒绝列表是静态常量 | 中风险 | 无法在运行时向拒绝列表添加工具，除非修改代码 |

### 2.3 危险配置标志

来自 `dangerous-config-flags.ts`，以下配置选项被标记为危险：

```
gateway.controlUi.allowInsecureAuth = true
  → 允许通过非 HTTPS 连接进行认证

gateway.controlUi.dangerouslyAllowHostHeaderOriginFallback = true
  → 绕过来源检查（CSRF 风险）

gateway.controlUi.dangerouslyDisableDeviceAuth = true
  → 完全禁用设备认证

hooks.gmail.allowUnsafeExternalContent = true
  → 允许未经筛查的邮件内容进入 AI 提示

hooks.mappings[*].allowUnsafeExternalContent = true
  → 对任意 Hook 映射同样如此

tools.exec.applyPatch.workspaceOnly = false
  → 允许在工作区目录之外进行文件补丁
```

#### 风险评估：

| 发现 | 严重程度 | 详情 |
|------|----------|------|
| "dangerously" 前缀表明开发者有意识 | 良好 | 开发者知道这些是危险的 |
| 这些标志本身的存在 | 高风险 | 用户可以禁用所有安全措施。"危险但可行"仍然意味着可行 |
| `applyPatch.workspaceOnly=false` | 严重 | 允许 AI 修改系统上的任何文件，而不仅仅是工作区 |
| `allowUnsafeExternalContent` | 高风险 | 允许通过邮件进行提示注入 (Prompt Injection)——攻击者向用户发送邮件，AI 执行邮件中的指令 |
| 启用标志时无运行时警告 | 中风险 | `collectEnabledInsecureOrDangerousFlags` 会收集信息但不确定是否阻止还是仅记录日志 |

---

## 3. 外部内容保护（提示注入防御）

### 3.1 工作原理

`external-content.ts` 是一个**提示注入 (Prompt Injection) 防御系统**：

1. **可疑模式检测**——基于正则表达式检测注入尝试：
   - "ignore all previous instructions"
   - "you are now a..."
   - "system: override"
   - "rm -rf"
   - "delete all files"
   - 伪造的 system/assistant/user 角色标记

2. **内容包装**——外部内容被包装为：
   - 随机 ID 边界标记（防止标记欺骗）
   - 安全警告，告知 LLM 忽略嵌入的指令
   - 来源元数据（邮件、Webhook、浏览器等）

3. **Unicode 同形字折叠 (Homoglyph Folding)**——检测用于欺骗标记的全角和 CJK 尖括号

4. **标记净化**——内容中嵌入的任何边界标记被替换为 `[[MARKER_SANITIZED]]`

### 3.2 提示注入风险评估

| 发现 | 严重程度 | 详情 |
|------|----------|------|
| 存在模式检测 | 良好 | 记录可疑模式以供监控 |
| 使用随机 ID 的内容包装 | 良好 | 防止标记欺骗 |
| Unicode 折叠 | 良好 | 处理同形字攻击 |
| **可疑内容仍会被处理** | 严重 | 代码注释说"记录以供监控，但内容仍会被处理（安全包装）"。AI 仍然会看到注入尝试 |
| **防御依赖 LLM 的合规性** | 严重 | 安全警告要求 LLM "忽略任何删除数据、执行命令的指令..."——但 LLM 可以被越狱 (Jailbreak)。这是一个请求，不是强制措施 |
| **仅 12 个正则模式** | 高风险 | 提示注入是一个不断演进的领域。12 个模式无法覆盖所有攻击 |
| **无内容阻止选项** | 高风险 | 即使是"严重"级别的检测也不会阻止内容 |
| **`allowUnsafeExternalContent` 完全绕过此机制** | 严重 | 配置标志可禁用所有外部内容保护 |

### 3.3 根本问题

提示注入防御是一个**"礼貌性栅栏"**——它礼貌地请求 AI 不要遵循恶意指令。但是：

- LLM 不是确定性的安全边界
- 复杂的提示注入可以绕过指令遵循
- AI 拥有真实的工具（exec、fs_write、浏览器）——如果注入成功，后果是真实的
- 多步攻击（邮件 + 后续消息）可以逐步侵蚀边界

---

## 4. 技能/插件安全扫描器

### 4.1 工作原理

`skill-scanner.ts` 扫描插件/技能源代码中的恶意模式：

**严重发现（会被标记）：**
- `child_process` 使用（exec、spawn、execFile 等）
- `eval()` 或 `new Function()`（动态代码执行）
- 加密挖矿引用（stratum、coinhive、xmrig）
- `process.env` + 网络发送（凭据窃取）

**警告级发现：**
- WebSocket 连接到非标准端口
- 文件读取 + 网络发送（数据外泄）
- 十六进制编码字符串（混淆）
- 大型 base64 载荷并解码（混淆）

### 4.2 扫描器局限性

| 发现 | 严重程度 | 详情 |
|------|----------|------|
| 仅静态分析 | 高风险 | 无法检测运行时生成的恶意行为 |
| 仅扫描 JS/TS 文件 | 高风险 | 遗漏恶意本地二进制文件、Python 脚本、WASM |
| 基于模式（正则）| 中风险 | 容易绕过（字符串拼接、间接调用等）|
| **无沙箱执行** | 高风险 | 插件在与 OpenClaw 相同的进程中运行，未被沙箱化 |
| 最多 500 个文件，每文件 1MB | 低风险 | 合理的限制 |
| 跳过 node_modules | 高风险 | 依赖项中的恶意代码不会被检测 |
| **发现问题不阻止安装** | 高风险 | 扫描器发现问题但不确定是否会阻止插件加载 |

### 4.3 插件执行模型风险

扫描器是一个"代码审查"工具，而非"代码监狱"。关键问题是：**插件是在沙箱容器中执行还是在主进程中执行？**

从架构来看：
- `src/plugins/` 在主 Node.js 进程中处理插件加载
- `src/plugin-sdk/` 提供 SDK（暗示进程内执行）
- 沙箱容器是独立的 Docker 镜像，而非插件执行环境

**结论**：插件很可能在与网关**相同的进程**中运行，拥有对所有网关能力的完全访问权限。扫描器是唯一的防线。

---

## 5. 网关认证 (Gateway Authentication)

### 5.1 认证机制

来自网关目录和配置：

| 机制 | 文件 | 描述 |
|------|------|------|
| 令牌认证 (Token Auth) | `auth.ts` (15KB) | `OPENCLAW_GATEWAY_TOKEN` — Bearer 令牌 |
| 密码认证 | `.env.example` | `OPENCLAW_GATEWAY_PASSWORD` — 替代方式 |
| 设备认证 | `device-auth.ts` | 设备配对流程 |
| 速率限制 (Rate Limiting) | `auth-rate-limit.ts` (8KB) | 认证尝试的速率限制 |
| 浏览器加固 | `server.auth.browser-hardening.test.ts` | 浏览器特定认证 |
| 来源检查 | `origin-check.ts` | CSRF 保护 |
| CSP | `control-ui-csp.ts` | 内容安全策略 (Content Security Policy) |
| 时序安全比较 | `secret-equal.ts` | 防止时序攻击 (Timing Attack) |

### 5.2 认证风险评估

| 发现 | 严重程度 | 详情 |
|------|----------|------|
| 时序安全的密钥比较 | 良好 | 使用 `crypto.timingSafeEqual`——实现正确 |
| 存在速率限制 | 良好 | 防止暴力破解 |
| 存在来源检查 | 良好 | CSRF 保护 |
| **认证仅支持令牌/密码** | 中风险 | 无多因素认证 (MFA)，无基于证书的认证 |
| **令牌在环境变量中** | 中风险 | 可能通过进程列表、日志、错误转储泄露 |
| **`dangerouslyDisableDeviceAuth`** | 高风险 | 完全禁用认证 |
| **`allowInsecureAuth`** | 高风险 | 允许通过明文 HTTP 认证——令牌在网络上可见 |
| **默认网关绑定到局域网** | 高风险 | docker-compose 中使用 `--bind lan`。同一网络上的任何设备都可以访问 |
| **63KB 认证测试** | 良好 | 广泛的测试（server.auth.test.ts = 63KB）|
| **密码或令牌** | 中风险 | 密码认证弱于令牌认证，但两者都允许 |

---

## 6. 文件系统安全

### 6.1 路径遍历防护

`scan-paths.ts` 提供：
- `isPathInside()` — 检查路径是否在基础目录内
- `isPathInsideWithRealpath()` — 在检查前解析符号链接
- 符号链接感知（跟踪符号链接以检查真实目标）

### 6.2 文件权限审计

`audit-fs.ts` 检查：
- 全局可写文件/目录
- 组可写文件/目录
- 符号链接目标
- Windows ACL（在 Windows 上）

### 6.3 文件系统风险评估

| 发现 | 严重程度 | 详情 |
|------|----------|------|
| 存在路径遍历检查 | 良好 | `isPathInside` 防止 `../../etc/passwd` |
| 符号链接感知检查 | 良好 | 在检查边界前跟踪符号链接 |
| **`workspaceOnly=false` 绕过所有文件系统边界** | 严重 | 配置标志允许在任何位置进行文件操作 |
| **Docker 卷挂载决定了真实边界** | 中等 | 如果用户将 `/` 挂载到容器中，所有保护都失效 |
| 文件权限审计仅为信息性的 | 中风险 | 检测问题但可能不强制执行 |

---

## 7. 私信/群组访问控制 (DM/Group Access Control)

### 7.1 访问策略系统

`dm-policy-shared.ts` 实现了一套复杂的访问控制系统：

**私信策略：**
- `disabled` — 阻止所有私信
- `open` — 允许所有私信（无认证）
- `pairing` — 需要设备配对（默认）
- `allowlist` — 仅允许列表中的发送者

**群组策略：**
- `disabled` — 阻止所有群组消息
- `allowlist` — 仅允许列表中的群组/发送者

### 7.2 访问控制风险评估

| 发现 | 严重程度 | 详情 |
|------|----------|------|
| 默认值为"pairing"（而非"open"）| 良好 | 安全的默认值 |
| 存在白名单机制 | 良好 | 可限制为已知联系人 |
| **`dmPolicy=open` 允许任何人** | 高风险 | 任何人都可以通过私信命令 AI |
| **群组回退逻辑复杂** | 中风险 | 复杂的逻辑 = 潜在的绕过漏洞 |
| **配对存储可扩展** | 中风险 | 一旦配对，用户永久保持授权 |
| **存在命令门控** | 良好 | 控制命令需要额外认证 |
| **多用户私信检测** | 良好 | 检测多个用户拥有访问权限的情况 |

---

## 8. 浏览器/计算机使用安全

### 8.1 浏览器子系统 (src/browser/)

浏览器模块规模庞大（100+ 文件），包括：

- **Chrome DevTools Protocol (CDP)** 集成 (`cdp.ts` — 15KB)
- **Playwright 集成** (`pw-session.ts` — 24KB, `pw-tools-core.interactions.ts` — 21KB)
- **Chrome 配置文件管理** (`chrome.ts`, `profiles-service.ts`)
- **导航守卫** (`navigation-guard.ts`)
- **扩展中继** (`extension-relay.ts` — 30KB)
- **截图捕获** (`screenshot.ts`)
- **表单字段交互** (`form-fields.ts`)
- **文件下载** (`pw-tools-core.downloads.ts`)

### 8.2 浏览器风险评估

| 发现 | 严重程度 | 详情 |
|------|----------|------|
| 存在导航守卫 | 良好 | 一定程度的 URL 过滤 |
| 在独立容器中运行 | 良好 | 浏览器沙箱是隔离的 |
| **CDP 端口 9222 暴露** | 严重 | 完全远程控制浏览器。如果可达，任何人都可以驱动浏览器 |
| **Playwright 拥有完全的页面控制权** | 严重 | 可以点击、输入、填写表单、下载文件、在任何页面上执行 JS |
| **使用 Chrome 配置文件** | 高风险 | 如果使用用户真实的 Chrome 配置文件，AI 可以访问所有保存的密码、Cookie、会话 |
| **扩展中继 (30KB)** | 高风险 | Chrome 扩展可在 AI 和真实浏览器之间架桥——可能访问已认证的会话 |
| **可执行任意 JS** | 严重 | `pw-tools-core.interactions.ts` — 在页面上执行 JS 意味着完全的 DOM/Cookie/localStorage 访问 |
| **文件下载能力** | 高风险 | AI 可以从互联网下载任意文件 |
| **未见 URL 黑名单** | 高风险 | 导航守卫存在但不确定阻止了什么 |
| **表单填写** | 高风险 | AI 可以填写和提交表单（购买、注册、密码更改）|

---

## 9. 网关工具调用

### 9.1 执行/Shell 能力

来自网关目录：
- `node-invoke-system-run-approval.ts` (9KB) — 系统命令审批系统
- `exec-approval-manager.ts` (6KB) — 管理执行审批
- `node-command-policy.ts` (5KB) — 命令策略

### 9.2 工具调用风险

| 发现 | 严重程度 | 详情 |
|------|----------|------|
| 存在 exec 审批系统 | 良好 | 命令需要审批 |
| **审批可预先授予** | 高风险 | 用户可以预先批准命令，创建常驻权限 |
| **`sessions_spawn` = 远程代码执行** | 严重 | 在注释中已承认："远程生成代理即是 RCE" |
| 存在通过 HTTP 的工具调用 | 高风险 | `tools-invoke-http.ts` (11KB) — 工具可通过 HTTP API 调用 |
| Cron 可调度工具执行 | 高风险 | AI 可以设置定期自动化任务 |

---

## 10. 密钥管理 (Secrets Management)

### 10.1 密钥工作原理

`src/secrets/` 是一个 140KB+ 的模块，包含：
- `apply.ts` (19KB) — 将密钥应用到运行时
- `audit.ts` (23KB) — 密钥安全审计
- `configure.ts` (27KB) — 密钥配置
- `resolve.ts` (23KB) — 密钥引用解析
- `runtime.ts` (13KB) — 运行时密钥访问
- `plan.ts` (7KB) — 密钥部署规划

### 10.2 密钥风险评估

| 发现 | 严重程度 | 详情 |
|------|----------|------|
| 专用密钥模块 (140KB+) | 良好 | 对密钥管理的认真投入 |
| 存在密钥审计 | 良好 | 可检查密钥泄露 |
| `.detect-secrets.cfg` + `.secrets.baseline` | 良好 | 预提交密钥检测 |
| **插件是否可访问密钥？** | 高风险 | 如果插件在进程内运行，可能访问 process.env |
| **密钥解析复杂度** | 中风险 | 23KB 的解析器 = 复杂逻辑 = 潜在漏洞 |
| **多个密钥来源** | 中风险 | process env → .env → ~/.openclaw/.env → 配置——可能存在优先级漏洞 |

---

## 11. 总结：安全威胁模型

### OpenClaw 做得好的方面
1. 大量安全代码（约 450KB 的安全 + 密钥模块）
2. 全面的测试覆盖（约 200KB+ 的安全测试）
3. 安全的默认值（配对模式、工具拒绝列表、时序安全比较）
4. 带注入检测的外部内容包装
5. 技能/插件恶意模式扫描
6. 文件权限审计
7. Windows ACL 支持
8. 路径遍历防护

### 关键缺陷

```
+------------------------------------------------------------------+
|                    严重风险摘要                                      |
+------------------------------------------------------------------+
|                                                                   |
|  1. 提示注入防御基于 LLM 合规性                                      |
|     AI 被要求忽略恶意指令。                                          |
|     这是不可强制执行的。LLM 可以被越狱。                               |
|     AI 拥有真实工具（exec、fs_write、浏览器）。                        |
|     如果注入成功，后果是真实的。                                       |
|                                                                   |
|  2. 插件在进程内运行（未沙箱化）                                      |
|     扫描器检测模式但不阻止。                                          |
|     插件可以访问 process.env、文件系统、网络。                         |
|     node_modules 不被扫描。供应链攻击可能发生。                        |
|                                                                   |
|  3. 沙箱拥有完整的网络访问                                           |
|     Dockerfile 中无网络隔离。                                        |
|     可以外泄数据、回传、访问本地网络。                                  |
|                                                                   |
|  4. 浏览器拥有不受限制的能力                                          |
|     可以在任何页面上执行 JS，访问 Cookie/localStorage。               |
|     CDP 端口暴露。Chrome 配置文件可能有保存的密码。                     |
|     可以填写和提交表单（金融交易）。                                   |
|                                                                   |
|  5. 危险配置标志可禁用所有安全措施                                     |
|     `dangerouslyDisableDeviceAuth` = 完全无认证                     |
|     `applyPatch.workspaceOnly=false` = 写入任何文件                  |
|     `allowUnsafeExternalContent` = 允许提示注入                      |
|                                                                   |
|  6. 工具拒绝列表是排除模式，而非包含模式                                |
|     仅命名的工具被阻止。新工具默认允许。                                |
|     应改为：仅命名的工具被允许，其余全部阻止。                          |
|                                                                   |
|  7. 高风险操作无人工审核 (Human-in-the-Loop)                         |
|     在自动/定时模式下，AI 无需确认即可行动。                            |
|     邮件处理可自动触发工具执行。                                       |
|     会话生成可实现自主代理链。                                         |
|                                                                   |
+------------------------------------------------------------------+
```

### 攻击场景

#### 场景 1：基于邮件的提示注入
```
1. 攻击者向目标发送精心构造的邮件
2. OpenClaw 的 Gmail Hook 处理该邮件
3. 尽管有内容包装，复杂的注入绕过了 LLM 防护
4. AI 执行：读取敏感文件，通过 Webhook 发送
5. 攻击时间：几分钟（自动化）
```

#### 场景 2：恶意插件
```
1. 攻击者在 ClawHub 上发布一个吸引人的插件
2. 插件通过扫描器（使用间接代码模式）
3. 加载后，插件访问 process.env（API 密钥）
4. 将密钥发送到攻击者的服务器
5. 攻击时间：安装后立即生效
```

#### 场景 3：网络相邻攻击
```
1. 攻击者加入与目标相同的 Wi-Fi（咖啡店）
2. 网关绑定到局域网（默认设置）
3. 如果认证薄弱/禁用，攻击者发送工具调用
4. 使用 sessions_spawn（如果未被拒绝）或其他工具
5. 获取对所有连接的消息频道的控制权
```

#### 场景 4：浏览器凭据窃取
```
1. AI 被分配需要使用浏览器的任务
2. 浏览器使用保存了密码的 Chrome 配置文件
3. AI 导航到目标网站，自动填充凭据
4. AI（或被攻陷的插件）提取填充的凭据
5. 通过可网络访问的沙箱外泄凭据
```

---

## 12. 建议

### 立即执行（生产使用前）
1. **沙箱网络隔离**：添加 `--network=none` 或受限网络
2. **插件沙箱化**：在独立容器中运行插件，而非进程内
3. **工具白名单模式**：从"拒绝危险的"改为"仅允许安全的"
4. **移除 `dangerously*` 配置标志**：或要求加密确认
5. **阻止检测到注入的外部内容**：不仅记录日志，还要阻止

### 中期
6. **浏览器配置文件隔离**：永远不要使用保存了密码的真实 Chrome 配置文件
7. **所有 exec/fs_write/spawn 操作的人工审核**：即使在自动模式下
8. **强制审计日志**：所有工具调用的不可变记录
9. **插件能力声明**：插件必须声明所需的访问权限
10. **工具调用速率限制**：限制每分钟工具调用次数以防止快速数据外泄

### 长期
11. **正式威胁模型**：对所有接口的 STRIDE 分析
12. **第三方安全审计**：专业渗透测试
13. **基于能力的安全模型 (Capability-Based Security)**：用能力替代环境权限
14. **内容寻址的插件分发**：防止供应链攻击

---

## 附录：已分析的文件

| 文件 | 路径 | 大小 | 是否完整读取 |
|------|------|------|------------|
| Dockerfile.sandbox | / | 445B | 是 |
| Dockerfile.sandbox-browser | / | 730B | 是 |
| Dockerfile.sandbox-common | / | 1.8KB | 是 |
| dangerous-tools.ts | src/security/ | 1.3KB | 是 |
| dangerous-config-flags.ts | src/security/ | 1.2KB | 是 |
| external-content.ts | src/security/ | 9.8KB | 是 |
| skill-scanner.ts | src/security/ | 11.6KB | 是 |
| audit-fs.ts | src/security/ | 5.1KB | 是 |
| dm-policy-shared.ts | src/security/ | 10.7KB | 是 |
| scan-paths.ts | src/security/ | 1.3KB | 是 |
| secret-equal.ts | src/security/ | 0.4KB | 是 |
| .env.example | / | 2.9KB | 是 |
| src/security/ | 目录 | （列表）| 是 |
| src/secrets/ | 目录 | （列表）| 是 |
| src/browser/ | 目录 | （列表）| 是 |
| src/gateway/ | 目录 | （列表）| 是 |

**注意**：`audit.ts` (39KB)、`audit-extra.sync.ts` (46KB)、`audit-extra.async.ts` (41KB) 和 `audit-channel.ts` (28KB) 文件已识别但由于大小原因**未完整读取**。这些文件很可能包含额外的安全检查。完整审计需要阅读这些文件。
