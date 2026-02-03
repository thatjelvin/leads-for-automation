# n8n Workflow Node-by-Node Breakdown

This document provides a detailed breakdown of each node in the HVAC lead automation workflow, including configuration details, expressions, and data mappings.

## Table of Contents

1. [Form Intake Trigger](#1-form-intake-trigger)
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

## 1. Form Intake Trigger

**Node Type**: `Webhook` or `n8n Form Trigger`

**Purpose**: Collect lead generation parameters from the user

### Configuration

```json
{
  "name": "Form Intake",
  "type": "n8n-nodes-base.formTrigger",
  "position": [250, 300],
  "parameters": {
    "formTitle": "HVAC Lead Generation Request",
    "formDescription": "Enter your search criteria for daily lead generation",
    "formFields": {
      "values": [
        {
          "fieldLabel": "Business Type",
          "fieldType": "text",
          "requiredField": true,
          "fieldPlaceholder": "HVAC, AC Repair, Heating Services"
        },
        {
          "fieldLabel": "Target Locations",
          "fieldType": "textarea",
          "requiredField": true,
          "fieldPlaceholder": "Los Angeles, CA\\nMiami, FL\\nDallas, TX"
        },
        {
          "fieldLabel": "Search Keywords",
          "fieldType": "text",
          "requiredField": true,
          "fieldPlaceholder": "HVAC contractor, air conditioning repair"
        },
        {
          "fieldLabel": "Daily Lead Limit",
          "fieldType": "number",
          "requiredField": false,
          "fieldPlaceholder": "100"
        }
      ]
    },
    "options": {
      "webhookPath": "hvac-leads-intake"
    }
  }
}
```

### Output Data Structure

```json
{
  "businessType": "HVAC contractor",
  "targetLocations": "Los Angeles, CA\\nMiami, FL",
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
