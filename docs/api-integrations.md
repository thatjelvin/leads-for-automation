# API Integrations Guide

## Overview

This document details all API integrations used in the HVAC lead automation workflow, including setup instructions, rate limits, costs, and alternatives.

---

## 1. Google Maps Scraping

### Option A: Outscraper (Recommended)

**Website**: https://outscraper.com

**Pricing**:
- Free: 100 requests/month
- Starter: $29/month (2,500 requests)
- Business: $99/month (15,000 requests)

**Rate Limits**: 20 requests/second

**Setup**:
1. Sign up at Outscraper
2. Navigate to API section
3. Copy API key
4. Add to n8n as Header Auth credential

**API Endpoint**:
```
GET https://api.outscraper.com/maps/search-v3
```

**Parameters**:
- `query`: Search query (e.g., "HVAC contractor in Los Angeles")
- `limit`: Results per request (max 500)
- `language`: Language code (default: en)
- `region`: Country code
- `skip`: Pagination offset

**Example Request**:
```bash
curl -X GET "https://api.outscraper.com/maps/search-v3?query=HVAC%20contractor%20Los%20Angeles&limit=20" \
  -H "X-API-Key: YOUR_API_KEY"
```

**Response Format**:
```json
{
  "data": [
    {
      "name": "ABC HVAC Services",
      "phone": "+1 310-555-1234",
      "site": "https://abchvac.com",
      "type": "HVAC contractor",
      "address": "123 Main St, Los Angeles, CA 90001",
      "rating": 4.5,
      "reviews": 150,
      "latitude": 34.0522,
      "longitude": -118.2437,
      "google_id": "ChIJxxxxx",
      "url": "https://maps.google.com/?cid=123456789"
    }
  ]
}
```

### Option B: SerpApi

**Website**: https://serpapi.com

**Pricing**:
- Free: 100 searches/month
- Developer: $50/month (5,000 searches)
- Production: $150/month (20,000 searches)

**API Endpoint**:
```
GET https://serpapi.com/search.json
```

**Parameters**:
- `engine`: google_maps
- `q`: Search query
- `ll`: Latitude/longitude (optional)
- `start`: Pagination

**n8n Configuration**:
```json
{
  "method": "GET",
  "url": "https://serpapi.com/search.json",
  "qs": {
    "engine": "google_maps",
    "q": "={{ $json.searchQuery }}",
    "api_key": "={{ $credentials.serpApiKey }}",
    "num": 20
  }
}
```

### Option C: ScraperAPI

**Website**: https://www.scraperapi.com

**Pricing**:
- Free: 1,000 requests/month
- Hobby: $29/month (50,000 requests)
- Startup: $99/month (500,000 requests)

**Features**:
- Automatic proxy rotation
- CAPTCHA solving
- JavaScript rendering

**API Endpoint**:
```
GET http://api.scraperapi.com/
```

**Parameters**:
- `api_key`: Your API key
- `url`: Target URL (Google Maps)
- `render`: true (for JS rendering)

---

## 2. Email Enrichment

### Option A: Hunter.io (Recommended)

**Website**: https://hunter.io

**Pricing**:
- Free: 25 searches/month
- Starter: $49/month (500 searches)
- Growth: $99/month (2,500 searches)

**Rate Limits**: 10 requests/second

**API Endpoint**:
```
GET https://api.hunter.io/v2/domain-search
```

**Parameters**:
- `domain`: Company domain
- `api_key`: Your API key
- `limit`: Results limit
- `type`: personal/generic

**Example Request**:
```bash
curl "https://api.hunter.io/v2/domain-search?domain=abchvac.com&api_key=YOUR_KEY"
```

**Response**:
```json
{
  "data": {
    "domain": "abchvac.com",
    "emails": [
      {
        "value": "john@abchvac.com",
        "type": "personal",
        "confidence": 95,
        "first_name": "John",
        "last_name": "Smith",
        "position": "Owner"
      }
    ]
  }
}
```

