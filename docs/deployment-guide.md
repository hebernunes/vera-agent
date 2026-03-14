# **OpenClaw/Vera Deployment - Phase 1 Summary**

**Date:** February 7-8, 2026  
**Last Updated:** February 16-17, 2026  
**Project:** Secure AI Agent Deployment (Vera)  
**Status:** ✅ Phase 1 Complete + Phase 2 Enhancements Operational

---

## **🎯 What We Achieved**

### **Core Infrastructure**

✅ **OpenClaw deployed in Docker**

* Container: `openclaw-test`
* Base image: Node.js on Ubuntu
* Running on Docker Desktop (macOS)
* Gateway listening on `ws://127.0.0.1:18789`

✅ **Security Hardening**

* Non-root user (uid 1000:1000)
* Loopback-only networking (127.0.0.1 - not exposed to internet)
* Port mapping: `127.0.0.1:18789->18789/tcp` (host access only)
* Environment-based secrets (no hardcoded credentials)
* `.env` file with `chmod 600` permissions

✅ **Authentication & API**

* Anthropic API key authentication (not session credentials)
* Model: `claude-haiku-4-5-20251001`
* Spending limit: $10 (set in Anthropic console)
* Fallback models: `claude-sonnet-4-5-20250514`, `openrouter/moonshotai/kimi-k2.5`
* API calls working and verified

✅ **Slack Integration**

* Socket Mode enabled (real-time messaging)
* Bot Token (`xoxb-...`) configured
* App Token (`xapp-...`) configured
* Event subscriptions: `app_mention`, `message.channels`, `message.im`
* Channel policy: `"open"` (responds in all channels when @mentioned)
* **Vera responds successfully to @mentions**

✅ **Agent Identity (SOUL.md)**

* Named: **Vera**
* Security-first behavioral constraints
* Trust boundary definitions (external vs internal sources)
* Prompt injection defense rules
* Least privilege execution principles
* Incident response protocols

✅ **Gmail Integration (gog CLI)**

* `gog` CLI built from source (Go 1.23.5)
* OAuth 2.0 authenticated with `your-sandbox@gmail.com`
* Gmail read, send, modify capabilities active
* Credentials persisted in `gog-config` Docker volume

✅ **Brave Search API**

* Provider: Brave Search (`BSA...` key format)
* Real-time web search enabled
* Daily AI & Security News Summary cron job active
* Configured via `tools` section in `openclaw.json`

✅ **Daily Logging & Memory System**

* Two-tier memory architecture implemented
* Raw daily logs: `~/.openclaw-host/workspace/memory/YYYY-MM-DD.md`
* Curated long-term memory: `~/.openclaw-host/workspace/MEMORY.md`
* Weekly review cron job: every Friday at 4:00 PM PST
* Vera autonomously logging events in real-time

---

## **📁 Final File Structure**

```
~/openclaw/                          # Project root
├── .env                            # Secrets (chmod 600)
│   ├── ANTHROPIC_API_KEY
│   ├── SLACK_BOT_TOKEN
│   ├── SLACK_APP_TOKEN
│   ├── GATEWAY_TOKEN
│   ├── OPENROUTER_API_KEY
│   ├── GOG_KEYRING_PASSWORD
│   └── BRAVE_API_KEY
├── docker-compose.simple.yml       # Container orchestration
└── .gitignore                      # Protects .env

~/.openclaw-host/                   # OpenClaw config (mounted into container)
├── openclaw.json                   # Agent & gateway config
└── workspace/
    ├── SOUL.md                     # Vera's identity & security rules
    ├── MEMORY.md                   # Curated long-term memory
    └── memory/                     # Daily raw logs
        └── YYYY-MM-DD.md
```

---

## **🔧 Key Configuration Files**

### **docker-compose.simple.yml**

