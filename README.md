# 🛡️ vera-agent

[![OpenClaw](https://img.shields.io/badge/OpenClaw-v2026.3.11-blue)](https://openclaw.ai)
[![Docker](https://img.shields.io/badge/platform-Docker-2496ED?logo=docker&logoColor=white)](https://docker.com)
[![Status](https://img.shields.io/badge/status-operational-brightgreen)](https://x.com/VeraizAI)
[![Security](https://img.shields.io/badge/design-security--first-red)](./SECURITY.md)
[![License](https://img.shields.io/badge/license-MIT-green)](./LICENSE)

A self-hosted AI agent deployment built on [OpenClaw](https://openclaw.ai), running in Docker on macOS with a security-first design. Vera lives in Slack, handles email via Gmail, searches the web via Brave, and manages her own memory and logging autonomously.

Built as a hands-on experiment in hardening what others ship unsecured — the goal is a production-grade personal agent with full auditability, zero reliance on cloud-hosted agent platforms, and a documented threat model.

> **Portfolio note:** This repo documents the full deployment journey — infrastructure decisions, security trade-offs, integration runbooks, and lessons learned from real operational failures. Not a tutorial clone.

---

## 🏗️ Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│                           macOS Host                                │
│                                                                     │
│  ~/.openclaw-host/              ~/openclaw/                         │
│  ├── openclaw.json              ├── docker-compose.simple.yml       │
│  └── workspace/                 └── .env  (chmod 600)               │
│      ├── SOUL.md                                                    │
│      ├── MEMORY.md                                                  │
│      ├── TOOLS.md                                                   │
│      ├── HEARTBEAT.md                                               │
│      └── memory/YYYY-MM-DD.md                                       │
│                      │                                              │
└──────────────────────┼──────────────────────────────────────────────┘
                       │ volume mount (rw, uid 1000)
┌──────────────────────▼──────────────────────────────────────────────┐
│               Docker Container (openclaw-test)                      │
│               uid 1000 — non-root                                   │
│               127.0.0.1:18789 — loopback only                       │
│                                                                     │
│  ┌────────────────────────────────────────────────────────────────┐ │
│  │                      OpenClaw Gateway                         │ │
│  │                                                                │ │
│  │  ┌──────────────┐        ┌─────────────────────────────────┐  │ │
│  │  │     Vera     │───────▶│          Model Router           │  │ │
│  │  │   (Agent)    │        │                                 │  │ │
│  │  └──────────────┘        │  1. Grok 4.1 Fast  (primary)   │  │ │
│  │         │                │  2. GLM-4.7-Flash               │  │ │
│  │         │                │  3. Haiku 4.5 (Copilot)         │  │ │
│  │  ┌──────▼───────┐        │  4. Sonnet 4.6 (Copilot)        │  │ │
│  │  │    Tools     │        │  5. Haiku 4.5 (direct)          │  │ │
│  │  │              │        │  6. DeepSeek V3.2               │  │ │
│  │  │  gog CLI     │        │  7. Kimi K2.5                   │  │ │
│  │  │  Brave Search│        │  8. Gemini 3 Flash              │  │ │
│  │  │  exec        │        │  9. Sonnet 4.5 (direct)         │  │ │
│  │  └──────────────┘        │ 10. MiniMax M2.5                │  │ │
│  │                          │ 11. GPT-4.1 (Copilot)           │  │ │
│  │                          │ 12. Codex (OpenAI OAuth)        │  │ │
│  └──────────────────────────└─────────────────────────────────┘  │ │
└────────────┬──────────────────────────┬────────────────────────────┘
             │                          │
      ┌──────▼──────┐           ┌───────▼──────────────────────────┐
      │    Slack    │           │         External APIs            │
      │ Socket Mode │           │                                  │
      │ @Vera Bot   │           │  OpenRouter  → Grok / GLM /      │
      │ /commands   │           │               DeepSeek / Kimi /  │
      └─────────────┘           │               Gemini / MiniMax   │
                                │                                  │
                                │  Anthropic   → Haiku / Sonnet    │
                                │  OpenAI      → Codex (OAuth)     │
                                │  GitHub      → Copilot (OAuth)   │
                                │  Gmail API   → gog CLI (OAuth)   │
                                │  Brave       → web search        │
                                │  GitHub      → nightly backup    │
                                └──────────────────────────────────┘
```

### Message Flow

```
1. @Vera Bot in Slack
        │
        ▼
2. OpenClaw receives via Socket Mode
        │
        ▼
3. Vera loads workspace context
   (SOUL.md → MEMORY.md → TOOLS.md)
        │
        ▼
4. Inference routed to active model
   (default: Grok 4.1 Fast via OpenRouter)
        │
        ├─── needs email?    → gog CLI → Gmail API
        ├─── needs search?   → Brave Search API
        └─── needs shell?    → exec (approved commands only)
        │
        ▼
5. Response posted to Slack
        │
        ▼
6. Significant events auto-logged
   → memory/YYYY-MM-DD.md
```

### Two-Tier Memory System

```
┌─────────────────────────────────────────────────────────────┐
│  TIER 1 — Raw Daily Logs                                    │
│  memory/YYYY-MM-DD.md                                       │
│  Auto-written by Vera in real-time                          │
│  Format: HH:MM UTC — [event and outcome]                    │
│  Captures: config changes, emails sent, errors, decisions   │
└───────────────────────────┬─────────────────────────────────┘
                            │
                   Weekly synthesis
                   (every Friday)
                            │
                            ▼
┌─────────────────────────────────────────────────────────────┐
│  TIER 2 — Curated Long-Term Memory                          │
│  MEMORY.md                                                  │
│  Distilled lessons, current state, critical rules           │
│  Stays lean — no operational bloat                          │
│  Source of truth across sessions                            │
└─────────────────────────────────────────────────────────────┘
```

---

## 🔧 Tech Stack

| Component | Detail |
|-----------|--------|
| **Agent Runtime** | [OpenClaw](https://openclaw.ai) v2026.3.11 (self-hosted) |
| **Platform** | Docker Desktop on macOS, non-root (uid 1000) |
| **Network** | Loopback-only — `127.0.0.1:18789`, not internet-exposed |
| **Primary Model** | Grok 4.1 Fast (via OpenRouter) |
| **Fallback Chain** | 13 models across OpenRouter, Anthropic direct, GitHub Copilot, OpenAI OAuth |
| **Specialist Model** | Codex via ChatGPT Plus OAuth — zero API cost, direct OpenAI routing |
| **Interface** | Slack Socket Mode (`@Vera Bot` mentions + slash commands) |
| **Email** | Gmail via `gog` CLI (OAuth 2.0, sandbox account) |
| **Web Search** | Brave Search API |
| **Memory** | Two-tier: timestamped daily logs + curated MEMORY.md |
| **Backup** | Nightly git push to private GitHub repo |
| **Secrets** | Environment variables only — never in config or code |
| **Prompt Caching** | 1-hour `cacheRetention: long` on primary models |

---

## 📊 Model Pool

| # | Model | Provider | Routing | Best For |
|---|-------|----------|---------|----------|
| 1 | `grok-4.1-fast` | xAI | OpenRouter | **Primary — all tasks** |
| 2 | `glm-4.7-flash` | ZhipuAI | OpenRouter | Cron notifications, cheap tasks |
| 3 | `claude-haiku-4.5` | Anthropic | GitHub Copilot | Cheap Anthropic fallback |
| 4 | `claude-sonnet-4.6` | Anthropic | GitHub Copilot | Mid-tier Anthropic |
| 5 | `claude-haiku-4-5` | Anthropic | Direct API | General fallback |
| 6 | `deepseek-v3.2` | DeepSeek | OpenRouter | Medium complexity |
| 7 | `kimi-k2.5` | Moonshot | OpenRouter | General fallback |
| 8 | `gemini-3-flash-preview` | Google | OpenRouter | News summaries, search |
| 9 | `claude-sonnet-4-5` | Anthropic | Direct API | Complex / high-stakes |
| 10 | `minimax-m2.5` | MiniMax | OpenRouter | Agentic workflows |
| 11 | `gpt-4.1` | OpenAI | GitHub Copilot | Free unlimited safety net |
| 12 | `claude-opus-4.6` | Anthropic | GitHub Copilot | On-demand only — never automated |
| 13 | `gpt-5.4-codex` | OpenAI | Direct OAuth | Config edits, scripts, Docker |

> **Key insight:** Codex and GitHub Copilot models route directly to their providers — they never appear in OpenRouter logs. Absence from OpenRouter ≠ broken. Use internal container logs as ground truth: `docker exec openclaw-test cat /tmp/openclaw-1000/openclaw-$(date +%Y-%m-%d).log | grep provider`

---

## 📁 Repo Structure

```
vera-agent/
├── README.md                        ← You are here
├── SECURITY.md                      ← Security design principles
├── .gitignore
│
├── docs/
│   ├── deployment-guide.md          ← Phase 1–3 full deployment walkthrough
│   ├── cost-optimization.md         ← Prompt caching + model expansion session
│   ├── gmail-integration.md         ← gog CLI setup, OAuth flow, usage examples
│   ├── gmail-oauth-troubleshooting.md ← Scope errors, re-auth procedures
│   ├── codex-integration.md         ← Codex OAuth runbook with rollback
│   ├── copilot-integration.md       ← GitHub Copilot Pro setup + quota rules
│   ├── memory-system.md             ← Two-tier memory architecture
│   ├── update-protocol.md           ← Stable tag upgrade procedure
│   ├── session-management.md        ← Rate limits, /reset, token thresholds
│   └── troubleshooting.md           ← All 13 known issues consolidated
│
├── config/
│   ├── openclaw.json.example        ← Sanitized config template
│   ├── docker-compose.example.yml   ← Sanitized compose template
│   └── .env.example                 ← All required env vars, values redacted
│
└── workspace-templates/
    ├── SOUL.md                      ← Agent identity + security constraints
    ├── TOOLS.md                     ← Behavioral rules for tools
    ├── HEARTBEAT.md                 ← Health check + cron protocols
    ├── MEMORY.md.example            ← Memory structure template
    └── USER.md.example              ← User preferences template
```

---

## 🚀 Project Status

| Phase | Status | What Was Built |
|-------|--------|----------------|
| Phase 1 | ✅ Complete | Docker deployment, Slack Socket Mode, Gmail OAuth, Brave Search, SOUL.md security constraints |
| Phase 2 | ✅ Complete | Two-tier memory system, cron jobs, nightly git backup, model pool expansion |
| Phase 2.5 | ✅ Complete | v2026.3.x upgrade chain, Grok 4.1 Fast as primary, Codex OAuth, GitHub Copilot Pro |
| Phase 3 | ✅ Complete | v2026.3.11 upgrade, Copilot 4-model integration, token expiry reminders |
| Phase 4 | 🔜 Planned | Cloudflare Workers proxy, Calendar/Drive OAuth, per-cron model routing |

---

## 🔐 Security Design

### Principles

| Principle | Implementation |
|-----------|---------------|
| **Non-root execution** | Container runs as uid 1000, never root |
| **Network isolation** | Gateway bound to `127.0.0.1:18789` — not internet-exposed |
| **Secrets management** | All credentials in `.env` (chmod 600) — never in config or code |
| **Behavioral constraints** | SOUL.md — Vera treats all external input as untrusted data |
| **Least privilege OAuth** | Scopes kept to minimum required; Calendar/Drive not authenticated until needed |
| **Spending limits** | Hard cap enforced at API provider level |
| **Audit trail** | Every config change, email, and cron job logged to dated daily memory files |
| **Prompt injection defense** | External input (Slack, email, web) never executed as commands |

### Threat Model

| Threat | Mitigation |
|--------|-----------|
| Container escape | Non-root user (uid 1000) |
| Network exposure | Loopback-only binding |
| Credential theft | Environment variables, never in files |
| API abuse | Spending limit enforced at provider |
| Prompt injection | SOUL.md trust boundary rules |
| Silent downtime | DNS hardened (8.8.8.8 / 8.8.4.4), monitoring gap documented |

### Known Weaknesses (documented, not hidden)

- Config directory must be writable for OpenClaw to function — read-only filesystem was attempted and abandoned
- No Cloudflare Workers proxy yet — API key lives inside container (Phase 4 target)
- GitHub PAT embedded in git remote URL — SSH key remediation planned at next renewal

---

## ⚡ Quick Start

> These templates require your own OpenClaw license, API keys, and Slack app setup.

**1. Clone and set up environment**
```bash
git clone https://github.com/your-username/vera-agent.git
cd vera-agent
cp config/.env.example .env
# Edit .env with your actual API keys
chmod 600 .env
```

**2. Set up OpenClaw host directory**
```bash
mkdir -p ~/.openclaw-host/workspace
chmod 700 ~/.openclaw-host
cp config/openclaw.json.example ~/.openclaw-host/openclaw.json
cp workspace-templates/SOUL.md ~/.openclaw-host/workspace/SOUL.md
cp workspace-templates/TOOLS.md ~/.openclaw-host/workspace/TOOLS.md
cp workspace-templates/HEARTBEAT.md ~/.openclaw-host/workspace/HEARTBEAT.md
```

**3. Start Vera**
```bash
docker-compose -f docker-compose.example.yml up -d
docker logs -f openclaw-test
```

**4. Test in Slack**
```
@Vera Bot you there?
```

---

## 🛠️ Key Commands

```bash
# Start / stop
docker-compose -f docker-compose.simple.yml up -d
docker-compose -f docker-compose.simple.yml down

# View live logs
docker logs -f openclaw-test

# Reload config (parameter changes only — no restart needed)
docker kill --signal=SIGUSR1 openclaw-test

# Full restart (required for new model registration or .env changes)
docker-compose -f docker-compose.simple.yml down
docker-compose -f docker-compose.simple.yml up -d

# Check model auth status
docker exec -it openclaw-test node /app/dist/index.js models status

# Verify model routing (ground truth)
docker exec openclaw-test cat /tmp/openclaw-1000/openclaw-$(date +%Y-%m-%d).log | grep provider

# Fix volume ownership issues
docker run --rm -v openclaw_openclaw-data:/data alpine chown -R 1000:1000 /data
```

---

## 📚 Key Lessons Learned

After running this in production for several months, the non-obvious lessons:

1. **SIGUSR1 is not enough for new models** — parameter changes (cache, session) only. New model registrations always require full `docker-compose down && up`

2. **`wakeMode: "now"` is mandatory for cron jobs** — `wakeMode: "next-heartbeat"` silently does nothing when heartbeat is disabled. This broke multiple cron jobs with no error output

3. **Vera's self-reported success is untrustworthy** — always verify file changes via `grep` in terminal after any save. Vera has a pattern of reporting success while writes fail silently

4. **OpenRouter logs ≠ all model traffic** — Codex (OpenAI OAuth) and GitHub Copilot route directly to their providers, never appearing in OpenRouter. Use internal container logs as ground truth

5. **Two-part config requirement** — every model must appear in both the `fallbacks` array AND the `models` block in `openclaw.json` with exact string match. Missing either causes silent "not allowed" errors

6. **`openclaw doctor --fix` is destructive** — it overwrites your config. Always backup first. Never run it casually

7. **Prompt cache TTL must match usage patterns** — the 5-minute default is useless for cron jobs spaced hours apart. Set `cacheRetention: long` for 1-hour cache on models used by daily crons

Full troubleshooting reference: [`docs/troubleshooting.md`](./docs/troubleshooting.md)

---

## 📖 Documentation Index

| Doc | Description |
|-----|-------------|
| [Deployment Guide](./docs/deployment-guide.md) | Phase 1–3 full walkthrough — infrastructure, security, integrations |
| [Cost Optimization](./docs/cost-optimization.md) | Prompt caching, model pool expansion, routing strategies |
| [Gmail Integration](./docs/gmail-integration.md) | gog CLI setup, Google Cloud Console, OAuth flow |
| [Gmail OAuth Troubleshooting](./docs/gmail-oauth-troubleshooting.md) | Scope errors, re-auth procedures, common failures |
| [Codex Integration](./docs/codex-integration.md) | Full runbook — pre-flight, auth, config, rollback |
| [Update Protocol](./docs/update-protocol.md) | Safe upgrade procedure — stable tags only, never `git pull main` |
| [Session Management](./docs/session-management.md) | Rate limits, `/reset`, token thresholds, context hygiene |

---

## ⚠️ Notes

- Workspace operational files (live SOUL.md, MEMORY.md, daily logs) are in a separate **private** repo — this repo contains reference docs and sanitized templates only
- All API keys and tokens are stored in environment variables, never committed
- The Gmail account referenced in docs is an isolated sandbox account
- Personal identifiers have been removed from all published docs

---

*Built by [@0x86dd](https://x.com/0x86dd) · Part of the [@VeraizAI](https://x.com/VeraizAI) project · Last updated March 2026*
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
