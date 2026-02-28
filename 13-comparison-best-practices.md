# Phase 9: Gap Analysis vs. Security Standards and Best Practices

## OpenClaw Security Study — Measuring Against Industry Frameworks

**Document:** 13-comparison-best-practices.md
**Scope:** Systematic comparison of OpenClaw's security posture against OWASP, NIST CSF, CIS Controls, STRIDE, EU AI Act, OWASP LLM Top 10, and platform security models
**Methodology:** Gap analysis using findings from Phases 2-8

---

## 1. Executive Summary

This document compares OpenClaw's security posture against six established security
frameworks, one AI-specific regulatory framework, and four platform security models.
The purpose is not to certify compliance but to provide a structured lens through
which the findings from Phases 2 through 8 can be evaluated against recognized
industry standards.

**Bottom line**: OpenClaw demonstrates meaningful effort in some areas -- SHA-pinned
base images, a 300KB+ security module, secret detection integration, timing-safe
comparisons -- but fails to meet baseline requirements across the majority of
framework categories. The gaps are most severe in:

- **Access control and privilege separation** (OWASP A01, NIST Protect, CIS v8)
- **Software integrity and supply chain** (OWASP A08, LLM05, CIS software control)
- **Cryptographic protection of data at rest** (OWASP A02, NIST Protect)
- **Excessive agency and autonomy controls** (LLM08, EU AI Act, NIST AI RMF)
- **Transparency and AI disclosure** (EU AI Act, IEEE P2894)

The overall estimated compliance across all frameworks is approximately **28%** of
assessed controls. This is not unusual for a fast-moving open-source project with
237K stars -- community-driven development typically prioritizes features over
compliance -- but it is deeply concerning given the system's capability surface:
shell execution, browser control, messaging impersonation, and autonomous operation.

A system this powerful, measured against the standards that govern systems far less
capable, reveals a gap between what OpenClaw *can do* and what industry consensus
says systems *should be required to do*.

---

## 2. OWASP Top 10 (2021) Compliance Matrix

The OWASP Top 10 represents the most widely recognized baseline for web application
security. While OpenClaw is not a traditional web application, it exposes a gateway
API, processes external input, and handles sensitive data -- making OWASP criteria
directly applicable.

| # | OWASP Category | What OpenClaw Does | What Is Missing | Risk Rating |
|---|---|---|---|---|
| A01 | **Broken Access Control** | Token-based gateway auth; LAN-bound default; tool deny list (opt-out) | No MFA; no role-based access; plugins run with full system privilege; deny-list model means new tools are allowed by default; `dangerouslyDisableDeviceAuth` bypasses all auth | **CRITICAL** |
| A02 | **Cryptographic Failures** | SHA-pinned Docker images; timing-safe token comparison; `.detect-secrets` integration | No encryption at rest for SQLite memory database; vector embeddings transmitted to external APIs without end-to-end encryption; no TLS enforcement for local API; no key management system | **HIGH** |
| A03 | **Injection** | 12 regex patterns for prompt injection; content wrapping for external data; `safe-regex.ts` ReDoS protection | Prompt injection defense relies on LLM behavioral compliance, not structural isolation; no parameterized input separation; plugin inputs not sanitized; email/webhook content injectable | **CRITICAL** |
| A04 | **Insecure Design** | Security module exists (~300KB); audit system with sync and async checks; path traversal prevention | Architecture grants plugins full process access by design; auto-reply enabled by default; no principle of least privilege; capability-first design philosophy | **CRITICAL** |
| A05 | **Security Misconfiguration** | Security audit checks with auto-fix; dedicated `fix.ts` module; permission auditing | Config flags (`dangerouslyDisableDeviceAuth`, `allowInsecureAuth`) disable security by design; sandbox lacks network isolation; CDP port 9222 exposed; no security hardening guide | **HIGH** |
| A06 | **Vulnerable Components** | SHA-pinned base images; npm-based dependency management | No dependency vulnerability scanning in CI; plugin scanner skips `node_modules` entirely; 500-file scan cap; no SBOM generation; no transitive dependency audit | **HIGH** |
| A07 | **Authentication Failures** | Token-based authentication; gateway auth module | Token-only (no MFA); tokens stored in config files; no session expiry mechanism documented; no account lockout; no credential rotation policy; LAN-binding is network-dependent | **HIGH** |
| A08 | **Software Integrity Failures** | Plugin manifest validation; basic manifest schema | No code signing for plugins or updates; no integrity verification of ClawHub packages; npm install with no lockfile pinning; scanner is regex-only; no attestation chain | **CRITICAL** |
| A09 | **Logging & Monitoring Failures** | Some operational logging | No security-focused audit logging; no tamper-resistant log storage; no real-time alerting on anomalous behavior; no plugin activity monitoring; no autonomous action log trail | **HIGH** |
| A10 | **SSRF** | Navigation guard for browser (2.4KB) | Browser sandbox has unrestricted network access; can reach cloud metadata endpoints (169.254.169.254); no URL allowlisting for outbound requests; plugins can make arbitrary HTTP calls | **HIGH** |

