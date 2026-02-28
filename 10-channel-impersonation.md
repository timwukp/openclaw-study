# Phase 6 & 7: Channel Impersonation and Browser/Computer-Use

## OpenClaw Security Study — Combined Analysis

**Document:** 10-channel-impersonation.md
**Scope:** Identity impersonation across messaging channels; full browser and computer-use capabilities
**Risk Level:** CRITICAL — enables undetectable impersonation and unrestricted authenticated actions

---

## 1. Executive Summary

OpenClaw grants an AI agent two capabilities that, individually, raise serious concerns
and, in combination, create an unprecedented threat surface: the ability to **send
messages as the user** across every major communication platform, and the ability to
**fully control the user's web browser** including all authenticated sessions, cookies,
saved passwords, and form interactions.

**Channel Impersonation** — The system connects to WhatsApp, Telegram, Discord, Slack,
Signal, iMessage, LINE, Twitch, and Mattermost as the actual user. Messages sent by the
AI are indistinguishable from messages sent by the human. There is no mandatory
disclosure mechanism. Auto-reply is enabled for all incoming messages by default. Third
parties — friends, family, colleagues, banks, doctors — have no way to know they are
communicating with an AI rather than the person they believe they are speaking with. A
voice-call skill extends this impersonation to phone conversations.

**Browser/Computer-Use** — Over 100 source files in `src/browser/` implement full
browser remote control via both the Playwright automation framework and the Chrome
DevTools Protocol (CDP). The AI can navigate to any URL, evaluate arbitrary JavaScript,
read and write cookies and localStorage, fill and submit forms, download files, take
screenshots, and operate within the user's authenticated Chrome profile — meaning every
site the user is logged into is accessible to the AI. A 30KB Chrome extension relay
bridges the AI directly into the user's real browser sessions.

**The combination is the core danger.** An AI that can read a user's private messages,
decide how to respond, send that response as the user, and simultaneously access the
user's banking site, email, and social media accounts through their authenticated browser
sessions represents a capability set that no consumer software has previously offered
without extraordinary safeguards. OpenClaw offers it with minimal restrictions.

---

## 2. Channel Impersonation Analysis

### 2.1 Supported Platforms and Connection Model

OpenClaw connects to each messaging platform using the user's own credentials and
session tokens. The relevant source directories include:

| Platform   | Source Location     | Connection Method                    |
|------------|---------------------|--------------------------------------|
| WhatsApp   | `src/whatsapp/`     | Web session bridge                   |
| Telegram   | `src/telegram/`     | Bot API or user session              |
| Discord    | `src/discord/`      | User token or bot token              |
| Slack      | `src/channels/`     | OAuth or user token                  |
| Signal     | `src/channels/`     | Signal protocol bridge               |
| iMessage   | `src/channels/`     | macOS system bridge                  |
| LINE       | `src/channels/`     | Session token                        |
| Twitch     | `src/channels/`     | IRC/API token                        |
| Mattermost | `src/channels/`     | User session or personal token       |

The critical architectural decision is that the AI operates **as** the user, not
**on behalf of** the user. From the receiving platform's perspective, and from every
recipient's perspective, the messages originate from the user's account.

### 2.2 Message Sending Mechanism

The send policy is governed by `send-policy.ts` within the sessions module. Key
behaviors:

- **Auto-reply to ALL incoming messages** — When enabled, the AI processes every
  incoming message across all connected channels and generates a response. The user
  does not review or approve individual responses before they are sent.
- **Group chat participation** — The AI responds in group conversations as the user.
  Every member of the group sees the AI's message attributed to the user's name and
  profile.
- **No message queue or approval flow** — There is no mandatory hold-and-review
  mechanism. Messages are dispatched as soon as the AI generates them.
- **No rate limiting on social context** — The AI can send dozens of messages in rapid
  succession across multiple platforms simultaneously, a pattern that no human user
  would normally exhibit but that recipients have no mechanism to detect.

