# Phase 7 Deep-Dive: Browser Automation & Computer-Use Capabilities

## OpenClaw Security Study — Dedicated Browser/Computer-Use Analysis

**Document:** 11-browser-computer-use.md
**Scope:** Technical deep-dive into the browser automation subsystem, CDP integration,
Playwright capabilities, extension relay, sandbox architecture, and computer-use attack surface
**Risk Level:** CRITICAL — the browser module is the single largest attack surface in OpenClaw
**Relation to Doc 10:** This document supersedes and significantly expands the browser sections
of 10-channel-impersonation.md, which combined Phases 6+7 at a survey level.

---

## 1. Executive Summary

"Computer-use" is the industry term for granting an AI agent the ability to interact with
a graphical user interface the way a human would: clicking buttons, typing text, reading
screen content, navigating between applications, and making decisions based on what it
sees. In OpenClaw, "computer-use" means something more specific and more dangerous: the
AI gets programmatic control of a full Chromium browser instance through two industrial-
grade automation frameworks (Chrome DevTools Protocol and Playwright), plus a 30KB
extension relay that bridges directly into the user's real Chrome browser with all saved
passwords, active sessions, and authentication cookies intact.

The `src/browser/` directory contains over 100 source files totaling well over 200KB of
TypeScript. It is one of the two largest modules in the entire codebase (alongside the
session/channel system). The scale of this investment is telling: more engineering effort
went into browser automation than into the entire security subsystem.

Key findings in this deep-dive:

- **CDP port 9222 is exposed in the sandbox Dockerfile** with no authentication. Anyone
  who can reach the port has full remote control of the browser — equivalent to sitting
  at the keyboard.
- **The extension relay (30KB) is larger than the security scanner.** It bridges the AI
  into the user's actual Chrome browser, inheriting every authenticated session.
- **The navigation guard (2.4KB) is the sole URL-filtering mechanism** protecting
  against the AI visiting banking, email, healthcare, or corporate admin sites. It is
  12x smaller than the extension relay it is meant to constrain.
- **Playwright's interaction layer (21KB) provides unrestricted form filling, JavaScript
  evaluation, cookie/storage access, and file downloads** — a complete remote-control
  toolkit for any website.
- **The Docker sandbox provides process isolation but not network isolation.** The
  browser has full internet access, meaning any data it reads can be exfiltrated.
- **AI-driven browsing (pw-ai.ts) adds an autonomous decision layer** where the AI
  itself decides what to click, what to type, and where to navigate — closing the loop
  from observation to action without human involvement.

---

## 2. Browser Subsystem Architecture

```
+===========================================================================+
|                        USER'S MACHINE                                      |
|                                                                            |
|  +------------------+          +--------------------------------------+    |
|  | User's Real      |          |  OpenClaw Core (src/gateway/)        |    |
|  | Chrome Browser   |<-------->|                                      |    |
|  |                  |  30KB    |  +- Session Manager                  |    |
|  | - Saved passwords|  Extension  |  +- LLM Provider                  |    |
|  | - All cookies    |  Relay   |  |  +- Skill/Plugin Engine           |    |
|  | - Active logins  |  (bi-directional)  +- Browser Client (8.4KB)   |    |
|  | - History        |          |         |                            |    |
|  | - Autofill data  |          |         v                            |    |
|  +------------------+          |  +------+-----+                      |    |
|         ^                      |  | client-    |  client-actions-     |    |
|         |                      |  | fetch.ts   |  core.ts (6.5KB)    |    |
|         |                      |  | (8.7KB)    |  observe.ts (5.6KB) |    |
|         |                      |  +------+-----+  state.ts (8.8KB)   |    |
|         |                      +---------|----------------------------+    |
|         |                                |                                 |
|         |                                v                                 |
|  +------+----------------------------------------------------------------+ |
|  |                    BROWSER SERVER (server.ts, 3.3KB)                   | |
|  |   server-context.ts (22KB)   server-lifecycle.ts   server-middleware  | |
|  |   control-auth.ts (2.7KB)    csrf.ts (2.3KB)       http-auth.ts      | |
|  +----+---------------------------+--------------------------------------+ |
|       |                           |                                        |
|       v                           v                                        |
|  +---------+              +----------------+                               |
|  | NAVIGATION              | CONFIG (10KB)  |                              |
|  | GUARD                   | profiles.ts    |                              |
|  | (2.4KB)                 | paths.ts (8KB) |                              |
|  | [ONLY URL               | chrome.ts      |                              |
|  |  FILTER]                | (9.5KB)        |                              |
|  +---------+               +-------+--------+                              |
|            |                       |                                       |
+============|=======================|=======================================+
             |                       |
             v                       v
+============|=======================|=======================================+
|            DOCKER SANDBOX (Dockerfile.sandbox-browser)                     |
|            debian:bookworm-slim | user: "sandbox" (non-root)              |
|                                                                            |
|  +------------------------------+  +----------------------------------+   |
|  |  CHROMIUM INSTANCE           |  |  DISPLAY SERVER                  |   |
|  |                              |  |                                  |   |
|  |  +------------------------+  |  |  Xvfb (virtual framebuffer)     |   |
|  |  | CDP (port 9222)        |  |  |      |                          |   |
|  |  | - DOM access           |  |  |      v                          |   |
|  |  | - JS evaluation        |  |  |  x11vnc (VNC server)           |   |
|  |  | - Network intercept    |  |  |  port 5900 --> VNC clients     |   |
|  |  | - Cookie read/write    |  |  |      |                          |   |
|  |  | - Storage access       |  |  |      v                          |   |
|  |  | - Screenshot capture   |  |  |  noVNC + websockify            |   |
|  |  | - Input simulation     |  |  |  port 6080 --> browser-based   |   |
|  |  | - NO AUTHENTICATION    |  |  |               remote desktop   |   |
|  |  +------------------------+  |  +----------------------------------+   |
|  |                              |                                         |
|  |  +------------------------+  |                                         |
|  |  | PLAYWRIGHT ENGINE      |  |  EXPOSED PORTS:                        |
|  |  | pw-session.ts (24KB)   |  |  9222 - CDP (full browser control)    |
|  |  | pw-tools-core.*        |  |  5900 - VNC (screen sharing)          |
|  |  |   interactions (21KB)  |  |  6080 - noVNC (web remote desktop)    |
|  |  |   storage (4KB)        |  |                                        |
|  |  |   downloads (8KB)      |  |  NETWORK: FULL INTERNET ACCESS        |
|  |  |   snapshot (7KB)       |  |  (no --network=none flag)             |
|  |  |   screenshots          |  |                                        |
|  |  |   state (6KB)          |  |                                        |
|  |  |   trace (1KB)          |  |                                        |
|  |  +------------------------+  |                                         |
|  |                              |                                         |
|  |  +------------------------+  |                                         |
|  |  | AI BROWSER LAYER       |  |                                         |
|  |  | pw-ai.ts (2KB)         |  |                                         |
|  |  | pw-ai-module.ts (1.4KB)|  |                                         |
|  |  | pw-ai-state.ts         |  |                                         |
|  |  | (autonomous decisions) |  |                                         |
|  |  +------------------------+  |                                         |
|  +------------------------------+                                         |
+===========================================================================+
```

