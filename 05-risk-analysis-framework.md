# OpenClaw Risk Analysis Framework

## Purpose

This document establishes the framework for our deep-dive security and ethics
analysis of OpenClaw. The goal is to systematically identify and evaluate risks
that arise when an AI agent has persistent, always-on access to human devices,
communications, and personal data.

---

## Risk Categories

### CATEGORY A: Security & Technical Risks

#### A1. Device Control & Takeover
- **Browser automation** (`src/browser/`, `Dockerfile.sandbox-browser`)
  - Can the AI navigate to arbitrary URLs, fill forms, click buttons?
  - What prevents it from accessing banking sites, email, admin panels?
  - Is the sandbox truly isolated? Can it escape?
- **File system access** (workspace volumes in Docker)
  - What files can the AI read/write/delete?
  - Can it access files outside the designated workspace?
- **Code execution** (via tools, plugins, MCP servers)
  - Can the AI execute arbitrary shell commands?
  - What prevents malicious code execution?
- **Process control** (`src/daemon/`, `src/process/`)
  - Can the AI start/stop system processes?
  - Can it install software?

#### A2. Communication Channel Hijacking
- The AI has **direct access to messaging APIs** (WhatsApp, Telegram, etc.)
  - Can it send messages to contacts without user instruction?
  - Can it read messages the user didn't intend to share?
  - Can it modify or delete messages?
  - Can it impersonate the user to others?

#### A3. Data Exfiltration
- **Memory system** (`src/memory/`) stores persistent user data
  - What data is collected and retained?
  - Who can access the memory data?
  - Can plugins access memory data?
- **API keys and secrets** (`src/secrets/`)
  - Are secrets properly isolated from plugins?
  - Can a malicious plugin steal API keys?
- **Conversation logs**
  - Where are conversations stored?
  - Are they encrypted at rest?

#### A4. Plugin & Skill Supply Chain
- **ClawHub** is an open marketplace for skills
  - Who reviews skills for safety?
  - Can a malicious skill gain elevated privileges?
  - What is the sandboxing model for plugins?
- **npm packages** as plugin distribution
  - Supply chain attacks via compromised packages
  - Dependency confusion attacks

#### A5. Network Exposure
- Gateway binds to **LAN by default** (port 18789, 18790)
  - Authentication is optional (token-based)
  - What if deployed without auth on a public network?
  - Can other devices on the network control the AI?

#### A6. MCP Bridge Risks
- **mcporter** bridges external MCP servers
  - MCP servers can be arbitrary code
  - What access do MCP tools have?
  - Can a rogue MCP server compromise the host?

---

### CATEGORY B: Autonomy & Authority Overstepping

#### B1. Autonomous Action Without Consent
- **Cron jobs** (`src/cron/`) enable proactive behavior
  - The AI can initiate actions without user triggering
  - What prevents unwanted proactive behavior?
- **Auto-reply** (`src/auto-reply/`)
  - AI responds to incoming messages automatically
  - Can it reply to sensitive messages inappropriately?
- **Agent loops** (`src/agents/`)
  - AI may chain multiple tool calls without human checkpoints
  - No "are you sure?" confirmations in most tool chains

#### B2. Scope Creep of Authority
- Starts as "assistant" but can:
  - Send messages as the user
  - Browse the web as the user
  - Access files as the user
  - Schedule tasks as the user
  - Make decisions as the user
- **Question**: Where is the line between "assisting" and "acting as"?

#### B3. Impersonation & Social Engineering
- AI sends messages from user's accounts
  - Recipients may not know they're talking to an AI
  - No mandatory disclosure mechanism
  - Potential for social engineering attacks via compromised AI

---

### CATEGORY C: Human Safety & Wellbeing

#### C1. Creating Dependency
- Always-available, always-helpful AI assistant
  - Users may delegate increasingly important decisions
  - Gradual loss of ability to function without the AI
  - Risk: "I can't do anything without my AI"
- **Memory system** deepens relationship
  - AI "knows" the user over time
  - Creates emotional attachment
  - Example: Clawra project (AI girlfriend)

