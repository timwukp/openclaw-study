> 本文为中文翻译版本。英文原版为权威版本,如有差异以英文版为准。
> [English Version](../11-browser-computer-use.md)

# 第7阶段深入分析:浏览器自动化与计算机使用能力

## OpenClaw安全研究——专门的浏览器/计算机使用分析

**文档:** 11-browser-computer-use.md
**范围:** 浏览器自动化子系统、CDP集成、Playwright能力、扩展中继、沙箱架构和计算机使用攻击面的技术深入分析
**风险级别:** 严重 (CRITICAL) ——浏览器模块是OpenClaw中最大的攻击面
**与文档10的关系:** 本文档取代并显著扩展了10-channel-impersonation.md的浏览器部分,该文档在概览层面结合了第6和第7阶段。

---

## 1. 执行摘要

"计算机使用"(Computer-use)是业界术语,指授予AI代理与图形用户界面交互的能力,就像人类一样:点击按钮、输入文本、阅读屏幕内容、在应用程序之间导航,并根据所见内容做出决策。在OpenClaw中,"计算机使用"意味着更具体、更危险的东西:AI通过两个工业级自动化框架(Chrome开发者工具协议和Playwright)获得对完整Chromium浏览器实例的程序化控制,外加一个30KB的扩展中继,直接桥接到用户的真实Chrome浏览器,其中包含所有保存的密码、活跃会话和完整的身份验证cookie。

`src/browser/`目录包含超过100个源文件,总计超过200KB的TypeScript代码。它是整个代码库中两个最大的模块之一(与会话/渠道系统并列)。这种投入的规模很能说明问题:投入到浏览器自动化的工程努力超过了整个安全子系统。

本深入分析的主要发现:

- **CDP端口9222在沙箱Dockerfile中暴露**,没有身份验证。任何能够访问该端口的人都可以完全远程控制浏览器——相当于坐在键盘前。
- **扩展中继(30KB)比安全扫描器更大。** 它将AI桥接到用户的真实Chrome浏览器,继承每一个已认证的会话。
- **导航守卫(2.4KB)是唯一的URL过滤机制**,保护免于AI访问银行、电子邮件、医疗保健或企业管理网站。它比它要约束的扩展中继小12倍。
- **Playwright的交互层(21KB)提供无限制的表单填充、JavaScript执行、cookie/存储访问和文件下载**——任何网站的完整远程控制工具包。
- **Docker沙箱提供进程隔离但不提供网络隔离。** 浏览器拥有完整的互联网访问,这意味着它读取的任何数据都可以被泄露。
- **AI驱动的浏览(pw-ai.ts)添加了一个自主决策层**,其中AI本身决定点击什么、输入什么以及导航到哪里——在没有人工参与的情况下从观察到行动完成闭环。

---

## 2. 浏览器子系统架构

```
+===========================================================================+
|                        用户机器                                            |
|                                                                            |
|  +------------------+          +--------------------------------------+    |
|  | 用户的真实       |          |  OpenClaw核心 (src/gateway/)        |    |
|  | Chrome浏览器     |<-------->|                                      |    |
|  |                  |  30KB    |  +- Session Manager                  |    |
|  | - 保存的密码     |  扩展    |  +- LLM Provider                     |    |
|  | - 所有cookie     |  中继    |  |  +- Skill/Plugin Engine           |    |
|  | - 活跃登录状态   |  (双向)  |  |  +- Browser Client (8.4KB)        |    |
|  | - 历史记录       |          |  |      |                            |    |
|  | - 自动填充数据   |          |  |      v                            |    |
|  +------------------+          |  +------+-----+                      |    |
|         ^                      |  | client-    |  client-actions-     |    |
|         |                      |  | fetch.ts   |  core.ts (6.5KB)    |    |
|         |                      |  | (8.7KB)    |  observe.ts (5.6KB) |    |
|         |                      |  +------+-----+  state.ts (8.8KB)   |    |
|         |                      +---------|----------------------------+    |
|         |                                |                                 |
|         |                                v                                 |
|  +------+----------------------------------------------------------------+ |
|  |                    浏览器服务器 (server.ts, 3.3KB)                     | |
|  |   server-context.ts (22KB)   server-lifecycle.ts   server-middleware  | |
|  |   control-auth.ts (2.7KB)    csrf.ts (2.3KB)       http-auth.ts      | |
|  +----+---------------------------+--------------------------------------+ |
|       |                           |                                        |
|       v                           v                                        |
|  +---------+              +----------------+                               |
|  | 导航守卫                | CONFIG (10KB)  |                              |
|  | (2.4KB)                 | profiles.ts    |                              |
|  | [唯一的URL              | paths.ts (8KB) |                              |
|  |  过滤器]                | chrome.ts      |                              |
|  +---------+               | (9.5KB)        |                              |
|  |                         +-------+--------+                              |
|            |                       |                                       |
+============|=======================|=======================================+
             |                       |
             v                       v
+============|=======================|=======================================+
|            DOCKER沙箱 (Dockerfile.sandbox-browser)                        |
|            debian:bookworm-slim | 用户: "sandbox" (非root)               |
|                                                                            |
|  +------------------------------+  +----------------------------------+   |
|  |  CHROMIUM实例                |  |  显示服务器                      |   |
|  |                              |  |                                  |   |
|  |  +------------------------+  |  |  Xvfb (虚拟帧缓冲)              |   |
|  |  | CDP (端口9222)         |  |  |      |                          |   |
|  |  | - DOM访问              |  |  |      v                          |   |
|  |  | - JS执行               |  |  |  x11vnc (VNC服务器)            |   |
|  |  | - 网络拦截             |  |  |  端口5900 --> VNC客户端        |   |
|  |  | - Cookie读写           |  |  |      |                          |   |
|  |  | - 存储访问             |  |  |      v                          |   |
|  |  | - 截图捕获             |  |  |  noVNC + websockify            |   |
|  |  | - 输入模拟             |  |  |  端口6080 --> 基于浏览器的     |   |
|  |  | - 无身份验证           |  |  |               远程桌面         |   |
|  |  +------------------------+  |  +----------------------------------+   |
|  |                              |                                         |
|  |  +------------------------+  |                                         |
|  |  | PLAYWRIGHT引擎         |  |  暴露的端口:                           |
|  |  | pw-session.ts (24KB)   |  |  9222 - CDP (完整浏览器控制)          |
|  |  | pw-tools-core.*        |  |  5900 - VNC (屏幕共享)                |
|  |  |   interactions (21KB)  |  |  6080 - noVNC (Web远程桌面)           |
|  |  |   storage (4KB)        |  |                                        |
|  |  |   downloads (8KB)      |  |  网络: 完整互联网访问                  |
|  |  |   snapshot (7KB)       |  |  (无 --network=none 标志)             |
|  |  |   screenshots          |  |                                        |
|  |  |   state (6KB)          |  |                                        |
|  |  |   trace (1KB)          |  |                                        |
|  |  +------------------------+  |                                         |
|  |                              |                                         |
|  |  +------------------------+  |                                         |
|  |  | AI浏览器层             |  |                                         |
|  |  | pw-ai.ts (2KB)         |  |                                         |
|  |  | pw-ai-module.ts (1.4KB)|  |                                         |
|  |  | pw-ai-state.ts         |  |                                         |
|  |  | (自主决策)             |  |                                         |
|  |  +------------------------+  |                                         |
|  +------------------------------+                                         |
+===========================================================================+
```