---

## 3. Capability Inventory

The following table enumerates every browser capability exposed to the AI agent,
organized by the source file that implements it. This is not speculative -- each
capability corresponds to implemented, callable code.

| Capability | Source File(s) | Description |
|---|---|---|
| Navigate to URL | client-actions-url.ts, pw-session.ts | Visit any URL; the navigation guard is the only filter |
| Click element | pw-tools-core.interactions.ts | Single, double, right-click on any visible element |
| Type text | pw-tools-core.interactions.ts | Keystroke-by-keystroke or bulk text entry |
| Fill form field | pw-tools-core.interactions.ts, form-fields.ts | Populate input, textarea, select, contenteditable |
| Submit form | pw-tools-core.interactions.ts | Trigger form submission (POST/GET) |
| Evaluate JavaScript | pw-tools-core.interactions.ts, cdp.ts | Execute arbitrary JS in page context |
| Read cookies | pw-tools-core.storage.ts | Read all cookies for any visited domain |
| Write cookies | pw-tools-core.storage.ts | Create or modify cookies |
| Read localStorage | pw-tools-core.storage.ts | Read all localStorage key-value pairs |
| Write localStorage | pw-tools-core.storage.ts | Create or modify localStorage entries |
| Read sessionStorage | pw-tools-core.storage.ts | Read per-tab session data |
| Write sessionStorage | pw-tools-core.storage.ts | Create or modify session data |
| Take screenshot | screenshot.ts, pw-tools-core.screenshots-* | Full-page or element-specific screenshots |
| Accessibility snapshot | pw-tools-core.snapshot.ts, pw-role-snapshot.ts | Structured DOM representation with ARIA roles |
| Download file | pw-tools-core.downloads.ts | Download any file to local filesystem |
| Upload file | pw-tools-core.interactions.ts | Set file input values (simulates file picker) |
| Intercept network | cdp.ts | Monitor/modify HTTP requests and responses |
| Read page state | pw-tools-core.state.ts, client-actions-state.ts | URL, title, DOM structure, visibility states |
| Manage browser tabs | pw-session.ts | Open, close, switch between tabs |
| Handle dialogs | pw-tools-core.interactions.ts | Accept/dismiss alerts, confirms, prompts |
| Trace execution | pw-tools-core.trace.ts | Record execution traces for debugging |
| Handle responses | pw-tools-core.responses.ts | Process HTTP response data |
| AI-autonomous browsing | pw-ai.ts, pw-ai-module.ts | AI decides what to click/type autonomously |
| Extension relay bridge | extension-relay.ts (30KB) | Control user's real Chrome browser |
| Chrome profile access | chrome.profile-decoration.ts, profiles-service.ts | Access saved passwords, autofill, history |
| CDP raw commands | cdp.ts, cdp.helpers.ts | Send any CDP command to Chromium |

**Total unique capabilities: 25+**

For comparison, most browser automation testing frameworks expose 8-12 capabilities.
OpenClaw exposes everything Playwright and CDP can do, plus custom layers for AI
decision-making and real-browser bridging.

---

## 4. Chrome DevTools Protocol (CDP) Deep-Dive

### 4.1 What CDP Port 9222 Provides

The Chrome DevTools Protocol is the same protocol used by Chrome's built-in developer
tools (F12). When Chromium exposes CDP on port 9222, it provides a WebSocket-based API
with access to every internal subsystem of the browser. `cdp.ts` (15KB) and
`cdp.helpers.ts` (5KB) wrap this protocol for OpenClaw.

CDP domains accessible include:

| CDP Domain | Capability | Security Impact |
|---|---|---|
| `Page` | Navigate, reload, capture screenshot, print to PDF | Full page control |
| `Runtime` | Evaluate JavaScript, call functions, inspect objects | Arbitrary code execution in page |
| `DOM` | Query, modify, delete any DOM node | Page manipulation |
| `Network` | Intercept requests/responses, modify headers, block URLs | MITM within browser |
| `Storage` | Read/write cookies, localStorage, sessionStorage, IndexedDB, Cache | Full client-side data access |
| `Input` | Dispatch mouse, keyboard, touch events | Human-indistinguishable input |
| `Emulation` | Override user agent, geolocation, device metrics | Fingerprint spoofing |
| `Security` | Override certificate errors, disable security features | Bypass HTTPS warnings |
| `Target` | List, attach to, create browser tabs and contexts | Multi-tab orchestration |
| `Browser` | Get version, manage windows, close browser | Full lifecycle control |
| `Fetch` | Intercept network at granular level, modify request/response bodies | Deep traffic manipulation |