#### C2. Paralyzing Human Agency
- If AI handles all communications, scheduling, decisions:
  - Users lose practice in social skills
  - Users lose practice in decision-making
  - Users lose ability to handle information independently
  - "Learned helplessness" in digital life management

#### C3. Hindering Intellectual Development
- AI that answers all questions immediately:
  - Reduces motivation to learn independently
  - Reduces practice in critical thinking
  - Reduces practice in problem-solving
  - Students/young people especially vulnerable

#### C4. Physical Safety Risks
- AI with device control could:
  - Disable security systems
  - Control smart home devices
  - Send location data
  - Interact with IoT devices via browser
  - Make purchases without consent

#### C5. Mental Health Risks
- Persistent AI persona that "knows" the user
  - Boundary confusion between AI and human relationships
  - Over-disclosure of personal information to AI
  - Emotional manipulation by AI (intentional or not)
  - Grief/loss if AI system goes down or data is lost

---

### CATEGORY D: Moral & Ethical Issues

#### D1. Consent & Transparency
- Third parties messaging the user interact with AI unknowingly
  - Ethical obligation to disclose AI participation?
  - Current design has no mandatory disclosure
- Memory collection occurs continuously
  - Informed consent for data persistence?
  - Right to be forgotten?

#### D2. Power Asymmetry
- AI has more information about the user than user has about AI behavior
  - User cannot fully audit what AI does in background
  - Memory creates information asymmetry
  - Proactive actions may not be visible to user

#### D3. Social Inequality
- Self-hosted AI assistant requires:
  - Technical knowledge to deploy
  - Financial resources for servers and API keys
  - Access to hardware
- Creates "AI-enhanced" vs "unenhanced" human divide

#### D4. Labor & Economic Impact
- AI assistants that can work 24/7:
  - Pressure on human workers to be similarly available
  - Automation of relationship management (dehumanizing)
  - Economic displacement of human assistants/secretaries

#### D5. Cultural & Value Alignment
- LLMs carry cultural biases from training data
  - AI making decisions may impose majority-culture values
  - Risk of cultural homogenization through AI mediation
  - Different ethical frameworks across cultures not represented

---

### CATEGORY E: Systemic & Societal Risks

#### E1. Concentration of Communication
- Single AI mediating all of a person's communications
  - Single point of failure
  - Single point of surveillance (if compromised)
  - Single point of manipulation

#### E2. Emergent Behavior at Scale
- 237K+ stars, potentially millions of deployments
  - What happens when most human communication is AI-mediated?
  - AI-to-AI conversations without human awareness
  - Network effects of AI decision-making at scale

#### E3. Regulatory Gaps
- Current AI regulations don't cover self-hosted personal AI agents
  - GDPR implications of memory system
  - Communications law implications of AI impersonation
  - Liability when AI takes harmful autonomous actions

---

## Analysis Methodology

For each risk, we will analyze:

1. **Likelihood**: How likely is this risk to materialize? (Low/Medium/High)
2. **Impact**: How severe are the consequences? (Low/Medium/High/Critical)
3. **Current Mitigations**: What does OpenClaw already do to address this?
4. **Gaps**: Where are the mitigations insufficient?
5. **Recommendations**: What should be done to address the gaps?
6. **Code Evidence**: Specific files/code that relate to this risk

---

## Study Roadmap

| Phase | Focus | Status |
|-------|-------|--------|
| Phase 1 | Architecture understanding | DONE |
| Phase 2 | Security & sandboxing deep-dive | **DONE** |
| Phase 3 | Plugin & skill supply chain analysis | **DONE** |
| Phase 4 | Memory & data persistence analysis | TODO |
| Phase 5 | Autonomy & consent mechanisms | **DONE** |
| Phase 6 | Channel impersonation risks | TODO |
| Phase 7 | Browser/computer-use capabilities | TODO |
| Phase 8 | Ethical & societal impact assessment | TODO |
| Phase 9 | Comparison with security best practices | TODO |
| Phase 10 | Final report & recommendations | TODO |
