# Implementation Guide

## Overview

This guide walks you through deploying both n8n workflows on your self-hosted instance:

- **Workflow 1 — Get Leads** (`workflow-1-get-leads.json`): scrapes Google Maps, enriches contact data, and stores leads in Google Sheets.
- **Workflow 2 — Send Personalised Emails** (`workflow-2-send-emails.json`): reads new leads from the sheet and sends a personalised outreach email to each business owner.

---

## Prerequisites

### 1. n8n Installation

```bash
# Install n8n on VPS (Ubuntu/Debian)
sudo apt update
sudo apt install nodejs npm -y
npm install -g n8n

# Start n8n
n8n start

# Or use Docker
docker run -it --rm \
  --name n8n \
  -p 5678:5678 \
  -v ~/.n8n:/home/node/.n8n \
  n8nio/n8n
```

### 2. Required API Keys

**Workflow 1 — Get Leads:**

1. **Google Maps Scraping** — [Outscraper](https://outscraper.com) (recommended), SerpApi, or ScraperAPI
2. **Email Enrichment** — [Hunter.io](https://hunter.io/api) (recommended), Clearbit, or Snov.io
3. **LinkedIn Scraping** — [RapidAPI LinkedIn Scraper](https://rapidapi.com)
4. **Google Sheets** — Google Cloud project with Sheets API and a Service Account

**Workflow 2 — Send Personalised Emails:**

5. **SMTP / Email** — Gmail, SendGrid, Mailgun, or any SMTP provider supported by n8n

### 3. System Requirements

- **VPS**: 2 GB RAM minimum, 4 GB recommended
- **Storage**: 20 GB available
- **OS**: Ubuntu 20.04+ or similar
- **Node.js**: Version 16+

---

## Step 1 — Set Up Google Sheets

1. Create a new Google Sheet named **"HVAC Leads Database"**
2. Create a sheet tab called **Leads**
3. Add these 13 column headers in row 1:

   ```
   Company Name | Owner First Name | Email | Phone Numbers | Website
   | LinkedIn URL | Category | Location | Google Maps URL | Date Scraped
   | Source Query | Email Sent | Email Sent Date
   ```

4. Copy the Sheet ID from the URL:
   ```
   https://docs.google.com/spreadsheets/d/{SHEET_ID}/edit
   ```
5. Open **both** workflow JSON files and replace every occurrence of `REPLACE_WITH_YOUR_SHEET_ID` with your actual Sheet ID.

---

## Step 2 — Configure Credentials in n8n

#### Google Sheets

1. Go to [Google Cloud Console](https://console.cloud.google.com)
2. Enable the Google Sheets API
3. Create a Service Account → download the JSON key
4. In n8n: **Settings → Credentials → New → Google Service Account** → upload the JSON
5. Share your Google Sheet with the service account email (Editor access)

#### Outscraper (Google Maps)

1. Sign up at [outscraper.com](https://outscraper.com) and get your API key
2. In n8n: **Settings → Credentials → New → Header Auth**
   - Name: `X-API-Key`
   - Value: your key

#### Hunter.io (Email Enrichment)

1. Sign up at [hunter.io](https://hunter.io) and get your API key
2. In n8n: **Settings → Credentials → New → Query Auth**
   - Name: `api_key`
   - Value: your key

#### RapidAPI (LinkedIn)

1. Sign up at [rapidapi.com](https://rapidapi.com) and subscribe to the LinkedIn Scraper API
2. In n8n: **Settings → Credentials → New → Header Auth**
   - Name: `X-RapidAPI-Key`
   - Value: your key

#### SMTP (for Workflow 2)

1. In n8n: **Settings → Credentials → New → SMTP**
2. Enter your mail server host, port, username, and password
3. Test the connection

---

## Step 3 — Import and Configure Workflow 1 (Get Leads)

1. In n8n: **Workflows → Import from File**
2. Select `workflow/workflow-1-get-leads.json`
3. Assign credentials to each node that requires them (Google Sheets, Outscraper, Hunter.io, RapidAPI)
4. Open the **Append to Leads Sheet** node and confirm your Sheet ID is correct
5. Open the **Success Notification** node and update `fromEmail` / `toEmail` to your addresses

### Test Workflow 1

1. Click the **Form Trigger** node and copy the **Test URL**
2. Open the URL in your browser and submit the form:
   - **Business Type**: HVAC contractor
   - **Target Locations**: Phoenix, AZ
   - **Search Keywords**: HVAC repair
   - **Daily Lead Limit**: 5
3. Watch the execution in the n8n **Executions** panel
4. Open your Google Sheet and verify 5 leads were appended with **Email Sent** left blank

---

## Step 4 — Import and Configure Workflow 2 (Send Personalised Emails)

1. In n8n: **Workflows → Import from File**
2. Select `workflow/workflow-2-send-emails.json`
3. Assign credentials to each node (Google Sheets, SMTP)
4. Open the **Send Personalised Email** node and set:
   - `fromEmail`: your sender address (must match your SMTP credentials)
5. Open the **Build Personalised Email** node and personalise the email template:
   - Replace `Your Name`, `Your Title`, `your@email.com`, and `your-phone-number` with your details
6. Open the **Read Leads Sheet** and **Mark Email Sent** nodes and confirm the Sheet ID

### Test Workflow 2

1. Manually trigger the workflow (click **Execute Workflow** in n8n)
2. It will read your sheet, find leads with an email address and blank **Email Sent**, and send emails
3. After each send, check that **Email Sent** = `Yes` and **Email Sent Date** is filled in the sheet

### Activate the Daily Schedule

1. Toggle the **Active** switch (top-right) on Workflow 2
2. It will now run automatically every day at 10 AM and email any new leads from the previous day's scrape

---

## Configuration Reference

### Changing the Email Schedule

Open the **Schedule Trigger** node in Workflow 2 and change `triggerAtHour`:

```json
{ "triggerAtHour": 10 }   // 10 AM (default)
{ "triggerAtHour": 8  }   // 8 AM
{ "triggerAtHour": 14 }   // 2 PM
```

### Adjusting the Scraping Rate Limiter (Workflow 1)

Open the **Rate Limiter** node:

```json
{ "amount": "={{ Math.floor(Math.random() * 3) + 2 }}" }  // 2–5 s (default, safe)
{ "amount": "={{ Math.floor(Math.random() * 5) + 5 }}" }  // 5–10 s (conservative)
```

### Adjusting the Email Rate Limiter (Workflow 2)

Open the **Rate Limiter** node:

```json
{ "amount": "={{ Math.floor(Math.random() * 30) + 30 }}" }  // 30–60 s (default)
{ "amount": "={{ Math.floor(Math.random() * 60) + 60 }}" }  // 60–120 s (conservative)
```

---

## Troubleshooting

### Workflow 1

| Problem | Likely Cause | Fix |
|---------|-------------|-----|
| Form URL returns 404 | n8n not running | Restart n8n; check webhook path |
| HTTP 403/429 from Outscraper | API quota exceeded | Check quota; increase Rate Limiter delay |
| No emails found | Hunter.io key invalid or no MX | Verify key; try a different domain |
| Duplicates in sheet | Deduplication node issue | Clear test data; check dedup function |

### Workflow 2

| Problem | Likely Cause | Fix |
|---------|-------------|-----|
| No emails sent | All leads already marked sent | Add new leads via Workflow 1 first |
| SMTP error | Wrong credentials | Re-enter SMTP details; check port (587/465) |
| Sheet not updating | Wrong Sheet ID or column name | Verify Sheet ID and column header spelling |
| Rate limit from email provider | Sending too fast | Increase Rate Limiter delay |

---

## Maintenance Schedule

| Frequency | Task |
|-----------|------|
| Daily | Check n8n execution logs for errors |
| Weekly | Review lead quality in Google Sheet |
| Monthly | Rotate API keys; analyse email reply rates |

---

## Next Steps

1. **Test thoroughly** with a small lead limit (5–10 leads) before scaling
2. **Monitor** the first few daily runs of both workflows
3. **Personalise** the email template in Workflow 2 with your real details
4. **Scale** Workflow 1 gradually to 100 leads/day
5. Review [Best Practices](best-practices.md) for compliance and optimisation