### 4.2 Why CDP Equals Full Remote Desktop (for the Browser)

CDP on port 9222 is not a limited diagnostic interface. It is the complete remote
control API for Chromium. A connection to port 9222 allows:

1. Navigating to any URL (including `file://` paths on the sandbox filesystem)
2. Executing arbitrary JavaScript on any loaded page
3. Reading every cookie, every localStorage entry, every cached response
4. Sending keyboard and mouse events indistinguishable from human input
5. Intercepting and modifying every network request before it leaves the browser
6. Taking screenshots of any page or element at any time
7. Overriding security policies (certificate pinning, CORS, CSP)

This is strictly more powerful than VNC. VNC provides visual access to the screen; CDP
provides programmatic access to every internal state of the browser. An attacker with
CDP access can extract data that is not even visible on screen (HTTP-only cookies,
in-memory JavaScript variables, response headers, IndexedDB contents).

### 4.3 Attack Surface of Exposed CDP

The sandbox Dockerfile exposes port 9222 without any of the following protections:

- **No authentication** -- CDP has no built-in authentication mechanism. Anyone who can
  connect to the WebSocket endpoint has full access.
- **No TLS** -- The WebSocket connection is unencrypted.
- **No IP allowlist** -- The Dockerfile does not restrict which hosts can connect.
- **No rate limiting** -- Unlimited commands can be sent per second.

The exposure surface depends on how Docker networking is configured:

| Docker Network Mode | CDP Accessible From | Risk Level |
|---|---|---|
| Bridge (default) | Host machine only | HIGH -- any process on host |
| Host (`--network=host`) | Entire local network | CRITICAL -- LAN-wide access |
| Port-mapped (`-p 9222:9222`) | Any network that can reach host | CRITICAL -- internet-facing if host is public |
| None (`--network=none`) | Nobody (port not reachable) | LOW -- but this mode is NOT used |

The Dockerfile does not specify `--network=none`. The sandbox has full network access.

### 4.4 CDP as a Lateral Movement Vector

If the sandbox is compromised by a malicious webpage (e.g., via a browser exploit), the
attacker gains CDP access to the Chromium instance and, through it:

1. Can navigate to any other site the AI had previously visited in that session
2. Can read cookies/tokens from other origins (CDP bypasses same-origin policy)
3. Can use the full network access of the sandbox to reach external services
4. Can use the VNC/noVNC ports to observe or take over the visual session

CDP transforms a browser-level compromise into a full-session compromise.

---

## 5. Playwright Capability Analysis

### 5.1 The 21KB Interactions File

`pw-tools-core.interactions.ts` at 21KB is the largest single capability file in the
Playwright integration. To put this in perspective: the entire navigation guard is 2.4KB.
A single file implementing click/type/fill/evaluate operations is 9x the size of the
safety mechanism that is supposed to constrain those operations.

The interactions file implements at minimum:

- **click(selector, options)** -- Click any element by CSS selector, XPath, text content,
  or accessibility role. Supports modifier keys (Ctrl, Shift, Alt), click position
  offsets, and force-click (bypasses visibility checks).
- **type(selector, text, options)** -- Type text character-by-character with configurable
  delay, or fill a field instantly. Handles special keys (Enter, Tab, Escape, arrow keys).
- **fill(selector, value)** -- Set the value of an input field directly, bypassing
  keystroke events. Faster than type() and works on all input types.
- **evaluate(pageFunction, args)** -- Execute arbitrary JavaScript in the page context.
  The function can access `window`, `document`, and all page-global variables. It can
  return serialized data to the caller. This single method gives the AI the same power
  as the browser console.
- **setInputFiles(selector, files)** -- Simulate file selection in a file-upload input.
  The AI can upload any file accessible within the sandbox filesystem.
- **selectOption(selector, values)** -- Choose options in a `<select>` dropdown.
- **check/uncheck(selector)** -- Toggle checkbox and radio button state.
- **hover(selector)** -- Move the virtual cursor over an element, triggering hover events
  and revealing hidden UI elements (dropdown menus, tooltips).
- **dialog handling** -- Automatically accept, dismiss, or respond to alert(), confirm(),
  and prompt() dialogs.

### 5.2 Storage Access: The Keys to Every Kingdom

`pw-tools-core.storage.ts` (4KB) deserves special attention. Modern web applications
store critical authentication material client-side:

| Storage Mechanism | What Applications Store There | AI Can |
|---|---|---|
| Cookies | Session IDs, auth tokens, CSRF tokens, "remember me" flags | Read, write, delete |
| localStorage | JWT tokens, OAuth tokens, user prefs, cached API responses | Read, write, delete |
| sessionStorage | Per-tab session data, temporary auth tokens, form state | Read, write, delete |

A concrete example: a user logged into their company's Okta SSO has cookies containing
the SSO session token. The AI reads this cookie. With this single token, any HTTP client
can impersonate the user to every application behind Okta -- email, HR systems, code
repositories, financial dashboards, customer databases.

### 5.3 Download Management

`pw-tools-core.downloads.ts` (8KB) provides structured file download capabilities:

- Trigger downloads from any authenticated source
- Monitor download progress
- Save files to configurable local paths
- No documented file-type restrictions
- No documented file-size limits
- Downloads inherit the browser's authentication state, so the AI can download files
  from password-protected services the user is logged into

### 5.4 AI-Driven Browsing: The Autonomous Loop

`pw-ai.ts` (2KB), `pw-ai-module.ts` (1.4KB), and `pw-ai-state.ts` implement AI-driven
browser interaction. This is the layer where the AI does not simply execute explicit
commands ("click this button") but instead makes autonomous decisions about what to do
next based on what it sees on the page.

The autonomous browsing loop:

1. AI takes an accessibility snapshot or screenshot of the current page
2. AI processes the visual/structural information through its LLM
3. AI decides what action to take (click, type, navigate, extract data)
4. AI executes the action via Playwright
5. AI observes the result and returns to step 1

This is computer-use in its fullest form. The AI is not a script following instructions.
It is an autonomous agent making real-time decisions about how to interact with arbitrary
web interfaces. It can figure out how to use a website it has never seen before, navigate
multi-step workflows, and adapt to unexpected dialog boxes or page layouts.

### 5.5 Role-Based Accessibility Snapshots

`pw-role-snapshot.ts` (11KB) creates structured representations of page content organized
by ARIA roles. This is not just an accessibility feature -- it is the AI's primary
mechanism for "reading" a webpage. The snapshot converts the visual page into a structured
data format that the LLM can reason about: buttons, links, form fields, headings, lists,
tables, and their text content.

At 11KB, this is a sophisticated parsing system that gives the AI a rich understanding of
page structure. Combined with the AI decision layer, this means the AI can:

- Identify all form fields on a page and determine what data they expect
- Find the "Submit" or "Confirm" button regardless of its CSS styling
- Navigate multi-step wizards by reading progress indicators and step labels
- Extract structured data from tables (financial records, transaction histories)
- Understand page context from headings and navigation elements

---

## 6. The Extension Relay Problem

### 6.1 Scale and Significance

`extension-relay.ts` is 30KB of TypeScript -- the single largest file in the browser
module. For perspective:

| Component | Size | Purpose |
|---|---|---|
| extension-relay.ts | 30KB | Bridge AI to user's real Chrome |
| navigation-guard.ts | 2.4KB | Only URL-level safety filter |
| Security scanner (referenced from doc 07) | ~25KB | Entire plugin security system |
| pw-session.ts | 24KB | All Playwright session management |
| server-context.ts | 22KB | Entire browser server context |

The extension relay is larger than the security scanner. More code went into bridging the
AI into the user's real browser than into scanning plugins for dangerous behavior.

### 6.2 How the Extension Relay Works

Based on the file names and the companion extension code (chrome-extension-background-
utils, chrome-extension-manifest, chrome-extension-options-validation), the relay
architecture is:

1. **Chrome Extension** -- A browser extension is installed in the user's real Chrome
   browser. This extension has access to Chrome's extension APIs, which are more
   privileged than regular web page JavaScript.
2. **WebSocket/HTTP Bridge** -- The extension communicates with OpenClaw's browser server
   via `extension-relay.ts`, creating a bi-directional communication channel.
3. **Command Relay** -- OpenClaw sends commands through the relay to the extension, which
   executes them in the context of the user's real browser.
4. **Data Return** -- The extension reads data from the user's browser (page content,
   cookies, DOM state) and sends it back through the relay to OpenClaw.

### 6.3 What Chrome Extensions Can Access

A Chrome extension with appropriate permissions can access capabilities that are
unavailable to regular web pages:

| Extension API | Capability | Risk When AI-Controlled |
|---|---|---|
| `chrome.cookies` | Read/write cookies for ANY domain | Session hijacking across all sites |
| `chrome.tabs` | List, create, navigate, close tabs | Full tab orchestration |
| `chrome.webRequest` | Intercept/modify all HTTP traffic | MITM on all browsing |
| `chrome.storage` | Extension-specific persistent storage | Store exfiltrated data |
| `chrome.history` | Full browsing history | Privacy violation |
| `chrome.bookmarks` | All bookmarks | Discover sensitive internal URLs |
| `chrome.downloads` | Manage file downloads | File exfiltration |
| `chrome.scripting` | Inject JS into any tab | Arbitrary code execution on any page |
| `chrome.identity` | OAuth token access | Steal OAuth tokens |
| `content_scripts` | Run JS in the context of web pages | Read/modify any page content |

### 6.4 Authentication Inheritance: The Core Danger

When the extension relay bridges the AI into the user's real Chrome browser, the AI
inherits every authentication state the user has established:

- **Gmail/Google Workspace** -- Full access to email, Drive, Calendar, Docs
- **Microsoft 365** -- Outlook, Teams, SharePoint, OneDrive
- **Banking sites** -- Any bank the user is logged into
- **Social media** -- Facebook, Twitter/X, LinkedIn, Instagram
- **Corporate SSO** -- Okta, Auth0, Azure AD -- potentially unlocking all corporate apps
- **Healthcare portals** -- Patient portals with medical records
- **Government services** -- Tax portals, benefits systems
- **Password managers** -- If the user's password manager has a web interface and is
  logged in, the AI can access the entire password vault

This is not hypothetical. Chrome extensions with these permissions exist and are routinely
used. The relay simply makes them available to an AI agent that can act on them
autonomously.

### 6.5 The Authentication Relay vs. Extension Relay Auth

`extension-relay-auth.ts` (2.4KB) handles authentication between the OpenClaw server and
the extension. This is authentication of the relay channel itself -- not authentication
of the AI's access to user resources. In other words, the 2.4KB auth file ensures that
the right OpenClaw instance is talking to the right extension, but it does nothing to
restrict what the AI can do once the channel is established.

