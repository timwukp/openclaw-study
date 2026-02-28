# OpenClaw Ecosystem

## Core Components

| Component | Repo | Description |
|---|---|---|
| OpenClaw Core | `openclaw/openclaw` | Main platform (237K stars) |
| ClawHub | `openclaw/clawhub` | Skill directory/marketplace |
| Website | `openclaw/openclaw.ai` | Official website |
| Nix Package | `openclaw/nix-openclaw` | Nix packaging |
| Ansible Deploy | `openclaw/openclaw-ansible` | Automated deployment |

## Community Projects (Notable)

### Lightweight Alternatives
| Project | Stars | Description |
|---|---|---|
| nanobot (HKUDS) | 26.5K | Ultra-lightweight OpenClaw |
| ClawWork (HKUDS) | 5.7K | OpenClaw as AI coworker |

### Skills & Plugins
| Project | Stars | Description |
|---|---|---|
| awesome-openclaw-skills | 22K | Curated skill collection |
| awesome-openclaw-usecases | 11.8K | Community use cases |
| memU (NevaMind-AI) | 11.2K | Memory for proactive agents |
| awesome-openclaw | 701 | Resource/tool/tutorial list |

### Deployment
| Project | Stars | Description |
|---|---|---|
| moltworker (Cloudflare) | 9.3K | Run on Cloudflare Workers |
| OpenClawInstaller | 2.1K | One-click deploy tool |
| OpenClaw-Docker-CN-IM | 2.2K | Docker with Chinese IM plugins |
| 1Panel | 33.7K | VPS management with OpenClaw |

### Infrastructure
| Project | Stars | Description |
|---|---|---|
| ClawRouter (BlockRunAI) | 3.7K | Smart LLM router |
| openclaw-studio | 707 | Web dashboard |
| openclaw-mission-control | 1.1K | Agent orchestration dashboard |
| secure-openclaw (Composio) | 1.4K | Security-hardened version |

### Localization
| Project | Stars | Description |
|---|---|---|
| OpenClawChineseTranslation | 1.7K | Chinese localization |
| openclaw-china | 1K | Chinese IM plugins (Feishu, DingTalk, QQ) |
| openclaw-wechat | 921 | WeChat integration |
| openclaw101 | 473 | Chinese tutorial site |

### Novel Applications
| Project | Stars | Description |
|---|---|---|
| Clawra (SumeLabs) | 1.8K | AI girlfriend persona |
| antfarm (snarktank) | 1.7K | Multi-agent team builder |
| MedgeClaw | 533 | Medical AI integration |

## MCP Integration

OpenClaw integrates MCP via **mcporter** (external bridge):
- GitHub: `steipete/mcporter`
- Add/remove MCP servers without restarting gateway
- Keeps MCP churn decoupled from core stability
- Preferred over building first-class MCP into core

## Plugin Distribution

- Preferred path: npm packages + local extension loading for development
- Plugin SDK: `src/plugin-sdk/`
- Plugin docs: `docs/tools/plugin.md`
- Community plugins: https://docs.openclaw.ai/plugins/community
- ClawHub for skill discovery: clawhub.ai

## Deployment Options

| Platform | Method |
|---|---|
| Docker | `docker-compose.yml` (primary) |
| Podman | `setup-podman.sh` |
| Fly.io | `fly.toml` |
| Render | `render.yaml` |
| Cloudflare Workers | Via `moltworker` |
| Ansible | Via `openclaw-ansible` |
| Nix | Via `nix-openclaw` |
| 1Panel | One-click VPS deploy |