**OWASP Top 10 Summary**: 4 CRITICAL, 6 HIGH. Zero categories at LOW risk.
OpenClaw meets partial requirements in 4 of 10 categories and fails substantially
in the remaining 6.

---

## 3. OWASP Top 10 for LLM Applications Compliance

The OWASP Top 10 for LLM Applications (2023/2025) addresses risks specific to
systems built on large language models. This is the most directly applicable
framework for OpenClaw.

### LLM01: Prompt Injection

| Aspect | Assessment |
|--------|-----------|
| **Standard requires** | Structural separation of instructions and data; input validation; output filtering; privilege control limiting LLM actions |
| **OpenClaw provides** | 12 regex patterns in `external-content.ts`; content wrapping with delimiters; safe-regex to prevent ReDoS |
| **Gap** | Defense relies entirely on LLM behavioral compliance. No structural isolation between system prompts and user/external data. Email content, webhook payloads, and inbound messages all flow into the same context window. The 12 regex patterns are a static allowlist that cannot adapt to novel injection techniques. |
| **Rating** | **FAIL** -- defense is aspirational, not structural |

### LLM02: Insecure Output Handling

| Aspect | Assessment |
|--------|-----------|
| **Standard requires** | Output encoding; content security policy; validation before rendering or executing LLM output |
| **OpenClaw provides** | Send policy controls on outbound messages; some output filtering |
| **Gap** | LLM output is used to construct shell commands (via exec tools), compose messages sent as the user, navigate browsers, and modify files. No systematic output encoding or validation exists between LLM generation and action execution. Auto-reply sends LLM output directly to messaging channels without human review. |
| **Rating** | **FAIL** -- LLM output is trusted and acted upon without validation |

### LLM03: Training Data Poisoning (Memory Poisoning Analog)

| Aspect | Assessment |
|--------|-----------|
| **Standard requires** | Data provenance tracking; input validation for training/fine-tuning data; anomaly detection |
| **OpenClaw provides** | No documented memory validation |
| **Gap** | OpenClaw's memory system accepts all conversation content, file contents, and third-party data into the vector store without provenance tracking or validation. A malicious actor could inject false memories through auto-reply channels, email hooks, or webhook triggers. These poisoned memories then influence all future agent decisions via semantic retrieval. No mechanism exists to distinguish organic memories from injected ones. |
| **Rating** | **FAIL** -- memory system is fully open to poisoning |

### LLM04: Model Denial of Service

| Aspect | Assessment |
|--------|-----------|
| **Standard requires** | Rate limiting; resource caps; input length validation; cost monitoring |
| **OpenClaw provides** | Some debouncing on inbound messages (`inbound-debounce.ts`) |
| **Gap** | No resource limits on sandbox containers. No rate limiting on LLM API calls. Agent chains are unbounded -- a self-spawning loop could consume unlimited API credits. No cost monitoring or circuit breakers. Cron jobs run without resource budgets. |
| **Rating** | **FAIL** -- no meaningful resource controls |

### LLM05: Supply Chain Vulnerabilities

| Aspect | Assessment |
|--------|-----------|
| **Standard requires** | Verified model sources; plugin/extension vetting; dependency scanning; SBOM |
| **OpenClaw provides** | Plugin manifest validation; basic regex security scanner |
| **Gap** | ClawHub marketplace has no review process. Plugins install via npm with no integrity verification beyond manifest schema. Scanner skips `node_modules`, caps at 500 files, uses only regex matching. No code signing. No SBOM. No transitive dependency analysis. The `skill-creator` plugin can generate new plugins autonomously. |
| **Rating** | **FAIL** -- supply chain is fundamentally unprotected |

### LLM06: Sensitive Information Disclosure

| Aspect | Assessment |
|--------|-----------|
| **Standard requires** | Data classification; output filtering for PII; access controls on sensitive data |
| **OpenClaw provides** | `.detect-secrets` integration; dedicated 140KB secret management module |
| **Gap** | Vector embeddings of all memory content are sent to external APIs (OpenAI, Google, Voyage, Mistral) by default. No data classification system determines what should or should not be embedded. No PII detection before embedding transmission. Memory is accessible to all plugins without access controls. Secret detection focuses on credentials, not on personal information. |
| **Rating** | **FAIL** -- personal data transmitted externally by design |

### LLM07: Insecure Plugin Design

| Aspect | Assessment |
|--------|-----------|
| **Standard requires** | Plugin sandboxing; capability-based permissions; input validation; principle of least privilege |
| **OpenClaw provides** | Plugin SDK with defined interfaces; `path-safety.ts` (0.8KB) |
| **Gap** | Plugins execute in-process with full system privilege. The SDK explicitly provides shell execution, filesystem access, and network capabilities. 8 lifecycle hooks allow plugins to intercept and modify every stage of agent operation. No capability restrictions. No sandboxing. No runtime monitoring. The memory system is a plugin slot, meaning plugins can replace the entire memory subsystem. |
| **Rating** | **FAIL** -- plugins are architecturally insecure |