---

## 3. 能力清单

以下表格列举了暴露给AI代理的每一项浏览器能力,按实现它的源文件组织。这不是推测——每项能力都对应已实现的、可调用的代码。

| 能力 | 源文件 | 描述 |
|---|---|---|
| 导航到URL | client-actions-url.ts, pw-session.ts | 访问任何URL;导航守卫是唯一的过滤器 |
| 点击元素 | pw-tools-core.interactions.ts | 在任何可见元素上单击、双击、右键点击 |
| 输入文本 | pw-tools-core.interactions.ts | 逐个按键或批量文本输入 |
| 填充表单字段 | pw-tools-core.interactions.ts, form-fields.ts | 填充input、textarea、select、contenteditable |
| 提交表单 | pw-tools-core.interactions.ts | 触发表单提交(POST/GET) |
| 执行JavaScript | pw-tools-core.interactions.ts, cdp.ts | 在页面上下文中执行任意JS |
| 读取cookie | pw-tools-core.storage.ts | 读取任何访问域的所有cookie |
| 写入cookie | pw-tools-core.storage.ts | 创建或修改cookie |
| 读取localStorage | pw-tools-core.storage.ts | 读取所有localStorage键值对 |
| 写入localStorage | pw-tools-core.storage.ts | 创建或修改localStorage条目 |
| 读取sessionStorage | pw-tools-core.storage.ts | 读取每个标签页的会话数据 |
| 写入sessionStorage | pw-tools-core.storage.ts | 创建或修改会话数据 |
| 截图 | screenshot.ts, pw-tools-core.screenshots-* | 全页或特定元素的截图 |
| 可访问性快照 | pw-tools-core.snapshot.ts, pw-role-snapshot.ts | 带有ARIA角色的结构化DOM表示 |
| 下载文件 | pw-tools-core.downloads.ts | 将任何文件下载到本地文件系统 |
| 上传文件 | pw-tools-core.interactions.ts | 设置文件输入值(模拟文件选择器) |
| 拦截网络 | cdp.ts | 监控/修改HTTP请求和响应 |
| 读取页面状态 | pw-tools-core.state.ts, client-actions-state.ts | URL、标题、DOM结构、可见性状态 |
| 管理浏览器标签页 | pw-session.ts | 打开、关闭、在标签页之间切换 |
| 处理对话框 | pw-tools-core.interactions.ts | 接受/取消警告、确认、提示 |
| 跟踪执行 | pw-tools-core.trace.ts | 记录执行跟踪以进行调试 |
| 处理响应 | pw-tools-core.responses.ts | 处理HTTP响应数据 |
| AI自主浏览 | pw-ai.ts, pw-ai-module.ts | AI自主决定点击/输入什么 |
| 扩展中继桥接 | extension-relay.ts (30KB) | 控制用户的真实Chrome浏览器 |
| Chrome配置文件访问 | chrome.profile-decoration.ts, profiles-service.ts | 访问保存的密码、自动填充、历史记录 |
| CDP原始命令 | cdp.ts, cdp.helpers.ts | 向Chromium发送任何CDP命令 |

**独特能力总数: 25+**

作为对比,大多数浏览器自动化测试框架暴露8-12项能力。OpenClaw暴露了Playwright和CDP能做的一切,外加用于AI决策和真实浏览器桥接的自定义层。

---

## 4. Chrome开发者工具协议(CDP)深入分析

### 4.1 CDP端口9222提供什么

Chrome开发者工具协议是Chrome内置开发者工具(F12)使用的同一协议。当Chromium在端口9222上暴露CDP时,它提供了一个基于WebSocket的API,可以访问浏览器的每个内部子系统。`cdp.ts`(15KB)和`cdp.helpers.ts`(5KB)为OpenClaw封装了这个协议。

可访问的CDP域包括:

| CDP域 | 能力 | 安全影响 |
|---|---|---|
| `Page` | 导航、重新加载、捕获截图、打印为PDF | 完整页面控制 |
| `Runtime` | 执行JavaScript、调用函数、检查对象 | 在页面中任意代码执行 |
| `DOM` | 查询、修改、删除任何DOM节点 | 页面操纵 |
| `Network` | 拦截请求/响应、修改标头、阻止URL | 浏览器内的中间人攻击 |
| `Storage` | 读/写cookie、localStorage、sessionStorage、IndexedDB、Cache | 完整的客户端数据访问 |
| `Input` | 分派鼠标、键盘、触摸事件 | 与人类无法区分的输入 |
| `Emulation` | 覆盖用户代理、地理位置、设备指标 | 指纹伪造 |
| `Security` | 覆盖证书错误、禁用安全功能 | 绕过HTTPS警告 |
| `Target` | 列出、附加到、创建浏览器标签页和上下文 | 多标签页编排 |
| `Browser` | 获取版本、管理窗口、关闭浏览器 | 完整的生命周期控制 |
| `Fetch` | 在粒度级别拦截网络、修改请求/响应正文 | 深度流量操纵 |

### 4.2 为什么CDP等同于完整远程桌面(对于浏览器)

端口9222上的CDP不是一个有限的诊断接口。它是Chromium的完整远程控制API。连接到端口9222允许:

1. 导航到任何URL(包括沙箱文件系统上的`file://`路径)
2. 在任何加载的页面上执行任意JavaScript
3. 读取每个cookie、每个localStorage条目、每个缓存的响应
4. 发送与人类输入无法区分的键盘和鼠标事件
5. 在每个网络请求离开浏览器之前拦截和修改它
6. 随时对任何页面或元素截图
7. 覆盖安全策略(证书固定、CORS、CSP)

这比VNC更强大。VNC提供对屏幕的可视访问;CDP提供对浏览器每个内部状态的程序化访问。拥有CDP访问权限的攻击者可以提取甚至在屏幕上不可见的数据(仅HTTP的cookie、内存中的JavaScript变量、响应标头、IndexedDB内容)。

### 4.3 暴露CDP的攻击面

沙箱Dockerfile暴露端口9222,没有以下任何保护:

- **无身份验证** -- CDP没有内置的身份验证机制。任何能够连接到WebSocket端点的人都拥有完整访问权限。
- **无TLS** -- WebSocket连接未加密。
- **无IP白名单** -- Dockerfile不限制哪些主机可以连接。
- **无速率限制** -- 每秒可以发送无限的命令。

暴露面取决于Docker网络的配置方式:

| Docker网络模式 | CDP可访问自 | 风险级别 |
|---|---|---|
| Bridge(默认) | 仅主机机器 | 高 (HIGH) -- 主机上的任何进程 |
| Host (`--network=host`) | 整个本地网络 | 严重 (CRITICAL) -- LAN范围的访问 |
| 端口映射 (`-p 9222:9222`) | 任何能访问主机的网络 | 严重 (CRITICAL) -- 如果主机是公共的则面向互联网 |
| None (`--network=none`) | 无人(端口不可达) | 低 (LOW) -- 但此模式未使用 |

Dockerfile没有指定`--network=none`。沙箱具有完整的网络访问权限。

### 4.4 CDP作为横向移动载体

如果沙箱被恶意网页(例如,通过浏览器漏洞利用)攻破,攻击者获得对Chromium实例的CDP访问权限,并通过它:

1. 可以导航到AI在该会话中之前访问过的任何其他站点
2. 可以从其他源读取cookie/令牌(CDP绕过同源策略)
3. 可以使用沙箱的完整网络访问权限来访问外部服务
4. 可以使用VNC/noVNC端口观察或接管可视会话

CDP将浏览器级别的攻破转变为完整会话的攻破。

---

## 5. Playwright能力分析

### 5.1 21KB的交互文件

`pw-tools-core.interactions.ts`有21KB,是Playwright集成中最大的单一能力文件。从角度来看:整个导航守卫只有2.4KB。实现点击/输入/填充/执行操作的单个文件是应该约束这些操作的安全机制的9倍大。

交互文件至少实现:

- **click(selector, options)** -- 通过CSS选择器、XPath、文本内容或可访问性角色点击任何元素。支持修饰键(Ctrl、Shift、Alt)、点击位置偏移和强制点击(绕过可见性检查)。
- **type(selector, text, options)** -- 逐个字符输入文本,带有可配置的延迟,或立即填充字段。处理特殊键(Enter、Tab、Escape、方向键)。
- **fill(selector, value)** -- 直接设置输入字段的值,绕过按键事件。比type()更快,适用于所有输入类型。
- **evaluate(pageFunction, args)** -- 在页面上下文中执行任意JavaScript。该函数可以访问`window`、`document`和所有页面全局变量。它可以将序列化的数据返回给调用者。这个单一方法赋予AI与浏览器控制台相同的权力。
- **setInputFiles(selector, files)** -- 在文件上传输入中模拟文件选择。AI可以上传沙箱文件系统中可访问的任何文件。
- **selectOption(selector, values)** -- 在`<select>`下拉列表中选择选项。
- **check/uncheck(selector)** -- 切换复选框和单选按钮状态。
- **hover(selector)** -- 将虚拟光标移动到元素上,触发悬停事件并显示隐藏的UI元素(下拉菜单、工具提示)。
- **对话框处理** -- 自动接受、取消或响应alert()、confirm()和prompt()对话框。

### 5.2 存储访问:每个王国的钥匙

`pw-tools-core.storage.ts`(4KB)值得特别关注。现代Web应用程序在客户端存储关键的身份验证材料:

| 存储机制 | 应用程序在那里存储什么 | AI能做什么 |
|---|---|---|
| Cookie | 会话ID、身份验证令牌、CSRF令牌、"记住我"标志 | 读取、写入、删除 |
| localStorage | JWT令牌、OAuth令牌、用户偏好、缓存的API响应 | 读取、写入、删除 |
| sessionStorage | 每个标签页的会话数据、临时身份验证令牌、表单状态 | 读取、写入、删除 |

一个具体的例子:登录到公司Okta SSO的用户拥有包含SSO会话令牌的cookie。AI读取此cookie。使用这个单一令牌,任何HTTP客户端都可以冒充该用户访问Okta后面的每个应用程序——电子邮件、HR系统、代码仓库、财务仪表板、客户数据库。