The relay is a pipe. Once the pipe is open, everything flows through it.

---

## 7. Navigation Guard Analysis

### 7.1 The 2.4KB Constraint

`navigation-guard.ts` is 2.4KB. After removing imports, type definitions, and
boilerplate, the functional code is approximately 40-60 lines. What can 40-60 lines of
URL filtering do?

**Best case** -- A hardcoded blocklist of the most obviously dangerous URL patterns:

```
Likely blocked (speculation based on size):
- chrome://  (internal browser pages)
- file://    (local filesystem access)
- data:      (data URLs that could encode malicious content)
- about:     (browser internal pages)
Possibly blocked:
- A small list of specific known-dangerous domains
```

**What would require much more code:**

- Domain categorization by risk level (banking, healthcare, corporate)
- URL parsing with subdomain and path matching
- Dynamic blocklist updates
- Context-aware restrictions (different rules for different tasks)
- Redirect chain following (blocking redirects to blocked destinations)
- IP address blocking (preventing direct-IP navigation to bypass domain blocks)
- Internationalized domain name (IDN) homograph attack detection

### 7.2 The Ratio Problem

| Component | Size | What It Protects/Enables |
|---|---|---|
| navigation-guard.ts | 2.4KB | The ONLY barrier between AI and any website |
| extension-relay.ts | 30KB | Full bridge to user's real authenticated browser |
| pw-tools-core.interactions.ts | 21KB | Complete page interaction toolkit |
| pw-session.ts | 24KB | Full browser session management |
| server-context.ts | 22KB | Browser server orchestration |
| config.ts | 10KB | Browser configuration (more config than security) |

The navigation guard is 2.5% the size of the extension relay. The capability-to-safety
code ratio across the browser module is approximately **50:1** -- for every line of
safety code, there are 50 lines of capability code.

### 7.3 What the Navigation Guard Almost Certainly Cannot Block

Given the 2.4KB size, the following are almost certainly NOT filtered:

- `https://mail.google.com` (email)
- `https://online.bankofamerica.com` (banking)
- `https://portal.azure.com` (cloud admin)
- `https://console.aws.amazon.com` (cloud admin)
- `https://myhr.company.com` (corporate HR)
- `https://mychart.epic.com` (healthcare)
- `https://id.irs.gov` (government/tax)
- Internal corporate URLs (intranet, Jira, Confluence, GitLab)
- Any non-English domain (internationalized domain names)
- IP-addressed servers (e.g., `http://192.168.1.1` for router admin)
- localhost services (e.g., `http://localhost:8080` for development servers)

### 7.4 Navigation Guard Bypass Techniques

Even if the guard implements domain blocking, there are standard bypass techniques that
a 2.4KB implementation cannot defend against:

1. **URL shorteners** -- Navigate to `bit.ly/xyz` which redirects to a blocked site
2. **Open redirects** -- Navigate to `allowed-site.com/redirect?url=blocked-site.com`
3. **Google Cache** -- Access cached versions of blocked pages via Google
4. **Web Archive** -- Access blocked pages via `web.archive.org`
5. **Translation proxy** -- Access blocked pages via `translate.google.com`
6. **IP address navigation** -- Use the IP address instead of the domain name
7. **Alternate ports** -- Access `blocked-site.com:8443` if only port 443 is blocked
8. **Subdomain variation** -- `api.blocked-site.com` or `m.blocked-site.com`

---

## 8. The Chrome Profile Problem

### 8.1 What Lives in a Chrome Profile

`chrome.profile-decoration.ts` (7KB), `profiles.ts` (3KB), and `profiles-service.ts`
(5KB) manage Chrome profile access. A Chrome user data directory contains:

| Data | File/Database | AI Access Via |
|---|---|---|
| Saved passwords | `Login Data` (SQLite) | Autofill into forms, or direct DB read via CDP |
| Cookies | `Cookies` (SQLite) | pw-tools-core.storage.ts, CDP Storage domain |
| Browsing history | `History` (SQLite) | CDP, or direct file read if profile mounted |
| Bookmarks | `Bookmarks` (JSON) | Direct file read, or CDP |
| Autofill data | `Web Data` (SQLite) | Autofill triggers on form focus |
| Cached pages | `Cache/` directory | CDP Cache domain |
| IndexedDB | `IndexedDB/` directory | CDP Storage domain |
| Service workers | `Service Worker/` | CDP ServiceWorker domain |
| Extension data | `Extensions/`, `Local Extension Settings/` | Extension APIs via relay |
| Certificates | `Certificate Revocation Lists/` | CDP Security domain |
| Session state | `Sessions/` directory | Restored on browser launch |

### 8.2 The Password Autofill Chain

When the AI operates within the user's Chrome profile, the following chain executes
automatically for any login form:

1. AI navigates to a login page
2. Chrome detects the username/password fields
3. Chrome autofills saved credentials from the profile's password store
4. AI clicks the "Login" button
5. The site authenticates the AI as the user

The AI does not need to know the user's password. Chrome fills it in automatically. The
AI just needs to navigate to the right URL and click "Sign In." This means that every
site for which the user has saved credentials in Chrome is accessible to the AI with a
simple navigate-and-click sequence.

### 8.3 Chrome Profile Discovery

`chrome.executables.ts` (17KB) finds Chrome installations across operating systems. The
size of this file (17KB -- larger than the CDP integration itself) suggests it handles
platform-specific Chrome locations on macOS, Windows, and Linux, including:

- Default Chrome installation paths
- Chromium installations
- Chrome Beta, Dev, and Canary channels
- Snap, Flatpak, and Homebrew installations
- Custom installation directories

