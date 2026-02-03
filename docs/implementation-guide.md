# Implementation Guide

## Step-by-Step Setup Instructions

This guide walks you through deploying the HVAC lead automation workflow on your self-hosted n8n instance.

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

You'll need to obtain API keys for the following services:

1. **Google Maps Scraping**
   - **Option A**: Outscraper API (https://outscraper.com)
   - **Option B**: SerpApi (https://serpapi.com)
   - **Option C**: ScraperAPI (https://www.scraperapi.com)

2. **Email Enrichment**
   - Hunter.io API (https://hunter.io/api)
   - Alternative: Clearbit, RocketReach, or Snov.io

3. **LinkedIn Scraping**
   - RapidAPI LinkedIn Scraper
   - Alternative: Proxycurl, PhantomBuster

4. **Google Sheets**
   - Google Cloud Project with Sheets API enabled
   - Service Account credentials

### 3. System Requirements

- **VPS**: 2GB RAM minimum, 4GB recommended
- **Storage**: 20GB available
- **OS**: Ubuntu 20.04+ or similar
- **Node.js**: Version 16+
- **Network**: Stable connection with proxy support (optional)

---

## Installation Steps

### Step 1: Import Workflow

1. Open n8n in your browser (http://your-server:5678)
2. Click **Workflows** → **Import from File**
3. Select `workflow/n8n-workflow.json` from this repository
4. The workflow will be imported with all nodes configured

### Step 2: Set Up Credentials

#### Google Sheets Credentials

1. Go to Google Cloud Console (https://console.cloud.google.com)
2. Create a new project or select existing
3. Enable Google Sheets API
4. Create Service Account:
   - IAM & Admin → Service Accounts
   - Create Service Account
   - Download JSON key file
5. In n8n:
   - Settings → Credentials → New
   - Type: Google Service Account
   - Upload JSON key file
6. Share your Google Sheet with the service account email

#### Outscraper API Credentials

1. Sign up at https://outscraper.com
2. Get API key from dashboard
3. In n8n:
   - Settings → Credentials → New
   - Type: Header Auth
   - Name: `X-API-Key`
   - Value: Your API key

#### Hunter.io Credentials

1. Sign up at https://hunter.io
2. Get API key from API section
3. In n8n:
   - Settings → Credentials → New
   - Type: Query Auth
   - Name: `api_key`
   - Value: Your API key

#### RapidAPI Credentials

1. Sign up at https://rapidapi.com
2. Subscribe to LinkedIn API
3. Copy your RapidAPI key
4. In n8n:
   - Settings → Credentials → New
   - Type: Header Auth
   - Name: `X-RapidAPI-Key`
   - Value: Your API key

### Step 3: Configure Google Sheet

1. Create a new Google Sheet
2. Name it "HVAC Leads Database"
3. Create a sheet tab called "Leads"
4. Add headers in row 1:
   ```
   Company Name | Owner First Name | Email | Phone Numbers | Website | LinkedIn URL | Category | Location | Google Maps URL | Date Scraped | Source Query
   ```
5. Copy the Sheet ID from URL:
   - URL format: `https://docs.google.com/spreadsheets/d/{SHEET_ID}/edit`
6. Update the workflow:
   - Open "Append to Leads Sheet" node
   - Enter your Sheet ID in credentials

### Step 4: Test the Workflow with Form Popup

#### Manual Test Run

1. With the workflow open in n8n, click the **"Execute Workflow"** button (top right)
2. A **form popup will appear** on your screen with fields:
   - **Business Type**: Enter your target industry
   - **Target Locations**: Enter cities (one per line)
   - **Search Keywords**: Enter comma-separated keywords
   - **Daily Lead Limit**: Enter number of leads (recommended: 5 for testing)

3. Example test values:
   ```
   Business Type: HVAC contractor
   Target Locations: 
   Phoenix, AZ
   
   Search Keywords: HVAC repair, AC service
   Daily Lead Limit: 5
   ```

4. Click **"Execute"** in the form popup
5. The workflow will run with your entered parameters
6. Monitor the execution:
   - Watch each node turn green as it completes
   - Check for any red (error) or yellow (warning) nodes

#### Verify Results

1. Open your Google Sheet
2. Verify 5 leads were added
3. Check data quality:
   - All fields populated?
   - No duplicates?
   - HVAC-relevant businesses?

### Step 5: Optional - Schedule Daily Execution

If you want the workflow to run automatically on a schedule:

1. Add a **Schedule Trigger** node to the workflow:
   - Click the "+" button to add a new node
   - Search for "Schedule Trigger"
   - Add it to the canvas

2. Configure the schedule:
   - **Trigger Interval**: Choose "Cron"
   - **Cron Expression**: `0 9 * * *` (runs at 9 AM daily)
   - **Timezone**: Select your local timezone

3. Configure default values for scheduled runs:
   - When using a Schedule Trigger, you'll need to provide default values
   - Add a "Set" node after the Schedule Trigger with default parameters
   - Connect: Schedule Trigger → Set (with defaults) → Input Validation

4. Connect the Execute Workflow Trigger:
   - Keep "Execute Workflow Trigger" connected for manual runs with form popup
   - Add "Schedule Trigger" → "Set" (defaults) → "Input Validation" for automated runs
   - This allows both manual (with form) and scheduled execution

5. Save and activate the workflow:
   - Click "Save" button
   - Toggle the "Active" switch (top-right) if you want scheduled runs

**Note**: With both triggers:
- Manual execution shows the form popup for you to enter values
- Scheduled execution uses the default values from the Set node

---

## Configuration Options

### Running Manual Tests

Each time you click "Execute Workflow", you'll see a form popup where you can:
- Adjust the lead limit (use 5-10 for quick tests)
- Change target locations
- Modify search keywords
- Update business type

### For Scheduled Runs

If you add a Schedule Trigger for automation:
1. Add a "Set" node after the Schedule Trigger
2. Configure default values in that Set node
3. Connect: Schedule Trigger → Set (defaults) → Input Validation

### Changing Default Form Values

Edit the "Execute Workflow Trigger" node:
- Double-click the node
- Modify the "Default Value" for each field
- These appear pre-filled in the form popup

### Rate Limiting

Adjust delay in "Rate Limiter" node:
```json
{
  "unit": "seconds",
  "amount": "={{ Math.floor(Math.random() * 3) + 2 }}"
}
```
- Current: 2-5 seconds (safe)
- Aggressive: 1-2 seconds (risky)
- Conservative: 5-10 seconds (very safe)

---

## Troubleshooting

### Common Issues

#### 1. Form Not Loading

**Problem**: Form URL returns 404

**Solution**:
- Check n8n is running
- Verify webhook path in Form Trigger node
- Restart n8n service

#### 2. Google Maps Scraping Fails

**Problem**: HTTP 403 or 429 errors

**Solutions**:
- Verify API key is correct
- Check API quota/limits
- Increase rate limiter delay
- Use proxy/VPN
- Switch to different scraping service

#### 3. No Emails Found

**Problem**: Email enrichment returns empty

**Solutions**:
- Verify Hunter.io API key
- Check if domain has MX records
- Try alternative email finder
- Increase search limit in Hunter.io node

#### 4. Duplicates in Sheet

**Problem**: Same companies appearing multiple times

**Solutions**:
- Check deduplication function
- Verify hash comparison logic
- Add Google Sheets lookup step
- Clear test data from sheet

#### 5. LinkedIn Not Matching

**Problem**: LinkedIn URLs not found

**Solutions**:
- Verify RapidAPI subscription is active
- Check company name formatting
- Increase search tolerance
- Use fuzzy matching

### Error Messages

#### "API quota exceeded"
- Wait for quota reset (usually 24 hours)
- Upgrade API plan
- Reduce daily lead limit

#### "Invalid credentials"
- Re-enter API keys
- Check credential type matches node expectation
- Verify no extra spaces in keys

#### "Timeout error"
- Increase timeout in HTTP node (30-60 seconds)
- Check internet connection
- Verify target site is accessible

---

## Performance Optimization

### Parallel Processing

Enable parallel execution in n8n settings:
```json
{
  "executions": {
    "mode": "queue",
    "concurrency": 3
  }
}
```

### Caching

Add caching for repeated lookups:
- Use Redis for deduplication cache
- Cache domain lookups (24 hours)
- Cache LinkedIn searches (7 days)

### Selective Enrichment

Only enrich high-quality leads:
- Skip enrichment if no website
- Skip LinkedIn if no owner name
- Skip email if already found

### Batch Processing

Process in batches to manage memory:
- 20-50 leads per batch
- Clear variables between batches
- Use workflow splitting

---

## Scaling Considerations

### 100+ Leads Per Day

- Use proxy rotation service
- Implement IP rotation
- Add delays between batches
- Monitor API quotas

### Multiple Niches

- Clone workflow for each niche
- Use shared Google Sheet with niche column
- Implement queue system

### Enterprise Scale (1000+ leads/day)

- Use dedicated scraping infrastructure
- Implement job queue (Bull/Redis)
- Add horizontal scaling
- Use enterprise API plans

---

## Maintenance Schedule

### Daily
- Check execution logs
- Verify lead quality
- Monitor API usage

### Weekly
- Review error logs
- Update search keywords
- Clean duplicate entries

### Monthly
- Analyze success metrics
- Update API credentials
- Review and optimize nodes
- Check API usage vs. cost

---

## Next Steps

1. **Test thoroughly** with small limits (5-10 leads)
2. **Monitor** first few daily runs
3. **Optimize** based on results
4. **Scale** gradually to 100 leads/day
5. **Review** [Best Practices](best-practices.md) for long-term success

---

## Support

For issues specific to:
- **n8n**: https://community.n8n.io
- **This workflow**: Open GitHub issue
- **API services**: Contact respective support teams

