# Workflow Architecture

## Visual Architecture Overview

```
┌──────────────────────────────────────────────────────────────────────────────┐
│                         HVAC LEAD AUTOMATION WORKFLOW                          │
└──────────────────────────────────────────────────────────────────────────────┘

                                INTAKE LAYER
┌───────────────────────────────────────────────────────────────────────────────┐
│                                                                                 │
│  ┌─────────────────┐         ┌──────────────────┐       ┌──────────────────┐ │
│  │  Form Trigger   │────────▶│  Input Validator │──────▶│  Initialize Vars │ │
│  │  (Webhook/Form) │         │   (Function)     │       │     (Set)        │ │
│  └─────────────────┘         └──────────────────┘       └──────────────────┘ │
│                                                                                 │
└───────────────────────────────────────────────────────────────────────────────┘
                                       │
                                       ▼
                              SCRAPING LAYER
┌───────────────────────────────────────────────────────────────────────────────┐
│                                                                                 │
│  ┌──────────────────┐      ┌────────────────────┐      ┌──────────────────┐  │
│  │  Loop Control    │─────▶│  Google Maps API   │─────▶│  Parse Results   │  │
│  │  (Function)      │      │  (HTTP Request)    │      │   (Function)     │  │
│  └──────────────────┘      └────────────────────┘      └──────────────────┘  │
│         │                            │                            │            │
│         │ While leadsCollected       │                            │            │
│         │ < leadLimit                │                            │            │
│         │                            ▼                            ▼            │
│         │                   ┌─────────────────┐        ┌──────────────────┐  │
│         └──────────────────▶│  Rate Limiter   │        │  Split Into Items│  │
│                              │  (Wait 2-5s)    │        │    (Function)    │  │
│                              └─────────────────┘        └──────────────────┘  │
│                                                                   │             │
└───────────────────────────────────────────────────────────────────┼───────────┘
                                                                     │
                                                                     ▼
                              ENRICHMENT LAYER
┌───────────────────────────────────────────────────────────────────────────────┐
│                                                                                 │
│  ┌──────────────────┐      ┌────────────────────┐      ┌──────────────────┐  │
│  │ Website Scraper  │      │  Email Enrichment  │      │ LinkedIn Matcher │  │
│  │ (HTTP + Parse)   │      │   (Hunter.io API)  │      │  (RapidAPI)      │  │
│  └──────────────────┘      └────────────────────┘      └──────────────────┘  │
│         │                            │                            │            │
│         │                            │                            │            │
│         └────────────────────────────┴────────────────────────────┘            │
│                                      │                                         │
│                                      ▼                                         │
│                           ┌─────────────────────┐                             │
│                           │  MCP Enrichment     │                             │
│                           │  (AI Agent/Function)│                             │
│                           └─────────────────────┘                             │
│                                      │                                         │
└──────────────────────────────────────┼─────────────────────────────────────── ┘
                                       │
                                       ▼
                           QUALITY CONTROL LAYER
┌───────────────────────────────────────────────────────────────────────────────┐
│                                                                                 │
│  ┌──────────────────┐      ┌────────────────────┐      ┌──────────────────┐  │
│  │  Deduplication   │─────▶│  Quality Filter    │─────▶│  Lead Scoring    │  │
│  │   (Function)     │      │  (IF + Function)   │      │   (Function)     │  │
│  └──────────────────┘      └────────────────────┘      └──────────────────┘  │
│                                      │                                         │
│                                      │ isQualified &&                          │
│                                      │ !isDuplicate                            │
│                                      ▼                                         │
└───────────────────────────────────────────────────────────────────────────────┘
                                       │
                                       ▼
                               STORAGE LAYER
┌───────────────────────────────────────────────────────────────────────────────┐
│                                                                                 │
│                           ┌─────────────────────┐                              │
│                           │  Google Sheets      │                              │
│                           │  Append Operation   │                              │
│                           │  (Append Row)       │                              │
│                           └─────────────────────┘                              │
│                                      │                                          │
└──────────────────────────────────────┼──────────────────────────────────────── ┘
                                       │
                                       ▼
                           NOTIFICATION LAYER
┌───────────────────────────────────────────────────────────────────────────────┐
│                                                                                 │
│  ┌──────────────────┐      ┌────────────────────┐                             │
│  │  Build Summary   │─────▶│  Send Notification │                             │
│  │   (Function)     │      │  (Email/Slack)     │                             │
│  └──────────────────┘      └────────────────────┘                             │
│                                                                                 │
└───────────────────────────────────────────────────────────────────────────────┘

                              ERROR HANDLING
┌───────────────────────────────────────────────────────────────────────────────┐
│                          (Runs on any node failure)                            │
│                                                                                 │
│  ┌──────────────────┐      ┌────────────────────┐      ┌──────────────────┐  │
│  │  Error Trigger   │─────▶│  Log Error         │─────▶│  Send Alert      │  │
│  │  (Error Catch)   │      │  (Function)        │      │  (Email)         │  │
│  └──────────────────┘      └────────────────────┘      └──────────────────┘  │
│                                                                                 │
└───────────────────────────────────────────────────────────────────────────────┘
```

