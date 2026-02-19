# Workflow Architecture

The automation is split into exactly **two independent workflows**:

| File | Name | Purpose |
|------|------|---------|
| `workflow-1-get-leads.json` | Workflow 1: Get Leads | Scrapes Google Maps, enriches contacts, deduplicates, and saves leads to Google Sheets |
| `workflow-2-send-emails.json` | Workflow 2: Send Personalised Emails | Reads leads from Google Sheets and sends a personalised outreach email to each business owner |

Both workflows share the same Google Sheet as the handoff point. Run Workflow 1 first to populate the sheet, then run (or schedule) Workflow 2 to send emails to the collected leads.

---

## Workflow 1: Get Leads

**Trigger**: n8n Form (fill in business type, target locations, keywords, and daily limit)

```
                                INTAKE LAYER
┌───────────────────────────────────────────────────────────────────────────────┐
│  ┌─────────────────┐         ┌──────────────────┐       ┌──────────────────┐ │
│  │  Form Trigger   │────────▶│  Input Validator │──────▶│  Initialize Vars │ │
│  │  (Webhook/Form) │         │   (Function)     │       │     (Set)        │ │
│  └─────────────────┘         └──────────────────┘       └──────────────────┘ │
└───────────────────────────────────────────────────────────────────────────────┘
                                       │
                                       ▼
                              SCRAPING LAYER
┌───────────────────────────────────────────────────────────────────────────────┐
│  ┌──────────────────┐      ┌────────────────────┐      ┌──────────────────┐  │
│  │  Loop Control    │─────▶│  Google Maps API   │─────▶│  Parse Results   │  │
│  │  (Function)      │      │  (HTTP Request)    │      │   (Function)     │  │
│  └──────────────────┘      └────────────────────┘      └──────────────────┘  │
│         ▲                                                         │            │
│         │ While leadsCollected < leadLimit                        ▼            │
│         │                                               ┌──────────────────┐  │
│         └───────────────────────────────────────────────│  Rate Limiter    │  │
│                                                          │  (Wait 2–5 s)    │  │
│                                                          └──────────────────┘  │
└───────────────────────────────────────────────────────────────────────────────┘
                                       │
                                       ▼
                         ENRICHMENT LAYER (parallel)
┌───────────────────────────────────────────────────────────────────────────────┐
│  ┌──────────────────┐      ┌────────────────────┐      ┌──────────────────┐  │
│  │ Website Scraper  │      │  Email Enrichment  │      │ LinkedIn Matcher │  │
│  │ (HTTP Request)   │      │   (Hunter.io API)  │      │  (RapidAPI)      │  │
│  └──────────────────┘      └────────────────────┘      └──────────────────┘  │
│         └────────────────────────────┴────────────────────────────┘            │
└───────────────────────────────────────────────────────────────────────────────┘
                                       │
                                       ▼
                           QUALITY CONTROL LAYER
┌───────────────────────────────────────────────────────────────────────────────┐
│  ┌──────────────────┐      ┌────────────────────┐                             │
│  │  Deduplication   │─────▶│  Filter Qualified  │                             │
│  │   (Function)     │      │  (IF Node)         │                             │
│  └──────────────────┘      └────────────────────┘                             │
└───────────────────────────────────────────────────────────────────────────────┘
                                       │ !isDuplicate
                                       ▼
                                STORAGE LAYER
┌───────────────────────────────────────────────────────────────────────────────┐
│                      ┌─────────────────────────┐                              │
│                       │  Append to Leads Sheet  │  ◀── writes all lead data   │
│                       │  (Google Sheets Append) │      Email Sent left blank  │
│                      └─────────────────────────┘                              │
└───────────────────────────────────────────────────────────────────────────────┘
                                       │
                                       ▼
                           NOTIFICATION LAYER
┌───────────────────────────────────────────────────────────────────────────────┐
│  ┌──────────────────┐      ┌────────────────────┐                             │
│  │  Build Summary   │─────▶│  Admin Notification│                             │
│  └──────────────────┘      └────────────────────┘                             │
└───────────────────────────────────────────────────────────────────────────────┘

                              ERROR HANDLING
┌───────────────────────────────────────────────────────────────────────────────┐
│  ┌──────────────────┐      ┌────────────────────┐      ┌──────────────────┐  │
│  │  Error Trigger   │─────▶│  Log Error         │─────▶│  Send Alert      │  │
│  └──────────────────┘      └────────────────────┘      └──────────────────┘  │
└───────────────────────────────────────────────────────────────────────────────┘
```

