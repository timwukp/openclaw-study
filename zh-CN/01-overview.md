> 本文为中文翻译版本。英文原版为权威版本，如有差异以英文版为准。
> [English Version](../01-overview.md)

# OpenClaw 概述

## 什么是 OpenClaw？

OpenClaw 是一个开源、可自托管的个人 AI 助手平台。
- **GitHub**：https://github.com/openclaw/openclaw
- **Stars**：237,000+（截至 2026 年 2 月）
- **编程语言**：TypeScript (Node.js)
- **许可证**：MIT
- **创建时间**：2025 年 11 月
- **曾用名**：Warelay -> Clawdbot -> Moltbot -> OpenClaw

## 核心理念

OpenClaw 运行在你自己的设备上，在你自己的消息渠道中，遵循你自己的规则。
它连接多个 LLM（大语言模型）提供商（Claude、GPT、Gemini 等），并将
AI 作为个人助手暴露在 WhatsApp、Telegram、Discord、Slack、
Signal、iMessage 等众多平台上。

## 关键特性

1. **自托管**：在你自己的基础设施上运行
2. **多平台**：支持 10 多个消息渠道
3. **LLM 无关性**：支持 Anthropic、OpenAI、Google、OpenRouter、本地模型
4. **可扩展**：带有 Skills（技能）和 ClawHub 市场的插件系统
5. **智能体能力**：可浏览网页、执行代码、管理文件、使用工具
6. **记忆功能**：跨对话的持久化记忆
7. **计算机操控**：可控制浏览器并与系统交互
8. **全天候在线**：作为守护进程（Daemon）/网关（Gateway）运行，7×24 小时

## 本研究的重要意义

OpenClaw 代表了一类具有以下特征的 AI 驱动的个人助手：
- 对个人设备和数据的深度访问权限
- 能够自主代表用户执行操作
- 关于用户的持久化记忆
- 对通信渠道的控制权
- 可能在无实时人工监督的情况下运行

本研究旨在深入分析此类系统的架构，识别潜在风险，
并评估其伦理影响。