---

## Data Flow Diagram

```
┌─────────────┐
│ Form Input  │
│             │
│ • Business  │
│ • Location  │
│ • Keywords  │
│ • Limit     │
└──────┬──────┘
       │
       ▼
┌──────────────────┐
│ Validated Config │
│                  │
│ {                │
│   businessType   │
│   locations[]    │
│   keywords[]     │
│   leadLimit      │
│ }                │
└────────┬─────────┘
         │
         ▼
┌────────────────────┐
│ Search Loop        │
│                    │
│ For each location: │
│   For each keyword:│
│     Search query   │
│     Page 1, 2, 3...│
└─────────┬──────────┘
          │
          ▼
┌───────────────────┐
│ Raw Business Data │
│                   │
│ • Company name    │
│ • Category        │
│ • Phone           │
│ • Website         │
│ • Address         │
│ • Maps URL        │
└─────────┬─────────┘
          │
          ▼
┌────────────────────┐
│ Enriched Lead      │
│                    │
│ + Owner name       │
│ + Email address    │
│ + LinkedIn URL     │
│ + Additional phone │
└─────────┬──────────┘
          │
          ▼
┌────────────────────┐
│ Quality Checked    │
│                    │
│ ✓ Not duplicate    │
│ ✓ HVAC relevant    │
│ ✓ Has contact info │
│ ✓ Score >= 60      │
└─────────┬──────────┘
          │
          ▼
┌────────────────────┐
│ Stored in Sheet    │
│                    │
│ All 11 columns     │
│ Timestamped        │
│ Append-only        │
└────────────────────┘
```

---

## Node Dependencies

```
Start
  │
  ├─ Form Trigger
  │    └─ Input Validation
  │         └─ Initialize Variables
  │              └─ Loop Control
  │                   ├─ Google Maps Scraper
  │                   │    └─ Parse Results
  │                   │         └─ Rate Limiter
  │                   │              └─ Loop Control (feedback)
  │                   │
  │                   └─ (On parsed results)
  │                        ├─ Website Scraper (parallel)
  │                        ├─ Email Enrichment (parallel)
  │                        └─ LinkedIn Matcher (parallel)
  │                             └─ MCP Enrichment
  │                                  └─ Deduplication
  │                                       └─ Quality Filter
  │                                            └─ Google Sheets
  │                                                 └─ Success Notification
  │
  └─ (Any failure)
       └─ Error Trigger
            └─ Log Error
                 └─ Send Alert
```

---

## Node Configuration Summary

| Node Name | Type | Purpose | Critical? | Continue on Fail? |
|-----------|------|---------|-----------|-------------------|
| Form Trigger | Webhook/Form | Collect inputs | Yes | No |
| Input Validation | Function | Validate data | Yes | No |
| Initialize Variables | Set | Setup vars | Yes | No |
| Loop Control | Function | Manage iteration | Yes | No |
| Google Maps Scraper | HTTP Request | Get businesses | Yes | No |
| Parse Results | Function | Extract data | Yes | No |
| Rate Limiter | Wait | Add delay | Yes | No |
| Website Scraper | HTTP Request | Get owner/email | No | Yes |
| Email Enrichment | HTTP Request | Find email | No | Yes |
| LinkedIn Matcher | HTTP Request | Find LinkedIn | No | Yes |
| MCP Enrichment | AI Agent | Enhance data | No | Yes |
| Deduplication | Function | Remove dupes | Yes | No |
| Quality Filter | IF + Function | Filter leads | Yes | No |
| Google Sheets | Sheets Node | Store data | Yes | No |
| Success Notification | Email/Slack | Send summary | No | Yes |
| Error Trigger | Error Catch | Handle errors | Yes | N/A |
| Log Error | Function | Log details | Yes | No |
| Send Alert | Email | Notify admin | No | Yes |

---

## Execution Flow States

### 1. Idle
- Workflow waiting for trigger
- No resources consumed

### 2. Intake (0-5 seconds)
- Form submitted
- Validation runs
- Variables initialized

### 3. Scraping (10-30 minutes)
- Loop through locations
- API calls to Google Maps
- Rate limiting between requests
- ~3-5 seconds per lead

### 4. Enrichment (10-20 minutes)
- Website scraping
- Email API calls
- LinkedIn API calls
- MCP processing
- ~2-3 seconds per lead