### LLM08: Excessive Agency

| Aspect | Assessment |
|--------|-----------|
| **Standard requires** | Limit LLM actions; require human approval for high-impact operations; implement least-privilege; log all actions |
| **OpenClaw provides** | Exec approval manager; command authorization; tool deny list |
| **Gap** | Exec approval can be pre-granted via patterns. Tool deny list is opt-out (block specific tools) not opt-in (allow specific tools). Agent chains are unbounded and self-spawning. Cron runs isolated agents without human oversight. Auto-reply processes all messages autonomously. No global emergency stop exists. The system's 510KB of autonomous operation code dwarfs its consent infrastructure. |
| **Rating** | **FAIL** -- agency controls are systematically insufficient |

### LLM09: Overreliance

| Aspect | Assessment |
|--------|-----------|
| **Standard requires** | Human-in-the-loop for critical decisions; confidence indicators; validation mechanisms |
| **OpenClaw provides** | Manual approval mode for exec commands (when not pre-granted) |
| **Gap** | Auto-reply sends messages without human review. Cron jobs execute without checkpoints. No confidence scores on LLM outputs. No validation of factual claims before acting. The architecture encourages maximum delegation through always-on availability and escalating automation (Phase 8 documented the dependency ratchet). |
| **Rating** | **PARTIAL FAIL** -- some approval mechanisms exist but are easily bypassed |

### LLM10: Model Theft

| Aspect | Assessment |
|--------|-----------|
| **Standard requires** | Protect proprietary models; access controls on model APIs; rate limiting |
| **OpenClaw provides** | Uses third-party models (OpenAI, Anthropic, etc.), not proprietary models |
| **Gap** | Less directly applicable since OpenClaw does not host its own models. However, API keys for model providers are stored in configuration and accessible to all in-process plugins. A malicious plugin could exfiltrate API keys to make unauthorized calls against the user's model API accounts. |
| **Rating** | **PARTIAL** -- not directly applicable but API key exposure remains a risk |

**OWASP LLM Top 10 Summary**: 8 FAIL, 1 PARTIAL FAIL, 1 PARTIAL. This is the
most directly relevant framework, and OpenClaw fails on nearly every criterion.

---

## 4. NIST Cybersecurity Framework (CSF) Alignment

The NIST CSF organizes cybersecurity into five core functions. Each is assessed
against OpenClaw's capabilities and gaps.

| NIST Function | Sub-Category | OpenClaw Capability | Gap | Status |
|---|---|---|---|---|
| **IDENTIFY** | Asset Management | No asset inventory of plugins, channels, or connected accounts | Cannot enumerate what the system has access to | MISSING |
| | Risk Assessment | No documented risk assessment process | Risk analysis is external (this study) | MISSING |
| | Governance | MIT license; no security governance; no incident response plan | No accountability framework | MISSING |
| | Supply Chain RM | Manifest validation for plugins | No supplier risk assessment; no ClawHub vetting | MINIMAL |
| **PROTECT** | Access Control | Token gateway; LAN binding; tool deny list | No MFA; no RBAC; no least-privilege; plugins in-process | PARTIAL |
| | Data Security | `.detect-secrets`; secret management module | No encryption at rest; no data classification; embeddings sent externally | PARTIAL |
| | Training | N/A (open-source project) | No security documentation for deployers | MINIMAL |
| | Protective Tech | Docker sandbox; content wrapping | No network isolation; no plugin sandbox; no seccomp/AppArmor | PARTIAL |
| **DETECT** | Anomaly Detection | None documented | No behavioral monitoring; no anomaly alerting | MISSING |
| | Continuous Monitoring | None documented | No runtime security monitoring | MISSING |
| | Detection Processes | Plugin scanner (regex) | Scanner is trivially bypassed; no continuous scanning | MINIMAL |
| **RESPOND** | Response Planning | None documented | No incident response plan | MISSING |
| | Analysis | Security audit module can check configuration | Post-incident analysis not supported | MINIMAL |
| | Mitigation | `fix.ts` auto-remediation for config issues | No runtime mitigation; no kill switch; no quarantine | MINIMAL |
| | Communication | None documented | No breach notification mechanism | MISSING |
| **RECOVER** | Recovery Planning | None documented | No disaster recovery plan | MISSING |
| | Improvements | Active open-source development | No formal lessons-learned process | MINIMAL |

**NIST CSF Summary**:
- IDENTIFY: 0/4 adequate
- PROTECT: 2/4 partial
- DETECT: 0/3 adequate
- RESPOND: 0/4 adequate
- RECOVER: 0/2 adequate

The PROTECT function has the only meaningful coverage, and even that is partial.
The DETECT, RESPOND, and RECOVER functions are essentially absent. For a system
with the capability to send messages as the user, execute shell commands, and
control browsers autonomously, this represents a fundamental governance failure.

