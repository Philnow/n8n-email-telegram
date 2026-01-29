# Email-to-Telegram Important Summary - Project Context

## Quick Resume
After restarting Claude Code, say:
> "Continue building the email-to-telegram workflow. Read PROJECT_CONTEXT.md for context."

---

## Project Goal
Build an n8n workflow that:
1. Monitors Gmail for new emails (multiple accounts)
2. Uses AI (Google Gemini) to classify importance
3. Sends Telegram notifications for important emails only
4. Includes a direct link to open the email in the correct Gmail inbox

## Decisions Made

| Decision | Choice |
|----------|--------|
| Email Provider | Gmail (OAuth2) - supports multiple accounts |
| AI Service | Google Gemini 2.5 Flash (free tier) |
| AI Node | Basic LLM Chain (not Agent - Agent requires tools) |
| Messaging | Telegram Bot (N8n517bot) |
| Importance Criteria | Business-critical, Action-required, VIP senders |
| Output Parsing | Code node with JSON.parse (not Structured Output Parser - incompatible with Gemini) |

## Classification Framework

### Priority Levels
- **HIGH**: urgent, invoice, payment, contract, legal, emergency, deadline, VIP senders
- **MEDIUM**: meeting, call, schedule, review, feedback, approval, questions
- **LOW**: newsletters, promotions, automated notifications (no alert sent)

### Categories
Client, Invoice, Meeting, Support, Newsletter, Notification, Personal, Other

---

## Workflow Architecture

```
Gmail Trigger(s) → Prepare Data → LLM Chain (Gemini) → Filter (HIGH/MEDIUM only) → Format Message → Telegram
```

### Nodes (7 total)
1. **Gmail Trigger** - Poll every minute for unread emails (Account 1: phil@yufora.com)
2. **Gmail Trigger1** - Poll every minute for unread emails (Account 2 - needs connecting to Prepare Email Data)
3. **Prepare Email Data** (Set node) - Extract sender, subject, body preview, emailId, recipientEmail
4. **Google Gemini** - Gemini 2.5 Flash model (sub-node connected to LLM Chain)
5. **AI Email Classifier** (Basic LLM Chain) - Classifies email, returns JSON with priority/category/summary
6. **Filter Important Only** (IF node) - Pass only HIGH/MEDIUM priority
7. **Format Telegram Message** (Code) - Rich formatting with emojis + Gmail deep link with authuser
8. **Send Telegram Alert** - Sends to chat ID 8469449994

---

## Key Technical Decisions & Fixes Applied

### Why Basic LLM Chain instead of Agent
- Agent node **requires at least one tool** sub-node connected
- We don't need tools, just a simple LLM classification call
- LLM Chain is simpler and more reliable for this use case

### Why no Structured Output Parser
- Gemini wraps JSON output in markdown code blocks (```json...```)
- The parser couldn't handle this and threw "Model output doesn't fit required format"
- Upgrading parser to v1.3 introduced a new "Model" input requirement
- **Solution**: Parse JSON directly in the Code node with fallback regex extraction

### Expression Format Fix
- n8n requires `=` prefix on expressions mixed with text (e.g., `=From: {{ $json.sender }}`)
- Without the prefix, n8n throws "WorkflowHasIssuesError" but doesn't say why
- Fixed using `n8n_autofix_workflow` tool

