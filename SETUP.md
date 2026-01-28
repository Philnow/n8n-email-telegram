# Setup Guide - Email-to-Telegram Workflow

Complete these steps to activate your workflow.

---

## Quick Start

```bash
# 1. Copy the template
cp .env.example .env

# 2. Edit with your credentials
nano .env  # or use any text editor
```

---

## Step 1: Telegram Bot (5 min)

### Create Your Bot
1. Open Telegram and search for **@BotFather**
2. Send `/newbot`
3. Choose a name (e.g., "Email Alert Bot")
4. Choose a username (must end in `bot`, e.g., `myemail_alert_bot`)
5. Copy the **token** - looks like: `123456789:ABCdefGHIjklMNOpqrsTUVwxyz`

### Get Your Chat ID
1. Search for **@userinfobot** on Telegram
2. Send any message
3. Copy the **Id** number (e.g., `123456789`)

### Important: Start Your Bot!
1. Find your bot by its username
2. Click **Start** or send `/start`
3. This allows the bot to message you

---

## Step 2: Google Gemini API (2 min)

1. Go to [Google AI Studio](https://aistudio.google.com/apikey)
2. Sign in with your Google account
3. Click **Create API Key**
4. Copy the key

**Free Tier Limits:**
- 60 requests per minute
- 1 million tokens per day

---

## Step 3: Configure n8n Credentials

Open your n8n instance: https://n8n.srv1299342.hstgr.cloud

### Gmail OAuth
1. Open the workflow: **Email-to-Telegram Important Summary**
2. Click the **Gmail Trigger** node
3. Under "Credential to connect with", click **Create New Credential**
4. Click **Sign in with Google**
5. Authorize access to your Gmail

### Google Gemini
1. Go to **Settings** (gear icon) > **Credentials**
2. Click **Add Credential**
3. Search for **Google Gemini** (or Google PaLM API)
4. Paste your API key
5. Save

### Telegram
1. Go to **Settings** > **Credentials**
2. Click **Add Credential**
3. Search for **Telegram API**
4. Paste your Bot Token
5. Save

---

## Step 4: Update Workflow Chat ID

Tell Claude your Telegram Chat ID and it will update the workflow:

> "My Telegram chat ID is 123456789"

Or manually:
1. Open the workflow
2. Click **Send Telegram Alert** node
3. Replace `YOUR_CHAT_ID_HERE` with your actual Chat ID
4. Save

---

## Step 5: Connect Credentials to Nodes

1. Click **Google Gemini** node > Select your Gemini credential
2. Click **Send Telegram Alert** node > Select your Telegram credential
3. Save the workflow

---

## Step 6: Activate & Test

1. Toggle the workflow to **Active** (top right)
2. Send yourself a test email with "URGENT" in the subject
3. Wait ~1 minute for the trigger to poll
4. Check Telegram for the alert!

---

## Troubleshooting

### No Telegram message received?
- Did you click "Start" on your bot?
- Is the Chat ID correct?
- Check workflow executions in n8n for errors

### Gmail not triggering?
- Is the workflow active?
- Are there unread emails?
- Check Gmail credential is connected

### AI classification not working?
- Verify Gemini API key is valid
- Check if you hit rate limits

---

## File Reference

| File | Purpose |
|------|---------|
| `.env.example` | Template - copy to `.env` |
| `.env` | Your actual credentials (keep private!) |
| `workflow.json` | Workflow definition backup |
| `PROJECT_CONTEXT.md` | Project documentation |
