# OpenClaw LLM Provider Support

## Supported Providers

| Provider | Models | Auth Method | Multi-Key |
|---|---|---|---|
| **Anthropic** | Claude 4.x, 3.5 Sonnet/Haiku, 3 Opus | `ANTHROPIC_API_KEY` | Yes (`ANTHROPIC_API_KEYS=k1,k2`) |
| **OpenAI** | GPT-4o, GPT-4, GPT-3.5, o1, o3 | `OPENAI_API_KEY` | Yes (`OPENAI_API_KEYS=k1,k2`) |
| **Google Gemini** | Gemini 2.x, 1.5 Pro/Flash | `GEMINI_API_KEY` / `GOOGLE_API_KEY` | Yes (`GEMINI_API_KEYS=k1,k2`) |
| **OpenRouter** | 100+ models via unified API | `OPENROUTER_API_KEY` | No |
| **GitHub Copilot** | Copilot-available models | OAuth device flow (token rotation) | N/A |
| **Qwen Portal** | Qwen models (Alibaba Cloud) | OAuth | No |
| **MiniMax** | MiniMax models | `MINIMAX_API_KEY` | No |
| **ZAI** | ZAI models | `ZAI_API_KEY` | No |
| **AI Gateway** | Various | `AI_GATEWAY_API_KEY` | No |
| **Kilocode** | Kilocode models | Custom auth | No |
| **OpenAI-compatible** | Ollama, LM Studio, vLLM, LocalAI, etc. | Custom endpoint + key | No |

## Routing Features

### Multi-Key Rotation
- Load-balance API calls across multiple keys
- Example: `OPENAI_API_KEYS=sk-key1,sk-key2,sk-key3`
- Helps with rate limiting and cost distribution

### Per-Channel Model Overrides
- Route different channels to different models
- Example: Telegram -> Claude, Discord -> GPT-4o, Slack -> Gemini
- Configured via `src/channels/model-overrides.ts`

### Session Continuity
- Keep the same model within a conversation thread
- Prevents confusing model switches mid-conversation
- Managed by `src/routing/session-key.ts`

### Account Bindings
- Map different users/accounts to different providers
- Managed by `src/routing/bindings.ts` and `src/routing/account-id.ts`

### Route Resolution Logic
- `src/routing/resolve-route.ts` (13KB, core routing logic)
- Considers: channel config, user bindings, session state, key availability
- Extensive test coverage (21KB of tests)

## External Routing (Community)

### ClawRouter (by BlockRunAI)
- Smart cost-optimized routing across providers
- Supports micropayments (stablecoin/USDC)
- GitHub: `BlockRunAI/ClawRouter`

## Security Considerations for Providers

- API keys stored in `.env` or `~/.openclaw/.env`
- Key precedence: process env > ./.env > ~/.openclaw/.env > openclaw.json
- `src/secrets/` handles secret management
- `.detect-secrets.cfg` + `.secrets.baseline` for leak prevention
- Keys can also be set in `openclaw.json` config (less secure than env vars)
