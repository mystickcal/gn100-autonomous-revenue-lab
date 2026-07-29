# Free n8n Invoice Automation Guide

A complete step-by-step guide to building an n8n workflow that reads incoming invoices from your email, extracts key details, and either sends you a Slack notification or auto-generates a PDF invoice to send out.

## What This Workflow Does

This n8n workflow automates the money-related email tasks that eat up time at the end of every week:

1. Triggers every hour via Cron to check for new emails
2. Scans Gmail for messages matching invoice/receipt/payment keywords
3. Uses OpenAI to extract key details: vendor name, amount, due date, description
4. Sends a structured summary to Slack or Telegram
5. Optionally generates a PDF invoice using a template
6. Saves all data to a Google Sheet for record-keeping

## Prerequisites

- **n8n instance** — self-hosted via Docker (free) or n8n Cloud (~$24/mo)
- **OpenAI API key** — gpt-4o-mini works fine, costs about $0.01 per batch
- **Gmail access** — via OAuth (n8n handles the OAuth flow)
- **Slack or Telegram** — for receiving notifications (free)

**Total monthly cost:** approximately $0.01 in API usage for typical use.

## Step 1: Set Up Your n8n Instance

### Option A: Self-hosted (free)

```bash
docker run -it --rm \
  --name n8n \
  -p 5678:5678 \
  -v ~/.n8n:/home/node/.n8n \
  n8nio/n8n
```

Visit http://localhost:5678 in your browser.

### Option B: n8n Cloud

Go to n8n.io and sign up. You'll get an instance in minutes.

## Step 2: Create the Workflow

1. Open n8n and click **New Workflow**
2. Name it: `Invoice Automation`
3. Click the **Cron** trigger node

## Step 3: Configure the Cron Trigger

- **Interval**: Every hour
- Leave **Minute** as `0` (checks at the top of each hour)

## Step 4: Add Gmail Read Node

1. Add a **Gmail** node
2. Connect it to the Cron trigger
3. Connect your Gmail account (OAuth flow)
4. Configure:
   - **Operation**: Read
   - **Message Fields**: from, subject, snippet, body, date
   - **Filter**: `is:unread -in:chat -in:trash`

## Step 5: Filter for Invoice-Related Emails

Add a **Filter** node after Gmail. Set the condition:

```
Subject OR Body contains: invoice OR receipt OR payment OR "due date" OR "amount due"
```

## Step 6: Extract Invoice Details with AI

Add an **OpenAI Chat Completion** node. Set the model to `gpt-4o-mini`.

### System Prompt

```
You are an invoice data extraction assistant. Given an email that contains invoice or payment information, extract the following fields and return valid JSON:

- vendor: name of the company/person
- amount: numeric amount as a number (e.g., 149.99)
- currency: 3-letter currency code (default: USD)
- due_date: date in YYYY-MM-DD format
- description: short description of what was invoiced
- email_body: the full email body text

Return your response as valid JSON only. Use "unknown" for any field you cannot determine.
If the email is not an invoice or payment, return {"is_invoice": false}.
```

### User Prompt

```
Email from: {{ $('Gmail Read').item.json.from }}
Subject: {{ $('Gmail Read').item.json.subject }}
Body: {{ $('Gmail Read').item.json.body }}

Extract the invoice details.
```

## Step 7: Send a Slack Notification

1. Add a **Filter** node: keep only items where `is_invoice` is `true`
2. Add a **Slack** node
3. Connect a Slack workspace
4. Set **Action**: `Send Message`
5. Set **Channel**: your preferred channel or direct message
6. Set **Message** to include the extracted fields

## Step 8: Save to Google Sheets

1. Add a **Google Sheets** node
2. Connect your Google account
3. Set **Operation**: `Add a Row`
4. Set your spreadsheet ID and sheet name
5. Map columns: Date, Vendor, Amount, Currency, Due Date, Description

## Step 9: Activate and Test

1. Click **Activate** on your workflow
2. Manually trigger with the **Execute Node** button
3. Check Slack for your first notification
4. Verify the Google Sheet has a new row

## What You Get

- Automatic invoice tracking without manual data entry
- Instant Slack/Telegram alerts when a new invoice arrives
- A running spreadsheet of all invoices for accounting
- Everything runs on autopilot from your inbox

## FAQ

### Q: Does this work with Outlook/Exchange?

Gmail/OAuth is the simplest path. Outlook works with the IMAP node in n8n.

### Q: How much does the AI cost?

With gpt-4o-mini, processing ~50 invoices per month costs less than $0.15.

### Q: Can I customize the extraction fields?

Yes. Just change the system prompt in the AI node to match your needs.

### Q: Does this work on n8n Cloud?

Yes. The workflow imports identically.

---

This guide is free. For a complete set of 14 n8n workflows including this one, see the n8n Starter Pack.