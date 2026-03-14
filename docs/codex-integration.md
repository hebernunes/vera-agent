# Codex OAuth Integration

**Status:** ✅ Confirmed Working
**Integration Type:** ChatGPT Plus OAuth (no API key required)
**Source:** Based on OpenClaw Codex Integration Runbook

---

## What This Is

Adds OpenAI Codex as a selectable model inside Vera's existing Slack session via ChatGPT Plus OAuth. No second bot, no new Slack tokens, no new Docker volumes.

**What changes:**
- `/model codex` becomes available inside Vera's Slack session
- One entry added to `agents.defaults.models` allowlist in `openclaw.json`
- Zero API cost — covered by ChatGPT Plus subscription

**What does not change:**
- Primary fallback chain — untouched
- All existing Slack slash commands
- Gmail, Brave Search, git backup — unaffected

---

## How Codex Routes

Codex does **not** appear in OpenRouter logs — this is expected and correct.

```
Vera → OpenClaw → OpenAI (direct OAuth) → Codex
                         ↑
                  Bypasses OpenRouter entirely
                  Zero OpenRouter cost
                  Covered by ChatGPT Plus subscription
```

**Ground truth verification:**
```bash
docker exec openclaw-test cat /tmp/openclaw-1000/openclaw-$(date +%Y-%m-%d).log \
  | grep -i "openai\|codex\|provider" | tail -20
```

Look for: `provider=openai-codex model=gpt-5.x-codex`

---

## Pre-Flight Checklist

Run every check before touching any file.

```bash
# 1. Container is healthy
docker ps --filter name=openclaw-test

# 2. Running as uid 1000 (not root)
docker exec -u 1000 openclaw-test id

# 3. Loopback-only binding
docker port openclaw-test
# Expected: 127.0.0.1:18789 — NOT 0.0.0.0

# 4. Backup openclaw.json
docker exec -u 1000 openclaw-test sh -lc \
  'cd /home/node/.openclaw && cp openclaw.json openclaw.json.backup-$(date +%Y%m%d)'

# 5. Verify Vera is alive
# In Slack: @Vera Bot you there?
```

---

## Step 1 — OAuth Authentication

```bash
docker exec -it -u 1000 openclaw-test sh -lc \
  'openclaw models auth login --provider openai-codex'
```

The CLI prints an authorization URL and waits. Do not close this terminal.

**In your browser:**
1. Copy the URL from the terminal
2. Open in browser on your Mac
3. Sign in with your ChatGPT Plus account
4. Approve the access request
5. Browser redirects to `http://localhost:1/?code=...&state=...`
6. Page shows `ERR_CONNECTION_REFUSED` — **this is expected**
7. Copy the full URL from the address bar
8. Paste it into the waiting terminal and press Enter

**Verify auth succeeded:**
```bash
docker exec -u 1000 openclaw-test sh -lc 'openclaw models auth list'
```

Expected: `openai-codex` appears in the list.

**If you get `context deadline exceeded`:**
```bash
docker exec -it -u 1000 openclaw-test sh -lc \
  'OPENCLAW_HTTP_TIMEOUT=120 openclaw models auth login --provider openai-codex'
```

---

## Step 2 — Update openclaw.json

Your `models` block is an allowlist — any model not listed is blocked from `/model` switching.

Open `~/.openclaw-host/openclaw.json` and add to the `models` block:

```json
"openai-codex/gpt-5.x-codex": {}
```

And add to the `fallbacks` array:

```json
"openai-codex/gpt-5.x-codex"
```

> Replace `gpt-5.x-codex` with the current model string from OpenClaw docs.

**Rules to respect during this edit:**
- Do not touch `agents.defaults.model.primary` or existing `fallbacks`
- Do not remove `thinkingDefault: "off"` — breaks Anthropic models
- Do not remove `commands.useAccessGroups: false` — breaks slash commands
- Do not remove `channels.slack.commands.native: true` — breaks slash commands

**Reload config:**
```bash
docker kill --signal=SIGUSR1 openclaw-test
```

Check logs for errors:
```bash
docker logs --tail 10 openclaw-test
```

---

## Step 3 — Verify

```bash
# Model auth status
docker exec -it openclaw-test node /app/dist/index.js models status
# Expected: openai-codex:default ok expires in Xd

# Switch to Codex in Slack
/model openai-codex/gpt-5.x-codex

# Test with a coding task
# Ask Vera to write a bash one-liner or edit a config

# Switch back to primary
/model openrouter/x-ai/grok-4.1-fast
```

---

## Token Expiry and Re-auth

Codex OAuth tokens expire approximately every 2 weeks.

**When token expires:** Codex silently fails and falls back to the next model in the chain — no visible error.

**Re-auth command:**
```bash
docker exec -it openclaw-test node /app/dist/index.js onboard --auth-choice openai-codex
```

**Steps:**
1. Run command above
2. Select **"Use existing values"** when config prompt appears
3. Copy the OAuth URL from terminal
4. Open in browser → sign in with ChatGPT Plus account
5. Copy full redirect URL from address bar
6. Paste back into terminal
7. Verify: `docker exec -it openclaw-test node /app/dist/index.js models status`

**Set a Slack reminder** so you don't miss the expiry:
```
@Vera Bot set a reminder in #vera-reminders 2 days before my Codex token expires
```

---

## When to Use Codex

| Task | Use Codex? |
|------|-----------|
| `openclaw.json` edits | ✅ Yes |
| Shell / bash scripts | ✅ Yes |
| Docker Compose changes | ✅ Yes |
| Cron job scripts | ✅ Yes |
| Security audit scripts | ✅ Yes |
| General conversation | ❌ Stay on primary |
| Web search / news | ❌ Stay on primary |
| Memory updates / daily ops | ❌ Stay on primary |

---

## Rollback

If anything breaks after the config edit:

```bash
# 1. Find last good backup
docker exec -u 1000 openclaw-test sh -lc \
  'ls -1t /home/node/.openclaw/openclaw.json.backup-* | head -5'

# 2. Restore
docker exec -u 1000 openclaw-test sh -lc \
  'cd /home/node/.openclaw && cp openclaw.json.backup-YYYYMMDD openclaw.json'

# 3. Reload
docker kill --signal=SIGUSR1 openclaw-test

# 4. Verify Vera is back
# In Slack: @Vera Bot you there?
```

---

## Key Lessons Learned

1. **OpenRouter logs will never show Codex** — it routes directly to OpenAI. This is correct behavior, not a bug
2. **Both locations required** — model must be in `fallbacks` array AND `models` block with exact string match
3. **Full restart required for new model registration** — SIGUSR1 is not sufficient; use `docker-compose down && up`
4. **Internal logs are ground truth** — grep for `provider=openai-codex` in the daily log file
5. **Token expiry is silent** — set a calendar reminder or Slack cron reminder before expiry
