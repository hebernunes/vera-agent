# Vera — AI Agent Deployment Docs

Personal documentation for my OpenClaw/Vera AI agent setup.
Not intended for production use — shared for reference only.

---

## 🤖 What is Vera?

Vera is a security-first AI assistant running on OpenClaw in Docker on macOS.
She lives in Slack, handles email via Gmail, searches the web via Brave Search,
and manages her own memory and logging autonomously.

---

## 📁 Contents

| File | Description |
|------|-------------|
| `OpenClaw_Vera_DeploymentV1.2_Phase1_Summary.md` | Full deployment guide — infrastructure, security, integrations, lessons learned |

*More docs added as the project evolves.*

---

## 🏗️ Stack

- **Agent:** OpenClaw 2026.2.23
- **Model:** Claude Haiku 4.5 (primary) → Sonnet 4.5 → DeepSeek V3.2 → Kimi K2.5 → GLM-4.7-Flash
- **Platform:** Docker on macOS
- **Channels:** Slack (Socket Mode)
- **Integrations:** Gmail, Brave Search, GitHub backup
- **Memory:** Two-tier system (daily logs + curated MEMORY.md)

---

## 📊 Project Status

| Phase | Status | Description |
|-------|--------|-------------|
| Phase 1 | ✅ Complete | Core infrastructure, Slack, Gmail, security hardening |
| Phase 2 | ✅ Complete | Memory system, cron jobs, git backup, model expansion |
| Phase 3 | 🔜 Planned | Cloudflare Workers proxy, production hardening |

---

## ⚠️ Notes

- Operational files (SOUL.md, MEMORY.md, TOOLS.md) are in a separate private repo
- This repo contains reference docs only — no secrets or credentials
- All API keys and tokens are stored in environment variables, never in code

---

*Last updated: February 2026*