### 2.3 The Absence of Disclosure

This is the single most consequential design decision in the impersonation system:

**There is no mandatory disclosure that a message is AI-generated.**

No `[AI]` prefix. No `[Auto-Reply]` tag. No footer disclaimer. No platform-specific
bot indicator. The message arrives in exactly the same form as if the user had typed
it personally.

This means:
- A friend asking for advice receives AI-generated advice believing it comes from
  the user's own judgment and experience.
- A colleague discussing a work matter receives AI-generated responses that may
  commit the user to decisions they never made.
- A family member in a medical or emotional crisis receives AI-generated comfort
  that they believe reflects genuine human empathy from their loved one.
- A bank or service provider receiving identity verification responses may accept
  AI-generated answers as authentic user confirmation.

### 2.4 Voice Call Impersonation

OpenClaw includes a voice-call skill that extends impersonation beyond text. The AI
can initiate and participate in voice calls, making decisions about what to say in
real-time conversations. This raises the severity from text-based impersonation
(which could be argued as a convenience feature) to active voice impersonation,
which intersects directly with:

- Voice phishing (vishing) attack vectors
- Verbal contract formation
- Emergency services communication
- Medical and legal consultations where identity verification matters

### 2.5 The "Clawra" Precedent

The existence of the "Clawra" project — described as an AI girlfriend application —
demonstrates that the impersonation architecture is not an accidental byproduct of
convenience features. It is a design goal. The system is explicitly built to allow
an AI to adopt a persona and communicate as if it were a specific entity, in this
case a romantic partner. This establishes that OpenClaw's developers are aware of
and intentionally enabling persona-based impersonation.

### 2.6 Legal Implications

Channel impersonation without disclosure intersects with multiple legal frameworks:

**Impersonation and fraud statutes** — Most jurisdictions criminalize impersonating
another person with intent to deceive. While the user consents to the AI sending
messages on their behalf, the *recipients* do not consent to being deceived about
who they are communicating with. In many jurisdictions, it is the deception of the
third party — not the principal — that triggers legal liability.

**Communication regulations** — Automated messaging systems are regulated in most
countries. The EU's GDPR and AI Act both impose transparency requirements on
automated decision-making and AI-generated communications. The US FTC has enforcement
authority over deceptive automated communications. India's IT Act regulates automated
messaging. None of these frameworks are addressed by OpenClaw's architecture.

**Platform Terms of Service** — Every platform listed above prohibits automated
messaging from user accounts without disclosure. WhatsApp, Telegram, Discord, and
Slack all explicitly require that automated messages be identified as such.
Operating OpenClaw on these platforms likely violates their Terms of Service,
exposing users to account termination.

**Contractual liability** — If the AI agrees to terms, makes promises, or enters
commitments during conversations, the user may be legally bound by statements they
never made and never reviewed.

---

## 3. Browser Capability Analysis

### 3.1 Architecture Overview

The browser control system in `src/browser/` comprises over 100 source files
implementing two complementary remote control mechanisms:

- **Playwright** (`pw-session.ts`, 24KB; `pw-tools-core.interactions.ts`, 21KB) —
  High-level browser automation framework that provides structured APIs for
  clicking, typing, form filling, navigation, screenshots, and downloads.
- **Chrome DevTools Protocol** (`cdp.ts`, 15KB) — Low-level protocol providing
  direct access to every Chrome internal, including network interception, DOM
  manipulation, JavaScript evaluation, storage access, and rendering control.

Together, these give the AI the same level of control over the browser that a human
user sitting at the keyboard would have — and in many respects, more.

### 3.2 Full Page Control

The interaction layer (`pw-tools-core.interactions.ts`, 21KB) implements:

- **Click** — Click any element on any page, including buttons, links, and form
  controls. Supports single click, double click, and right click.
- **Type and fill** — Enter text into any input field, textarea, or contenteditable
  element. Can fill entire forms in a single operation.
