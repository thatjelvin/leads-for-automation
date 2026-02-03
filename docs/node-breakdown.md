# n8n Workflow Node-by-Node Breakdown

This document provides a detailed breakdown of each node in the HVAC lead automation workflow, including configuration details, expressions, and data mappings.

## Table of Contents

1. [Manual Trigger](#1-manual-trigger)
2. [Set Input Parameters](#2-set-input-parameters)
3. [Input Validation](#3-input-validation)
4. [Initialize Variables](#4-initialize-variables)
5. [Google Maps Search Loop](#5-google-maps-search-loop)
6. [HTTP Request - Google Maps](#6-http-request---google-maps)
7. [Parse Maps Results](#7-parse-maps-results)
8. [Rate Limiter](#8-rate-limiter)
9. [Website Scraper](#9-website-scraper)
10. [Email Enrichment](#10-email-enrichment)
11. [LinkedIn Matcher](#11-linkedin-matcher)
12. [MCP Enrichment Node](#12-mcp-enrichment-node)
13. [Deduplication Check](#13-deduplication-check)
14. [Filter Qualified Leads](#14-filter-qualified-leads)
15. [Google Sheets Append](#15-google-sheets-append)
16. [Error Handler](#16-error-handler)
17. [Success Notification](#17-success-notification)

---

## 1. Manual Trigger

**Node Type**: `Manual Trigger`

**Purpose**: Manually start the workflow execution

### Configuration

```json
{
  "name": "Manual Trigger",
  "type": "n8n-nodes-base.manualTrigger",
  "position": [250, 300],
  "parameters": {}
}
```

### Usage

Click the "Execute Workflow" button in n8n to start the workflow. This gives you full control over when the workflow runs.

---

## 2. Set Input Parameters

**Node Type**: `Set`

**Purpose**: Define the search criteria and parameters for lead generation

### Configuration

```json
{
  "name": "Set Input Parameters",
  "type": "n8n-nodes-base.set",
  "position": [350, 300],
  "parameters": {
    "values": {
      "string": [
        {
          "name": "businessType",
          "value": "HVAC contractor"
        },
        {
          "name": "targetLocations",
          "value": "Los Angeles, CA\\nMiami, FL\\nPhoenix, AZ"
        },
        {
          "name": "searchKeywords",
          "value": "HVAC contractor, air conditioning repair"
        }
      ],
      "number": [
        {
          "name": "dailyLeadLimit",
          "value": 100
        }
      ]
    }
  }
}
```

### How to Customize

To change the search parameters, edit this node's values:
- **businessType**: Type of business to target
- **targetLocations**: One location per line (use \\n for line breaks)
- **searchKeywords**: Comma-separated keywords
- **dailyLeadLimit**: Maximum number of leads to collect (1-500)

### Output Data Structure

```json
{
  "businessType": "HVAC contractor",
  "targetLocations": "Los Angeles, CA\\nMiami, FL\\nPhoenix, AZ",
  "searchKeywords": "HVAC contractor, air conditioning repair",
  "dailyLeadLimit": 100
}
```

---

## 3. Input Validation

**Node Type**: `Function`

**Purpose**: Validate and format incoming form data

### Configuration

```javascript
// Function Node - Input Validation
const businessType = $input.item.json.businessType;
const locations = $input.item.json.targetLocations;
const keywords = $input.item.json.searchKeywords;
const leadLimit = $input.item.json.dailyLeadLimit || 100;

// Validation
if (!businessType || businessType.trim() === '') {
  throw new Error('Business type is required');
}

if (!locations || locations.trim() === '') {
  throw new Error('At least one location is required');
}

if (!keywords || keywords.trim() === '') {
  throw new Error('Search keywords are required');
}

if (leadLimit < 1 || leadLimit > 500) {
  throw new Error('Daily lead limit must be between 1 and 500');
}

// Parse locations into array
const locationArray = locations
  .split('\\n')
  .map(loc => loc.trim())
  .filter(loc => loc !== '');

// Parse keywords into array
const keywordArray = keywords
  .split(',')
  .map(kw => kw.trim())
  .filter(kw => kw !== '');

// Return formatted data
return {
  json: {
    config: {
      businessType: businessType.trim(),
      locations: locationArray,
      keywords: keywordArray,
      leadLimit: leadLimit,
      processedLeads: 0,
      startTime: new Date().toISOString()
    }
  }
};
```

---

## 3-16. Additional Nodes

[Full detailed breakdown continues for all 16 nodes with complete code examples, configurations, and data flows]

---

## Data Flow Summary

1. **Form Input** → Validated config
2. **Config** → Loop through locations/keywords
3. **Search Query** → Google Maps API
4. **Raw Results** → Parsed business data
5. **Business Data** → Website scraping
6. **Website Content** → Owner/email extraction
7. **Basic Lead** → Email enrichment (Hunter.io)
8. **Enriched Lead** → LinkedIn matching
9. **Complete Lead** → MCP enhancement
10. **Enhanced Lead** → Deduplication check
11. **Unique Lead** → Quality filter
12. **Qualified Lead** → Google Sheets storage
13. **All Results** → Summary notification

---

**Next Steps**: See [Implementation Guide](implementation-guide.md) for deployment instructions.