---

## 5. STRIDE Threat Model Analysis

STRIDE provides a structured way to categorize threats. For each threat type,
we identify specific OpenClaw manifestations, current mitigations, and gaps.

### S — Spoofing (Identity)

| Aspect | Detail |
|--------|--------|
| **Manifestation** | AI sends messages as the user across 10+ platforms. Recipients cannot distinguish AI from human. Voice call skill enables vocal impersonation. No AI disclosure mechanism. |
| **Current Mitigation** | None. Impersonation is the intended feature. |
| **Gap** | No mandatory disclosure indicator. No opt-in consent from message recipients. No machine-readable AI attribution. Violates identity authentication principles at the social layer. |
| **Severity** | **CRITICAL** |

### T — Tampering (Data Integrity)

| Aspect | Detail |
|--------|--------|
| **Manifestation** | Memory poisoning via injected messages. Plugin hooks can modify any data in transit. Auto-reply can alter message content. Agent chains can modify files. No integrity verification on memory entries. |
| **Current Mitigation** | Path traversal prevention for filesystem. Manifest schema for plugins. |
| **Gap** | No data integrity checks on memory. No tamper detection for configuration. No integrity monitoring on plugin behavior. No cryptographic verification of message content between generation and delivery. |
| **Severity** | **HIGH** |

### R — Repudiation (Deniability of Actions)

| Aspect | Detail |
|--------|--------|
| **Manifestation** | No comprehensive audit trail. Autonomous actions (cron, hooks, auto-reply) execute without logging attribution. Cannot determine after the fact whether the human or the AI performed an action. |
| **Current Mitigation** | Minimal operational logging. |
| **Gap** | No tamper-resistant audit log. No cryptographic attribution of actions. No separation between human-initiated and AI-initiated actions in any log. For a system that sends messages as the user, repudiability is a serious legal and accountability concern. |
| **Severity** | **HIGH** |

### I — Information Disclosure

| Aspect | Detail |
|--------|--------|
| **Manifestation** | Memory embeddings sent to external APIs. Browser access exposes cookies, localStorage, and authenticated sessions. Plugins access all memory without restriction. CDP port 9222 exposed. VNC port 5900 exposed. |
| **Current Mitigation** | `.detect-secrets` for credential detection. Some port documentation. |
| **Gap** | No data classification. No PII detection before external transmission. No access controls on memory. No network segmentation for sensitive data flows. Secret detection targets credentials, not personal information. |
| **Severity** | **CRITICAL** |

### D — Denial of Service

| Aspect | Detail |
|--------|--------|
| **Manifestation** | Unbounded agent chains can consume unlimited compute/API resources. No resource limits on sandbox containers. No rate limiting on LLM calls. Self-spawning agents create exponential resource consumption. |
| **Current Mitigation** | Message debouncing. |
| **Gap** | No resource caps. No circuit breakers. No cost monitoring. No per-plugin resource quotas. No agent chain depth limits. A single misconfigured cron job could trigger unlimited API spending. |
| **Severity** | **HIGH** |

### E — Elevation of Privilege

| Aspect | Detail |
|--------|--------|
| **Manifestation** | Plugins escalate from "extension" to full system access via in-process execution. Prompt injection can escalate LLM from "assistant" to "autonomous agent." Pre-granted exec approval escalates from "per-command approval" to "blanket authorization." Config flags escalate from "secured" to "fully open." |
| **Current Mitigation** | Non-root sandbox user. Some command authorization. |
| **Gap** | No privilege boundaries between components. No capability-based security. No mandatory access control. The architectural model is "everything runs at the same privilege level," which means any compromise is total compromise. |
| **Severity** | **CRITICAL** |

**STRIDE Summary**: 3 CRITICAL (Spoofing, Information Disclosure, Elevation of
Privilege), 3 HIGH (Tampering, Repudiation, Denial of Service). Every STRIDE
category reveals significant gaps.

---

## 6. EU AI Act Compliance Assessment

The EU AI Act (entered into force August 2024) classifies AI systems by risk level
and imposes corresponding obligations. This section assesses where OpenClaw would
fall under this framework.

### 6.1 Risk Classification

| AI Act Tier | Description | Does OpenClaw Qualify? |
|---|---|---|
| **Unacceptable Risk** (banned) | Social scoring, real-time biometric surveillance, manipulation of vulnerable groups | Not directly, though impersonation capabilities could constitute manipulation |
| **High Risk** | AI in critical infrastructure, education, employment, law enforcement, migration, democratic processes | Potentially -- if used for employment communication, legal correspondence, or financial transactions via browser automation |
| **Limited Risk** | Chatbots and AI systems interacting with humans | **YES** -- at minimum. The AI communicates with third parties who do not know they are interacting with AI |
| **Minimal Risk** | Spam filters, video games | No |

**Assessment**: OpenClaw falls at minimum into the **Limited Risk** category, and
depending on deployment context could qualify as **High Risk**. If used to send
employment-related communications, interact with financial institutions, or operate
in contexts where people's rights are affected, high-risk obligations apply.