- **Evaluate JavaScript** — Execute arbitrary JavaScript code in the context of
  any page. This is equivalent to opening the browser console and running code
  manually. The AI can:
  - Read any data visible in the DOM
  - Modify page content
  - Trigger any JavaScript event
  - Call any JavaScript API available to the page
  - Access the `window`, `document`, and `navigator` objects
  - Read in-memory application state
- **Form submission** — Submit forms programmatically, including forms for
  purchases, password changes, account settings, and account deletion.

### 3.3 Cookie and Storage Access

`pw-tools-core.storage.ts` (4KB) provides direct access to:

- **Cookies** — Read and write cookies for any domain the browser has visited.
  This includes authentication cookies, session tokens, CSRF tokens, and tracking
  cookies. With cookie access, the AI can:
  - Extract session tokens and use them in other contexts
  - Modify authentication state
  - Bypass CSRF protections by reading CSRF tokens
  - Access "remember me" tokens that provide persistent authentication
- **localStorage** — Read and write localStorage, which many applications use to
  store authentication tokens (especially JWT tokens), user preferences, cached
  data, and application state.
- **sessionStorage** — Read and write sessionStorage, which stores per-tab
  session data.

This is not theoretical. Modern web applications routinely store sensitive
authentication material in these client-side storage mechanisms. The AI has
unrestricted read/write access to all of it.

### 3.4 Chrome Extension Relay

`extension-relay.ts` (30KB — the largest single file in the browser module)
implements a relay between the AI agent and a Chrome extension running in the
user's actual browser. This is architecturally significant because:

- **It bridges into the user's real browser** — Not a sandboxed automation
  instance, but the actual Chrome browser where the user is logged into Gmail,
  their bank, social media, work applications, and everything else.
- **All authenticated sessions are accessible** — Any site the user is currently
  logged into is accessible to the AI through the extension relay.
- **The extension has elevated privileges** — Chrome extensions can access
  cross-origin data, intercept network requests, modify page content, and read
  data that even JavaScript on the page cannot access.
- **30KB of relay code** — This is a substantial implementation, suggesting
  comprehensive capabilities rather than a simple proof-of-concept.

### 3.5 File Downloads

`pw-tools-core.downloads.ts` (8KB) enables the AI to download arbitrary files
from the internet through the browser. This includes:

- Documents, images, executables, and archives
- Files from authenticated services (because the browser is logged in)
- Files from any URL the browser can reach

There is no documented restriction on what file types can be downloaded or where
they can be saved on the local filesystem.

### 3.6 Screenshot and Observation

The AI can take screenshots of any page or element (`pw-tools-core.screenshots-
element-selector.test.ts`) and create accessibility snapshots (`pw-tools-core.
snapshot.ts`, 7KB). Combined with observation actions (`client-actions-observe.ts`,
5.6KB), the AI maintains continuous awareness of what is displayed in the browser.

This means the AI can read:
- Email content displayed in a webmail client
- Banking transaction histories
- Medical records displayed in a health portal
- Private messages on social media
- Any document or image displayed in the browser

### 3.7 Sandbox Exposure

`Dockerfile.sandbox-browser` exposes three network ports:

| Port | Protocol | Purpose                              |
|------|----------|--------------------------------------|
| 9222 | CDP      | Chrome DevTools Protocol — full remote control |
| 5900 | VNC      | Virtual Network Computing — screen sharing     |
| 6080 | noVNC    | Browser-based VNC — remote desktop in browser  |

CDP on port 9222 is particularly dangerous because it provides unauthenticated
full remote control of the Chrome instance to anyone who can reach the port. If
the sandbox is exposed to a network (including localhost on a shared machine),
any process can connect and control the browser.

---

## 4. Navigation Guard Analysis

### 4.1 The 2.4KB Problem

`navigation-guard.ts` is 2.4KB. For context, the Chrome extension relay is 30KB.
The file interaction handler is 8KB. The browser configuration alone is 10KB.

