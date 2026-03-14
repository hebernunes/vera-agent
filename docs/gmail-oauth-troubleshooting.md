# Gmail OAuth Troubleshooting Guide

**Applies to:** gog CLI + OpenClaw Docker deployment

---

## Quick Reference — Services to Scopes Mapping

When running `gog auth add` with `--services`, these are the OAuth scopes requested:

| Service Flag | Required OAuth Scopes |
|-------------|----------------------|
| `gmail` | `gmail.modify`, `gmail.settings.basic`, `gmail.settings.sharing`, `userinfo.email`, `openid` |
| `calendar` | `calendar` |
| `drive` | `drive` |

---

## Critical Rule — Match Services to Configured Scopes

The `--services` flag must match what is configured in your GCP OAuth Consent Screen. Requesting a service whose scope isn't on the consent screen causes `Error 400: invalid_scope`.

**Check current auth status:**
```bash
/home/node/.local/bin/gog auth list
```

---

## Re-Authentication Commands

### Gmail only (default starting point)
```bash
GOG_HTTP_TIMEOUT=120 /home/node/.local/bin/gog auth add your-sandbox@gmail.com \
  --services gmail --manual
```

### Gmail + Calendar
```bash
GOG_HTTP_TIMEOUT=120 /home/node/.local/bin/gog auth add your-sandbox@gmail.com \
  --services gmail,calendar --manual
```
**Required first:** Add `calendar` scope to GCP OAuth Consent Screen.

### Gmail + Calendar + Drive
```bash
GOG_HTTP_TIMEOUT=120 /home/node/.local/bin/gog auth add your-sandbox@gmail.com \
  --services gmail,calendar,drive --manual
```
**Required first:** Add `calendar` and `drive` scopes to GCP OAuth Consent Screen.

---

## Common Errors

### Error 400: invalid_scope

**Cause:** GCP OAuth Consent Screen is missing scopes that gog is requesting.

**Fix:**
1. Go to GCP Console → OAuth Consent Screen → **EDIT APP**
2. Click **ADD OR REMOVE SCOPES**
3. Add the missing scopes (see table above)
4. Click **UPDATE** → **SAVE AND CONTINUE**
5. Re-run the `gog auth add` command

---

### context deadline exceeded

**Cause:** Docker networking timeout during OAuth token exchange.

**Fix:** Use extended timeout:
```bash
GOG_HTTP_TIMEOUT=120 /home/node/.local/bin/gog auth add your-sandbox@gmail.com \
  --services gmail --manual
```

---

### ERR_CONNECTION_REFUSED at localhost callback

**Cause:** The OAuth callback goes to `localhost:1` inside the container — not your Mac's localhost.

**This is expected behavior.** Copy the full URL from your browser's address bar (including the `code=` parameter) and paste it back to Vera in Slack.

---

### 403 insufficientPermissions

**Cause:** OAuth token was granted without the required scopes.

**Fix:**
1. Remove the existing token: `gog auth remove your-sandbox@gmail.com`
2. Add missing scopes to GCP OAuth Consent Screen (see above)
3. Re-authenticate: `gog auth add your-sandbox@gmail.com --services gmail --manual`

---

### Credentials file not found

**Symptom:** `credentials.json: no such file or directory`

**Fix:**
```bash
# Verify file exists on host
ls -la ~/.openclaw-host/gmail_credentials.json

# Verify it's visible inside container
docker exec openclaw-test ls -la /home/node/.openclaw/gmail_credentials.json

# Register with gog
docker exec -it openclaw-test /home/node/.local/bin/gog auth credentials \
  /home/node/.openclaw/gmail_credentials.json
```

---

### No tokens stored after auth

**Fix — run in order:**
```bash
# 1. Verify credentials file is registered
gog auth credentials /home/node/.openclaw/gmail_credentials.json

# 2. Re-authenticate
GOG_HTTP_TIMEOUT=120 gog auth add your-sandbox@gmail.com --services gmail --manual

# 3. Verify tokens stored
gog auth list
```

---

## Workflow Checklist — Adding a New cog Cron Job

Before adding any cron job that uses gog commands:

- [ ] Identify what gog command it uses (`gog gmail`, `gog calendar`, `gog drive`)
- [ ] Check current authenticated services: `gog auth list`
- [ ] If new service needed: add required scopes to GCP → re-authenticate
- [ ] Test the command manually before scheduling
- [ ] Verify `wakeMode: "now"` is set on the cron job

---

## Diagnostic Commands

```bash
# Check auth status
/home/node/.local/bin/gog auth list

# Check credentials file
ls -la /home/node/.config/gogcli/credentials.json

# Check what client_id is registered
cat /home/node/.config/gogcli/credentials.json | grep client_id
```
