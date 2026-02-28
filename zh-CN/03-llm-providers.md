> 本文为中文翻译版本。英文原版为权威版本，如有差异以英文版为准。
> [English Version](../03-llm-providers.md)

# OpenClaw LLM 提供商支持

## 支持的提供商

| 提供商 | 模型 | 认证方式 | 多密钥支持 |
|---|---|---|---|
| **Anthropic** | Claude 4.x、3.5 Sonnet/Haiku、3 Opus | `ANTHROPIC_API_KEY` | 是（`ANTHROPIC_API_KEYS=k1,k2`） |
| **OpenAI** | GPT-4o、GPT-4、GPT-3.5、o1、o3 | `OPENAI_API_KEY` | 是（`OPENAI_API_KEYS=k1,k2`） |
| **Google Gemini** | Gemini 2.x、1.5 Pro/Flash | `GEMINI_API_KEY` / `GOOGLE_API_KEY` | 是（`GEMINI_API_KEYS=k1,k2`） |
| **OpenRouter** | 通过统一 API 提供 100+ 模型 | `OPENROUTER_API_KEY` | 否 |
| **GitHub Copilot** | Copilot 可用模型 | OAuth 设备流（令牌轮换） | 不适用 |
| **Qwen Portal**（通义千问） | Qwen 模型（阿里云） | OAuth | 否 |
| **MiniMax** | MiniMax 模型 | `MINIMAX_API_KEY` | 否 |
| **ZAI** | ZAI 模型 | `ZAI_API_KEY` | 否 |
| **AI Gateway** | 多种模型 | `AI_GATEWAY_API_KEY` | 否 |
| **Kilocode** | Kilocode 模型 | 自定义认证 | 否 |
| **OpenAI 兼容端点** | Ollama、LM Studio、vLLM、LocalAI 等 | 自定义端点 + 密钥 | 否 |

## 路由功能

### 多密钥轮换（Multi-Key Rotation）
- 在多个密钥之间负载均衡 API 调用
- 示例：`OPENAI_API_KEYS=sk-key1,sk-key2,sk-key3`
- 有助于应对速率限制和分散成本

### 按渠道模型覆盖（Per-Channel Model Overrides）
- 将不同渠道路由到不同模型
- 示例：Telegram -> Claude、Discord -> GPT-4o、Slack -> Gemini
- 通过 `src/channels/model-overrides.ts` 进行配置

### 会话连续性（Session Continuity）
- 在同一对话线程中保持使用相同模型
- 防止对话中途出现令人困惑的模型切换
- 由 `src/routing/session-key.ts` 管理

### 账户绑定（Account Bindings）
- 将不同用户/账户映射到不同的提供商
- 由 `src/routing/bindings.ts` 和 `src/routing/account-id.ts` 管理

### 路由解析逻辑
- `src/routing/resolve-route.ts`（13KB，核心路由逻辑）
- 考虑因素：渠道配置、用户绑定、会话状态、密钥可用性
- 具有广泛的测试覆盖（21KB 测试代码）

## 外部路由（社区项目）

### ClawRouter（由 BlockRunAI 开发）
- 跨提供商的智能成本优化路由
- 支持微支付（稳定币/USDC）
- GitHub：`BlockRunAI/ClawRouter`

## 提供商相关的安全注意事项

- API 密钥存储在 `.env` 或 `~/.openclaw/.env` 中
- 密钥优先级：进程环境变量 > `./.env` > `~/.openclaw/.env` > `openclaw.json`
- `src/secrets/` 处理密钥管理
- `.detect-secrets.cfg` + `.secrets.baseline` 用于防止密钥泄露
- 密钥也可在 `openclaw.json` 配置中设置（安全性低于环境变量）
