# OpenClaw Update Protocol

**Rule:** Always update to a stable tagged release. Never run `git pull` on main.

---

## Release Channels

| Channel | Source | Risk |
|---------|--------|------|
| `stable` | Tagged releases (`vYYYY.M.D`) | ✅ Safe for production |
| `beta` | Prerelease tags (`vYYYY.M.D-beta.N`) | ⚠️ Some rough edges |
| `dev` | Moving head of `main` branch | ❌ Avoid — unstable |

---

## Correct Update Process

```bash
# Step 1 — Backup everything first
cp -r ~/openclaw ~/openclaw-backup-$(date +%Y%m%d)
cp -r ~/.openclaw-host ~/.openclaw-host-backup-$(date +%Y%m%d)

# Step 2 — Fetch latest tags
cd ~/openclaw
git fetch --tags

# Step 3 — List available stable tags
git tag --sort=-version:refname | grep -v beta | grep -v dev | head -5

# Step 4 — Checkout the latest stable tag
git checkout vYYYY.M.D

# Step 5 — Rebuild
docker-compose -f docker-compose.simple.yml down
docker-compose -f docker-compose.simple.yml build --no-cache
docker-compose -f docker-compose.simple.yml up -d

# Step 6 — Verify version
docker exec openclaw-test node dist/index.js --version

# Step 7 — Run doctor (no --fix flag)
docker exec -it openclaw-test node dist/index.js doctor
```

---

## Post-Update Verification Checklist

Confirm all of these in Slack before considering the update complete:

- [ ] `/status` — Vera responds normally
- [ ] `/model list` — all registered models showing
- [ ] `@Vera Bot you there?` — clean response, no error codes
- [ ] `docker logs openclaw-test --tail 20` — no config errors
- [ ] Cron jobs still active (check `/status` output)
- [ ] Gmail auth still valid: `gog auth list`
- [ ] Model auth status: `docker exec -it openclaw-test node /app/dist/index.js models status`

---

## Config Integrity Check

Run before and after every upgrade:

```bash
docker exec -u 1000 openclaw-test sh -lc \
  'cd /home/node/.openclaw && jq "{
    primary: .agents.defaults.model.primary,
    thinkingDefault: .agents.defaults.thinkingDefault,
    session_location: (if .session then "top-level OK" else "missing" end),
    useAccessGroups: .commands.useAccessGroups,
    slashCommands: .channels.slack.commands.native
  }" openclaw.json'
```

**Expected values:**
- `thinkingDefault`: `"off"`
- `session_location`: `"top-level OK"`
- `useAccessGroups`: `false`
- `slashCommands`: `true`

---

## Known Pitfalls

### Running `git pull` on main
Pulling main can jump hundreds of commits and introduce unstable dev changes. Always use a specific stable tag.

### `thinkingDefault` key (required v2026.2.23+)
Add to `agents.defaults` to prevent reasoning/reasoning_effort conflict on Anthropic models:

```json
"agents": {
  "defaults": {
    "thinkingDefault": "off"
  }
}
```

Only `thinkingDefault` is valid. Keys `thinking`, `reasoning`, and `reasoningLevel` all cause errors.

### MiniMax incompatibility
MiniMax models are incompatible with the global `thinkingDefault` fix in some versions. Leave MiniMax as `{}` in the models section until per-model override is supported.

### `openclaw doctor --fix` is destructive
Never run `doctor --fix` after an upgrade without a fresh backup. It may overwrite your config with defaults.

---

## Rollback Procedure

If the upgrade breaks something:

```bash
# Restore from backup
cd ~/openclaw
git checkout <previous-stable-tag>

docker-compose -f docker-compose.simple.yml down
docker-compose -f docker-compose.simple.yml build --no-cache
docker-compose -f docker-compose.simple.yml up -d

# Restore config if needed
cp ~/.openclaw-host-backup-YYYYMMDD/openclaw.json ~/.openclaw-host/openclaw.json
docker kill --signal=SIGUSR1 openclaw-test
```
