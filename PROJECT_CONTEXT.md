# N8n Automation Project - Context

## Quick Resume
After restarting Claude Code, say:
> "Continue working on my n8n workflows. Read PROJECT_CONTEXT.md for context."

---

## Workflows Overview

| Workflow | ID | Status | Purpose |
|----------|----|--------|---------|
| Email-to-Telegram | `arJbFrfKsJguy9tp` | Active | AI email classification + Telegram alerts |
| Stock Market Alert | `782rqSIRL2xZ3wqv` | Active | US + TSX stock price alerts |
| Error Alert | `Fhmq0omGZpu98HN3` | Active | Sends Telegram on any workflow failure |

**n8n Instance**: https://n8n.srv1299342.hstgr.cloud

---

## 1. Email-to-Telegram Important Summary

### Goal
Monitor Gmail for new emails, classify importance with AI (Google Gemini), send Telegram notifications for HIGH/MEDIUM priority only, with a direct link to open the email.

### Decisions

| Decision | Choice |
|----------|--------|
| Email Provider | Gmail (OAuth2) - multiple accounts |
| AI Service | Google Gemini 2.5 Flash (free tier) |
| AI Node | Basic LLM Chain (not Agent - Agent requires tools) |
| Messaging | Telegram Bot (N8n517bot) |
| Importance Criteria | Business-critical, Action-required, VIP senders |
| Output Parsing | Code node with JSON.parse (not Structured Output Parser - incompatible with Gemini) |

### Architecture
```
Gmail Trigger(s) → Prepare Data → LLM Chain (Gemini) → Filter (HIGH/MEDIUM only) → Format Message → Telegram
```

### Nodes (8)
1. **Gmail Trigger** - Poll every minute for unread emails (Account 1: phil@yufora.com)
2. **Gmail Trigger1** - Poll every minute for unread emails (Account 2 - needs connecting)
3. **Prepare Email Data** (Set) - Extract sender, subject, body preview, emailId, recipientEmail
4. **Google Gemini** - Gemini 2.5 Flash model (sub-node connected to LLM Chain)
5. **AI Email Classifier** (Basic LLM Chain) - Classifies email, returns JSON
6. **Filter Important Only** (IF) - Pass only HIGH/MEDIUM priority
7. **Format Telegram Message** (Code) - Rich formatting with emojis + Gmail deep link
8. **Send Telegram Alert** - Sends to chat ID 8469449994

### Classification Framework

**Priority**: HIGH (urgent, invoice, payment, contract, legal, emergency) | MEDIUM (meeting, call, review, feedback, approval) | LOW (newsletter, promotion, automated - no alert)

**Categories**: Client, Invoice, Meeting, Support, Newsletter, Notification, Personal, Other

---

## 2. Stock Market Alert

### Goal
Monitor US and Toronto (TSX/TSXV) stocks from a Google Sheets watchlist, check live prices, send Telegram alerts when stocks cross upper/lower price limits with cooldown to prevent spam.

### Decisions

| Decision | Choice |
|----------|--------|
| US stocks API | Twelve Data (free: 800 calls/day, 8/min) |
| Canadian stocks API | Yahoo Finance (free, unlimited) |
| Watchlist | Google Sheets (Sheet ID: `18nROXFSRms0OYLcl3Eye7bJ_iB3bdkusydLI4vm8rMU`) |
| Alerts | Telegram (reuse N8n517bot) |
| Polling | Every 5 min, Mon-Fri 9:30-16:00 ET |
| Timezone | America/Toronto |

### Architecture
```
Schedule Trigger (market hours) → Google Sheets → Parse Watchlist → Fetch Price (HTTP) → Smart Alert Logic → IF alert → Format + Telegram → Update Sheet
```

### Nodes (9)
1. **Market Hours Trigger** (Schedule) - Every 5 min, Mon-Fri, 9:30-16:00 ET
2. **Read Stock Watchlist** (Google Sheets) - Read watchlist
3. **Parse Watchlist Data** (Code) - Validate rows, route to correct API (Twelve Data for US, Yahoo Finance for Canadian)
4. **Fetch Live Stock Price** (HTTP Request) - URL from `$json.apiUrl`, User-Agent header for Yahoo
5. **Smart Alert Logic** (Code) - Compare price vs limits, check cooldown, handle both API response formats
6. **Filter Alerts** (IF) - Pass only `shouldAlert === 'true'`
7. **Format Alert Message** (Code) - Rich Telegram message with price/change/limits
8. **Send Telegram Alert** - To chat 8469449994
9. **Update Alert History** (Google Sheets) - Write back last_alert_price/time