```yaml
services:
  openclaw:
    build: .
    container_name: openclaw-test
    user: "1000:1000"                         # Non-root
    ports:
      - "127.0.0.1:18789:18789"               # Loopback only
    environment:
      ANTHROPIC_API_KEY: ${ANTHROPIC_API_KEY}
      SLACK_BOT_TOKEN: ${SLACK_BOT_TOKEN}
      SLACK_APP_TOKEN: ${SLACK_APP_TOKEN}
      OPENCLAW_GATEWAY_TOKEN: ${GATEWAY_TOKEN}
      OPENROUTER_API_KEY: ${OPENROUTER_API_KEY}
      GOG_KEYRING_PASSWORD: ${GOG_KEYRING_PASSWORD}
      BRAVE_API_KEY: ${BRAVE_API_KEY}
      PATH: "/home/node/.local/bin:/usr/local/go/bin:${PATH}"
    volumes:
      - ~/.openclaw-host:/home/node/.openclaw:rw   # Writable mount
      - openclaw-data:/tmp/openclaw
      - local-bin:/home/node/.local/bin:rw          # gog persists here
      - go-install:/usr/local/go:rw                 # Go compiler persists here
      - gog-config:/home/node/.config:rw            # Gmail OAuth tokens
    dns:
      - 8.8.8.8
      - 8.8.4.4
    restart: unless-stopped

volumes:
  openclaw-data:
  local-bin:
  go-install:
  gog-config:
```

### **openclaw.json**

```json
{
  "agents": {
    "defaults": {
      "model": {
        "primary": "anthropic/claude-haiku-4-5-20251001",
        "fallbacks": [
          "anthropic/claude-sonnet-4-5-20250514",
          "openrouter/moonshotai/kimi-k2.5"
        ]
      }
    }
  },
  "gateway": {
    "mode": "local",
    "bind": "loopback",
    "port": 18789
  },
  "channels": {
    "slack": {
      "enabled": true,
      "groupPolicy": "open"
    }
  },
  "tools": {
    "web": {
      "search": {
        "provider": "brave",
        "apiKey": "${BRAVE_API_KEY}",
        "maxResults": 5,
        "timeoutSeconds": 30,
        "cacheTtlMinutes": 15
      }
    }
  }
}
```

---

## **⚠️ Where We Fell Short**

### **1. Security vs Functionality Trade-offs**

**Original Goal:** Read-only container filesystem  
**Reality:** Had to make writable for OpenClaw to function

**Issue:** OpenClaw needs to persist plugin state, sessions, and credentials. Read-only filesystem blocked:

* Plugin auto-enable
* Session storage (`~/.openclaw/agents/main/sessions`)
* OAuth credentials (`~/.openclaw/credentials`)
* Canvas feature
* Cron scheduler

**Solution:** Mounted entire `~/.openclaw-host` directory as read-write  
**Security Impact:** Medium - container can modify its own config  
**Mitigation:** Directory owned by host user (uid 1000), not root

---

### **2. Permission Challenges**

**Problem:** Volume mount ownership mismatches caused EACCES errors throughout deployment

**Root Causes:**

* Individual file mounts created root-owned files
* Running `docker exec -u 0` (as root) created root-owned directories
* User 1000 inside container couldn't write to root-owned paths

**Final Solution:**

```bash
# Mount entire directory (not individual files)
- ~/.openclaw-host:/home/node/.openclaw:rw

# Fix volume ownership when needed
docker run --rm -v openclaw_openclaw-data:/data alpine chown -R 1000:1000 /data
```

**Lesson:** When using non-root containers, mount directories (not files) and ensure host ownership matches container UID

---

### **3. Slack Configuration Complexity**

**Issues Encountered:**

1. **Socket Mode + Event Subscriptions confusion**
   * Initially enabled Event Subscriptions (which requires public URL)
   * Actually needed Socket Mode (for private deployments)
   * Both were enabled at once, causing confusion
2. **Missing scopes**
   * Required: `app_mentions:read`, `chat:write`, `channels:read`, `channels:history`
   * Initially missing `app_mentions:read` - Vera couldn't hear @mentions
