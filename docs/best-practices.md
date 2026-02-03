# Best Practices Guide

## Scaling, Compliance, and Reliability

This document outlines best practices for running the HVAC lead automation workflow at scale while maintaining compliance, reliability, and data quality.

---

## 1. Compliance & Legal

### Terms of Service (ToS)

#### Google Maps
- **Risk**: Scraping violates Google Maps ToS
- **Mitigation**:
  - Use official Google Places API where possible
  - Use third-party services (Outscraper, SerpApi) that handle compliance
  - Add delays between requests (2-5 seconds)
  - Rotate IPs using proxy services
  - Never use Google Maps Platform API keys for scraping

#### LinkedIn
- **Risk**: Automated scraping violates LinkedIn ToS
- **Mitigation**:
  - Use official LinkedIn API where available
  - Use third-party services with proper licenses
  - Don't create fake accounts
  - Limit request frequency

### Data Privacy (GDPR, CCPA)

#### Compliance Requirements

1. **Data Collection**:
   - Only collect publicly available information
   - Have legitimate business purpose
   - Implement data minimization

2. **Data Storage**:
   - Encrypt data at rest
   - Limit access to authorized personnel
   - Implement data retention policies (delete after 1 year)

3. **Data Usage**:
   - Use only for B2B outreach
   - Provide opt-out mechanism
   - Don't sell data to third parties

4. **Privacy Policy**:
   - Maintain clear privacy policy
   - Explain data collection methods
   - Provide contact information

#### GDPR-Specific

```javascript
// Add to enrichment nodes
const gdprCompliance = {
  dataController: "Your Company Name",
  legalBasis: "Legitimate interest (B2B marketing)",
  retentionPeriod: "12 months",
  rightsInfo: "Subjects can request deletion via privacy@yourcompany.com"
};
```

#### Implementation

- Add "Date Collected" column in Google Sheets
- Implement automated deletion after 12 months
- Create opt-out form/process
- Log consent for email communications

---

## 2. Rate Limiting & Anti-Detection

### Request Patterns

#### Human-Like Behavior

```javascript
// Variable delays (not fixed intervals)
const getRandomDelay = () => {
  // 2-5 seconds with occasional longer pauses
  const base = Math.floor(Math.random() * 3000) + 2000;
  const occasional = Math.random() < 0.1 ? 10000 : 0; // 10% chance of 10s pause
  return base + occasional;
};
```

#### Request Headers

```javascript
// Rotate user agents
const userAgents = [
  'Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36',
  'Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36',
  'Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/537.36'
];

const headers = {
  'User-Agent': userAgents[Math.floor(Math.random() * userAgents.length)],
  'Accept-Language': 'en-US,en;q=0.9',
  'Accept': 'text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8',
  'Referer': 'https://www.google.com/'
};
```

### Time-Based Limits

```javascript
// Hourly limits
const MAX_REQUESTS_PER_HOUR = 200;
const MAX_REQUESTS_PER_DAY = 3000;

// Track in workflow
const requestLog = {
  hour: 0,
  day: 0,
  lastReset: new Date()
};
```

### Circuit Breaker Pattern

```javascript
// Stop workflow if too many errors
const errorThreshold = 10;
const errorWindow = 60000; // 1 minute

if (recentErrors.length > errorThreshold) {
  throw new Error('Circuit breaker triggered - too many errors');
}
```

---

## 3. Data Quality

### Validation Rules

#### Phone Numbers

```javascript
function validatePhone(phone) {
  // Remove formatting
  const cleaned = phone.replace(/[^\d+]/g, '');
  
  // US numbers: 10 digits or +1 and 10 digits
  if (cleaned.length === 10 || (cleaned.startsWith('+1') && cleaned.length === 12)) {
    return true;
  }
  
  // International: + followed by 7-15 digits
  if (cleaned.startsWith('+') && cleaned.length >= 8 && cleaned.length <= 16) {
    return true;
  }
  
  return false;
}
```

#### Emails

