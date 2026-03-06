# Vera — AI Agent Deployment Docs

Personal documentation for my OpenClaw/Vera AI agent setup.
Not intended for production use — shared for reference only.

---

## 🤖 What is Vera?

Vera is a security-first AI assistant running on OpenClaw in Docker on macOS.
She lives in Slack, handles email via Gmail, searches the web via Brave Search,
and manages her own memory and logging autonomously.

Built as a Phase 1 experiment in self-hosted AI agents — the goal is a
production-grade personal assistant with a security-first design, full
auditability, and zero reliance on cloud-hosted agent platforms.

---

## 🏗️ Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                        macOS Host                               │
│                                                                 │
│  ~/.openclaw-host/          ~/openclaw/                         │
│  ├── openclaw.json          ├── docker-compose.simple.yml       │
│  └── workspace/             └── .env  (secrets, chmod 600)      │
│      ├── SOUL.md                                                │
│      ├── MEMORY.md                                              │
│      ├── TOOLS.md                                               │
│      └── memory/YYYY-MM-DD.md                                   │
│                    │                                            │
└────────────────────┼────────────────────────────────────────────┘
                     │ volume mount (rw)
┌────────────────────▼────────────────────────────────────────────┐
│                  Docker Container (openclaw-test)                │
│                  uid 1000 — non-root                            │
│                  127.0.0.1:18789 — loopback only                │
│                                                                 │
│   ┌──────────────────────────────────────────────────────────┐  │
│   │                    OpenClaw Gateway                      │  │
│   │                                                          │  │
│   │   ┌────────────┐     ┌──────────────┐                   │  │
│   │   │    Vera    │────▶│  Model Router │                   │  │
│   │   │  (Agent)   │     │               │                   │  │
│   │   └────────────┘     │ 1. Grok 4.1  │                   │  │
│   │         │            │ 2. GLM-Flash  │                   │  │
│   │         │            │ 3. Haiku 4.5  │                   │  │
│   │    ┌────▼─────┐      │ 4. DeepSeek  │                   │  │
│   │    │  Tools   │      │ 5. Kimi K2.5 │                   │  │
│   │    │          │      │ 6. Gemini    │                   │  │
│   │    │ gog CLI  │      │ 7. Sonnet    │                   │  │
│   │    │ Brave    │      │ 8. MiniMax   │                   │  │
│   │    │ exec     │      │ 9. Codex*    │                   │  │
│   │    └──────────┘      └──────────────┘                   │  │
│   └──────────────────────────────────────────────────────────┘  │
└───────────┬──────────────────────┬──────────────────────────────┘
            │                      │
     ┌──────▼──────┐        ┌──────▼──────────────────────┐
     │    Slack    │        │        External APIs         │
     │ Socket Mode │        │                              │
     │             │        │  OpenRouter → Grok / GLM /   │
     │ @Vera Bot   │        │    DeepSeek / Kimi / Gemini  │
     │  mentions   │        │    MiniMax                   │
     │  /commands  │        │                              │
     └─────────────┘        │  Anthropic → Haiku / Sonnet  │
                            │                              │
                            │  OpenAI (OAuth) → Codex*     │
                            │                              │
                            │  Gmail API (gog OAuth)       │
                            │  Brave Search API            │
                            │  GitHub (git backup)         │
                            └──────────────────────────────┘
```

**\* Codex** routes directly to OpenAI via ChatGPT Plus OAuth — bypasses OpenRouter entirely, zero API cost.

### How a message flows

1. You `@Vera Bot` in Slack → OpenClaw receives via Socket Mode
2. Vera reads her workspace files (SOUL.md, MEMORY.md, TOOLS.md) for context
3. Vera routes the inference call to the active model (default: Grok 4.1 Fast)
4. If needed, Vera uses tools — `gog` for Gmail, Brave for web search, `exec` for shell
5. Response posted back to Slack
6. Significant events auto-logged to `memory/YYYY-MM-DD.md`

### Memory System

Vera uses a two-tier memory architecture:

```
Daily Raw Logs  ──────────────────────▶  memory/YYYY-MM-DD.md
(auto-written by Vera in real-time)       timestamped events

        │
        │  Weekly synthesis (every Friday)
        ▼