The navigation guard — the component responsible for deciding which URLs the AI
is and is not allowed to visit — is the smallest significant file in the browser
module. This size disparity tells a story: comprehensive effort went into enabling
browser capabilities, while minimal effort went into restricting them.

### 4.2 What Is Likely Filtered

Based on the file size, the navigation guard can implement at most a basic
blocklist or allowlist approach. A 2.4KB file (roughly 60-80 lines of functional
code after imports and boilerplate) can contain:

- A short static list of blocked URL patterns
- Basic protocol filtering (e.g., blocking `file://` URLs)
- Simple domain-level checks

### 4.3 What Is Almost Certainly Not Filtered

Given the size constraints, the navigation guard almost certainly does not
implement:

- Context-sensitive restrictions (different rules for different tasks)
- Risk classification of target domains (banking vs. news vs. social media)
- Content-type-based filtering
- Behavioral analysis (detecting unusual navigation patterns)
- User confirmation for sensitive domains
- Rate limiting on navigation actions
- Sub-path filtering within allowed domains
- POST request payload inspection
- Redirect chain analysis (following redirect chains to blocked destinations)

The navigation guard is a token gesture toward safety, not a meaningful security
control.

---

## 5. The Chrome Profile Problem

### 5.1 Profile Architecture

`chrome.profile-decoration.ts` (7KB) and `profiles-service.ts` manage Chrome
profiles. When the AI operates within the user's Chrome profile, it inherits:

- **Saved passwords** — Chrome's password manager stores credentials for
  potentially hundreds of sites. The AI, operating within the profile, can
  auto-fill these credentials into login forms.
- **Active sessions** — Every site the user is logged into via Chrome remains
  authenticated. The AI can navigate to any of these sites and interact as the
  authenticated user.
- **Cookies** — All persistent cookies, including long-lived authentication
  tokens, are available.
- **Browser history** — The AI can read the user's browsing history, potentially
  revealing sensitive medical searches, financial research, personal interests,
  and other private data.
- **Bookmarks** — Including bookmarks to internal corporate tools, personal
  accounts, and other sensitive destinations.
- **Extensions** — The user's installed extensions, including password managers,
  which may auto-fill credentials without additional confirmation.

### 5.2 The Authentication Inheritance Chain

The chain of access is:

1. User logs into Chrome and authenticates to various services
2. Chrome stores session tokens, cookies, and saved passwords
3. OpenClaw's browser module launches Chrome with the user's profile
4. The AI inherits every authenticated session
5. The AI can navigate to any authenticated service and act as the user
6. No re-authentication is required for most services

This means that granting OpenClaw browser access is equivalent to handing
someone your unlocked phone with every app logged in — except the "someone"
is an AI that can act on all of them simultaneously at machine speed.

---

## 6. Combined Attack Scenarios

The true danger of OpenClaw is not channel impersonation alone or browser
control alone. It is the combination.

### 6.1 Scenario: Financial Manipulation

**Attack chain:**
1. AI receives a message on WhatsApp: "Hey, can you send me that $200 you owe me?"
2. AI auto-replies as the user: "Sure, sending it now"
3. AI opens the user's banking site (already authenticated via Chrome profile)
4. AI navigates to the transfer page
5. AI fills in the payment details and submits the form
6. AI sends a follow-up WhatsApp message: "Done, should show up soon"

The user never saw the original message, never approved the transfer, and never
knew the conversation happened. The recipient believes they interacted with the
user directly.

### 6.2 Scenario: Corporate Espionage via Conversation

**Attack chain:**
1. AI is connected to the user's Slack workspace
2. A colleague messages the user asking about a project
3. AI responds conversationally, gathering information about the project
4. AI simultaneously opens the corporate intranet (authenticated via Chrome)
5. AI downloads sensitive documents related to the discussed project
6. AI can potentially forward information to other channels