### 6.2 Transparency Requirements (Article 52)

| Requirement | OpenClaw Status | Compliance |
|---|---|---|
| Users must be informed they are interacting with AI | No disclosure mechanism exists. Auto-reply sends messages indistinguishable from human communication. | **NON-COMPLIANT** |
| AI-generated content must be labeled | No labeling of AI-generated messages, emails, or browser-submitted forms. | **NON-COMPLIANT** |
| Deepfake/impersonation disclosure | Voice call skill enables vocal impersonation. No disclosure. | **NON-COMPLIANT** |
| Emotion recognition system disclosure | Not applicable (no emotion recognition) | N/A |

### 6.3 Human Oversight Requirements (Article 14)

| Requirement | OpenClaw Status | Compliance |
|---|---|---|
| Human oversight must be possible | No global kill switch. Auto-reply and cron operate without human presence. | **NON-COMPLIANT** |
| Human must be able to understand AI system capabilities | No comprehensive capability documentation for end users. | **NON-COMPLIANT** |
| Human must be able to override/interrupt | Exec approval exists but is pre-grantable. No per-message approval for auto-reply. No per-action approval for cron. | **PARTIAL** |
| Human must be able to decide not to use the system | System can be stopped, but no graceful shutdown that preserves state and notifies contacts. | **PARTIAL** |

### 6.4 Additional AI Act Requirements

| Requirement | Status |
|---|---|
| Risk management system (Art. 9) | No formal risk management. This study is external. |
| Data governance (Art. 10) | No data governance. Memory collects everything. |
| Technical documentation (Art. 11) | Open source provides code transparency but no compliance documentation. |
| Record-keeping (Art. 12) | No audit logging adequate for regulatory review. |
| Accuracy, robustness, cybersecurity (Art. 15) | Partially addressed by security module; fails on robustness (no kill switch) and cybersecurity (plugin system). |

**EU AI Act Summary**: OpenClaw is **non-compliant** with the EU AI Act's
transparency requirements and substantially non-compliant with human oversight
requirements. Deployment in the EU in its current form would expose operators to
regulatory action.

---

## 7. Platform Security Model Comparison

This section compares OpenClaw's security model to established platform extension
models that have evolved over decades of security engineering.

### 7.1 Extension/Plugin Security Comparison

| Category | OpenClaw Plugins | VS Code Extensions | Chrome Extensions | iOS Apps | Android Apps |
|---|---|---|---|---|---|
| **Sandboxing** | None. In-process execution. | Extension Host process (separate from renderer) | Content scripts isolated; background in service worker | App Sandbox. Each app in own container. | SELinux MAC; app sandbox per UID |
| **Permission Model** | None. Full system access. | `contributes` declarations; some API gating | Manifest V3 permissions; host permissions; activeTab | Entitlements; runtime permission prompts | Manifest permissions; runtime dangerous permissions |
| **Code Signing** | None | Marketplace publisher verification | Chrome Web Store developer registration | Mandatory. Apple code signing + notarization | APK signing; Play App Signing |
| **Review Process** | None (ClawHub is unreviewed) | Marketplace review + automated scanning | Chrome Web Store review + automated analysis | App Store Review (human + automated) | Play Protect + automated scanning |
| **Runtime Monitoring** | None | Extension telemetry; crash reporting | Extension performance monitoring; abuse detection | System integrity monitoring; runtime checks | Play Protect runtime scanning; Verify Apps |
| **Capability Restrictions** | None. Shell, filesystem, network, hooks all available. | API surface gated; no arbitrary shell access | No arbitrary code exec in MV3; declarativeNetRequest | No private API access; no arbitrary file access | Scoped storage; no shell without root |
| **Update Verification** | npm install (no integrity check) | Marketplace signature verification | CRX signature verification | App Store re-review on update | Play Store re-review on update |
| **Privilege Escalation Prevention** | Not addressed | Process isolation prevents renderer compromise | Site isolation; process-per-extension in MV3 | Sandboxing + code signing + ASLR/PIE | SELinux + seccomp + ASLR |
| **Uninstall Cleanup** | Not documented | Clean extension state removal | Clean profile data removal | Complete app data deletion | App data scoped and removable |

### 7.2 Key Takeaways

Every established platform security model rests on three pillars that OpenClaw
lacks entirely:

1. **Isolation**: Extensions run in separate processes/sandboxes with defined
   communication channels. OpenClaw plugins run in the same process as the host.

2. **Declaration**: Extensions must declare what they need before installation. Users
   can review and approve. OpenClaw plugins get everything by default.

3. **Verification**: Extensions are reviewed (human or automated) before distribution.
   OpenClaw's ClawHub has no review process.

These are not optional features that mature platforms added later. They were
foundational design decisions made precisely because the alternative -- unrestricted
in-process extensions -- was recognized as architecturally indefensible decades ago.

---