```javascript
function validateEmail(email) {
  // RFC 5322 simplified
  const regex = /^[a-zA-Z0-9.!#$%&'*+/=?^_`{|}~-]+@[a-zA-Z0-9](?:[a-zA-Z0-9-]{0,61}[a-zA-Z0-9])?(?:\.[a-zA-Z0-9](?:[a-zA-Z0-9-]{0,61}[a-zA-Z0-9])?)*$/;
  
  if (!regex.test(email)) return false;
  
  // Blacklist common fake domains
  const blacklist = ['example.com', 'test.com', 'mail.com', 'email.com'];
  const domain = email.split('@')[1].toLowerCase();
  
  return !blacklist.includes(domain);
}
```

#### Websites

```javascript
function validateWebsite(url) {
  try {
    const parsed = new URL(url);
    
    // Must be http or https
    if (!['http:', 'https:'].includes(parsed.protocol)) {
      return false;
    }
    
    // Must have valid domain
    if (!parsed.hostname.includes('.')) {
      return false;
    }
    
    // Blacklist common placeholder domains
    const blacklist = ['example.com', 'yoursite.com', 'website.com'];
    return !blacklist.includes(parsed.hostname);
    
  } catch {
    return false;
  }
}
```

### Lead Scoring

```javascript
function scoreLead(lead) {
  let score = 0;
  
  // Has company name (required)
  if (lead.companyName) score += 10;
  
  // Has phone
  if (lead.phoneNumbers && lead.phoneNumbers.length > 0) score += 20;
  
  // Has email
  if (lead.email) score += 25;
  
  // Has website
  if (lead.website) score += 15;
  
  // Has owner name
  if (lead.ownerFirstName) score += 15;
  
  // Has LinkedIn
  if (lead.linkedinCompanyUrl) score += 10;
  
  // High rating (4.0+)
  if (lead.rating >= 4.0) score += 5;
  
  // Many reviews (100+)
  if (lead.reviewCount >= 100) score += 5;
  
  return score; // Max: 105 points
}

// Only keep leads with score >= 60
const qualifiedLeads = leads.filter(lead => scoreLead(lead) >= 60);
```

### Data Enrichment Quality

```javascript
// Track enrichment success rate
const enrichmentStats = {
  totalLeads: 0,
  withEmail: 0,
  withOwner: 0,
  withLinkedIn: 0,
  fullyEnriched: 0
};

// A fully enriched lead has: name, phone, email, website, owner, LinkedIn
if (lead.email && lead.ownerFirstName && lead.linkedinCompanyUrl) {
  enrichmentStats.fullyEnriched++;
}

// Target: >80% enrichment rate
const enrichmentRate = (enrichmentStats.fullyEnriched / enrichmentStats.totalLeads) * 100;
```

---

## 4. Error Handling

### Comprehensive Error Catching

```javascript
// Wrap API calls in try-catch
try {
  const response = await fetch(apiUrl);
  const data = await response.json();
  return data;
} catch (error) {
  // Log error with context
  console.error('API Error:', {
    url: apiUrl,
    error: error.message,
    timestamp: new Date().toISOString(),
    lead: lead.companyName
  });
  
  // Return null instead of failing workflow
  return null;
}
```

### Graceful Degradation

```javascript
// If email enrichment fails, continue without email
lead.email = await getEmailFromHunter(lead.website) || null;

// If LinkedIn fails, continue without LinkedIn
lead.linkedinUrl = await getLinkedInProfile(lead.companyName) || null;

// Only fail if critical data is missing
if (!lead.companyName || (!lead.phoneNumbers?.length && !lead.website)) {
  throw new Error('Insufficient lead data');
}
```

### Error Notification

```javascript
// Set up error thresholds
const ERROR_THRESHOLD = 10; // Alert if more than 10 errors

if (errorCount > ERROR_THRESHOLD) {
  // Send alert email/Slack
  await sendAlert({
    severity: 'HIGH',
    message: `Workflow has ${errorCount} errors`,
    workflow: 'HVAC Lead Generation',
    timestamp: new Date().toISOString()
  });
}
```

---

## 5. Performance Optimization

### Parallel Processing

```javascript
// Process enrichment in parallel
const enrichedLead = await Promise.all([
  getEmailFromHunter(lead.website),
  getLinkedInProfile(lead.companyName),
  scrapeWebsiteForOwner(lead.website)
]).then(([email, linkedin, owner]) => ({
  ...lead,
  email,
  linkedinUrl: linkedin,
  ownerFirstName: owner
}));
```

### Caching Strategy

```javascript
// Cache domain lookups (24 hours)
const domainCache = new Map();

async function getEmailWithCache(domain) {
  const cacheKey = `email:${domain}`;
  
  // Check cache
  if (domainCache.has(cacheKey)) {
    const cached = domainCache.get(cacheKey);
    if (Date.now() - cached.timestamp < 86400000) { // 24 hours
      return cached.data;
    }
  }
  
  // Fetch from API
  const email = await getEmailFromHunter(domain);
  
  // Store in cache
  domainCache.set(cacheKey, {
    data: email,
    timestamp: Date.now()
  });
  
  return email;
}
```

### Batch Processing

```javascript
// Process in batches of 25
const BATCH_SIZE = 25;