### Gmail Data Field Mapping
- Gmail Trigger with `simple: true` returns flat fields: `$json.From`, `$json.Subject`, `$json.snippet`
- NOT nested fields like `$json.from.text` (that's the non-simple format)

### Gmail Link with Correct Account
- Format: `https://mail.google.com/mail/u/?authuser=EMAIL#all/EMAILID`
- Uses `recipientEmail` (To field) to open the correct inbox
- Works across multiple Gmail/Workspace accounts

### Telegram Operation Parameter
- The Telegram node **requires** `operation: "sendMessage"` explicitly
- This gets lost during full workflow updates via API - must be re-set

---

## Credentials Configured

| Service | Credential | n8n Credential ID |
|---------|-----------|-------------------|
| Gmail (Account 1) | Gmail OAuth2 (phil@yufora.com) | QKqspapsDqWJkzDa |
| Gmail (Account 2) | Gmail OAuth2 (2nd account) | Configured in UI |
| Google Gemini | Google PaLM API | x4jAdG2mahUmKbmW |
| Telegram | Telegram API (N8n517bot) | KSDajeVj98U3LS8y |

## Deployed Workflow

- **Workflow ID**: `arJbFrfKsJguy9tp`
- **URL**: https://n8n.srv1299342.hstgr.cloud/workflow/arJbFrfKsJguy9tp
- **Status**: Active (Published)
- **Validation**: Valid (0 errors)
- **Error Workflow**: `Fhmq0omGZpu98HN3` (Error Alert → Telegram)

### Error Handling
- Dedicated error workflow sends Telegram alert on any failure
- Google Gemini node: retryOnFail (3 attempts, 5s delay)
- Send Telegram Alert node: retryOnFail (3 attempts, 5s delay)

---

## Current Status

| Task | Status |
|------|--------|
| Plan approved | ✅ |
| Workflow design complete | ✅ |
| MCP connection | ✅ Connected |
| Deploy workflow to n8n | ✅ |
| Configure credentials | ✅ |
| Fix data mapping | ✅ |
| Fix AI node (Agent → LLM Chain) | ✅ |
| Fix Structured Output Parser | ✅ Removed, using Code node |
| Fix expression format | ✅ |
| Fix Telegram auth | ✅ |
| Add Gmail deep link | ✅ |
| Add second Gmail account | ⚠️ Trigger added, needs connecting |
| Setup documentation | ✅ |
| Workflow live & working | ✅ |
| Error notification workflow | ✅ Fhmq0omGZpu98HN3 |
| Retry on flaky nodes | ✅ Gemini + Telegram |

---

## Project Files

| File | Purpose |
|------|---------|
| `.env.example` | Credential template with instructions |
| `.env` | Actual credentials (private) |
| `.gitignore` | Protects secrets from git |
| `SETUP.md` | Step-by-step setup guide |
| `workflow.json` | Original workflow definition (outdated - live version has fixes) |
| `.mcp.json` | MCP config for n8n API |
| `PROJECT_CONTEXT.md` | This file |

---

## Stock Market Alert Workflow

- **Workflow ID**: `782rqSIRL2xZ3wqv`
- **URL**: https://n8n.srv1299342.hstgr.cloud/workflow/782rqSIRL2xZ3wqv
- **Status**: Created (needs credentials)
- **Error Workflow**: Linked to `Fhmq0omGZpu98HN3`

### Architecture
```
Schedule Trigger (every 5 min, Mon-Fri 9:30-16:00 ET) → Google Sheets watchlist → Parse → Twelve Data API → Smart Alert Logic → IF alert → Telegram + Update Sheet
```

### Setup Required
1. Get Twelve Data API key (free) at https://twelvedata.com/register
2. Create Google Sheet with columns: symbol | upper_limit | lower_limit | direction | cooldown_minutes | last_alert_price | last_alert_time
3. Connect Google Sheets + Telegram credentials in n8n UI
4. Replace `YOUR_GOOGLE_SHEET_ID_HERE` and `YOUR_TWELVE_DATA_API_KEY` in workflow nodes

### Stock Symbol Format
- US: `AAPL`, `TSLA`, `MSFT`
- Toronto (TSX): `SHOP.TO`, `TD.TO`, `RY.TO`

---

## Pending / Future

1. **Connect Gmail Trigger1** → Prepare Email Data (drag connection in n8n UI)
2. ~~Consider adding error notification workflow~~ ✅ Done
3. Consider testing AI reliability with sample emails
4. Optionally export updated workflow.json from n8n as backup
