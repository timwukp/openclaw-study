> 本文为中文翻译版本。英文原版为权威版本，如有差异以英文版为准。
> [English Version](../04-ecosystem.md)

# OpenClaw 生态系统

## 核心组件

| 组件 | 仓库 | 说明 |
|---|---|---|
| OpenClaw 核心 | `openclaw/openclaw` | 主平台（237K stars） |
| ClawHub | `openclaw/clawhub` | 技能目录/市场 |
| 官方网站 | `openclaw/openclaw.ai` | 官方网站 |
| Nix 包 | `openclaw/nix-openclaw` | Nix 打包 |
| Ansible 部署 | `openclaw/openclaw-ansible` | 自动化部署 |

## 社区项目（知名项目）

### 轻量级替代方案
| 项目 | Stars | 说明 |
|---|---|---|
| nanobot (HKUDS) | 26.5K | 超轻量级 OpenClaw |
| ClawWork (HKUDS) | 5.7K | OpenClaw 作为 AI 协作者 |

### 技能与插件
| 项目 | Stars | 说明 |
|---|---|---|
| awesome-openclaw-skills | 22K | 精选技能合集 |
| awesome-openclaw-usecases | 11.8K | 社区使用案例 |
| memU (NevaMind-AI) | 11.2K | 面向主动式智能体的记忆系统 |
| awesome-openclaw | 701 | 资源/工具/教程列表 |

### 部署方案
| 项目 | Stars | 说明 |
|---|---|---|
| moltworker (Cloudflare) | 9.3K | 在 Cloudflare Workers 上运行 |
| OpenClawInstaller | 2.1K | 一键部署工具 |
| OpenClaw-Docker-CN-IM | 2.2K | 带中国即时通讯插件的 Docker 版本 |
| 1Panel | 33.7K | 带 OpenClaw 的 VPS 管理面板 |

### 基础设施
| 项目 | Stars | 说明 |
|---|---|---|
| ClawRouter (BlockRunAI) | 3.7K | 智能 LLM 路由器 |
| openclaw-studio | 707 | Web 管理面板 |
| openclaw-mission-control | 1.1K | 智能体编排仪表盘 |
| secure-openclaw (Composio) | 1.4K | 安全加固版本 |

### 本地化
| 项目 | Stars | 说明 |
|---|---|---|
| OpenClawChineseTranslation | 1.7K | 中文本地化 |
| openclaw-china | 1K | 中国即时通讯插件（飞书、钉钉、QQ） |
| openclaw-wechat | 921 | 微信集成 |
| openclaw101 | 473 | 中文教程站点 |

### 创新应用
| 项目 | Stars | 说明 |
|---|---|---|
| Clawra (SumeLabs) | 1.8K | AI 女友人设 |
| antfarm (snarktank) | 1.7K | 多智能体团队构建器 |
| MedgeClaw | 533 | 医疗 AI 集成 |

## MCP 集成

OpenClaw 通过 **mcporter**（外部桥接）集成 MCP（Model Context Protocol，模型上下文协议）：
- GitHub：`steipete/mcporter`
- 无需重启网关即可添加/移除 MCP 服务器
- 将 MCP 的变动与核心稳定性解耦
- 优先采用此方案，而非将 MCP 作为核心一等功能构建

## 插件分发

- 推荐方式：npm 包 + 本地扩展加载（用于开发）
- 插件 SDK：`src/plugin-sdk/`
- 插件文档：`docs/tools/plugin.md`
- 社区插件：https://docs.openclaw.ai/plugins/community
- 通过 ClawHub 发现技能：clawhub.ai

## 部署选项

| 平台 | 方式 |
|---|---|
| Docker | `docker-compose.yml`（主要方式） |
| Podman | `setup-podman.sh` |
| Fly.io | `fly.toml` |
| Render | `render.yaml` |
| Cloudflare Workers | 通过 `moltworker` |
| Ansible | 通过 `openclaw-ansible` |
| Nix | 通过 `nix-openclaw` |
| 1Panel | 一键 VPS 部署 |