**n8n Node Configuration**:
```json
{
  "name": "Hunter Email Finder",
  "type": "n8n-nodes-base.httpRequest",
  "parameters": {
    "url": "https://api.hunter.io/v2/domain-search",
    "qs": {
      "domain": "={{ $json.website.replace('https://', '').split('/')[0] }}",
      "api_key": "={{ $credentials.hunterApiKey }}",
      "type": "personal",
      "limit": 3
    }
  }
}
```

### Option B: Clearbit

**Website**: https://clearbit.com

**Pricing**: Starting at $99/month

**API Endpoint**:
```
GET https://person.clearbit.com/v2/combined/find
```

### Option C: Snov.io

**Website**: https://snov.io

**Pricing**:
- Trial: 50 credits
- Starter: $39/month (1,000 credits)

---

## 3. LinkedIn Scraping

### Option A: RapidAPI LinkedIn Scraper

**Website**: https://rapidapi.com/rockapis-rockapis-default/api/linkedin-api8

**Pricing**:
- Free: 100 requests/month
- Basic: $15/month (1,000 requests)
- Pro: $50/month (10,000 requests)

**Endpoints**:
1. Company Search: `/search/company`
2. Person Search: `/search/person`
3. Company Profile: `/get-company-details`

**Example - Company Search**:
```bash
curl -X GET "https://linkedin-api8.p.rapidapi.com/search/company?keywords=ABC%20HVAC" \
  -H "X-RapidAPI-Key: YOUR_KEY" \
  -H "X-RapidAPI-Host: linkedin-api8.p.rapidapi.com"
```

**n8n Configuration**:
```json
{
  "name": "LinkedIn Company Search",
  "type": "n8n-nodes-base.httpRequest",
  "parameters": {
    "method": "GET",
    "url": "https://linkedin-api8.p.rapidapi.com/search/company",
    "headers": {
      "X-RapidAPI-Key": "={{ $credentials.rapidApiKey }}",
      "X-RapidAPI-Host": "linkedin-api8.p.rapidapi.com"
    },
    "qs": {
      "keywords": "={{ $json.companyName }}"
    }
  }
}
```

### Option B: Proxycurl

**Website**: https://nubela.co/proxycurl

**Pricing**: Pay-as-you-go, ~$0.01-0.02/request

**Features**:
- Real-time LinkedIn data
- No rate limits
- High accuracy

### Option C: PhantomBuster

**Website**: https://phantombuster.com

**Pricing**: Starting at $30/month

---

## 4. Google Sheets Integration

### Setup Instructions

1. **Create Google Cloud Project**:
   - Go to https://console.cloud.google.com
   - Click "New Project"
   - Name: "HVAC Lead Automation"

2. **Enable Google Sheets API**:
   - APIs & Services → Library
   - Search "Google Sheets API"
   - Click Enable

3. **Create Service Account**:
   - APIs & Services → Credentials
   - Create Credentials → Service Account
   - Name: "n8n-sheets-access"
   - Role: Editor
   - Create and download JSON key

4. **Configure in n8n**:
   - Settings → Credentials → New
   - Type: Google Service Account
   - Upload JSON file
   - Test connection

5. **Share Sheet**:
   - Open your Google Sheet
   - Click Share
   - Add service account email (found in JSON)
   - Give Editor access

**API Limits**:
- 300 requests/minute per user
- 500 requests/100 seconds per project

**n8n Node Configuration**:
```json
{
  "name": "Google Sheets Append",
  "type": "n8n-nodes-base.googleSheets",
  "parameters": {
    "operation": "append",
    "sheetId": "YOUR_SHEET_ID",
    "range": "Leads!A:K",
    "options": {
      "valueInputMode": "USER_ENTERED"
    }
  }
}
```

---

## 5. Proxy Services (Optional but Recommended)

### Why Use Proxies?

- Avoid IP bans
- Distribute requests across IPs
- Bypass rate limits
- Appear as different users

### Recommended Providers

#### Bright Data (formerly Luminati)

**Website**: https://brightdata.com

**Pricing**: Starting at $500/month

**Features**:
- 72M+ residential IPs
- City/state targeting
- 99.9% uptime

#### Smartproxy

**Website**: https://smartproxy.com

