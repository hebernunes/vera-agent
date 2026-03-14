# Session Management

**Purpose:** Avoid rate limits and manage context window effectively.

---

## Quick Reference Commands

```bash
/status          # Check current token usage
/context list    # See what's consuming context
/reset           # Start fresh session (clears context)
/new             # Same as /reset with greeting
/compact         # Compress session history (preserves context)
```

---

## Rate Limits vs Spending Limits

These are two different things:

| Type | Cause | Error | Solution |
|------|-------|-------|----------|
| **Rate limit** | Too many tokens per minute | HTTP 429 | Reset session or wait 60s |
| **Spending limit** | Monthly budget exceeded | Different message | Increase limit at provider |

Rate limits are about **speed**, not cost. Hitting a rate limit does not mean you've spent money.

---

## Context Window Thresholds

Every message re-sends the **entire conversation history**. A large session = large input on every message = rate limits hit faster.

| Usage | Status | Action |
|-------|--------|--------|
| 0–50% | ✅ Safe | Continue normally |
| 50–70% | ⚠️ Warning | Consider `/compact` |
| 70%+ | 🚨 Critical | Use `/reset` soon |
| 90%+ | 💣 Emergency | `/reset` immediately |

**Example:** A 123k token session sends 123k tokens with every single message. At a 50k/min rate limit, you can only send 2–3 messages before hitting the limit.

---

## Session Management Options

### Option 1: Reset (Recommended for new tasks)
```bash
/reset
```
Clears all context. Fresh start. Best when starting a new task or after hitting rate limits.

### Option 2: Compact (Preserve context)
```bash
/compact
```
Summarizes conversation history. Reduces token count while keeping key information. Best for ongoing projects where context matters.

### Option 3: Delete session files (Nuclear option)
```bash
# Stop OpenClaw first
docker-compose -f ~/openclaw/docker-compose.simple.yml down

# Delete session files
rm -rf ~/.openclaw-host/agents/main/sessions/*.jsonl
rm -f ~/.openclaw-host/agents/main/sessions/sessions.json

# Restart
docker-compose -f ~/openclaw/docker-compose.simple.yml up -d
```

---

## Session Configuration

Current session config in `openclaw.json`:

```json
{
  "session": {
    "reset": {
      "mode": "idle",
      "idleMinutes": 480
    }
  }
}
```

Session resets automatically after 8 hours of inactivity.

> **Important:** `session` must be a **top-level key** in `openclaw.json`. Never nest it under `agents.defaults` — this causes a startup error.

---

## What Counts as Input Tokens

Input tokens include everything sent with each message:

- Your message text
- Entire conversation history
- System prompts (SOUL.md, MEMORY.md, TOOLS.md loaded at session start)
- Tool outputs stored in session

This is why large sessions hit rate limits quickly even for short messages.

---

## Debugging Rate Limits

```bash
# Step 1: Check usage
/status

# Step 2: See what's consuming tokens
/context list

# Step 3: Act based on usage
# < 50% → wait 60 seconds and retry
# > 50% → /compact or /reset

# Step 4: Verify after reset
/status
# Should show near 0% usage
```

---

## Daily Workflow Recommendations

**Before a long session:**
```bash
/status
# If > 70%, reset first
/reset
/status  # Confirm fresh start
```

**During work (every 2–3 hours):**
```bash
/status
# If approaching 100k tokens, use /compact or /reset
```

---

## Session Storage Locations

```bash
# Session metadata
~/.openclaw-host/agents/main/sessions/sessions.json

# Session transcripts (conversation history)
~/.openclaw-host/agents/main/sessions/*.jsonl
```