### 5. Quality Control (1-2 minutes)
- Deduplication checks
- Quality filtering
- Lead scoring

### 6. Storage (1-2 minutes)
- Batch append to Google Sheets
- ~100 rows

### 7. Completion (0-5 seconds)
- Summary generation
- Notification sent
- Workflow ends

**Total Time**: 30-60 minutes for 100 leads

---

## Memory & Resource Usage

### Per Node Memory

| Node | Memory | Notes |
|------|--------|-------|
| Functions | 10-50 MB | Depends on array size |
| HTTP Requests | 5-20 MB | Per concurrent request |
| Google Sheets | 50-100 MB | Batch operations |
| Total Workflow | 500 MB - 1 GB | Peak during enrichment |

### Optimization Tips

1. **Process in batches**: Don't load all 100 leads in memory
2. **Clear variables**: Release memory after each batch
3. **Limit concurrency**: Max 3-5 parallel HTTP requests
4. **Use streaming**: For large API responses

---

## Failure Scenarios

### Scenario 1: API Rate Limit

**Symptom**: HTTP 429 errors

**Handling**:
1. Wait node backs off (exponential)
2. Error caught by Error Trigger
3. Workflow pauses for 60 seconds
4. Retries from last checkpoint

### Scenario 2: Invalid API Key

**Symptom**: HTTP 401 errors

**Handling**:
1. Node fails immediately
2. Error Trigger activates
3. Alert sent to admin
4. Workflow stops (requires manual fix)

### Scenario 3: Google Sheets Quota

**Symptom**: Sheets API error

**Handling**:
1. Batch saved to temp storage (Redis/file)
2. Retry after 60 seconds
3. If still fails, alert admin
4. Data not lost

### Scenario 4: Network Timeout

**Symptom**: Request timeout after 30s

**Handling**:
1. Node continues on fail
2. Lead marked as incomplete
3. Logged for manual retry
4. Workflow continues

---

## Scaling Architecture

### Current (100 leads/day)
```
┌─────────────┐
│  Single n8n │
│   Instance  │──▶ APIs ──▶ Google Sheets
│  (1 worker) │
└─────────────┘
```

### Medium (500 leads/day)
```
┌─────────────┐
│    n8n      │
│   (Queue)   │──▶ Workers (3) ──▶ APIs ──▶ Sheets
│             │     ▲
└─────────────┘     │
                 Redis Queue
```

### Large (1000+ leads/day)
```
┌──────────────┐
│  Load        │
│  Balancer    │
└──────┬───────┘
       │
       ├─▶ n8n Instance 1 ──▶ APIs (with proxy) ──┐
       ├─▶ n8n Instance 2 ──▶ APIs (with proxy) ──┼──▶ PostgreSQL
       └─▶ n8n Instance 3 ──▶ APIs (with proxy) ──┘
            │
            └──▶ Export to Sheets (scheduled)
```

---

## Performance Metrics

### Target Benchmarks

| Metric | Target | Acceptable | Poor |
|--------|--------|------------|------|
| Execution Time | <45 min | <60 min | >60 min |
| Leads/Minute | 2-3 | 1-2 | <1 |
| Error Rate | <2% | <5% | >5% |
| Enrichment Rate | >85% | >70% | <70% |
| Duplicate Rate | <3% | <5% | >5% |
| API Success | >98% | >95% | <95% |

### Monitoring Dashboard

Track these in real-time:

1. **Current Status**: Running / Idle / Error
2. **Progress**: Leads collected / Target
3. **Success Rate**: % of successful operations
4. **Enrichment**: % with email, owner, LinkedIn
5. **Errors**: Count and types
6. **API Usage**: Calls remaining
7. **ETA**: Estimated completion time

---

## Security Layers

### 1. Input Security
- Validate all form inputs
- Sanitize strings (SQL injection prevention)
- Rate limit form submissions

### 2. API Security
- Store keys in n8n credentials (encrypted)
- Use environment variables
- Rotate keys monthly
- IP whitelist where possible

### 3. Data Security
- Encrypt data at rest (Google Sheets)
- Use HTTPS for all API calls
- No sensitive data in logs
- Implement data retention policy

### 4. Access Control
- n8n authentication required
- Role-based access (if team)
- Audit logs for all executions
- Separate prod/dev environments

---

## Next Steps

1. **Import Workflow**: Use `workflow/n8n-workflow.json`
2. **Configure APIs**: Follow [API Integrations Guide](../docs/api-integrations.md)
3. **Test**: Run with 5 leads first
4. **Monitor**: Watch first few executions
5. **Scale**: Gradually increase to 100/day

See [Implementation Guide](../docs/implementation-guide.md) for detailed setup.