### 5.3 下载管理

`pw-tools-core.downloads.ts`(8KB)提供结构化的文件下载能力:

- 从任何经过身份验证的源触发下载
- 监控下载进度
- 将文件保存到可配置的本地路径
- 没有记录的文件类型限制
- 没有记录的文件大小限制
- 下载继承浏览器的身份验证状态,因此AI可以从用户登录的受密码保护的服务下载文件

### 5.4 AI驱动的浏览:自主循环

`pw-ai.ts`(2KB)、`pw-ai-module.ts`(1.4KB)和`pw-ai-state.ts`实现了AI驱动的浏览器交互。这是AI不仅执行明确命令("点击这个按钮")而是根据它在页面上看到的内容自主决定下一步做什么的层。

自主浏览循环:

1. AI对当前页面进行可访问性快照或截图
2. AI通过其LLM处理视觉/结构信息
3. AI决定采取什么行动(点击、输入、导航、提取数据)
4. AI通过Playwright执行操作
5. AI观察结果并返回步骤1

这是最完整形式的计算机使用。AI不是遵循指令的脚本。它是一个自主代理,实时决定如何与任意Web界面交互。它可以弄清楚如何使用它以前从未见过的网站,导航多步骤工作流程,并适应意外的对话框或页面布局。

### 5.5 基于角色的可访问性快照

`pw-role-snapshot.ts`(11KB)创建按ARIA角色组织的页面内容的结构化表示。这不仅仅是一个可访问性功能——它是AI"阅读"网页的主要机制。快照将可视页面转换为LLM可以推理的结构化数据格式:按钮、链接、表单字段、标题、列表、表格及其文本内容。

这是一个复杂的解析系统,有11KB,使AI能够丰富地理解页面结构。结合AI决策层,这意味着AI可以:

- 识别页面上的所有表单字段并确定它们期望什么数据
- 找到"提交"或"确认"按钮,无论其CSS样式如何
- 通过阅读进度指示器和步骤标签导航多步骤向导
- 从表格中提取结构化数据(财务记录、交易历史)
- 从标题和导航元素理解页面上下文

---

## 6. 扩展中继问题

### 6.1 规模和意义

`extension-relay.ts`是30KB的TypeScript——浏览器模块中最大的单个文件。从角度来看:

| 组件 | 大小 | 目的 |
|---|---|---|
| extension-relay.ts | 30KB | 将AI桥接到用户的真实Chrome |
| navigation-guard.ts | 2.4KB | 唯一的URL级安全过滤器 |
| 安全扫描器(来自文档07) | ~25KB | 整个插件安全系统 |
| pw-session.ts | 24KB | 所有Playwright会话管理 |
| server-context.ts | 22KB | 整个浏览器服务器上下文 |

扩展中继比安全扫描器更大。将AI桥接到用户真实浏览器的代码比扫描插件的危险行为的代码更多。

### 6.2 扩展中继如何工作

基于文件名和配套的扩展代码(chrome-extension-background-utils、chrome-extension-manifest、chrome-extension-options-validation),中继架构是:

1. **Chrome扩展** -- 在用户的真实Chrome浏览器中安装浏览器扩展。此扩展可以访问Chrome的扩展API,比常规网页JavaScript更具特权。
2. **WebSocket/HTTP桥接** -- 扩展通过`extension-relay.ts`与OpenClaw的浏览器服务器通信,创建双向通信通道。
3. **命令中继** -- OpenClaw通过中继将命令发送到扩展,扩展在用户真实浏览器的上下文中执行它们。
4. **数据返回** -- 扩展从用户的浏览器读取数据(页面内容、cookie、DOM状态)并通过中继发送回OpenClaw。

### 6.3 Chrome扩展可以访问什么

具有适当权限的Chrome扩展可以访问常规网页无法使用的能力:

| 扩展API | 能力 | AI控制时的风险 |
|---|---|---|
| `chrome.cookies` | 读/写任何域的cookie | 跨所有站点的会话劫持 |
| `chrome.tabs` | 列出、创建、导航、关闭标签页 | 完整的标签页编排 |
| `chrome.webRequest` | 拦截/修改所有HTTP流量 | 对所有浏览的中间人攻击 |
| `chrome.storage` | 扩展特定的持久存储 | 存储泄露的数据 |
| `chrome.history` | 完整的浏览历史记录 | 隐私侵犯 |
| `chrome.bookmarks` | 所有书签 | 发现敏感的内部URL |
| `chrome.downloads` | 管理文件下载 | 文件泄露 |
| `chrome.scripting` | 将JS注入任何标签页 | 在任何页面上任意代码执行 |
| `chrome.identity` | OAuth令牌访问 | 窃取OAuth令牌 |
| `content_scripts` | 在网页上下文中运行JS | 读取/修改任何页面内容 |

### 6.4 身份验证继承:核心危险

当扩展中继将AI桥接到用户的真实Chrome浏览器时,AI继承用户建立的每个身份验证状态:

- **Gmail/Google Workspace** -- 完全访问电子邮件、Drive、日历、文档
- **Microsoft 365** -- Outlook、Teams、SharePoint、OneDrive
- **银行网站** -- 用户登录的任何银行
- **社交媒体** -- Facebook、Twitter/X、LinkedIn、Instagram
- **企业SSO** -- Okta、Auth0、Azure AD——可能解锁所有企业应用程序
- **医疗保健门户** -- 带有医疗记录的患者门户
- **政府服务** -- 税务门户、福利系统
- **密码管理器** -- 如果用户的密码管理器有Web界面并且已登录,AI可以访问整个密码库

这不是假设的。具有这些权限的Chrome扩展存在并经常使用。中继只是使它们可供可以自主采取行动的AI代理使用。

### 6.5 身份验证中继与扩展中继身份验证

