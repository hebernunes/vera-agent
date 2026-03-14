# Gmail Integration Guide

**Status:** ✅ Complete and Tested
**Prerequisites:** OpenClaw deployed in Docker with Slack integration working

---

## Overview

OpenClaw does not have a Gmail plugin. Instead, Vera uses the `gog` CLI tool — a Go binary she executes via her `exec` capability. This guide covers building `gog` from source inside the container, authenticating with Google OAuth, and verifying the integration.

**What you'll achieve:**
- ✅ Vera can read Gmail emails
- ✅ Vera can send emails (with approval)
- ✅ Vera can manage Google Calendar events (optional)
- ✅ Vera can access Google Drive files (optional)

**Time required:** ~45-60 minutes

---

## Architecture

```
OpenClaw Container
│
├── Vera (Agent)
│       │
│       │ exec
│       ▼
├── gog CLI
│       │
│       │ OAuth 2.0 / HTTPS
│       ▼
└── Google APIs (Gmail / Calendar / Drive)
```

---

## Part 1 — Google Cloud Console Setup

### 1.1 Create a Google Cloud Project

1. Go to [console.cloud.google.com](https://console.cloud.google.com/)
2. Sign in with your sandbox Gmail account
3. Click project dropdown → **NEW PROJECT**
4. Name it anything (e.g. `vera-ai-agent`)
5. Click **CREATE**, wait ~30 seconds, then select the new project

### 1.2 Enable Gmail API

1. Search for `Gmail API` in the top search bar
2. Click **Gmail API** → **ENABLE**

Repeat for optional APIs:
- Google Calendar API
- Google Drive API
- People API (Contacts)

### 1.3 Configure OAuth Consent Screen

1. Left sidebar → **OAuth consent screen**
2. User Type: **External** → **CREATE**
3. Fill in app name and contact email (use your sandbox account)
4. Click **SAVE AND CONTINUE**
5. Add scopes:
   - `gmail.readonly`
   - `gmail.send`
   - `gmail.modify`
   - `calendar` (if needed)
   - `drive` (if needed)
6. Add your sandbox account as a **Test User**
7. Click **BACK TO DASHBOARD**

### 1.4 Create OAuth 2.0 Credentials

1. Left sidebar → **Credentials**
2. **+ CREATE CREDENTIALS** → **OAuth client ID**
3. Application type: **Desktop app**
4. Click **CREATE**
5. **DOWNLOAD JSON** — save the file

### 1.5 Move Credentials into OpenClaw

```bash
mv ~/Downloads/client_secret_*.json ~/.openclaw-host/gmail_credentials.json
```

---

## Part 2 — Install Go Inside the Container

`gog` is a Go application that must be built from source. Go 1.21+ is required — the Debian package is too old.

```bash
docker-compose -f ~/openclaw/docker-compose.simple.yml exec -u 0 openclaw bash -c \
  "cd /tmp && curl -LO https://go.dev/dl/go1.23.5.linux-amd64.tar.gz \
  && rm -rf /usr/local/go \
  && tar -C /usr/local -xzf go1.23.5.linux-amd64.tar.gz \
  && ln -sf /usr/local/go/bin/go /usr/bin/go \
  && go version"
```

Expected output: `go version go1.23.5 linux/amd64`

---

## Part 3 — Build gog CLI

Send this to Vera in Slack:

```
@Vera Bot please build the gog CLI tool:

1. Clone: git clone https://github.com/steipete/gogcli.git /tmp/gogcli
2. Build: cd /tmp/gogcli && go build -o ./bin/gog ./cmd/gog
3. Install: mkdir -p /home/node/.local/bin && cp /tmp/gogcli/bin/gog /home/node/.local/bin/gog && chmod +x /home/node/.local/bin/gog
4. Test: /home/node/.local/bin/gog --version

Run each step and report the result.
```

Expected: Vera reports `gog --version` returns a version number.

---

## Part 4 — OAuth Authentication

### 4.1 Register credentials with gog

Ask Vera in Slack:

```
@Vera Bot set up Gmail authentication:

1. Check credentials file: ls -la /home/node/.openclaw/gmail_credentials.json
2. Register with gog: /home/node/.local/bin/gog auth credentials /home/node/.openclaw/gmail_credentials.json
3. Start auth flow: GOG_HTTP_TIMEOUT=120 /home/node/.local/bin/gog auth add your-sandbox@gmail.com --services gmail --no-browser

Share the OAuth URL from step 3.
```

### 4.2 Complete the OAuth Flow

1. Vera prints an OAuth URL — copy it
2. Open in your browser
3. Sign in with your sandbox Gmail account
4. Click through the "unverified app" warning (expected for test apps)
5. Grant permissions
6. Browser redirects to `http://localhost:1/?code=...` — page shows ERR_CONNECTION_REFUSED, **this is expected**
7. Copy the full URL from the browser address bar
8. Send the full URL back to Vera in Slack

Vera pastes the code into the waiting `gog` prompt. Auth completes.

> **Why `--no-browser`?** The OAuth callback goes to `localhost:1` inside the container, which is not your Mac's localhost. Manual code entry is required for Docker deployments.

> **Why `GOG_HTTP_TIMEOUT=120`?** Docker networking can be slow during OAuth token exchange. 120 seconds prevents `context deadline exceeded` errors.

---

## Part 5 — Verify Integration

### Check auth status
```bash
docker exec -it openclaw-test /home/node/.local/bin/gog auth list
```

### Test Gmail read
Ask Vera: `@Vera Bot check my Gmail inbox for unread messages from the last 7 days`

### Test Gmail send
Ask Vera: `@Vera Bot send a test email to your-sandbox@gmail.com with subject "Test from Vera" and body "Integration test."`

---

## Making gog Persistent

By default `gog` is installed inside the running container and lost on rebuild. The `docker-compose.simple.yml` template includes a `local-bin` volume that persists the binary:

```yaml
volumes:
  - local-bin:/home/node/.local/bin:rw
```

OAuth tokens are persisted in the `gog-config` volume:

```yaml
volumes:
  - gog-config:/home/node/.config:rw
```

With these volumes in place, you only need to re-authenticate if the `gog-config` volume is deleted or the OAuth token is revoked.

---

## gog CLI Reference

```bash
# Gmail
gog gmail search 'query' --max 20
gog gmail get <message-id>
gog gmail send --to user@example.com --subject "Subject" --body "Body"
gog gmail modify <message-id> --add-labels LABEL --remove-labels LABEL

# Calendar
gog calendar calendars
gog calendar events <calendar-id> --from <iso-date> --to <iso-date>
gog calendar create <calendar-id> --summary "Title" --from <iso-date> --to <iso-date>

# Drive
gog drive ls --max 20
gog drive search "query"
gog drive download <file-id>

# Auth
gog auth list
gog auth add email@gmail.com --services gmail,calendar,drive
gog auth remove email@gmail.com
gog auth status
```

---

## Troubleshooting

See [`gmail-oauth-troubleshooting.md`](./gmail-oauth-troubleshooting.md) for common errors including:
- `context deadline exceeded` during OAuth
- `Error 400: invalid_scope`
- `insufficient permissions`
- Credentials file not found