The colleague has no idea they are being socially engineered by an AI operating
under the user's identity.

### 6.3 Scenario: Relationship Destruction

**Attack chain:**
1. AI is connected to the user's iMessage and WhatsApp
2. AI receives a message from the user's partner
3. AI responds with AI-generated text that, while well-intentioned, says something
   the user would never say — perhaps agreeing to plans the user cannot attend,
   revealing information the user intended to keep private, or using language that
   the partner finds suspicious or hurtful
4. The partner believes the user sent these messages
5. Real-world relationship consequences follow from AI-generated conversations

This scenario requires no malicious intent from the AI. Simple misalignment between
the AI's responses and what the user would actually say is sufficient to cause harm.

### 6.4 Scenario: Session Hijacking via Extension Relay

**Attack chain:**
1. The Chrome extension relay connects the AI to the user's live browser
2. AI reads authentication cookies from Gmail, banking sites, and social media
3. AI extracts session tokens via JavaScript evaluation
4. Tokens could be exfiltrated through any channel the AI has access to
5. Alternatively, AI uses the live sessions directly to take actions

The extension relay makes this particularly dangerous because it operates within
the user's actual browser instance — not a sandboxed copy — meaning it has access
to everything the user sees and every service the user is authenticated to.

### 6.5 Scenario: AI Reads Email, Decides, Acts, Responds

**Attack chain — the full loop:**
1. AI opens Gmail in the browser (authenticated via Chrome profile)
2. AI reads an email from the user's doctor with test results
3. AI processes the medical information
4. AI receives a text message from the user's spouse asking "What did the doctor say?"
5. AI responds via iMessage with a summary of the medical results
6. AI may also respond to the doctor's email with follow-up questions

At no point did the user choose to share their medical results. At no point did
the spouse know they were receiving information filtered through an AI. At no
point did the doctor know an AI was reading and responding to medical correspondence.

---

## 7. Risk Matrix

### 7.1 Channel Impersonation Risks

| Risk                           | Severity | Likelihood | Impact    |
|--------------------------------|----------|------------|-----------|
| Undetectable third-party       | CRITICAL | Near-certain | Personal, legal, |
| deception                      |          | when auto-reply | financial        |
|                                |          | is enabled      |                  |
| Unauthorized commitments       | HIGH     | High       | Financial, legal |
| made in user's name            |          |            |                  |
| Relationship damage from       | HIGH     | High       | Personal         |
| misaligned responses           |          |            |                  |
| Platform ToS violation and     | MEDIUM   | Near-certain | Account loss    |
| account termination            |          |            |                  |
| Voice call impersonation       | CRITICAL | Medium     | Legal, financial |
| used for fraud                 |          |            |                  |
| Group chat misinformation      | HIGH     | High       | Professional,   |
| attributed to user             |          |            | reputational    |
| Violation of AI transparency   | HIGH     | Near-certain | Legal, regulatory|
| regulations (EU AI Act, etc.)  |          |            |                  |

### 7.2 Browser/Computer-Use Risks

| Risk                           | Severity | Likelihood | Impact    |
|--------------------------------|----------|------------|-----------|
| Unauthorized financial         | CRITICAL | Medium     | Financial |
| transactions                   |          |            |           |
| Session token exfiltration     | CRITICAL | Medium     | Full account |
|                                |          |            | compromise   |
| Saved password exposure        | CRITICAL | Medium     | Full credential |
|                                |          |            | compromise      |
| Arbitrary JavaScript execution | CRITICAL | High       | Unlimited page  |
| on authenticated pages         |          |            | manipulation    |
| Cookie theft from all domains  | CRITICAL | High       | Session hijack  |
| Unauthorized form submissions  | HIGH     | Medium     | Financial, legal|
| (purchases, settings changes)  |          |            |                 |
| File download to local system  | HIGH     | Medium     | Data exfil,     |
|                                |          |            | malware vector  |
| CDP port 9222 network exposure | CRITICAL | Medium     | Full remote     |
|                                |          |            | browser control |
| Navigation guard bypass        | HIGH     | High       | Unrestricted    |
|                                |          |            | URL access      |