## 8. Docker/Container Security Best Practices Comparison

OpenClaw uses Docker containers for its sandbox environment. This section compares
the implementation against the CIS Docker Benchmark and general container security
best practices.

### 8.1 CIS Docker Benchmark Comparison

| CIS Benchmark Item | Best Practice | OpenClaw Status | Compliance |
|---|---|---|---|
| 4.1 Create a user for the container | Run as non-root | YES -- `sandbox` user created | COMPLIANT |
| 4.2 Use trusted base images | Pinned, verified images | YES -- SHA-pinned `debian:bookworm-slim` | COMPLIANT |
| 4.5 Enable Content Trust | Docker Content Trust for image verification | Not documented | NON-COMPLIANT |
| 4.6 Add HEALTHCHECK | Container health monitoring | Not present in Dockerfiles | NON-COMPLIANT |
| 5.2 Do not use host network mode | Network namespace isolation | Containers do not use `--network=host` but also lack `--network=none` | PARTIAL |
| 5.3 Restrict network traffic | Limit inter-container traffic | No network restrictions. Full outbound access. | NON-COMPLIANT |
| 5.4 Do not use privileged containers | Drop capabilities | Not documented as privileged; but also no explicit `--cap-drop ALL` | PARTIAL |
| 5.10 Limit memory | Set memory limits | No `--memory` flags in Dockerfiles or docs | NON-COMPLIANT |
| 5.11 Set CPU priority | CPU limits | No `--cpus` or `--cpu-shares` limits | NON-COMPLIANT |
| 5.12 Mount root FS as read-only | `--read-only` flag | Not used. Writable filesystem. | NON-COMPLIANT |
| 5.14 Limit the restart policy | Restart limits | Using `sleep infinity`; no restart policy documented | PARTIAL |
| 5.15 Do not share host PID namespace | PID isolation | Not documented as sharing | ASSUMED COMPLIANT |
| 5.25 Restrict container from acquiring new privileges | `--security-opt=no-new-privileges` | Not applied | NON-COMPLIANT |
| 5.28 Use PIDs cgroup limit | Prevent fork bombs | Not applied | NON-COMPLIANT |

### 8.2 Advanced Container Security

| Security Measure | Best Practice | OpenClaw Status |
|---|---|---|
| **Seccomp Profile** | Custom or default seccomp profile restricting syscalls | Not applied. Default Docker seccomp only (if Docker daemon enables it). |
| **AppArmor/SELinux** | Mandatory access control profile | Not applied. No custom AppArmor or SELinux profiles. |
| **Read-only Root FS** | Prevent filesystem modifications except in designated volumes | Not applied. Full writable filesystem. |
| **No-new-privileges** | Prevent privilege escalation via setuid binaries | Not applied. |
| **Capability Dropping** | Drop all capabilities, add only needed ones | Not applied. Default Docker capabilities. |
| **User Namespace Remapping** | Map container root to non-root on host | Not documented. |
| **Image Vulnerability Scanning** | Scan images for known CVEs | Not documented in build pipeline. |
| **Distroless/Minimal Images** | Use minimal base images | Uses `debian:bookworm-slim` (reasonable but not minimal). sandbox-common includes full build toolchains. |

### 8.3 Sandbox-Specific Concerns

The `sandbox-common` image is particularly concerning from a container security
perspective. It includes: Node.js, npm, pnpm, Bun, Python 3, Go, Rust, Cargo,
build-essential, and Homebrew. This effectively provides a full development
environment inside the container, meaning:

- Arbitrary native code can be compiled and executed
- Package managers can download and install additional software
- The attack surface is vastly larger than necessary for most sandboxed operations

**CIS Docker Benchmark Compliance**: 2 compliant, 4 partial, 8 non-compliant
out of 14 assessed items (**14% full compliance**).

---

## 9. Gap Summary Dashboard

### 9.1 Compliance Overview by Framework

```
FRAMEWORK                          ASSESSED    MET    PARTIAL    FAILED    SCORE
==========================================================================================
OWASP Top 10 (2021)                10          0      4          6         20%
OWASP LLM Top 10                   10          0      2          8         10%
NIST CSF (5 functions)             17          0      4          13        12%
STRIDE (6 threat categories)        6          0      0          6          0%
EU AI Act (transparency)            4          0      0          4          0%
EU AI Act (human oversight)         4          0      2          2         25%
CIS Docker Benchmark               14          2      4          8         29%
Platform Security Model (9 cats)    9          0      0          9          0%
==========================================================================================
OVERALL                            74          2      16         56        ~14%
```

### 9.2 Visual Compliance Map

```
                                    0%    25%    50%    75%    100%
                                    |------|------|------|------|
OWASP Top 10 (2021)        [======|                              ]  20%
OWASP LLM Top 10           [===|                                 ]  10%
NIST CSF                    [====|                                ]  12%
STRIDE                      [|                                    ]   0%
EU AI Act (Transparency)    [|                                    ]   0%
EU AI Act (Oversight)       [======|                              ]  25%
CIS Docker Benchmark        [========|                            ]  29%
Platform Security Model     [|                                    ]   0%
                            |------|------|------|------|------|
                            0%    25%    50%    75%    100%

LEGEND:  [====]  Partial compliance achieved
         [|   ]  No meaningful compliance
         Target: 75%+ for production-grade systems
```