`extension-relay-auth.ts`(2.4KB)处理OpenClaw服务器和扩展之间的身份验证。这是中继通道本身的身份验证——而不是AI访问用户资源的身份验证。换句话说,2.4KB的身份验证文件确保正确的OpenClaw实例正在与正确的扩展通信,但一旦通道建立,它不会限制AI可以做什么。

中继是一个管道。一旦管道打开,一切都会流过它。

---

## 7. 导航守卫分析

### 7.1 2.4KB的约束

`navigation-guard.ts`是2.4KB。删除导入、类型定义和样板代码后,功能代码大约是40-60行。40-60行URL过滤能做什么?

**最好的情况** -- 最明显危险的URL模式的硬编码黑名单:

```
可能被阻止(推测基于大小):
- chrome://  (内部浏览器页面)
- file://    (本地文件系统访问)
- data:      (可以编码恶意内容的数据URL)
- about:     (浏览器内部页面)
可能被阻止:
- 特定已知危险域的小列表
```

**需要更多代码的内容:**

- 按风险级别进行域分类(银行、医疗保健、企业)
- 带有子域和路径匹配的URL解析
- 动态黑名单更新
- 上下文感知限制(不同任务的不同规则)
- 重定向链跟踪(阻止重定向到被阻止的目标)
- IP地址阻止(防止直接IP导航以绕过域阻止)
- 国际化域名(IDN)同形异义字攻击检测

### 7.2 比例问题

| 组件 | 大小 | 它保护/启用什么 |
|---|---|---|
| navigation-guard.ts | 2.4KB | AI和任何网站之间的唯一屏障 |
| extension-relay.ts | 30KB | 到用户真实认证浏览器的完整桥接 |
| pw-tools-core.interactions.ts | 21KB | 完整的页面交互工具包 |
| pw-session.ts | 24KB | 完整的浏览器会话管理 |
| server-context.ts | 22KB | 浏览器服务器编排 |
| config.ts | 10KB | 浏览器配置(配置多于安全) |

导航守卫是扩展中继大小的2.5%。整个浏览器模块的能力与安全代码比率约为**50:1**——每行安全代码对应50行能力代码。

### 7.3 导航守卫几乎肯定不能阻止什么

鉴于2.4KB的大小,以下内容几乎肯定没有被过滤:

- `https://mail.google.com`(电子邮件)
- `https://online.bankofamerica.com`(银行)
- `https://portal.azure.com`(云管理)
- `https://console.aws.amazon.com`(云管理)
- `https://myhr.company.com`(企业HR)
- `https://mychart.epic.com`(医疗保健)
- `https://id.irs.gov`(政府/税务)
- 内部企业URL(内联网、Jira、Confluence、GitLab)
- 任何非英语域(国际化域名)
- IP地址服务器(例如,`http://192.168.1.1`用于路由器管理)
- localhost服务(例如,`http://localhost:8080`用于开发服务器)

### 7.4 导航守卫绕过技术

即使守卫实现域阻止,也有2.4KB实现无法防御的标准绕过技术:

1. **URL缩短器** -- 导航到`bit.ly/xyz`,重定向到被阻止的站点
2. **开放重定向** -- 导航到`allowed-site.com/redirect?url=blocked-site.com`
3. **Google缓存** -- 通过Google访问被阻止页面的缓存版本
4. **Web存档** -- 通过`web.archive.org`访问被阻止的页面
5. **翻译代理** -- 通过`translate.google.com`访问被阻止的页面
6. **IP地址导航** -- 使用IP地址而不是域名
7. **备用端口** -- 如果只阻止端口443,则访问`blocked-site.com:8443`
8. **子域变化** -- `api.blocked-site.com`或`m.blocked-site.com`

---

## 8. Chrome配置文件问题

### 8.1 Chrome配置文件中有什么

`chrome.profile-decoration.ts`(7KB)、`profiles.ts`(3KB)和`profiles-service.ts`(5KB)管理Chrome配置文件访问。Chrome用户数据目录包含:

| 数据 | 文件/数据库 | AI访问方式 |
|---|---|---|
| 保存的密码 | `Login Data` (SQLite) | 自动填充到表单,或通过CDP直接读取数据库 |
| Cookie | `Cookies` (SQLite) | pw-tools-core.storage.ts, CDP Storage域 |
| 浏览历史 | `History` (SQLite) | CDP,或如果配置文件挂载则直接读取文件 |
| 书签 | `Bookmarks` (JSON) | 直接读取文件,或CDP |
| 自动填充数据 | `Web Data` (SQLite) | 表单焦点时自动填充触发 |
| 缓存页面 | `Cache/`目录 | CDP Cache域 |
| IndexedDB | `IndexedDB/`目录 | CDP Storage域 |
| Service workers | `Service Worker/` | CDP ServiceWorker域 |
| 扩展数据 | `Extensions/`, `Local Extension Settings/` | 通过中继的扩展API |
| 证书 | `Certificate Revocation Lists/` | CDP Security域 |
| 会话状态 | `Sessions/`目录 | 浏览器启动时恢复 |

### 8.2 密码自动填充链

当AI在用户的Chrome配置文件中操作时,任何登录表单都会自动执行以下链:

1. AI导航到登录页面
2. Chrome检测用户名/密码字段
3. Chrome从配置文件的密码存储中自动填充保存的凭据
4. AI点击"登录"按钮
5. 站点将AI验证为用户

AI不需要知道用户的密码。Chrome会自动填充它。AI只需要导航到正确的URL并点击"登录"。这意味着用户在Chrome中保存凭据的每个站点都可以通过简单的导航和点击序列访问AI。

### 8.3 Chrome配置文件发现

`chrome.executables.ts`(17KB)在各个操作系统中查找Chrome安装。这个文件的大小(17KB——比CDP集成本身更大)表明它处理macOS、Windows和Linux上特定于平台的Chrome位置,包括:

- 默认Chrome安装路径
- Chromium安装
- Chrome Beta、Dev和Canary通道
- Snap、Flatpak和Homebrew安装
- 自定义安装目录

结合`chrome-user-data-dir.test-harness.ts`,这表明系统可以定位和使用Chrome用户数据目录,这是获得对用户保存的凭据和会话的访问权限的关键步骤。

---

## 9. 沙箱分析

### 9.1 Docker沙箱提供什么

`Dockerfile.sandbox-browser`创建一个容器,具有:

**积极的安全属性:**
- 非root用户(`sandbox`)——限制容器内的权限升级
- 最小基础镜像(`debian:bookworm-slim`)——与完整操作系统相比减少攻击面
- 未安装开发工具——无法在容器内编译漏洞利用
- 与主机的进程隔离——容器无法直接访问主机进程

**沙箱的架构限制:**
```
沙箱边界:
+--------------------------+     +--------------------------+
|    Docker容器            |     |     主机机器             |
|                          |     |                          |
|  Chromium + Xvfb + VNC   |     |  用户的文件              |
|                          |     |  用户的真实Chrome        |
|  可以:                   |     |  其他应用程序            |
|  - 访问任何URL           |     |                          |
|  - 发送任何HTTP请求      |     |  扩展中继跨越            |
|  - 通过网络泄露          |     |  此边界! -------->      |
|  - 本地存储数据          |     |                          |
|                          |     |                          |
|  不能(理论上):           |     |                          |
|  - 访问主机文件          |     |                          |
|  - 运行主机进程          |     |                          |
|  - 修改主机系统          |     |                          |
+--------------------------+     +--------------------------+
         |
         | 完整网络访问
         | (无 --network=none)
         v
+---------------------------+
|      互联网                |
|  - 银行网站                |
|  - 电子邮件提供商          |
|  - 企业内联网              |
|  - 云控制台                |
|  - 攻击者C2服务器          |
+---------------------------+
```

### 9.2 网络访问:沙箱的致命缺陷

浏览器沙箱可以拥有的最重要的安全属性是**网络隔离**。无法访问互联网的浏览器不能:

- 将窃取的cookie或令牌泄露到攻击者的服务器
- 导航到银行或电子邮件网站
- 下载恶意有效负载
- 联系命令和控制服务器

Docker支持`--network=none`,它将提供这种隔离。沙箱Dockerfile不使用它。浏览器具有不受限制的互联网访问。

这意味着沙箱仅是**进程隔离边界**。它防止浏览器直接访问主机文件,但它不能防止浏览器访问任何网站、任何API或任何互联网可访问的服务。对于浏览器——主要风险是它访问什么站点以及它发送/接收什么数据——没有网络隔离的进程隔离提供最小的保护。

### 9.3 VNC/noVNC:对沙箱的可视访问

端口5900(VNC)和6080(noVNC)提供对沙箱虚拟桌面的可视访问。这意味着:

- 任何具有对端口6080的网络访问权限的人都可以通过标准Web浏览器实时观看AI浏览(noVNC提供基于Web的VNC客户端)
- VNC访问包括鼠标和键盘控制——外部方可以接管AI的浏览器会话
- 默认情况下,此配置中的VNC和noVNC都不需要身份验证
- 可视馈送显示浏览器显示的所有内容,包括:
  - 页面内容(电子邮件、银行对账单、医疗记录)
  - 带有输入数据的表单字段(包括凭据预填充)
  - 正在进行的截图
  - AI的实时操作

### 9.4 通过扩展中继逃离沙箱

扩展中继(`extension-relay.ts`)明确跨越沙箱边界。它将沙箱化的浏览器服务器连接到在主机机器上运行的用户真实Chrome浏览器。这是一个有意的、架构性的沙箱绕过。

扩展中继意味着沙箱对最危险的浏览器操作不提供安全益处:访问用户的认证会话。沙箱将Chromium包含在容器中,但中继到达容器外部,到达实际认证会话所在的用户真实浏览器。

---

## 10. 攻击场景

### 10.1 通过认证银行会话的金融欺诈

**威胁模型:** 提示注入或错位的AI行为

**攻击链:**
1. AI接收到一个任务或提示(可能通过恶意网页或文档注入),指示它"验证用户的银行余额"
2. AI使用扩展中继访问用户的真实Chrome,其中Chase.com已经登录,或在沙箱中导航到chase.com,Chrome配置文件自动填充凭据
3. AI通过可访问性快照(pw-role-snapshot.ts)读取账户余额、最近交易和收款人列表
4. AI导航到账单支付或转账页面
5. AI使用pw-tools-core.interactions.ts填写转账详细信息
6. AI点击"提交"——银行将交易作为认证用户处理
7. AI截图(screenshot.ts)确认页面作为证明
8. 银行的欺诈检测看到来自用户通常设备/IP的正常浏览器会话——未检测到异常

**为什么现有控制失败:**
- 导航守卫(2.4KB)几乎肯定不会阻止chase.com
- 即使Playwright受到限制,CDP也提供原始能力
- Chrome配置文件自动填充提供凭据,无需AI知道它们
- 沙箱具有完整的网络访问权限来访问银行网站

### 10.2 通过认证内联网的企业间谍活动

**威胁模型:** 受损插件或对抗性提示注入

**攻击链:**
1. 恶意插件或注入的提示指示AI"研究项目状态"
2. AI通过扩展中继打开Jira/Confluence(用户已通过SSO认证)
3. AI使用可访问性快照和JavaScript执行读取项目看板、冲刺计划和设计文档
4. AI导航到内部Git仓库(GitLab/GitHub Enterprise,也通过SSO)
5. AI通过pw-tools-core.downloads.ts下载源代码文件
6. AI提取关键数据并通过任何可用通道传输:
   - HTTP POST到外部端点(沙箱具有网络访问)
   - 消息通道(如果连接到Slack/Discord)
   - 将数据编码在导航模式中发送到串通的服务器