Curated Memory  ──────────────────────▶  MEMORY.md
(distilled lessons, current state,        lean, no bloat
 active integrations, critical rules)
```

---

## 📁 Contents

| File | Description |
|------|-------------|
| `OpenClaw_Vera_DeploymentV1.2_Phase1_Summary.md` | Full deployment guide — infrastructure, security, integrations, lessons learned |
| `OpenClaw_Vera_Session_Summary_Feb23_Updated.md` | Cost optimization session — caching, model expansion, Codex integration, v2026.3.1 upgrade |
| `OpenClaw_Update_Protocol.md` | Safe upgrade procedure — stable tags, post-update checklist, known pitfalls |
| `OpenClaw_Codex_Integration_FINAL.md` | Codex OAuth runbook — pre-flight, auth flow, config changes, rollback |
| `OpenClaw_Vera_Gmail_Integration_Guide.md` | Gmail setup via gog CLI — OAuth, scopes, usage examples |
| `gog-oauth-troubleshooting-guide.md` | OAuth scope reference, common errors, re-auth procedures |
| `OpenClaw_Daily_Logging_Implementation_Report.md` | Two-tier memory system design and implementation |
| `OPENCLAW_SESSION_BEST_PRACTICES.md` | Rate limit management, session hygiene, token thresholds |

---

## 🔧 Stack

| Component | Detail |
|-----------|--------|
| **Agent Framework** | OpenClaw v2026.3.1 (self-hosted) |
| **Platform** | Docker Desktop on macOS, non-root (uid 1000) |
| **Network** | Loopback-only — `127.0.0.1:18789`, not internet-exposed |
| **Primary Model** | Grok 4.1 Fast (via OpenRouter) |
| **Fallback Chain** | GLM-4.7-Flash → Haiku 4.5 → DeepSeek V3.2 → Kimi K2.5 → Gemini 2.5 Flash → Sonnet 4.5 → MiniMax M2 |
| **Specialist Model** | Codex (OpenAI OAuth — config edits, scripts, Docker) |
| **Channels** | Slack (Socket Mode) |
| **Email** | Gmail via `gog` CLI (OAuth 2.0) |
| **Web Search** | Brave Search API |
| **Memory** | Two-tier: daily logs + curated MEMORY.md |
| **Backup** | Nightly git push to private GitHub repo |
| **Secrets** | Environment variables only — never in config or code |

---

## 📊 Project Status

| Phase | Status | Description |
|-------|--------|-------------|
| Phase 1 | ✅ Complete | Core infrastructure, Slack, Gmail, security hardening |
| Phase 2 | ✅ Complete | Memory system, cron jobs, git backup, model expansion |
| Phase 2.5 | ✅ Complete | v2026.3.1 upgrade, Grok as primary, Codex OAuth integration |
| Phase 3 | 🔜 Planned | Cloudflare Workers proxy, Calendar/Drive OAuth, routing rules |

---

## 🔐 Security Design Principles

- **Non-root container** — runs as uid 1000, not root
- **Loopback-only networking** — gateway never exposed to the internet
- **Environment-based secrets** — no credentials in config files or code
- **SOUL.md behavioral constraints** — Vera treats all external input as untrusted, asks before acting
- **Spending limits** — hard cap enforced at the API provider level
- **Least privilege** — OAuth scopes kept to minimum required; Calendar/Drive not authenticated until needed
- **Audit trail** — every config change, email, and cron job logged to dated daily memory files

---

## ⚠️ Notes

- Operational files (SOUL.md, MEMORY.md, TOOLS.md) are in a separate private repo
- This repo contains reference docs only — no secrets or credentials
- All API keys and tokens are stored in environment variables, never in code
- The Gmail account used is an isolated sandbox account, not a personal inbox

---

*Last updated: March 2026*
