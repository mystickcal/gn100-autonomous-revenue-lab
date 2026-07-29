# How to Set Up an n8n Email Triage Bot That Classifies Your Inbox Every Morning

**Free step-by-step guide. No coding required.**

---

## The Problem Nobody Talks About

Your inbox is a to-do list that never ends. Every morning you open email and see 50+ unread messages. You can't tell which are urgent, which need a reply, which are worth keeping, and which are trash. So you start reading from the top, spend an hour, and still feel behind before your real work begins.

The email triage bot solves this by scanning your inbox every morning and delivering a clean, categorized digest to Telegram or Slack -- so you see exactly what needs attention, not everything that exists.

---

## What This Guide Covers

This guide shows you how to build an n8n workflow that:

1. Checks your Gmail inbox every morning at 8am
2. Reads unread emails from the last 24 hours
3. Uses AI to classify each email as: **URGENT**, **REPLY TODAY**, **FYI**, or **JUNK**
4. Sends a clean, prioritized digest to Telegram or Slack
5. Marks emails as read automatically

No coding required. If you can follow instructions and paste in API keys, you can build this.

---

## Prerequisites

You need three things:

- **An n8n instance** -- self-hosted (free, docker) or n8n Cloud (~$24/mo)
- **An OpenAI API key** -- gpt-4o-mini works fine, costs a few cents per batch
- **Gmail access** -- via OAuth (n8n handles the OAuth flow)

**Total cost per month: approximately a few cents for API usage. After that, free.**

---

## Step 1: Set Up Your n8n Instance

### Option A: Self-hosted (Recommended, free)

```bash
docker run -it --rm \
  --name n8n \
  -p 5678:5678 \
  -v ~/.n8n:/home/node/.n8n \
  n8nio/n8n
```

Visit http://localhost:5678 in your browser. Done.

### Option B: n8n Cloud

Go to n8n.io and sign up. You'll get an instance in minutes.

---

## Step 2: Create a New Workflow

1. Open n8n and click **New Workflow**
2. Name it: `Email Triage Bot`
3. Click the **Cron** trigger node (or search for "Cron" in the node palette)

---

## Step 3: Configure the Cron Trigger

Set the schedule:

- **Interval:** Every day
- **Hour:** 8 (8am)
- **Minute:** 0
- **Timezone:** Your local timezone

This ensures the triage runs while you sleep and the results are ready when you start your workday.

---

## Step 4: Add Gmail Read Node

1. Click **Add Node** and search for **Gmail**
2. Choose the **Read** operation
3. Connect it to the Cron trigger
4. Connect your Gmail account (n8n will guide you through OAuth)

Configure the Gmail node:

- **Operation:** Read
- **Message Fields:** `from`, `subject`, `snippet`, `body`, `date`
- **Filter:**

```
labelIds: INBOX
query: is:unread after:$(=new Date(Date.now() - 24*60*60*1000).toISOString().split('T')[0])
```

This reads unread emails from the last 24 hours.

---

## Step 5: Add the AI Classification Node

This is the brain. The AI will read each email and assign it a category.

