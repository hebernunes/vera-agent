# Troubleshooting

Consolidated reference for all known issues with the OpenClaw/Vera Docker deployment.

---

## Issue 1 — `openclaw doctor --fix` overwrites config

**Symptom:** Running `doctor --fix` modifies `openclaw.json` and removes custom settings.

**Rule:** Never run `doctor --fix` without a fresh timestamped backup first.

```bash
cp ~/.openclaw-host/openclaw.json ~/.openclaw-host/openclaw.json.backup-$(date +%Y%m%d)
docker exec -it openclaw-test node dist/index.js doctor
# No --fix flag
```

---

## Issue 2 — Container crashes with "Config invalid"

**Symptom:** Container exits immediately on startup with config error in logs.

**Diagnosis:**
```bash
docker logs openclaw-test
```

**Fix:** Verify JSON syntax. Common causes:
- Trailing comma in JSON
- `session` nested under `agents.defaults` instead of top-level
- Invalid key name (e.g. `reasoning` instead of `thinkingDefault`)

---

## Issue 3 — Slack messages not received

**Symptom:** Vera is connected but doesn't respond to `@Vera Bot` mentions.

**Check logs for:**
```bash
docker logs openclaw-test | grep -i "drop\|channel\|group"
```

**Fix:** Ensure `groupPolicy: "open"` in `openclaw.json`:
```json
"channels": {
  "slack": {
    "groupPolicy": "open"
  }
}
```

---

## Issue 4 — Env var not picked up after `.env` edit

**Symptom:** New key added to `.env` is not visible inside the container.

**Cause:** `docker-compose restart` does not reload `.env` — only a full recreate does.

**Fix:**
```bash
docker-compose -f docker-compose.simple.yml down
docker-compose -f docker-compose.simple.yml up -d

# Verify
docker exec openclaw-test printenv | grep YOUR_KEY_NAME
```

---

## Issue 5 — YAML syntax error in docker-compose

**Symptom:** `yaml: line X: did not find expected key`

**Cause:** Wrong indentation or duplicate content from copy/paste.

**Fix:** Environment variables under `environment:` need consistent indentation. Validate with:
```bash
docker-compose -f docker-compose.simple.yml config
```

---

## Issue 6 — `session` key causes startup error

**Symptom:** Container fails with `Unrecognized key: "session"` under `agents.defaults`.

**Cause:** `session` must be a top-level key, not nested under `agents.defaults`.

**Wrong:**
```json
{
  "agents": {
    "defaults": {
      "session": { ... }  ❌
    }
  }
}
```

**Correct:**
```json
{
  "agents": { ... },
  "session": {           ✅
    "reset": {
      "mode": "idle",
      "idleMinutes": 480
    }
  }
}
```

---

## Issue 7 — Slash commands return `dispatch_failed`

**Symptom:** `/reset`, `/new`, `/context`, `/model`, `/help` all fail with `dispatch_failed`.

**Cause:** Two required fields missing from `openclaw.json`.

**Fix:** Both fields must be present:
```json
{
  "commands": {
    "useAccessGroups": false
  },
  "channels": {
    "slack": {
      "commands": {
        "native": true
      }
    }
  }
}
```

---

## Issue 8 — GitHub PAT token expiry

**Symptom:** Nightly git backup fails. `#git-backup` Slack channel shows authentication error.

**Fix:**
1. Go to GitHub → Settings → Developer settings → Personal access tokens
2. Regenerate token with `repo` scope
3. Update `~/openclaw/.env` with new token
4. Full restart to pick up `.env` change:
```bash
docker-compose -f docker-compose.simple.yml down
docker-compose -f docker-compose.simple.yml up -d
```
5. Update git remote URL inside container:
```bash
docker exec openclaw-test bash -c \
  'cd /home/node/.openclaw/workspace && git remote set-url origin \
  https://your-username:'"${GITHUB_TOKEN}"'@github.com/your-username/your-workspace-repo.git'
```
6. Test: `docker exec openclaw-test bash -c 'cd /home/node/.openclaw/workspace && git push origin main'`

---

## Issue 9 — Cron jobs silently not firing

**Symptom:** Cron jobs show as active in config but never execute. No error output.

**Cause:** `wakeMode: "next-heartbeat"` does nothing when heartbeat is disabled.

**Fix:** Set `wakeMode: "now"` on every cron job:
```json
{
  "wakeMode": "now"
}
```

**Rule:** Always use `wakeMode: "now"` when heartbeat is disabled.

---

## Issue 10 — Reasoning conflict on Anthropic models (v2026.2.23+)

**Symptom:** 400 errors on all Anthropic model calls after an upgrade.

**Cause:** `reasoning` / `reasoning_effort` conflict introduced in newer OpenClaw builds.

**Fix:** Add to `agents.defaults` in `openclaw.json`:
```json
"agents": {
  "defaults": {
    "thinkingDefault": "off"
  }
}
```

**Note:** Only `thinkingDefault` is valid. Keys `thinking`, `reasoning`, and `reasoningLevel` all cause errors.

---

## Issue 11 — Copilot token missing after container rebuild

**Symptom:** `/model github-copilot/claude-haiku-4.5` returns auth error after a full image rebuild.

**Cause:** The Copilot token is written to `openclaw.json` on the host mount and survives most restarts, but may need re-auth after a full image rebuild.

**Fix:**
```bash
docker exec -it -u 1000 openclaw-test node /app/dist/index.js models auth login-github-copilot
```

Re-auth takes ~60 seconds via the device code flow.

---

## Issue 12 — `docker-compose down` fails on unset optional env vars

**Symptom:** `docker-compose down` errors with unset variable warnings (`OPENCLAW_CONFIG_DIR`, `OPENCLAW_WORKSPACE_DIR`, etc.).

**Cause:** Optional env vars referenced in compose file but not set in `.env`.

**Workaround:** Use `docker restart` for config reloads in this case:
```bash
docker restart openclaw-test
```

**Note:** For `.env` changes, full `down` + `up` is still required — `docker restart` does not reload `.env`.

---

## Issue 13 — New models not appearing after SIGUSR1

**Symptom:** Added new models to `openclaw.json`, sent SIGUSR1, but `/model list` shows wrong count.

**Cause:** SIGUSR1 only reloads parameter changes. New model registrations require a full restart.

**Fix:**
```bash
docker-compose -f docker-compose.simple.yml down
docker-compose -f docker-compose.simple.yml up -d
```

**Rule:**
- SIGUSR1 → parameter changes only (cache retention, session config)
- Full `down && up` → new model registrations, new config keys, `.env` changes

---

## General Debugging Order

When something breaks, always follow this order:

1. **Check logs first** → `docker logs openclaw-test`
2. **Verify config syntax** → `docker exec -it openclaw-test node dist/index.js doctor`
3. **Check environment variables** → `docker exec openclaw-test printenv | grep KEY_NAME`
4. **Test API directly** → bypass OpenClaw to isolate the issue
5. **Check model routing** → `docker exec openclaw-test cat /tmp/openclaw-1000/openclaw-$(date +%Y-%m-%d).log | grep provider`
6. **Restore from backup** → if config is corrupted

---

## Volume Ownership Fix

If you see `EACCES` permission errors on volume mounts:

```bash
docker run --rm -v openclaw_<volume-name>:/data alpine chown -R 1000:1000 /data
```

Replace `<volume-name>` with the affected volume: `openclaw-data`, `local-bin`, `go-install`, or `gog-config`.