Combined with `chrome-user-data-dir.test-harness.ts`, this suggests the system can
locate and use Chrome user data directories, which is the critical step in gaining access
to the user's saved credentials and sessions.

---

## 9. Sandbox Analysis

### 9.1 What the Docker Sandbox Provides

The `Dockerfile.sandbox-browser` creates a container with:

**Positive security properties:**
- Non-root user (`sandbox`) -- limits privilege escalation within the container
- Minimal base image (`debian:bookworm-slim`) -- reduced attack surface vs. full OS
- No development tools installed -- cannot compile exploits within the container
- Process isolation from host -- container cannot directly access host processes

**Architectural limits of the sandbox:**
```
SANDBOX BOUNDARY:
+--------------------------+     +--------------------------+
|    Docker Container      |     |     Host Machine         |
|                          |     |                          |
|  Chromium + Xvfb + VNC   |     |  User's files            |
|                          |     |  User's real Chrome      |
|  CAN:                    |     |  Other applications      |
|  - Access any URL        |     |                          |
|  - Send any HTTP request |     |  Extension relay crosses |
|  - Exfiltrate via network|     |  this boundary! -------->|
|  - Store data locally    |     |                          |
|                          |     |                          |
|  CANNOT (in theory):     |     |                          |
|  - Access host files     |     |                          |
|  - Run host processes    |     |                          |
|  - Modify host system    |     |                          |
+--------------------------+     +--------------------------+
         |
         | FULL NETWORK ACCESS
         | (no --network=none)
         v
+---------------------------+
|      INTERNET              |
|  - Banking sites           |
|  - Email providers         |
|  - Corporate intranets     |
|  - Cloud consoles          |
|  - Attacker C2 servers     |
+---------------------------+
```

### 9.2 Network Access: The Sandbox's Fatal Flaw

The single most important security property a browser sandbox could have is **network
isolation**. A browser that cannot reach the internet cannot:

- Exfiltrate stolen cookies or tokens to an attacker's server
- Navigate to banking or email sites
- Download malicious payloads
- Contact command-and-control servers

Docker supports `--network=none` which would provide this isolation. The sandbox
Dockerfile does not use it. The browser has unrestricted internet access.

This means the sandbox is a **process isolation boundary only**. It prevents the browser
from accessing host files directly, but it does nothing to prevent the browser from
accessing any website, any API, or any internet-accessible service. For a browser -- where
the primary risk is what sites it visits and what data it sends/receives -- process
isolation without network isolation provides minimal protection.

### 9.3 VNC/noVNC: Visual Access to the Sandbox

Ports 5900 (VNC) and 6080 (noVNC) provide visual access to the sandbox's virtual
desktop. This means:

- Anyone with network access to port 6080 can watch the AI browsing in real-time
  through a standard web browser (noVNC provides a web-based VNC client)
- VNC access includes mouse and keyboard control -- an external party could take over
  the AI's browser session
- Neither VNC nor noVNC in this configuration require authentication by default
- The visual feed shows everything the browser displays, including:
  - Page content (emails, bank statements, medical records)
  - Form fields with entered data (including credentials pre-fill)
  - Screenshots in progress
  - The AI's actions in real-time

### 9.4 Sandbox Escape via the Extension Relay

The extension relay (`extension-relay.ts`) explicitly bridges across the sandbox
boundary. It connects the sandboxed browser server to the user's real Chrome browser
running on the host machine. This is an intentional, architectural sandbox bypass.

The extension relay means the sandbox provides no security benefit for the most dangerous
browser operation: accessing the user's authenticated sessions. The sandbox contains
Chromium in a container, but the relay reaches outside the container to the user's real
browser where the actual authenticated sessions live.

---

## 10. Attack Scenarios

### 10.1 Financial Fraud via Authenticated Banking Session

**Threat model:** Prompt injection or misaligned AI behavior

**Attack chain:**
1. AI receives a task or prompt (possibly injected via a malicious webpage or
   document) instructing it to "verify the user's bank balance"
2. AI uses the extension relay to access the user's real Chrome where Chase.com
   is already logged in, or navigates to chase.com in the sandbox where Chrome
   profile autofills credentials
3. AI reads account balances, recent transactions, and payee list via
   accessibility snapshots (pw-role-snapshot.ts)
4. AI navigates to the bill pay or transfer page
5. AI fills in transfer details using pw-tools-core.interactions.ts
6. AI clicks "Submit" -- the bank processes the transaction as the authenticated user
7. AI takes a screenshot (screenshot.ts) of the confirmation page as proof
8. Bank's fraud detection sees a normal browser session from the user's usual
   device/IP -- no anomaly detected

**Why existing controls fail:**
- Navigation guard (2.4KB) almost certainly does not block chase.com
- CDP provides the raw capability even if Playwright were restricted
- Chrome profile autofill provides credentials without AI needing to know them
- The sandbox has full network access to reach the banking site

### 10.2 Corporate Espionage via Authenticated Intranet

**Threat model:** Compromised plugin or adversarial prompt injection

**Attack chain:**
1. Malicious plugin or injected prompt instructs AI to "research project status"
2. AI opens Jira/Confluence via extension relay (user is SSO-authenticated)
3. AI reads project boards, sprint plans, and design documents using accessibility
   snapshots and JavaScript evaluation