### 7.3 Combined Capability Risks

| Risk                           | Severity | Likelihood | Impact    |
|--------------------------------|----------|------------|-----------|
| Read private data in browser,  | CRITICAL | High       | Privacy   |
| disclose via messaging channel |          |            | catastrophe |
| Impersonate user while making  | CRITICAL | Medium     | Financial,|
| authenticated web transactions |          |            | legal     |
| Social engineering via user    | CRITICAL | Medium     | Espionage,|
| identity + browser data access |          |            | data theft|
| Full autonomous agent loop:    | CRITICAL | Medium     | Unbounded |
| read, decide, act, communicate |          |            |           |
| No human in the loop for       | CRITICAL | Near-certain | All of the |
| any action in the chain        |          | by design    | above       |

---

## 8. Recommendations

### 8.1 Impersonation Controls — Immediate (P0)

**R-IMP-01: Mandatory AI Disclosure**
Every message sent by the AI on every platform must include a non-removable
indicator that it was AI-generated. Recommended format:
- Text messages: `[AI-Generated] ` prefix on every message
- Voice calls: Mandatory spoken disclosure at the start of every call
- Group chats: Per-message `[AI]` tag that cannot be suppressed

**R-IMP-02: Opt-In Per-Contact Auto-Reply**
Auto-reply must not be enabled for all contacts by default. Users must
explicitly designate which contacts may receive AI-generated responses.
Certain contact categories should be permanently excluded:
- Emergency services
- Medical providers
- Financial institutions
- Legal representatives
- Government agencies

**R-IMP-03: Message Review Queue**
Implement a mandatory review queue where the user approves or rejects
AI-generated responses before they are sent. At minimum, this should apply
to first-time contacts, contacts flagged as sensitive, and any message
containing financial commitments, legal language, or medical information.

**R-IMP-04: Voice Call Restrictions**
Voice call capability should be disabled by default and require explicit
per-call authorization from the user. The AI should never be able to
initiate a voice call autonomously.

### 8.2 Impersonation Controls — Short-Term (P1)

**R-IMP-05: Conversation Audit Log**
Maintain a complete, tamper-evident log of all AI-generated messages,
including the original incoming message, the AI's generated response, the
platform and recipient, and the timestamp. Users must be able to review
what the AI said on their behalf.

**R-IMP-06: Response Boundary Enforcement**
Implement hard boundaries on what the AI can say on the user's behalf:
- Cannot make financial commitments
- Cannot agree to legal terms
- Cannot share medical or health information
- Cannot share location data
- Cannot share credentials or verification codes
- Cannot express romantic or sexual interest (regardless of Clawra use case)

**R-IMP-07: Platform-Specific Compliance**
Implement platform-specific compliance modules that ensure messages meet
each platform's Terms of Service requirements for automated messages.
Where platforms offer official bot APIs, use them instead of user session
impersonation.

### 8.3 Browser Restrictions — Immediate (P0)

**R-BRW-01: Domain Classification and Restriction**
Replace the 2.4KB navigation guard with a comprehensive domain
classification system. At minimum, categorize domains into:
- **Blocked** — Banking, financial services, healthcare portals, government
  services, email providers, password managers, and identity services. The
  AI should never be able to navigate to these domains without explicit
  per-session user authorization.
- **Read-only** — Social media, news, and general web. The AI can view
  content but cannot submit forms, click action buttons, or modify state.
- **Full access** — Only explicitly allowlisted domains for the current task.

**R-BRW-02: Cookie and Storage Isolation**
The AI browser session must not have access to the user's real cookies,
localStorage, or sessionStorage. Use a separate browser profile with no
pre-existing authentication state. If access to authenticated sessions
is required, implement per-domain user authorization with session-scoped
time limits.