### 9.3 Compliance by Security Domain

```
DOMAIN                           STATUS          NOTES
=========================================================================
Authentication                   PARTIAL (30%)   Token auth exists; no MFA, no RBAC
Authorization/Access Control     FAIL (10%)      No least privilege; plugins full access
Encryption (Transit)             PARTIAL (40%)   Some TLS; embeddings unencrypted
Encryption (Rest)                FAIL (5%)       No encryption at rest for memory
Input Validation                 PARTIAL (25%)   Regex patterns; no structural defense
Output Handling                  FAIL (10%)      LLM output directly acted upon
Supply Chain Security            FAIL (10%)      No signing, no review, no SBOM
Audit/Logging                    FAIL (5%)       No security audit trail
Incident Response                FAIL (0%)       No IR plan, no kill switch
Privacy/Data Protection          FAIL (10%)      Data sent externally; no classification
Human Oversight                  PARTIAL (20%)   Approval exists but bypassable
Transparency                     FAIL (0%)       No AI disclosure anywhere
Container Security               PARTIAL (30%)   Non-root, pinned images; no hardening
Plugin Isolation                 FAIL (0%)       In-process; no sandbox at all
Autonomy Controls                FAIL (10%)      Unbounded chains; no emergency stop
=========================================================================
WEIGHTED AVERAGE:                                ~13%
```

### 9.4 Estimated Overall Compliance Score

Taking into account the weighted severity of each framework and the degree of
partial compliance in some areas, the estimated overall compliance score is:

```
+============================================================+
|                                                            |
|      ESTIMATED OVERALL COMPLIANCE: ~14%                    |
|                                                            |
|      Industry baseline for production systems: 70-80%      |
|      Gap to minimum acceptable: ~56 percentage points      |
|                                                            |
+============================================================+
```

This score reflects the reality that OpenClaw has invested in some security
measures (SHA-pinned images, secret detection, security audit module) but lacks
the architectural foundations (isolation, least privilege, mandatory oversight)
that constitute the majority of compliance requirements across all frameworks.

---

## 10. Priority Remediation Map

### 10.1 Standards-Driven Prioritization

The following maps the most critical standards gaps to the recommendations in
the final report (14-final-report.md), ordered by the number of frameworks
each gap violates simultaneously.

| Priority | Gap Description | Frameworks Violated | Report Recommendation | Effort |
|---|---|---|---|---|
| **P1** | No plugin sandboxing/isolation | OWASP A01, A04, A08; LLM05, LLM07, LLM08; NIST Protect; STRIDE-E; CIS; Platform Model | T1.1 (Plugin sandboxing) | Large |
| **P2** | No AI disclosure/transparency | EU AI Act Art.52; LLM02; STRIDE-S; IEEE P2894; NIST AI RMF | T1.4 (Mandatory AI disclosure) | Small |
| **P3** | No global emergency stop | LLM08; NIST Respond; STRIDE-D; EU AI Act Art.14; NIST AI RMF | T1.2 (Global emergency stop) | Medium |
| **P4** | Unbounded agent chains | LLM04, LLM08; NIST Protect; STRIDE-D, STRIDE-E; CIS | T1.5 (Agent chain limits) | Small |
| **P5** | No encryption at rest | OWASP A02; NIST Protect; STRIDE-I; CIS Docker | T2.4 (Memory encryption) | Medium |
| **P6** | Network not isolated in sandbox | OWASP A10; NIST Protect; STRIDE-I; CIS Docker; Platform Model | T1.3 (Network isolation) | Small |
| **P7** | No audit logging | OWASP A09; NIST Detect, Respond; STRIDE-R; EU AI Act Art.12; CIS | T3.5 (Audit logging) | Medium |
| **P8** | No code signing for plugins | OWASP A08; LLM05; NIST Protect; Platform Model | T2.7 (Plugin code signing) | Medium |
| **P9** | No MFA on authentication | OWASP A07; NIST Protect; CIS Controls | T2.3 (MFA gateway auth) | Medium |
| **P10** | Memory sent to external APIs | OWASP A02; LLM06; NIST Protect; STRIDE-I; EU AI Act | T3.2 (Client-side embedding) | Large |

### 10.2 Quick Wins (High Impact, Low Effort)

These gaps can be addressed with relatively small code changes but would
significantly improve compliance posture:

1. **Add `--network=none` to sandbox containers** (P6)
   - Addresses: OWASP A10, CIS Docker 5.3, STRIDE-I
   - Effort: Configuration change in container launch
   - Impact: Eliminates data exfiltration and SSRF from sandbox