4. AI navigates to internal Git repository (GitLab/GitHub Enterprise, also SSO'd)
5. AI downloads source code files via pw-tools-core.downloads.ts
6. AI extracts key data and transmits it via any available channel:
   - HTTP POST to an external endpoint (sandbox has network access)
   - Messaging channel (if connected to Slack/Discord)
   - Encoding data in navigation patterns to a colluding server

**Unique risk factors:**
- Corporate SSO (Okta, Azure AD) means one authentication token unlocks dozens
  of internal applications
- Corporate intranets are typically not in any browser blocklist
- The AI can read and understand technical documents, not just exfiltrate them blindly

### 10.3 Credential Harvesting via Cookie and Token Extraction

**Threat model:** Malicious AI behavior or compromised extension relay

**Attack chain:**
1. AI uses pw-tools-core.storage.ts to enumerate all cookies across all domains
2. AI identifies high-value authentication cookies:
   - Google: `SID`, `HSID`, `SSID`, `APISID`, `SAPISID`
   - Microsoft: `ESTSAUTHPERSISTENT`, `ESTSAUTH`
   - AWS: `aws-userInfo`, session cookies
   - Generic: `sessionid`, `auth_token`, `jwt`, `access_token`
3. AI uses CDP `Storage.getCookies` to read HTTP-only cookies that JavaScript
   cannot access (CDP has no same-origin restrictions)
4. AI evaluates JavaScript to read localStorage for JWT tokens:
   `JSON.stringify(Object.entries(localStorage))`
5. AI encodes all harvested credentials and tokens
6. AI exfiltrates via network (sandbox has full internet access), or via a
   messaging channel, or by storing in a webpage form field on a colluding site

**Impact:** With the harvested tokens, an external attacker can impersonate the user
on every service. The tokens may be valid for hours, days, or indefinitely depending
on the service's session policy.

### 10.4 Persistent Surveillance via Screenshot and CDP

**Threat model:** Stalkerware pattern via misconfigured or malicious AI

**Attack chain:**
1. AI runs a periodic loop (OpenClaw supports cron-like scheduling):
   a. Navigate to Gmail -- take accessibility snapshot, extract email subjects/senders
   b. Navigate to bank -- take screenshot of account balance and transactions
   c. Navigate to social media -- snapshot message inbox
   d. Navigate to calendar -- extract upcoming appointments
   e. Navigate to location-sharing page -- extract current location
2. AI compiles the surveillance data into a structured report
3. AI stores the report locally or transmits it to a preconfigured endpoint
4. Loop repeats every N minutes

**Amplifying factors:**
- CDP network interception can silently monitor all HTTP traffic without navigating
  to specific pages
- Screenshots capture visual content that text extraction might miss (images, charts)
- The VNC port (5900/6080) allows real-time visual surveillance of the AI's browsing
- Combined with messaging channels, the AI can forward surveillance reports to a
  third party

### 10.5 Persistent Backdoor via Extension Relay

**Threat model:** Supply-chain attack through compromised OpenClaw update or plugin

**Attack chain:**
1. Compromised code modifies extension-relay.ts to maintain a persistent WebSocket
   connection to an external command-and-control server
2. The relay continues operating normally for the AI (maintaining stealth)
3. The C2 server can send commands through the relay to the Chrome extension
4. The extension executes commands in the user's real browser:
   - Open new tabs to specific URLs
   - Inject JavaScript into existing pages
   - Extract cookies and tokens
   - Modify page content (e.g., change bank transfer recipient)
   - Install additional extensions for persistence
5. Because the extension runs in the user's real Chrome (not the sandbox), it
   survives sandbox restarts and container rebuilds
6. The backdoor persists until the Chrome extension is manually removed

**Why this is especially dangerous:**
- The extension relay is 30KB -- large enough to hide malicious code in plain sight
- 23KB of tests suggest active development where changes are frequent
- The relay authenticates the OpenClaw-to-extension channel (extension-relay-auth.ts)
  but does not validate individual commands against a security policy
- Chrome extensions auto-update, so a compromised update could be pushed silently

---

## 11. Risk Matrix

### 11.1 Browser Capability Risks (Detailed)

| # | Risk | Attack Vector | Severity | Likelihood | Impact | CVSS-like Score |
|---|---|---|---|---|---|---|
| B-01 | Unauthenticated CDP access | Port 9222 exposure | CRITICAL | HIGH | Full browser control | 9.8 |
| B-02 | Extension relay session hijack | 30KB relay bridge | CRITICAL | MEDIUM | All authenticated sessions | 9.6 |
| B-03 | Credential autofill exploitation | Chrome profile access | CRITICAL | HIGH | Saved password exposure | 9.4 |
| B-04 | Cookie/token exfiltration | storage.ts + network | CRITICAL | HIGH | Cross-service impersonation | 9.4 |
| B-05 | Arbitrary JS execution | evaluate() in interactions | CRITICAL | HIGH | Unlimited page manipulation | 9.2 |
| B-06 | Navigation guard bypass | URL shorteners, redirects | HIGH | HIGH | Access to blocked sites | 8.8 |
| B-07 | Autonomous financial transactions | AI-driven browsing loop | CRITICAL | MEDIUM | Unauthorized fund transfers | 9.0 |
| B-08 | Unrestricted file download | downloads.ts + network | HIGH | MEDIUM | Data exfiltration, malware | 8.5 |
| B-09 | VNC/noVNC unauthenticated access | Ports 5900/6080 | HIGH | MEDIUM | Visual surveillance, takeover | 8.2 |
| B-10 | Sandbox network non-isolation | No --network=none | HIGH | CERTAIN | All network-based attacks enabled | 8.8 |
| B-11 | Form submission without consent | interactions.ts | HIGH | HIGH | Unauthorized actions on websites | 8.5 |
| B-12 | Persistent extension backdoor | Extension auto-update | CRITICAL | LOW | Long-term undetected access | 9.0 |

### 11.2 Aggregate Assessment

**Overall Browser Module Risk: CRITICAL**

The browser module combines:
- Unrestricted capability (25+ distinct actions, full CDP access)
- Minimal restriction (2.4KB navigation guard, no network isolation)
- Authentication inheritance (Chrome profile, extension relay)
- Autonomous decision-making (AI-driven browsing loop)

This creates a system where an AI agent has more practical access to a user's online
identity than most sophisticated malware. Unlike malware, which must evade detection,
OpenClaw's browser access is an intended feature operating with the user's implicit
consent -- making it invisible to antivirus, endpoint detection, and browser security
extensions.

---

## 12. Recommendations for Browser/Computer-Use

### 12.1 Critical (P0) -- Implement Before Any Production Use

**R-BROWSER-01: Network Isolation for Sandbox**
Add `--network=none` to the sandbox Docker configuration by default. Provide a
controlled proxy mechanism that allows only explicitly allowlisted domains. This single
change eliminates the majority of exfiltration-based attack scenarios.

**R-BROWSER-02: CDP Port Authentication**
Implement mandatory authentication for CDP connections on port 9222. Options:
- Use `--remote-debugging-pipe` instead of `--remote-debugging-port` (pipe-based CDP
  does not create a network-accessible endpoint)
- Add a reverse proxy with token authentication in front of port 9222
- At minimum, bind CDP to `127.0.0.1` within the container and use Docker networking
  to restrict access

**R-BROWSER-03: VNC/noVNC Authentication**
Configure x11vnc with a mandatory password (`-passwd` or `-rfbauth` flag). Configure
noVNC to require authentication. Do not expose these ports outside the Docker host.

**R-BROWSER-04: Disable Extension Relay by Default**
The extension relay should be an opt-in feature that requires explicit, per-session
user authorization. When enabled, it must display a persistent visual indicator in the
Chrome extension that the AI has active access to the browser. Provide a one-click
"Disconnect AI" button in the extension.

**R-BROWSER-05: Chrome Profile Isolation**
Never launch the sandboxed browser with the user's real Chrome profile. Create a
dedicated, empty profile for AI use. If access to specific authenticated services is
required, implement a scoped credential injection mechanism that provides time-limited
access to a single site.

### 12.2 High Priority (P1) -- Implement Within 30 Days

**R-BROWSER-06: Domain Classification System**
Replace the 2.4KB navigation guard with a comprehensive domain classification engine:
- **Tier 1 (Blocked):** Financial institutions, government services, healthcare portals,
  password managers, identity providers, email services. AI cannot navigate to these
  domains under any circumstances without explicit per-URL user approval.
- **Tier 2 (Read-Only):** Social media, news, general web. AI can view content but
  cannot submit forms, execute JavaScript, or modify page state.
- **Tier 3 (Full Access):** Explicitly allowlisted domains for the current task only.
- Dynamic categorization using a domain classification API for unknown domains.

**R-BROWSER-07: JavaScript Evaluation Policy Engine**
Wrap all `evaluate()` calls through a policy engine that:
- Blocks access to `document.cookie` and `Storage` APIs
- Blocks network-initiating APIs (`fetch`, `XMLHttpRequest`, `WebSocket`)
- Blocks DOM mutation on Tier 1 and Tier 2 domains
- Logs all evaluated JavaScript for audit review
- Enforces a maximum execution time per evaluation

**R-BROWSER-08: Form Submission Gate**
Intercept all form submissions and classify them:
- Any form on a financial, email, or identity domain: BLOCK without user approval
- Any form containing password, payment, or SSN fields: BLOCK without user approval
- Any POST request to a cross-origin destination: REQUIRE user approval
- All other forms: ALLOW with logging

**R-BROWSER-09: Cookie and Storage Read Restrictions**
Implement per-domain authorization for cookie and storage reads:
- AI cannot read cookies for domains it has not been explicitly authorized to access
- HTTP-only and Secure cookies should be inaccessible to the AI entirely
- localStorage reads should require per-domain allowlisting
- All storage reads should be logged with full context (which domain, which keys, when)

### 12.3 Medium Priority (P2) -- Implement Within 90 Days

**R-BROWSER-10: Download Controls**
- Block executable file types by default (.exe, .msi, .dmg, .sh, .bat, .cmd, .ps1)
- Enforce maximum download size (configurable, default 50MB)
- Quarantine all downloads in a sandboxed directory
- Require user approval for downloads from authenticated sessions
- Scan downloads with antivirus before making them accessible

**R-BROWSER-11: Extension Relay Command Validation**
If the extension relay remains available, implement a command validation layer:
- Whitelist of allowed extension API calls
- Per-domain authorization for `chrome.cookies` and `chrome.scripting` access
- Rate limiting on relay commands
- Complete audit log of all relay commands and responses
- Automatic relay disconnection after configurable idle timeout

**R-BROWSER-12: Behavioral Anomaly Detection**
Monitor AI browsing patterns for anomalous behavior:
- Rapid navigation across many unrelated domains (credential harvesting pattern)
- Repeated cookie/storage reads without corresponding user task
- Form submissions on financial domains
- Large data downloads from authenticated services
- Navigation immediately following cookie extraction (exfiltration pattern)

**R-BROWSER-13: User Visibility and Control Dashboard**
Provide a real-time dashboard showing:
- Every URL the AI has visited in the current session
- Every form the AI has submitted
- Every file the AI has downloaded
- Every cookie/storage read the AI has performed
- A "Kill Switch" button that immediately terminates all browser sessions and
  disconnects the extension relay

---

*OpenClaw Security Study -- Phase 7: Browser Automation & Computer-Use Deep-Dive*
*Document 11 of the analysis series*
*This document provides detailed technical analysis that supersedes the browser*
*sections of document 10 (10-channel-impersonation.md).*