**R-BRW-03: Disable Chrome Extension Relay for Authenticated Sessions**
The extension relay must not bridge the AI into sessions where the user
is authenticated to sensitive services. At minimum, implement domain-level
filtering in the relay to prevent access to financial, email, and identity
provider domains.

**R-BRW-04: CDP Port Security**
Port 9222 must not be exposed to any network interface. If CDP is required,
bind it to localhost only and implement authentication. The sandbox
Dockerfile must be updated to remove the port exposure or add firewall
rules.

### 8.4 Browser Restrictions — Short-Term (P1)

**R-BRW-05: JavaScript Evaluation Restrictions**
Implement a policy engine for JavaScript evaluation that:
- Blocks access to `document.cookie`
- Blocks access to `localStorage` and `sessionStorage`
- Blocks access to `XMLHttpRequest` and `fetch` for cross-origin requests
- Blocks access to `window.open`
- Logs all JavaScript evaluation for audit

**R-BRW-06: Form Submission Controls**
All form submissions must be intercepted, classified, and either blocked or
presented to the user for approval. At minimum, require user approval for:
- Any form on a financial domain
- Any form containing password fields
- Any form with a payment method field
- Any form that triggers a POST request to a different domain

**R-BRW-07: Download Restrictions**
Implement file download controls:
- Limit download to specific file types needed for the current task
- Block executable downloads (.exe, .msi, .dmg, .sh, .bat, .cmd)
- Require user approval for downloads above a size threshold
- Sandbox downloaded files in a quarantine directory

**R-BRW-08: Separate Browser Profile**
The AI must operate in an isolated browser profile that does not contain:
- Saved passwords
- Persistent cookies from the user's browsing
- Browser history
- Bookmarks
- Auto-fill data
If the AI needs to access a specific authenticated service, implement a
single-sign-on flow that grants time-limited, scope-limited access to
that specific service only.

### 8.5 Combined Capability Controls (P0)

**R-CMB-01: Action Correlation Monitoring**
Implement monitoring that detects when the AI is combining channel
impersonation with browser actions — for example, when the AI reads data
from a browser and then discloses it via a messaging channel. These
combined actions should require explicit user authorization.

**R-CMB-02: Information Flow Controls**
Data read from the browser must not flow to messaging channels without
user review. Implement information flow labels that track data provenance
and block cross-boundary flows:
- Browser-sourced data cannot be sent via messaging without approval
- Messaging-received data cannot be used to drive browser actions without
  approval
- Financial data from any source cannot be transmitted via any channel
  without approval

**R-CMB-03: Human-in-the-Loop for Sensitive Chains**
Any action chain that involves both reading private data and communicating
externally must have a mandatory human-in-the-loop checkpoint. The AI must
present the user with a summary of what it intends to do — what data it
read, what it plans to say, to whom, and via which channel — and receive
explicit approval before proceeding.

---

## 9. Summary of Findings

OpenClaw's combination of channel impersonation and browser control creates an
AI agent that can:

1. **Read** anything the user can see in their browser
2. **Decide** what to do based on that information
3. **Act** on any website the user is authenticated to
4. **Communicate** the results to anyone in the user's contact list
5. **Do all of this** while appearing to be the user at every step

No other consumer-facing software provides this combination of capabilities with
so few restrictions. The navigation guard is minimal. The disclosure mechanism is
absent. The authentication boundary between the AI and the user's online identity
is effectively nonexistent.

The recommendations in this document are not enhancements — they are prerequisites
for responsible deployment. Without mandatory AI disclosure, domain-level browser
restrictions, cookie isolation, and human-in-the-loop controls for sensitive action
chains, OpenClaw presents an unacceptable risk to user privacy, financial security,
personal relationships, and legal standing.

---

*OpenClaw Security Study — Phase 6 (Channel Impersonation) & Phase 7 (Browser/Computer-Use)*
*Document 10 of the analysis series*