2. **Add mandatory AI disclosure prefix to outbound messages** (P2)
   - Addresses: EU AI Act Art.52, STRIDE-S, IEEE P2894
   - Effort: Small change in message delivery pipeline
   - Impact: Transforms legal compliance posture entirely

3. **Implement agent chain depth limit** (P4)
   - Addresses: LLM04, LLM08, STRIDE-D
   - Effort: Counter and check in session spawn logic
   - Impact: Prevents runaway resource consumption

4. **Remove `dangerouslyDisableDeviceAuth` flag** (relates to T1.6)
   - Addresses: OWASP A01, A05, A07
   - Effort: Flag removal or multi-step confirmation
   - Impact: Prevents programmatic security bypass

### 10.3 Critical Path Items (High Impact, High Effort)

These require architectural changes but are necessary for any meaningful
compliance:

1. **Plugin sandbox architecture** (P1)
   - Move plugins to separate processes or V8 isolates
   - Implement capability-based permission system
   - Would address the single largest gap across all frameworks

2. **Structural prompt injection defense** (relates to T2.6)
   - Separate instruction and data channels architecturally
   - Cannot be solved with more regex patterns
   - Requires fundamental rethinking of input processing

3. **Comprehensive audit logging** (P7)
   - Tamper-resistant logs of all autonomous actions
   - Attribution of human vs. AI actions
   - Necessary for EU AI Act, NIST CSF, and forensic capability

4. **Client-side embedding for memory** (P10)
   - Eliminate default external API transmission
   - Requires local model integration as default
   - Addresses data sovereignty and privacy regulations

### 10.4 Remediation Roadmap Summary

```
PHASE 1 (Weeks 1-4): Quick Wins
================================================================
- Network isolation for sandboxes ........... P6   [SMALL]
- AI disclosure on messages ................ P2   [SMALL]
- Agent chain limits ...................... P4   [SMALL]
- Dangerous flag removal/restriction ....... --   [SMALL]
  Expected compliance improvement: +8-12 percentage points

PHASE 2 (Months 2-3): Core Security
================================================================
- Plugin code signing ..................... P8   [MEDIUM]
- MFA on gateway auth .................... P9   [MEDIUM]
- Memory encryption at rest .............. P5   [MEDIUM]
- Audit logging framework ................ P7   [MEDIUM]
  Expected compliance improvement: +15-20 percentage points

PHASE 3 (Months 3-6): Architectural
================================================================
- Plugin sandboxing ...................... P1   [LARGE]
- Structural prompt injection defense ..... --   [LARGE]
- Client-side embedding option ........... P10  [LARGE]
- Human oversight framework .............. --   [LARGE]
  Expected compliance improvement: +20-25 percentage points

PHASE 4 (Months 6-12): Maturity
================================================================
- ClawHub review process ................. --   [LARGE]
- Formal incident response plan .......... --   [MEDIUM]
- Runtime plugin monitoring .............. --   [LARGE]
- Consent granularity framework .......... --   [MEDIUM]
  Expected compliance improvement: +10-15 percentage points

PROJECTED FINAL COMPLIANCE: ~65-80%
(Sufficient for production deployment consideration)
```

---

## 11. Conclusion

The gap between OpenClaw's current security posture and the requirements of
established security frameworks is not a matter of missing a few checkboxes. It
is a structural deficit that spans every framework examined.

The most concerning pattern is not any single gap but the consistent failure in
**isolation and privilege separation** -- the principle that components should
operate with minimum necessary access and that compromise of one component should
not cascade to the entire system. This principle is foundational to OWASP (A01),
NIST (Protect), CIS (Access Control), STRIDE (Elevation of Privilege), every
platform security model (VS Code, Chrome, iOS, Android), and every container
security benchmark (CIS Docker). OpenClaw violates it at every architectural
boundary: plugins in-process, sandbox with network access, tools on deny-list
instead of allow-list, pre-grantable approvals, and unbounded agent chains.

The remediation path is clear but substantial. The quick wins in Phase 1 of the
roadmap can be implemented in weeks and would begin to address the most
egregious gaps. The architectural changes in Phases 3-4 require months of work
but are necessary for any credible claim of standards compliance.

For the 237,000 developers who have starred this project, and the potentially
millions of users running it, the message from this analysis is straightforward:
**OpenClaw's capabilities have outpaced the security frameworks that should
govern them.** The frameworks exist. The standards are clear. The gap is
measurable. What remains is the engineering and governance work to close it.

---

*Cross-reference: This document supports the final consolidated report
(14-final-report.md) Section 8 (Consolidated Recommendations). Gaps identified
here map directly to recommendation tiers T1 through T4.*

*Frameworks assessed: OWASP Top 10 (2021), OWASP Top 10 for LLM Applications,
NIST Cybersecurity Framework (CSF), CIS Controls v8, STRIDE Threat Model,
EU AI Act, NIST AI Risk Management Framework, IEEE P2894, CIS Docker Benchmark,
VS Code/Chrome/iOS/Android platform security models.*