### Node Summary — Workflow 1

| Node | Type | Purpose | Continue on Fail? |
|------|------|---------|-------------------|
| Form Trigger | Webhook/Form | Collect search parameters | No |
| Input Validation | Function | Validate and format inputs | No |
| Initialize Variables | Set | Set loop counters | No |
| Loop Control | Function | Iterate locations/keywords | No |
| Google Maps Scraper | HTTP Request | Search businesses via Outscraper | No |
| Parse Maps Results | Function | Extract structured data | No |
| Rate Limiter | Wait | Respectful delay (2–5 s) | No |
| Website Scraper | HTTP Request | Scrape owner/email from site | Yes |
| Email Enrichment | HTTP Request | Hunter.io email lookup | Yes |
| LinkedIn Matcher | HTTP Request | RapidAPI LinkedIn search | Yes |
| Deduplication Check | Function | Flag duplicate leads | No |
| Filter Qualified | IF | Drop duplicates | No |
| Append to Leads Sheet | Google Sheets | Save lead rows | No |
| Build Summary | Function | Count results | No |
| Success Notification | Email | Notify admin | Yes |
| Error Trigger | Error Catch | Catch any failure | N/A |
| Log Error | Function | Format error details | No |
| Send Error Alert | Email | Alert admin | Yes |

---

## Workflow 2: Send Personalised Emails

**Trigger**: Schedule (daily at 10 AM) — runs automatically to email leads collected by Workflow 1

```
                              INTAKE LAYER
┌───────────────────────────────────────────────────────────────────────────────┐
│  ┌─────────────────┐      ┌──────────────────────┐      ┌──────────────────┐ │
│  │ Schedule Trigger│─────▶│  Read Leads Sheet    │─────▶│ Filter Unsent    │ │
│  │  (Daily 10 AM)  │      │  (Google Sheets)     │      │ Leads (Function) │ │
│  └─────────────────┘      └──────────────────────┘      └──────────────────┘ │
└───────────────────────────────────────────────────────────────────────────────┘
                                       │ rows with Email + no Email Sent
                                       ▼
                             EMAIL SENDING LAYER
┌───────────────────────────────────────────────────────────────────────────────┐
│  ┌──────────────────┐      ┌────────────────────┐      ┌──────────────────┐  │
│  │ Has Leads to Send│─────▶│  Build Personalised│─────▶│  Rate Limiter    │  │
│  │ (IF Node)        │      │  Email (Function)  │      │  (Wait 30–60 s)  │  │
│  └──────────────────┘      └────────────────────┘      └──────────────────┘  │
│                                                                   │             │
│                                                                   ▼             │
│                                                        ┌──────────────────┐   │
│                                                        │ Send Personalised│   │
│                                                        │ Email (SMTP)     │   │
│                                                        └──────────────────┘   │
└───────────────────────────────────────────────────────────────────────────────┘
                                       │
                                       ▼
                                TRACKING LAYER
┌───────────────────────────────────────────────────────────────────────────────┐
│                      ┌─────────────────────────┐                              │
│                       │  Mark Email Sent        │  ◀── updates "Email Sent"   │
│                       │  (Google Sheets Update) │      and "Email Sent Date"  │
│                      └─────────────────────────┘                              │
└───────────────────────────────────────────────────────────────────────────────┘
                                       │
                                       ▼
                           NOTIFICATION LAYER
┌───────────────────────────────────────────────────────────────────────────────┐
│  ┌──────────────────┐      ┌────────────────────┐                             │
│  │  Build Summary   │─────▶│  Admin Notification│                             │
│  └──────────────────┘      └────────────────────┘                             │
└───────────────────────────────────────────────────────────────────────────────┘

                              ERROR HANDLING
┌───────────────────────────────────────────────────────────────────────────────┐
│  ┌──────────────────┐      ┌────────────────────┐      ┌──────────────────┐  │
│  │  Error Trigger   │─────▶│  Log Error         │─────▶│  Send Alert      │  │
│  └──────────────────┘      └────────────────────┘      └──────────────────┘  │
└───────────────────────────────────────────────────────────────────────────────┘
```

