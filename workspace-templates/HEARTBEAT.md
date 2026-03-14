# HEARTBEAT.md - Periodic Check Tasks

**Last Updated:** February 20, 2026

## Daily Email Report (10:00 AM PST, Mon-Fri only)

**Send email to:** your-sandbox@gmail.com  
**Subject:** "Vera Daily Report for Cowboy - [DATE]"

### Content:
- **💰 API Spending**:
  - Yesterday's cost
  - Week-to-date total
  - Month-to-date total vs $20 limit
  - Percentage of budget used

- **📊 Activity Summary**:
  - Total messages processed (Slack)
  - Tools executed count
  - Token usage (input/output)

- **🚨 Alerts** (if any):
  - Rate limit events
  - Auth errors
  - Unusual activity

- **📅 Calendar**: Today's events (when calendar has data)
- **📬 Gmail**: Unread count + flagged emails (when inbox has activity)

### Alert Thresholds:
- API spending > $16.00: ⚠️ Warning — include in daily email report
- API spending > $19.00: 🚨 Critical — alert immediately via Slack AND email, pause and await manual approval
- Auth failures > 3 in 24h: 🚨 Investigate — alert via Slack
- Container restarts: 🚨 Check logs — alert via Slack

---

## Weekly Summary (Monday 9:00 AM PST)
- Weekly spending summary
- Security posture check
- Gmail/Calendar integration health

---

## Monthly Tasks (1st of month, 9:00 AM PST)
- Monthly API spending report + reset tracking
- Check for OpenClaw updates

---

## Emergency Procedures

**If API spending hits $19.00:**
1. Pause all non-critical operations
2. Alert Cowboy immediately via Slack + Email
3. Await manual approval before resuming

---

## Notes
- Currently using test inbox (your-sandbox@gmail.com)
- Calendar and inbox sections will be minimal until real data added
- Main value: Daily spending tracking and system health monitoring
