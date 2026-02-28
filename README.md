# OpenClaw Deep-Dive Study

> **License**: [CC BY 4.0](LICENSE) — free to share and adapt with attribution
> **Disclaimer**: [Read full disclaimer](DISCLAIMER.md) — this is independent research, not security advice
> **Not affiliated with** the OpenClaw project or any company mentioned herein

## About This Study

This is a systematic analysis of OpenClaw, the open-source self-hosted AI
personal assistant platform (237K+ GitHub stars). The study focuses on understanding
the technical architecture and identifying potential risks across security, ethics,
human safety, and societal impact dimensions.

**Motivation**: AI-powered personal assistants like OpenClaw have unprecedented
access to human devices, communications, and personal data. They can act
autonomously, impersonate users, and operate 24/7 without real-time oversight.
This study aims to catalog and analyze the risks this creates.

**Approach**: We analyze the public source code and architecture WITHOUT cloning
the repository, working entirely through GitHub API inspection and documentation
review. We do NOT execute any OpenClaw code.

---

## Document Index

### Foundation Documents

| File | Description |
|---|---|
| [01-overview.md](01-overview.md) | What is OpenClaw, key characteristics, why this study matters |
| [02-architecture.md](02-architecture.md) | Full technical architecture, diagrams, source code map |
| [03-llm-providers.md](03-llm-providers.md) | Supported LLM models, routing features, provider details |
| [04-ecosystem.md](04-ecosystem.md) | Community projects, plugins, deployment options |
| [05-risk-analysis-framework.md](05-risk-analysis-framework.md) | Comprehensive risk categories and analysis methodology |

### Deep-Dive Analysis

| File | Phase | Description |
|---|---|---|
| [06-security-sandboxing.md](06-security-sandboxing.md) | Phase 2 | Sandboxing, auth, prompt injection, plugin security |
| [07-plugin-supply-chain.md](07-plugin-supply-chain.md) | Phase 3 | Plugin trust model, supply chain, 48 bundled skills |
| [08-memory-data-analysis.md](08-memory-data-analysis.md) | Phase 4 | Memory system, data collection, external transmission, privacy |
| [09-autonomy-consent.md](09-autonomy-consent.md) | Phase 5 | Autonomous AI, consent gaps, human-in-the-loop |
| [10-channel-impersonation.md](10-channel-impersonation.md) | Phase 6+7 | Channel impersonation + browser/computer-use |
| [11-browser-computer-use.md](11-browser-computer-use.md) | Phase 7 | Browser deep-dive: CDP, Playwright, extension relay, sandbox |
| [12-ethical-assessment.md](12-ethical-assessment.md) | Phase 8 | Dependency, agency erosion, intelligence attenuation, ethics |
| [13-comparison-best-practices.md](13-comparison-best-practices.md) | Phase 9 | Gap analysis vs. OWASP, NIST, STRIDE, EU AI Act, platform models |
| [14-final-report.md](14-final-report.md) | Phase 10 | Consolidated findings, risk dashboard, recommendations |

---

## Key Questions This Study Answers

1. **Can OpenClaw take actions without explicit consent?** YES — cron, auto-reply, hooks, agent chains all operate autonomously
2. **Can it impersonate the user without disclosure?** YES — no mandatory AI disclosure on any channel
3. **How effective is the sandboxing?** WEAK — full network access, plugins in-process, config flags disable all security
4. **Can malicious plugins compromise security?** YES — plugins have full process access, scanner is trivially bypassed
5. **What data does memory collect?** EVERYTHING — conversations, files, preferences, sent to external APIs for embedding
6. **Does it create harmful dependency?** HIGH RISK — always-on, handles all communications, emotional attachment via memory
7. **What happens when AI mediates communications?** IMPERSONATION — third parties don't know, no disclosure, legal implications
8. **Who is liable for harmful actions?** NOBODY — MIT license, self-hosted, no accountability framework
9. **How does it affect cognitive development?** ATTENUATION — reduces practice in critical thinking, social skills, decision-making
10. **What safeguards are missing?** MANY — no kill switch, no plugin sandbox, no consent framework, no audit trail

---

## Study Statistics

- **Total documents**: 15 (including LICENSE and DISCLAIMER)
- **Total lines of analysis**: ~7,200+
- **Source files examined**: 100+ files across 20+ directories
- **Risk categories covered**: Security, Supply Chain, Memory/Privacy, Autonomy, Impersonation, Browser Control, Ethics, Societal Impact
- **Critical findings**: 25+ discrete risks identified
- **Recommendations**: 22+ tiered recommendations

## Study Started: 2026-02-28
## Study Completed: 2026-02-28
## Researcher: Tim Wu + Claude (Anthropic)

---

*This study is licensed under [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/).
See [DISCLAIMER.md](DISCLAIMER.md) for important limitations and legal notices.*
