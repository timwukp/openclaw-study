> 本文为中文翻译版本。英文原版为权威版本，如有差异以英文版为准。
> [English Version](../README.md)

# OpenClaw 深度研究报告

> **许可协议**: [CC BY 4.0](../LICENSE) — 在注明出处的前提下可自由分享和改编
> **免责声明**: [阅读完整免责声明](DISCLAIMER.md) — 本研究为独立研究，非安全建议
> **未隶属于** OpenClaw 项目或本文提及的任何公司

## 关于本研究

本文是对 OpenClaw 的系统性分析。OpenClaw 是一个开源、自托管的 AI 个人助手平台（GitHub 星标 237K+）。本研究聚焦于理解其技术架构，并识别安全、伦理、人身安全及社会影响等维度的潜在风险。

**研究动机**：像 OpenClaw 这样的 AI 个人助手对人类设备、通信和个人数据拥有前所未有的访问权限。它们可以自主行动、冒充用户身份，并在无实时监督的情况下全天候运行。本研究旨在对由此产生的风险进行编目和分析。

**研究方法**：我们在不克隆代码仓库的前提下，完全通过 GitHub API 检查和文档审阅来分析公开源代码和架构。我们不执行任何 OpenClaw 代码。

---

## 文档索引

### 基础文档

| 文件 | 说明 |
|---|---|
| [01-overview.md](01-overview.md) | OpenClaw 是什么、关键特征、本研究的意义 |
| [02-architecture.md](02-architecture.md) | 完整技术架构、图表、源代码映射 |
| [03-llm-providers.md](03-llm-providers.md) | 支持的 LLM 模型、路由功能、提供商详情 |
| [04-ecosystem.md](04-ecosystem.md) | 社区项目、插件、部署选项 |
| [05-risk-analysis-framework.md](05-risk-analysis-framework.md) | 综合风险类别与分析方法论 |

### 深度分析

| 文件 | 阶段 | 说明 |
|---|---|---|
| [06-security-sandboxing.md](06-security-sandboxing.md) | 第 2 阶段 | 沙箱机制、身份认证、Prompt Injection（提示注入）、插件安全 |
| [07-plugin-supply-chain.md](07-plugin-supply-chain.md) | 第 3 阶段 | 插件信任模型、供应链安全、48 个内置技能 |
| [08-memory-data-analysis.md](08-memory-data-analysis.md) | 第 4 阶段 | 记忆系统、数据收集、外部传输、隐私 |
| [09-autonomy-consent.md](09-autonomy-consent.md) | 第 5 阶段 | AI 自主性、同意机制缺口、人类参与回路（Human-in-the-Loop） |
| [10-channel-impersonation.md](10-channel-impersonation.md) | 第 6+7 阶段 | 通道冒充 + 浏览器/计算机使用 |
| [11-browser-computer-use.md](11-browser-computer-use.md) | 第 7 阶段 | 浏览器深度分析：CDP、Playwright、扩展程序中继、沙箱 |
| [12-ethical-assessment.md](12-ethical-assessment.md) | 第 8 阶段 | 依赖性、自主能力削弱、智力衰减、伦理问题 |
| [13-comparison-best-practices.md](13-comparison-best-practices.md) | 第 9 阶段 | 对标 OWASP、NIST、STRIDE、EU AI Act（欧盟人工智能法案）及平台模型的差距分析 |
| [14-final-report.md](14-final-report.md) | 第 10 阶段 | 综合发现、风险仪表盘、建议措施 |

---

## 本研究回答的关键问题

1. **OpenClaw 能否在未经明确同意的情况下采取行动？** 是 — cron 定时任务、自动回复、钩子（hooks）、Agent 链均可自主运行
2. **它能否在未告知的情况下冒充用户？** 是 — 所有通道均无强制性 AI 身份披露
3. **沙箱机制的效果如何？** 薄弱 — 拥有完整网络访问权限，插件在进程内运行，配置标志可禁用所有安全措施
4. **恶意插件能否危害安全？** 是 — 插件拥有完整进程访问权限，扫描器可被轻易绕过
5. **记忆系统收集哪些数据？** 一切 — 对话、文件、偏好设置，均被发送至外部 API 进行向量化（Embedding）
6. **它是否会造成有害依赖？** 高风险 — 全天候运行、处理所有通信、通过记忆系统产生情感依附
7. **AI 介入通信会产生什么后果？** 冒充 — 第三方不知情、无披露机制、存在法律影响
8. **有害行为由谁承担责任？** 无人 — MIT 许可协议、自托管部署、无问责框架
9. **对认知发展有何影响？** 衰减 — 减少批判性思维、社交技能和决策能力的锻炼
10. **缺少哪些安全保障？** 众多 — 无紧急停止开关（Kill Switch）、无插件沙箱、无同意框架、无审计追踪

---

## 研究统计数据

- **文档总数**：15 份（含 LICENSE 和 DISCLAIMER）
- **分析总行数**：约 7,200+ 行
- **检查的源文件**：20+ 个目录中的 100+ 个文件
- **覆盖的风险类别**：安全、供应链、记忆/隐私、自主性、冒充、浏览器控制、伦理、社会影响
- **关键发现**：识别出 25+ 项独立风险
- **建议措施**：22+ 项分级建议

## 研究开始时间：2026-02-28
## 研究完成时间：2026-02-28
## 研究人员：Tim Wu + Claude (Anthropic)

---

*本研究基于 [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/) 许可协议发布。
请参阅 [DISCLAIMER.md](DISCLAIMER.md) 了解重要限制条件和法律声明。*
