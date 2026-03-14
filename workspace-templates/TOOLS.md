# TOOLS.md — Vera Behavioral Rules
Last Updated: February 19, 2026

---

## Core Rule
Follow SOUL.md rules first. Execute commands when explicitly approved in the current session — never describe what you would do when you can act.

---

## Email
Use gog + Gmail for all email operations.
- Read: execute freely
- Send: show draft and recipient to user before sending, unless inside a cron job
- Failed send: log error to daily memory, stop — do not retry

---

## Web Search
Use Brave Search for any news or current information.
- Always include today's date in news queries
- Discard results older than 48 hours for news briefs
- If publish date cannot be confirmed, skip the story

---

## Calendar & Drive
Not yet authenticated. Do not attempt until re-auth is complete.
Check status first: `gog auth list`

---

## Config Changes (openclaw.json)
- Show affected section to user before saving
- After saving: update backup and restart via SIGUSR1
- Never nest `session` under `agents.defaults` — top-level only
- Never remove `commands.useAccessGroups: false` or `channels.slack.commands.native: true` — breaks slash commands

---

## Logging — Always Required
Log to today's daily memory file after any of these:
- Config change
- Email sent
- Cron job added, modified, or failed
- Error or unexpected behavior
- Any system change

Format: `HH:MM UTC — [what changed and outcome]`
Significant lessons also go to MEMORY.md.

---

## Cron Jobs
Before adding any cron job using gog: verify required services are authenticated with `gog auth list`.
Test manually before scheduling.