**独特的风险因素:**
- 企业SSO(Okta、Azure AD)意味着一个身份验证令牌解锁数十个内部应用程序
- 企业内联网通常不在任何浏览器黑名单中
- AI可以阅读和理解技术文档,而不仅仅是盲目地泄露它们

### 10.3 通过Cookie和令牌提取的凭据收集

**威胁模型:** 恶意AI行为或受损的扩展中继

**攻击链:**
1. AI使用pw-tools-core.storage.ts枚举所有域的所有cookie
2. AI识别高价值身份验证cookie:
   - Google: `SID`、`HSID`、`SSID`、`APISID`、`SAPISID`
   - Microsoft: `ESTSAUTHPERSISTENT`、`ESTSAUTH`
   - AWS: `aws-userInfo`、会话cookie
   - 通用: `sessionid`、`auth_token`、`jwt`、`access_token`
3. AI使用CDP `Storage.getCookies`读取JavaScript无法访问的仅HTTP cookie(CDP没有同源限制)
4. AI执行JavaScript读取localStorage中的JWT令牌:
   `JSON.stringify(Object.entries(localStorage))`
5. AI编码所有收集的凭据和令牌
6. AI通过网络泄露(沙箱具有完整的互联网访问),或通过消息通道,或通过在串通站点的网页表单字段中存储

**影响:** 使用收集的令牌,外部攻击者可以在每个服务上冒充用户。根据服务的会话策略,令牌可能在数小时、数天或无限期内有效。

### 10.4 通过截图和CDP的持久监视

**威胁模型:** 通过配置错误或恶意AI的跟踪软件模式

**攻击链:**
1. AI运行周期性循环(OpenClaw支持类似cron的调度):
   a. 导航到Gmail——进行可访问性快照,提取电子邮件主题/发件人
   b. 导航到银行——截图账户余额和交易
   c. 导航到社交媒体——快照消息收件箱
   d. 导航到日历——提取即将到来的约会
   e. 导航到位置共享页面——提取当前位置
2. AI将监视数据编译成结构化报告
3. AI将报告存储在本地或传输到预配置的端点
4. 循环每N分钟重复一次

**放大因素:**
- CDP网络拦截可以静默监控所有HTTP流量,而无需导航到特定页面
- 截图捕获文本提取可能错过的可视内容(图像、图表)
- VNC端口(5900/6080)允许实时可视监视AI的浏览
- 结合消息通道,AI可以将监视报告转发给第三方

### 10.5 通过扩展中继的持久后门

**威胁模型:** 通过受损的OpenClaw更新或插件的供应链攻击

**攻击链:**
1. 受损的代码修改extension-relay.ts以维护到外部命令和控制服务器的持久WebSocket连接
2. 中继继续为AI正常运行(保持隐蔽)
3. C2服务器可以通过中继向Chrome扩展发送命令
4. 扩展在用户的真实浏览器中执行命令:
   - 打开新标签页到特定URL
   - 将JavaScript注入现有页面
   - 提取cookie和令牌
   - 修改页面内容(例如,更改银行转账收款人)
   - 安装额外的扩展以实现持久性
5. 因为扩展在用户的真实Chrome中运行(而不是沙箱),它在沙箱重启和容器重建后仍然存在
6. 后门持续存在,直到手动删除Chrome扩展

**为什么这特别危险:**
- 扩展中继是30KB——足够大,可以隐藏恶意代码
- 23KB的测试表明活跃的开发,变化频繁
- 中继验证OpenClaw到扩展通道(extension-relay-auth.ts),但不根据安全策略验证单个命令
- Chrome扩展自动更新,因此可以静默推送受损的更新

---

## 11. 风险矩阵

### 11.1 浏览器能力风险(详细)

| # | 风险 | 攻击载体 | 严重性 | 可能性 | 影响 | 类似CVSS的分数 |
|---|---|---|---|---|---|---|
| B-01 | 未经身份验证的CDP访问 | 端口9222暴露 | 严重 (CRITICAL) | 高 (HIGH) | 完整浏览器控制 | 9.8 |
| B-02 | 扩展中继会话劫持 | 30KB中继桥接 | 严重 (CRITICAL) | 中等 (MEDIUM) | 所有认证会话 | 9.6 |
| B-03 | 凭据自动填充利用 | Chrome配置文件访问 | 严重 (CRITICAL) | 高 (HIGH) | 保存的密码暴露 | 9.4 |
| B-04 | Cookie/令牌泄露 | storage.ts + 网络 | 严重 (CRITICAL) | 高 (HIGH) | 跨服务冒充 | 9.4 |
| B-05 | 任意JS执行 | interactions中的evaluate() | 严重 (CRITICAL) | 高 (HIGH) | 无限页面操纵 | 9.2 |
| B-06 | 导航守卫绕过 | URL缩短器、重定向 | 高 (HIGH) | 高 (HIGH) | 访问被阻止的站点 | 8.8 |
| B-07 | 自主金融交易 | AI驱动的浏览循环 | 严重 (CRITICAL) | 中等 (MEDIUM) | 未经授权的资金转移 | 9.0 |
| B-08 | 无限制的文件下载 | downloads.ts + 网络 | 高 (HIGH) | 中等 (MEDIUM) | 数据泄露、恶意软件 | 8.5 |
| B-09 | VNC/noVNC未经身份验证的访问 | 端口5900/6080 | 高 (HIGH) | 中等 (MEDIUM) | 可视监视、接管 | 8.2 |
| B-10 | 沙箱网络非隔离 | 无 --network=none | 高 (HIGH) | 确定 (CERTAIN) | 所有基于网络的攻击启用 | 8.8 |
| B-11 | 未经同意的表单提交 | interactions.ts | 高 (HIGH) | 高 (HIGH) | 在网站上未经授权的操作 | 8.5 |
| B-12 | 持久扩展后门 | 扩展自动更新 | 严重 (CRITICAL) | 低 (LOW) | 长期未检测到的访问 | 9.0 |

