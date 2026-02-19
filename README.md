# HVAC Lead Automation — Two-Workflow n8n Implementation

## Overview

This repository contains two focused n8n automation workflows that work together to find HVAC business leads and send personalised outreach emails to their owners.

| # | Workflow | File | What it does |
|---|---------|------|-------------|
| 1 | **Get Leads** | `workflow/workflow-1-get-leads.json` | Scrapes Google Maps, enriches contact data, deduplicates, and stores leads in Google Sheets |
| 2 | **Send Personalised Emails** | `workflow/workflow-2-send-emails.json` | Reads new leads from the sheet and sends a personalised outreach email to each business owner |

Run Workflow 1 to fill your leads sheet, then Workflow 2 (scheduled daily) picks up every lead that has an email address and hasn't been contacted yet.

---

## Repository Structure

```
.
├── README.md
├── workflow/
│   ├── workflow-1-get-leads.json        # Import into n8n: lead scraping workflow
│   ├── workflow-2-send-emails.json      # Import into n8n: email outreach workflow
│   └── workflow-architecture.md        # Architecture diagrams for both workflows
├── docs/
│   ├── implementation-guide.md         # Step-by-step setup for both workflows
│   ├── node-breakdown.md               # Detailed node-by-node reference
│   ├── api-integrations.md             # API keys and service configurations
│   └── best-practices.md              # Scaling, compliance, and optimisation
└── config/
    ├── form-template.json              # Lead generation form configuration
    └── sheets-template.json           # Google Sheets schema (13 columns)
```

---

## Quick Start

### Step 1 — Set up your Google Sheet

Create a sheet with a **Leads** tab containing these 13 columns (in order):

```
Company Name | Owner First Name | Email | Phone Numbers | Website | LinkedIn URL
| Category | Location | Google Maps URL | Date Scraped | Source Query
| Email Sent | Email Sent Date
```

Copy the Sheet ID from the URL (`https://docs.google.com/spreadsheets/d/{SHEET_ID}/edit`) and replace `REPLACE_WITH_YOUR_SHEET_ID` in **both** workflow JSON files.

### Step 2 — Import Workflow 1: Get Leads

1. Open n8n → **Workflows** → **Import from File**
2. Select `workflow/workflow-1-get-leads.json`
3. Add your API credentials (Outscraper, Hunter.io, RapidAPI, Google Sheets)
4. Open the Form Trigger node and copy the **Production URL**
5. Submit the form with your target business type, locations, and keywords

### Step 3 — Import Workflow 2: Send Personalised Emails

1. Open n8n → **Workflows** → **Import from File**
2. Select `workflow/workflow-2-send-emails.json`
3. Open the **Send Personalised Email** node and set your `fromEmail` address
4. Add your SMTP credentials
5. Activate the workflow — it runs daily at 10 AM, picks up all leads with an email address that haven't been contacted yet, and sends each one a personalised outreach email

---

## How the Two Workflows Connect

```
WORKFLOW 1: GET LEADS
  Form Trigger
      │
      ▼
  Validate → Scrape Google Maps → Enrich (website, email, LinkedIn) → Deduplicate
      │
      ▼
  Append row to Google Sheet  ◀── Email Sent and Email Sent Date left blank
      │
      ▼
  Notify admin

            ────────────── Google Sheet is the handoff ──────────────

WORKFLOW 2: SEND PERSONALISED EMAILS  (runs daily at 10 AM)
  Schedule Trigger
      │
      ▼
  Read all rows from Google Sheet
      │
      ▼
  Filter: has Email AND Email Sent is blank
      │
      ▼
  Build personalised HTML email (uses owner name, company, location, category)
      │
      ▼
  Rate limiter (30–60 s between sends)  →  Send Email  →  Mark Email Sent in Sheet
      │
      ▼
  Notify admin with summary
```

---

## Target Metrics

| Metric | Target |
|--------|--------|
| Daily Leads (Workflow 1) | ~100 qualified leads |
| Data Enrichment Rate | >80% with email / owner info |
| Duplicate Rate | <5% |
| Emails Sent per Day (Workflow 2) | Up to 100 (matches new leads) |
| Execution Time (Workflow 1) | 30–45 minutes |

---

## Prerequisites

### n8n
- Self-hosted n8n instance (version 1.0.0+)
- VPS with 2 GB+ RAM recommended

### API Keys (Workflow 1)
- **Outscraper** — Google Maps scraping
- **Hunter.io** — Email enrichment
- **RapidAPI** (LinkedIn Scraper) — LinkedIn matching
- **Google Sheets API** — Lead storage

### Email (Workflow 2)
- SMTP credentials (Gmail, SendGrid, or any provider)
- A sender email address configured in the **Send Personalised Email** node

---

## Documentation

- [Workflow Architecture](workflow/workflow-architecture.md) — diagrams and node tables for both workflows
- [Implementation Guide](docs/implementation-guide.md) — step-by-step setup
- [API Integrations](docs/api-integrations.md) — API keys and configurations
- [Node Breakdown](docs/node-breakdown.md) — detailed per-node reference
- [Best Practices](docs/best-practices.md) — scaling, compliance, and optimisation

---

## License

MIT License — see LICENSE file for details.

## Author

Jelvin Kiteete — HVAC Automation Specialist

---

**Ready to deploy?** Start with the [Implementation Guide](docs/implementation-guide.md).