**Pricing**:
- Starter: $50/month (5GB)
- Business: $200/month (25GB)

**Setup**:
```javascript
// In HTTP Request nodes
const proxyUrl = 'http://user:pass@proxy.smartproxy.com:10001';

// Add to node
{
  "proxy": proxyUrl
}
```

#### Oxylabs

**Website**: https://oxylabs.io

**Pricing**: Starting at $300/month

---

## 6. Alternative Data Sources

### ZoomInfo

**Use Case**: Enterprise B2B data

**Pricing**: Custom (expensive)

**Integration**: REST API

### Apollo.io

**Use Case**: B2B contact database

**Pricing**: Free tier available

**API**: https://apolloio.github.io/apollo-api-docs/

### Crunchbase

**Use Case**: Startup/company information

**Pricing**: Starting at $29/month

---

## API Cost Calculator

For 100 leads/day (3,000/month):

| Service | Usage | Cost |
|---------|-------|------|
| Outscraper | 3,000 searches | $99/month |
| Hunter.io | 3,000 email searches | $99/month |
| RapidAPI LinkedIn | 3,000 searches | $50/month |
| Google Sheets | Free (under limits) | $0 |
| **Total** | | **$248/month** |

**Cost per lead**: $0.08

---

## Rate Limiting Strategy

### Per-API Limits

```javascript
// Outscraper: 20 req/sec = 0.05s between requests
// Hunter.io: 10 req/sec = 0.1s between requests
// LinkedIn: No official limit, use 1 req/sec = 1s

// Combined strategy: 2-5 second delays
const delay = Math.floor(Math.random() * 3000) + 2000; // 2-5 seconds
await new Promise(resolve => setTimeout(resolve, delay));
```

### Daily Quotas

Monitor usage in n8n:

```javascript
// Track API calls
const apiUsage = {
  outscraper: 0,
  hunter: 0,
  linkedin: 0,
  date: new Date().toISOString().split('T')[0]
};

// Store in workflow static data or external database
```

---

## Error Handling

### API-Specific Error Codes

#### Outscraper
- 401: Invalid API key
- 429: Rate limit exceeded
- 500: Server error

#### Hunter.io
- 401: Unauthorized
- 429: Too many requests
- 404: Domain not found

#### LinkedIn (RapidAPI)
- 403: Subscription expired
- 429: Rate limit
- 500: Service unavailable

### Retry Strategy

```javascript
// Exponential backoff
async function retryWithBackoff(fn, maxRetries = 3) {
  for (let i = 0; i < maxRetries; i++) {
    try {
      return await fn();
    } catch (error) {
      if (i === maxRetries - 1) throw error;
      const delay = Math.pow(2, i) * 1000; // 1s, 2s, 4s
      await new Promise(resolve => setTimeout(resolve, delay));
    }
  }
}
```

---

## Security Best Practices

1. **Environment Variables**: Store API keys in n8n credentials, never in code
2. **Rotate Keys**: Change keys monthly
3. **Monitor Usage**: Set up alerts for unusual activity
4. **IP Whitelist**: Where possible, restrict API access to your VPS IP
5. **Audit Logs**: Keep track of all API calls

---

## Testing APIs

### Test Each API Individually

```bash
# Test Outscraper
curl -X GET "https://api.outscraper.com/maps/search-v3?query=HVAC%20Phoenix&limit=1" \
  -H "X-API-Key: YOUR_KEY"

# Test Hunter.io
curl "https://api.hunter.io/v2/domain-search?domain=example.com&api_key=YOUR_KEY"

# Test LinkedIn
curl -X GET "https://linkedin-api8.p.rapidapi.com/search/company?keywords=HVAC" \
  -H "X-RapidAPI-Key: YOUR_KEY"
```

### Validate Responses

Check for:
- Correct status code (200)
- Valid JSON structure
- Expected data fields
- No error messages

---

## Next Steps

1. **Sign up** for each required API service
2. **Test** each API with provided curl commands
3. **Add credentials** to n8n
4. **Configure nodes** with your API keys
5. **Test workflow** with small lead limit

See [Implementation Guide](implementation-guide.md) for complete setup.