3. **Token truncation**
   * App Token was accidentally truncated in `.env` (missing last digit)
   * Caused `invalid_auth` errors
   * Hard to debug because tokens are partially hidden
4. **Channel allowlist policy**
   * Default `groupPolicy: "allowlist"` blocked all messages
   * Changed to `groupPolicy: "open"` to allow all channels

**Lesson:** Slack app setup requires careful attention to Socket Mode vs Event Subscriptions, complete token copying, OAuth scopes, and channel policies.

---

### **4. Non-Essential Features Disabled**

**Failed to start (non-critical):**

* ❌ Emoji reactions (`missing_scope: reactions:write`)
* ❌ Status updates (`missing_scope: users:write`)

> **Note:** Canvas and Cron scheduler were previously failing with EACCES errors but are now operational following volume and permission fixes.

**Should we fix?** Optional - low priority

---

## **📚 Lessons Learned**

### **1. OpenClaw Architecture Insights**

**Discovery:** OpenClaw expects to manage its own state directory with full read-write access

* Cannot use fully read-only filesystems
* Must trust OpenClaw to manage `~/.openclaw` directory
* Config file (`openclaw.json`) is both input AND output (OpenClaw modifies it)

```yaml
# Don't do this (individual file mounts)
volumes:
  - ./openclaw.json:/home/node/.openclaw/openclaw.json:ro  # ❌ Breaks state persistence

# Do this (directory mount)
volumes:
  - ~/.openclaw-host:/home/node/.openclaw:rw  # ✅ Allows state management
```

---

### **2. Docker Volume Ownership Pattern**

**Problem:** Non-root containers + volume mounts = permission hell

```bash
# 1. Create host directory owned by your user
mkdir -p ~/.openclaw-host
chmod 700 ~/.openclaw-host

# 2. Mount directory (not files) with matching UID
services:
  openclaw:
    user: "1000:1000"  # Matches host user UID
    volumes:
      - ~/.openclaw-host:/home/node/.openclaw:rw

# 3. If volumes get root-owned, fix with Alpine:
docker run --rm -v volume_name:/data alpine chown -R 1000:1000 /data
```

---

### **3. Environment Variable Management**

**Critical lessons:**

* Docker Compose reads `.env` automatically, but only for variables explicitly listed in `docker-compose.yml`
* Always use full `down` + `up` cycle (not just `restart`) to pick up `.env` changes
* Always verify env vars are actually inside the container after restart:

```bash
docker exec openclaw-test printenv | grep KEY_NAME
```

* YAML is indentation-sensitive — always validate `docker-compose.yml` after manual edits
* Brave API keys start with `BSA` — use as quick validation check

---

### **4. Debugging Strategy That Worked**

**Effective troubleshooting order:**

1. **Check logs first** (`docker logs openclaw-test`)
2. **Verify config syntax** (`docker exec ... node dist/index.js doctor`)
3. **Check environment variables** (`docker exec ... printenv`)
4. **Test API directly** (bypass OpenClaw complexity)
5. **Use diagnostic commands** (`docker exec ... node dist/index.js config get channels.slack`)

---

### **5. Token Management**

**Critical insight:** Always verify the full token, not just the prefix

```bash
# Verify in container
docker exec openclaw-test printenv BRAVE_API_KEY
docker exec openclaw-test printenv | grep SLACK
```

---

## **🚀 Current Operational Status**

### **What Works**

✅ Vera responds to @mentions in Slack  
✅ Claude Haiku 4.5 generates responses  
✅ Fallback models configured (Sonnet, Kimi K2.5)  
✅ API spending tracked ($10 limit enforced)  
✅ Socket Mode real-time messaging  
✅ SOUL.md security constraints active  
✅ Gateway accessible on localhost only  
✅ Container runs as non-root  
✅ Gmail read/send via `gog` CLI  
✅ Google Calendar access  
✅ Brave Search web search active  
✅ Daily AI & Security News Summary cron job running  
✅ Daily memory logs (memory/YYYY-MM-DD.md)  
✅ Weekly memory review cron (Fridays 4pm PST)  
✅ Canvas feature operational  
✅ Cron scheduler operational  