for (let i = 0; i < leads.length; i += BATCH_SIZE) {
  const batch = leads.slice(i, i + BATCH_SIZE);
  
  // Process batch
  const enrichedBatch = await enrichLeads(batch);
  
  // Save to sheets
  await appendToSheet(enrichedBatch);
  
  // Delay between batches
  await delay(5000);
}
```

---

## 6. Monitoring & Analytics

### Key Metrics

Track these metrics daily:

1. **Volume Metrics**:
   - Total leads scraped
   - Qualified leads
   - Duplicates removed
   - Leads saved to sheet

2. **Quality Metrics**:
   - Enrichment rate (% with email)
   - Owner identification rate
   - LinkedIn match rate
   - Average lead score

3. **Performance Metrics**:
   - Execution time
   - API response times
   - Error rate
   - Success rate

4. **Cost Metrics**:
   - API calls per day
   - Cost per lead
   - Cost per qualified lead

### Dashboard Setup

```javascript
// Generate daily report
const dailyReport = {
  date: new Date().toISOString().split('T')[0],
  metrics: {
    leadsScraped: 100,
    qualifiedLeads: 87,
    duplicates: 13,
    withEmail: 72,
    withOwner: 65,
    withLinkedIn: 45,
    executionTime: 2340, // seconds
    errors: 3,
    apiCalls: {
      googleMaps: 150,
      hunter: 87,
      linkedin: 87
    },
    costs: {
      total: 8.70,
      perLead: 0.087
    }
  }
};

// Save to monitoring sheet
await appendToSheet('Metrics', dailyReport);
```

### Alerting

Set up alerts for:
- Error rate >5%
- Enrichment rate <70%
- Execution time >60 minutes
- API errors
- Daily lead count <80

---

## 7. Maintenance

### Daily Tasks

- [ ] Check workflow execution status
- [ ] Review error logs
- [ ] Verify lead quality (spot check 10 leads)
- [ ] Monitor API usage

### Weekly Tasks

- [ ] Analyze enrichment success rates
- [ ] Update search keywords if needed
- [ ] Review and remove duplicates
- [ ] Check API quotas/billing

### Monthly Tasks

- [ ] Performance review (all metrics)
- [ ] Cost analysis
- [ ] Keyword optimization
- [ ] Clean up old leads (>12 months)
- [ ] Update API credentials
- [ ] Review and update documentation

### Quarterly Tasks

- [ ] Full workflow audit
- [ ] API provider evaluation
- [ ] Scale planning
- [ ] Process improvement review

---

## 8. Scaling Guidelines

### 100 Leads/Day (Current)

- Standard setup
- Single workflow instance
- Basic rate limiting
- Free/low-tier API plans

### 500 Leads/Day

- Upgrade API plans
- Add proxy rotation
- Implement Redis caching
- Split workflows by region

### 1,000+ Leads/Day

- Multiple workflow instances
- Dedicated scraping infrastructure
- Enterprise API plans
- Queue-based processing (Bull/Redis)
- Database instead of Google Sheets
- Dedicated monitoring/alerting

---

## 9. Backup & Recovery

### Data Backup

```javascript
// Daily backup to CSV
const backup = await exportSheetToCSV();
await uploadToS3(backup, `backup-${date}.csv`);
```

### Workflow Backup

- Export workflow JSON weekly
- Version control in Git
- Document any manual changes

### Recovery Plan

1. **Workflow Failure**:
   - Check error logs
   - Restore from last working version
   - Resume from last successful batch

2. **Data Loss**:
   - Restore from daily CSV backup
   - Re-scrape if necessary
   - Verify data integrity

3. **API Outage**:
   - Switch to alternative provider
   - Queue failed requests for retry
   - Continue with available APIs

---

## 10. Testing Strategy

### Test Scenarios

1. **Happy Path**:
   - 5 leads, all fields enriched
   - Verify all data in sheet
   - Check for proper formatting

2. **Missing Data**:
   - Leads without website
   - Leads without phone
   - Verify graceful handling

3. **Duplicates**:
   - Submit same lead twice
   - Verify deduplication works
   - Check no duplicates in sheet

4. **API Failures**:
   - Simulate API errors
   - Verify fallback behavior
   - Check error logging

5. **Rate Limiting**:
   - High volume test (50 leads)
   - Verify delays working
   - Check no API blocks

### Testing Checklist

- [ ] Form validation
- [ ] Input parsing
- [ ] Google Maps scraping
- [ ] Website scraping
- [ ] Email enrichment
- [ ] LinkedIn matching
- [ ] Deduplication
- [ ] Quality filtering
- [ ] Sheet append
- [ ] Error handling
- [ ] Notifications

---

## Summary

| Priority | Practice | Impact |
|----------|----------|--------|
| **HIGH** | Rate limiting | Avoid bans |
| **HIGH** | Error handling | Reliability |
| **HIGH** | Data validation | Quality |
| **MEDIUM** | GDPR compliance | Legal |
| **MEDIUM** | Monitoring | Visibility |
| **MEDIUM** | Caching | Performance |
| **LOW** | Parallel processing | Speed |
| **LOW** | Lead scoring | Targeting |

---

**Next Steps**:
1. Implement rate limiting
2. Set up error handling
3. Add data validation
4. Configure monitoring
5. Test thoroughly before production

See [Implementation Guide](implementation-guide.md) for deployment.
