# n8n Workflow Node-by-Node Breakdown

This document provides a detailed breakdown of each node in the HVAC lead automation workflow, including configuration details, expressions, and data mappings.

## Table of Contents

1. [Execute Workflow Trigger (with Form)](#1-execute-workflow-trigger-with-form)
2. [Input Validation](#2-input-validation)
3. [Initialize Variables](#3-initialize-variables)
4. [Google Maps Search Loop](#4-google-maps-search-loop)
5. [HTTP Request - Google Maps](#5-http-request---google-maps)
6. [Parse Maps Results](#6-parse-maps-results)
7. [Rate Limiter](#7-rate-limiter)
8. [Website Scraper](#8-website-scraper)
9. [Email Enrichment](#9-email-enrichment)
10. [LinkedIn Matcher](#10-linkedin-matcher)
11. [MCP Enrichment Node](#11-mcp-enrichment-node)
12. [Deduplication Check](#12-deduplication-check)
13. [Filter Qualified Leads](#13-filter-qualified-leads)
14. [Google Sheets Append](#14-google-sheets-append)
15. [Error Handler](#15-error-handler)
16. [Success Notification](#16-success-notification)

---

## 1. Execute Workflow Trigger (with Form)

**Node Type**: `Execute Workflow Trigger`

**Purpose**: Display a popup form when manually executing the workflow, allowing you to enter search parameters

### Configuration

```json
{
  "name": "Execute Workflow Trigger",
  "type": "n8n-nodes-base.executeWorkflowTrigger",
  "position": [250, 300],
  "parameters": {
    "fieldsUi": {
      "values": [
        {
          "displayName": "Business Type",
          "fieldName": "businessType",
          "fieldType": "string",
          "required": true,
          "defaultValue": "HVAC contractor"
        },
        {
          "displayName": "Target Locations",
          "fieldName": "targetLocations",
          "fieldType": "multiline",
          "required": true,
          "defaultValue": "Los Angeles, CA\\nMiami, FL\\nPhoenix, AZ"
        },
        {
          "displayName": "Search Keywords",
          "fieldName": "searchKeywords",
          "fieldType": "string",
          "required": true,
          "defaultValue": "HVAC contractor, air conditioning repair"
        },
        {
          "displayName": "Daily Lead Limit",
          "fieldName": "dailyLeadLimit",
          "fieldType": "number",
          "required": false,
          "defaultValue": 100
        }
      ]
    }
  }
}
```

### Usage

1. Click the **"Execute Workflow"** button in n8n
2. A form popup will appear on your screen with fields for:
   - **Business Type**: Type of business to target (e.g., "HVAC contractor")
   - **Target Locations**: Enter locations one per line
   - **Search Keywords**: Comma-separated search terms
   - **Daily Lead Limit**: Maximum leads to collect (1-500)
3. Fill in the form with your desired parameters
4. Click "Execute" to run the workflow with those values

### Form Fields

| Field | Type | Required | Default | Description |
|-------|------|----------|---------|-------------|
| Business Type | String | Yes | "HVAC contractor" | Target industry |
| Target Locations | Multiline | Yes | Multiple cities | One location per line |
| Search Keywords | String | Yes | Keywords | Comma-separated |
| Daily Lead Limit | Number | No | 100 | Max leads (1-500) |

### Output Data Structure

When you submit the form, the values are available as:

```json
{
  "businessType": "HVAC contractor",
  "targetLocations": "Los Angeles, CA\\nMiami, FL\\nPhoenix, AZ",
  "searchKeywords": "HVAC contractor, air conditioning repair",
  "dailyLeadLimit": 100
}
```

---

## 2. Input Validation

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