### **What Doesn't Work (Non-Critical)**

⚠️ Emoji reactions (missing scope: `reactions:write`)  
⚠️ Status updates "typing..." (missing scope: `users:write`)  

### **Usage Pattern**

* **Trigger:** @mention `@Vera Bot` in any Slack channel
* **Response:** Vera replies in ~3-8 seconds
* **Cost:** ~$0.0001 per simple message (Haiku pricing)
* **Logs:** Real-time in terminal showing all activity

---

## **🔐 Security Posture**

### **Strengths**

✅ No session credential exposure  
✅ API key with spending limit  
✅ Network isolation (loopback only)  
✅ Non-root execution  
✅ Environment-based secrets  
✅ SOUL.md behavioral constraints  
✅ DNS hardened (8.8.8.8 / 8.8.4.4)  

### **Weaknesses**

⚠️ Config directory is writable (required for functionality)  
⚠️ No Cloudflare Workers proxy yet (API key exposed in container)  

### **Threat Model**

* **Container escape:** Mitigated by non-root user, read-only where possible
* **Network exposure:** Mitigated by loopback-only binding
* **Credential theft:** Mitigated by environment variables, but still in container
* **API abuse:** Mitigated by $10 spending limit
* **Prompt injection:** Mitigated by SOUL.md constraints

---

## **📊 Resource Usage**

* **Docker image:** ~6.36 GB
* **API costs so far:** ~$0.001 (testing)
* **Memory limit:** 2GB (not enforced yet)
* **CPU limit:** 1.0 core (not enforced yet)
* **Disk (volumes):** ~50MB (`openclaw-data` volume)

---

## **🛠️ Commands Reference**

### **Daily Operations**

```bash
# Start Vera
cd ~/openclaw
docker-compose -f docker-compose.simple.yml up -d

# Stop Vera
docker-compose -f docker-compose.simple.yml down

# View logs
docker logs -f openclaw-test

# Check status
docker ps | grep openclaw
```

### **Troubleshooting**

```bash
# Restart from scratch (picks up .env changes)
docker-compose -f docker-compose.simple.yml down
docker-compose -f docker-compose.simple.yml up -d

# Verify environment variables in container
docker exec openclaw-test printenv | grep BRAVE
docker exec openclaw-test printenv | grep SLACK

# Fix permissions
docker run --rm -v openclaw_openclaw-data:/data alpine chown -R 1000:1000 /data

# Check config
docker exec -it openclaw-test node dist/index.js config get channels.slack

# Run doctor
docker exec -it openclaw-test node dist/index.js doctor --fix
```

### **Testing**

```bash
# Test Anthropic API directly (bypass OpenClaw)
curl https://api.anthropic.com/v1/messages \
  -H "x-api-key: $(cat ~/openclaw/.env | grep ANTHROPIC_API_KEY | cut -d'=' -f2)" \
  -H "anthropic-version: 2023-06-01" \
  -H "content-type: application/json" \
  -d '{"model":"claude-haiku-4-5-20251001","max_tokens":100,"messages":[{"role":"user","content":"test"}]}'

# Test Brave Search API directly
curl -H "X-Subscription-Token: YOUR_BRAVE_KEY" \
  "https://api.search.brave.com/res/v1/web/search?q=test"
```

---

## **🎯 Next Steps (Phase 3 Options)**

### **Option A: Cloudflare Workers Proxy (Recommended)**

**Goal:** Hide real API key from container

```
OpenClaw → Worker (https://your-worker.workers.dev) → Anthropic API
           ↑
      Holds real API key
      Rate limiting
      Request validation
```

**Benefits:** API key never touches container, centralized rate limiting, easy key rotation

---

### **Option B: Add Missing Slack Scopes (Polish)**

**Goal:** Enable emoji reactions and status updates