1. Add an **OpenAI Chat Completion** node
2. Connect it after the Gmail Read node
3. Enter your OpenAI API key (or use n8n's credential manager)
4. Set the model: **gpt-4o-mini**

### System Prompt

```
You are an email triage assistant. Given an email with sender, subject, and content, classify it into exactly ONE of these categories:

- URGENT: Requires immediate action. Your boss, a client deadline, a payment issue, a time-sensitive problem.
- REPLY TODAY: Needs a response today but not time-critical. A colleague asking a question, a meeting invite, a partner update.
- FYI: Information only. No action needed. Newsletters, CC emails, announcements.
- JUNK: Spam, promotional, newsletter you didn't ask for, irrelevant.

Return your response as valid JSON only, with this exact structure:
{"category": "URGENT|REPLY TODAY|FYI|JUNK", "reason": "one-sentence explanation"}

Do NOT include any text outside the JSON object.
```

### User Prompt

```
Email from: {{ $('Gmail Read').item.json.from }}
Subject: {{ $('Gmail Read').item.json.subject }}
Body: {{ $('Gmail Read').item.json.snippet }}

Classify this email.
```

**Configuration:** Function Mode: Yes, Output Key: `classification`, Batch Size: 1

---

## Step 6: Format the Digest

Add a **Code** node after the AI classification:

```javascript
// Collect all classified emails
const emails = [];
for (const item of $input.all()) {
  emails.push({
    from: item.json.from,
    subject: item.json.subject,
    category: item.json.classification?.category || 'REPLY TODAY',
    reason: item.json.classification?.reason || 'Unclassified'
  });
}

// Sort by priority
const priority = { URGENT: 0, 'REPLY TODAY': 1, FYI: 2, JUNK: 3 };
emails.sort((a, b) => priority[a.category] - priority[b.category]);

// Build the message
const urgent = emails.filter(e => e.category === 'URGENT');
const reply = emails.filter(e => e.category === 'REPLY TODAY');
const fyi = emails.filter(e => e.category === 'FYI');
const junk = emails.filter(e => e.category === 'JUNK');

let message = '📧 Daily Email Digest\n\n';

if (urgent.length > 0) {
  message += '🔴 URGENT (' + urgent.length + ')\n';
  for (const e of urgent) {
    message += '  • ' + e.subject + '\n  ' + e.from + '\n';
  }
  message += '\n';
}

if (reply.length > 0) {
  message += '🟡 REPLY TODAY (' + reply.length + ')\n';
  for (const e of reply) {
    message += '  • ' + e.subject + '\n  ' + e.from + '\n';
  }
  message += '\n';
}

if (fyi.length > 0) {
  message += '🟢 FYI (' + fyi.length + ')\n';
  for (const e of fyi.slice(0, 5)) {
    message += '  • ' + e.subject + '\n';
  }
  if (fyi.length > 5) message += '  ... and ' + (fyi.length - 5) + ' more\n';
  message += '\n';
}

message += 'Total: ' + emails.length + ' emails scanned.';

return [{ json: { message } }];
```

---

## Step 7: Send to Telegram or Slack

### Telegram Setup

1. Open Telegram, search for @BotFather
2. Create a new bot, get the API token
3. Search for your bot, send it /start
4. Find your chat ID: visit `https://api.telegram.org/bot<YOUR_TOKEN>/getUpdates` and look for `chat.id`
5. In n8n, add a **Telegram** node
6. Set operation: **SendMessage**
7. Paste your chat ID and use the message field from the previous node

### Slack Setup

1. In Slack, create an app and add a webhook URL
2. In n8n, add an **HTTP Request** node
3. POST to your webhook URL with the message from the previous node

---

## Step 8: Mark Emails as Read

Add a **Gmail** node:

- **Operation:** Update
- **Message ID:** Reference each processed email
- **Label Operation:** Remove label INBOX OR add label Read

This prevents the triage bot from re-processing the same emails tomorrow.

---

## Step 9: Activate and Test

1. Click **Activate** on your workflow
2. Manually trigger it using the **Execute Node** button (right-click on the Cron node)
3. Check Telegram or Slack for your first digest
4. Verify the classifications make sense

---

## What This Bot Does for You

| Before | After |
|--------|-------|
| Manually sorting through a flooded inbox each morning | Receiving a clean, prioritized digest |
| No idea which emails matter | Clear priority list |
| Important emails get buried | Urgent emails are highlighted first |
| Manual categorization | Automatic every morning |
| Spending the start of your day on email | Starting your day with what matters |

---

## Want the Done-For-You Version?

This guide walks you through building the email triage bot yourself -- about 20-30 minutes if you follow it carefully. But the **n8n Starter Pack** includes all 14 workflows, setup guides, and annotated nodes.

**The n8n Starter Pack is $97 for all 14 workflows.**

Get it at: https://buy.stripe.com/7sYeVc1SB0Os4Rkb6D3oA05

---

## FAQ

**Q: Does this work with Outlook/Exchange?**
A: Gmail/OAuth is the simplest path. Outlook works with the IMAP node in n8n, but Gmail has a more mature integration.

**Q: Will this miss anything?**
A: The AI is very good at classification but not perfect. The digest format means you can see all categories at a glance. If something's marked JUNK and you wanted it, it's still in your inbox -- the bot only marks it as read.

**Q: How much does the AI cost?**
A: With gpt-4o-mini, processing a batch of emails costs about a fraction of a cent per run. A few cents per month. Essentially free.

**Q: Can I customize the categories?**
A: Yes. Just change the system prompt in the AI node. You could add a "SELL" category for lead inquiries, for example.

**Q: Does this work on n8n Cloud?**
A: Yes. The workflow file imports identically.

**Q: What if I don't want to set up n8n myself?**
A: n8n Cloud handles the infrastructure. You still need to build the workflow, but n8n Cloud manages servers, updates, and uptime.