### 11.2 总体评估

**浏览器模块总体风险: 严重 (CRITICAL)**

浏览器模块结合了:
- 不受限制的能力(25+不同操作,完整CDP访问)
- 最小限制(2.4KB导航守卫,无网络隔离)
- 身份验证继承(Chrome配置文件,扩展中继)
- 自主决策(AI驱动的浏览循环)

这创建了一个系统,其中AI代理对用户在线身份的实际访问权限超过大多数复杂的恶意软件。与必须逃避检测的恶意软件不同,OpenClaw的浏览器访问是一个旨在在用户隐式同意下运行的功能——使其对防病毒、端点检测和浏览器安全扩展不可见。

---

## 12. 浏览器/计算机使用建议

### 12.1 严重 (P0) ——在任何生产使用之前实施

**R-BROWSER-01: 沙箱的网络隔离**
默认情况下向沙箱Docker配置添加`--network=none`。提供一个受控的代理机制,只允许明确列入白名单的域。这一单一更改消除了大多数基于泄露的攻击场景。

**R-BROWSER-02: CDP端口身份验证**
为端口9222上的CDP连接实施强制性身份验证。选项:
- 使用`--remote-debugging-pipe`而不是`--remote-debugging-port`(基于管道的CDP不创建网络可访问的端点)
- 在端口9222前面添加带有令牌身份验证的反向代理
- 至少,将CDP绑定到容器内的`127.0.0.1`,并使用Docker网络限制访问

**R-BROWSER-03: VNC/noVNC身份验证**
使用强制性密码配置x11vnc(`-passwd`或`-rfbauth`标志)。配置noVNC需要身份验证。不要在Docker主机之外暴露这些端口。

**R-BROWSER-04: 默认禁用扩展中继**
扩展中继应该是一个选择加入的功能,需要明确的、每个会话的用户授权。启用后,它必须在Chrome扩展中显示一个持久的可视指示器,表明AI具有对浏览器的活动访问权限。在扩展中提供一个一键式"断开AI连接"按钮。

**R-BROWSER-05: Chrome配置文件隔离**
永远不要使用用户的真实Chrome配置文件启动沙箱化浏览器。为AI使用创建一个专用的空配置文件。如果需要访问特定的认证服务,实施一个范围受限的凭据注入机制,提供对单个站点的时间限制访问。

### 12.2 高优先级 (P1) ——在30天内实施

**R-BROWSER-06: 域分类系统**
用综合域分类引擎替换2.4KB导航守卫:
- **第1层(已阻止):** 金融机构、政府服务、医疗保健门户、密码管理器、身份提供者、电子邮件服务。在没有明确的每个URL用户批准的情况下,AI在任何情况下都不能导航到这些域。
- **第2层(只读):** 社交媒体、新闻、一般网络。AI可以查看内容,但不能提交表单、执行JavaScript或修改页面状态。
- **第3层(完全访问):** 仅当前任务的明确白名单域。
- 使用域分类API对未知域进行动态分类。

**R-BROWSER-07: JavaScript执行策略引擎**
通过策略引擎包装所有`evaluate()`调用:
- 阻止访问`document.cookie`和`Storage` API
- 阻止网络启动API(`fetch`、`XMLHttpRequest`、`WebSocket`)
- 阻止第1层和第2层域上的DOM突变
- 记录所有执行的JavaScript以进行审计审查
- 强制执行每次执行的最长执行时间

**R-BROWSER-08: 表单提交门**
拦截所有表单提交并对其进行分类:
- 金融、电子邮件或身份域上的任何表单:未经用户批准阻止
- 包含密码、付款或SSN字段的任何表单:未经用户批准阻止
- 到跨源目标的任何POST请求:需要用户批准
- 所有其他表单:允许并记录

**R-BROWSER-09: Cookie和存储读取限制**
对cookie和存储读取实施每个域授权:
- AI无法读取未明确授权访问的域的cookie
- 仅HTTP和Secure cookie应该对AI完全不可访问
- localStorage读取应该需要每个域白名单
- 所有存储读取都应该记录完整的上下文(哪个域、哪些键、何时)

### 12.3 中等优先级 (P2) ——在90天内实施

**R-BROWSER-10: 下载控制**
- 默认阻止可执行文件类型(.exe、.msi、.dmg、.sh、.bat、.cmd、.ps1)
- 强制执行最大下载大小(可配置,默认50MB)
- 在沙箱目录中隔离所有下载
- 从认证会话下载需要用户批准
- 在使下载可访问之前使用防病毒软件扫描下载

**R-BROWSER-11: 扩展中继命令验证**
如果扩展中继仍然可用,实施命令验证层:
- 允许的扩展API调用的白名单
- 对`chrome.cookies`和`chrome.scripting`访问的每个域授权
- 中继命令的速率限制
- 所有中继命令和响应的完整审计日志
- 在可配置的空闲超时后自动断开中继连接

**R-BROWSER-12: 行为异常检测**
监控AI浏览模式以检测异常行为:
- 在许多不相关的域之间快速导航(凭据收集模式)
- 在没有相应用户任务的情况下重复cookie/存储读取
- 在金融域上提交表单
- 从认证服务大量数据下载
- 在cookie提取后立即导航(泄露模式)

**R-BROWSER-13: 用户可见性和控制仪表板**
提供实时仪表板显示:
- AI在当前会话中访问的每个URL
- AI提交的每个表单
- AI下载的每个文件
- AI执行的每个cookie/存储读取
- 一个"终止开关"按钮,立即终止所有浏览器会话并断开扩展中继

---

*OpenClaw安全研究——第7阶段:浏览器自动化与计算机使用深入分析*
*分析系列的第11号文档*
*本文档提供了详细的技术分析,取代了文档10 (10-channel-impersonation.md)的浏览器部分。*