1. Add scopes: `reactions:write`, `users:write`
2. Reinstall app to workspace
3. Restart OpenClaw

---

### **Option C: Production Hardening**

* Add resource limits to docker-compose (memory/CPU)
* Set up log rotation
* Add health check endpoint monitoring
* Create backup/restore procedures for `.openclaw-host`
* Set up alerting for spending limits

---

### **Option D: Monitoring & Observability**

* Parse logs for metrics (messages/hour, API costs)
* Dashboard for Vera's activity
* Alert on errors or unusual behavior
* Track spending vs. $10 limit

---

## **🐛 Known Issues & Workarounds**

### **Issue 1: Config file gets overwritten by doctor**

**Symptom:** `openclaw doctor --fix` modifies `openclaw.json`  
**Workaround:** Keep backup: `cp ~/.openclaw-host/openclaw.json ~/.openclaw-host/openclaw.json.backup`  
**Permanent fix:** Use version control for config

### **Issue 2: Container crashes with "Config invalid"**

**Symptom:** Invalid config format causes immediate exit  
**Diagnosis:** Check logs: `docker logs openclaw-test`  
**Fix:** Verify JSON syntax, check against schema in logs

### **Issue 3: Slack messages not received**

**Symptom:** Vera connected but doesn't respond  
**Check:** Look for "drop message" or "drop channel" in logs  
**Fix:** Ensure `groupPolicy: "open"` in config

### **Issue 4: Env var not picked up after .env edit**

**Symptom:** New key in `.env` not visible inside container  
**Fix:** Must do full `down` + `up` cycle — `restart` alone does not reload `.env` variables

### **Issue 5: YAML syntax error in docker-compose**

**Symptom:** `yaml: line X: did not find expected key`  
**Cause:** Wrong indentation or duplicate content from copy/paste  
**Fix:** Validate indentation carefully — environment variables need 6 spaces of indentation

---

## **📝 Documentation Notes**

### **Files to backup:**

* `~/openclaw/.env` (secrets)
* `~/.openclaw-host/openclaw.json` (config)
* `~/.openclaw-host/workspace/SOUL.md` (agent identity)
* `~/.openclaw-host/workspace/MEMORY.md` (curated memory)
* `~/openclaw/docker-compose.simple.yml` (orchestration)

### **Sensitive data locations:**

* API keys: `~/openclaw/.env`
* Slack tokens: `~/openclaw/.env`
* Gmail OAuth tokens: Docker volume `gog-config`
* Session data: Docker volume `openclaw-data`

### **Log locations:**

* OpenClaw gateway: `/tmp/openclaw/openclaw-YYYY-MM-DD.log` (inside container)
* Docker logs: `docker logs openclaw-test`
* Daily memory logs: `~/.openclaw-host/workspace/memory/YYYY-MM-DD.md`

---

## **✅ Success Criteria Met**

| Requirement | Status | Notes |
| --- | --- | --- |
| API-only auth (no session) | ✅ | Using Anthropic API key |
| Spending limit enforced | ✅ | $10 limit in Anthropic console |
| Slack integration working | ✅ | Responds to @mentions |
| Security-first design | ✅ | SOUL.md + container isolation |
| Non-root container | ✅ | Runs as uid 1000 |
| Network isolation | ✅ | Loopback-only binding |
| Agent responds correctly | ✅ | Vera identifies herself, follows SOUL.md |
| Gmail integration | ✅ | Read/send via gog CLI |
| Web search | ✅ | Brave Search API active |
| Daily memory logging | ✅ | Automated daily logs + weekly review |
| Cron jobs | ✅ | Daily news summary + weekly memory review |

---

**Phase 1 Status: ✅ COMPLETE**  
**Phase 2 Status: ✅ COMPLETE**  
**Deployment Date:** February 8, 2026  
**Last Updated:** February 17, 2026  
**Vera Status:** 🟢 Operational  
**Ready for:** Phase 3 (Cloudflare Workers proxy or production hardening)