### Google Sheets Watchlist Columns
`symbol | upper_limit | lower_limit | direction | cooldown_minutes | last_alert_price | last_alert_time`

### Stock Symbol Formats
- US: `AAPL`, `TSLA`, `MSFT`
- Toronto (TSX): `SHOP.TO`, `TD.TO`, `RY.TO`
- Toronto (TSXV): `ODV.V`
- Note: Twelve Data uses `.TRT`/`.TRV` suffixes but small-cap Canadian stocks are behind their paywall — Yahoo Finance handles all Canadian stocks for free

### Dual-API Routing
The Parse Watchlist Data node pre-computes `apiUrl` per stock:
- Canadian (`.TO`, `.V`, `.TRT`, `.TRV`) → Yahoo Finance: `https://query1.finance.yahoo.com/v8/finance/chart/SYMBOL?interval=1d&range=1d`
- US → Twelve Data: `https://api.twelvedata.com/quote?symbol=SYMBOL&apikey=KEY`

The Smart Alert Logic node handles both response formats (Yahoo's `chart.result[0].meta` vs Twelve Data's flat `close`/`change`).

---

## 3. Error Alert Workflow

### Architecture
```
Error Trigger → Format Error Message (Code) → Send Telegram Error Alert
```

Linked to other workflows via `errorWorkflow` setting. Sends formatted Telegram message with workflow name, error message, timestamp, and execution link.

---

## Key Technical Lessons

| Issue | Root Cause | Fix |
|-------|-----------|-----|
| Agent node fails | Requires at least one tool sub-node | Use Basic LLM Chain instead |
| Structured Output Parser fails with Gemini | Gemini wraps JSON in markdown code blocks | Parse JSON in Code node with regex fallback |
| Expression format errors | n8n requires `=` prefix on mixed expressions | Use `n8n_autofix_workflow` tool |
| Gmail field mapping | `simple: true` returns flat fields (`$json.From`) not nested | Use `$json.From`, `$json.Subject`, `$json.snippet` |
| Telegram `operation` lost on update | Full workflow update via API drops it | Always re-set `operation: "sendMessage"` |
| IF node boolean vs string | `typeValidation: "strict"` rejects boolean for string comparison | Output `shouldAlert` as string `'true'`/`'false'` |
| Canadian stocks 404 on Twelve Data | Small-cap TSX/TSXV behind paywall | Use Yahoo Finance for all Canadian stocks |
| n8n editor not reflecting API changes | Editor caches workflow in memory | Refresh browser after MCP API updates |
| Cron running at wrong time | Server timezone Europe/Berlin | Set `settings.timezone: "America/Toronto"` |
| `n8n_update_partial_workflow` syntax | `updateNode` needs `updates` key not `properties` | Use `updates` key |

---

## Credentials Configured

| Service | Credential | n8n Credential ID |
|---------|-----------|-------------------|
| Gmail (Account 1) | Gmail OAuth2 (phil@yufora.com) | QKqspapsDqWJkzDa |
| Gmail (Account 2) | Gmail OAuth2 (2nd account) | Configured in UI |
| Google Gemini | Google PaLM API | x4jAdG2mahUmKbmW |
| Telegram | Telegram API (N8n517bot) | KSDajeVj98U3LS8y |
| Google Sheets | Google Sheets OAuth2 (separate account) | Configured in UI |
| Twelve Data | HTTP Query Auth (`TwelveApiKey`) | Configured in UI |

---

## Project Files

| File | Purpose |
|------|---------|
| `workflow_email.json` | Clean export of Email-to-Telegram workflow (credentials stripped) |
| `workflow_stock.json` | Clean export of Stock Market Alert workflow (credentials stripped) |
| `workflow_error.json` | Clean export of Error Alert workflow (credentials stripped) |
| `PROJECT_CONTEXT.md` | This file |
| `SETUP.md` | Step-by-step setup guide |
| `.env.example` | Credential template with instructions |
| `.env` | Actual credentials (private, gitignored) |
| `.mcp.json` | MCP config for n8n API (private, gitignored) |
| `.gitignore` | Protects secrets from git |
| `twelvedata_canadian_stocks.csv` | TSX stocks list from Twelve Data (1,108) |
| `twelvedata_canadian_venture_stocks.csv` | TSXV stocks list from Twelve Data (1,700) |

**GitHub**: https://github.com/Philnow/n8n-email-telegram

---

## Pending / Future

1. **Connect Gmail Trigger1** → Prepare Email Data (drag connection in n8n UI)
2. Consider testing AI reliability with sample emails
3. Consider adding more stock alert features (volume alerts, daily summary)