### Node Summary — Workflow 2

| Node | Type | Purpose | Continue on Fail? |
|------|------|---------|-------------------|
| Schedule Trigger | Schedule | Run daily at 10 AM | N/A |
| Read Leads Sheet | Google Sheets | Read all rows | No |
| Filter Unsent Leads | Function | Keep rows with email but no Email Sent | No |
| Has Leads to Send | IF | Skip if nothing to send | No |
| Build Personalised Email | Function | Generate HTML email per lead | No |
| Rate Limiter | Wait | Delay 30–60 s between sends | No |
| Send Personalised Email | Email Send | Send to business owner | Yes |
| Mark Email Sent | Google Sheets | Update Email Sent + Email Sent Date | Yes |
| Build Summary | Function | Count emails sent | No |
| Success Notification | Email | Notify admin | Yes |
| Error Trigger | Error Catch | Catch any failure | N/A |
| Log Error | Function | Format error details | No |
| Send Error Alert | Email | Alert admin | Yes |

---

## Data Flow Between Workflows

```
┌──────────────────────────────────────────────────────────────────┐
│                      WORKFLOW 1: GET LEADS                        │
│                                                                    │
│  Form Input ──▶ Validate ──▶ Scrape Maps ──▶ Enrich ──▶ Dedupe  │
│                                                         │         │
└─────────────────────────────────────────────────────────┼─────────┘
                                                          │
                                                          ▼
                                              ┌───────────────────┐
                                              │   Google Sheet    │
                                              │   (Leads tab)     │
                                              │                   │
                                              │  Company Name     │
                                              │  Owner First Name │
                                              │  Email            │
                                              │  Phone Numbers    │
                                              │  Website          │
                                              │  LinkedIn URL     │
                                              │  Category         │
                                              │  Location         │
                                              │  Google Maps URL  │
                                              │  Date Scraped     │
                                              │  Source Query     │
                                              │  Email Sent    ◀──┼─┐
                                              │  Email Sent Date◀─┼─┤
                                              └───────────────────┘  │
                                                          │           │
┌─────────────────────────────────────────────────────────┼──────────┘
│                      WORKFLOW 2: SEND EMAILS             │
│                                                          │
│  Schedule ──▶ Read Sheet ──▶ Filter Unsent ──▶ Build    │
│              Email ──▶ Rate Limit ──▶ Send ──▶ Mark Sent┘
└──────────────────────────────────────────────────────────┘
```

---

## Next Steps

1. **Import both workflows** into your n8n instance
2. **Configure APIs**: Follow [API Integrations Guide](../docs/api-integrations.md)
3. **Set up your Google Sheet** with all 13 columns (see `config/sheets-template.json`)
4. **Update the Sheet ID** in both workflow files (replace `REPLACE_WITH_YOUR_SHEET_ID`)
5. **Test Workflow 1** with a small lead limit (5–10 leads)
6. **Test Workflow 2** manually, then activate the daily schedule

See [Implementation Guide](../docs/implementation-guide.md) for detailed setup.
