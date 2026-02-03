# HVAC Lead Automation Workflow - n8n Implementation

## Overview

This repository contains a complete n8n automation workflow designed to scrape and enrich B2B leads for HVAC businesses at scale using the n8n Model Context Protocol (MCP). The system targets ~100 qualified leads per day with full enrichment and deduplication.

## Features

- **Manual Execution**: Manually trigger workflow runs with customizable parameters
- **Google Maps Scraping**: Automated business discovery with pagination
- **Contact Enrichment**: Owner identification, email extraction, and LinkedIn matching
- **Intelligent Deduplication**: Multi-field matching to prevent duplicates
- **Rate Limiting**: Safe, compliant scraping with configurable delays
- **Google Sheets Integration**: Automatic data storage with append-only writes
- **Error Handling**: Comprehensive logging and retry mechanisms
- **Daily Automation**: Can be scheduled for daily execution with configurable limits

## Repository Structure

```
.
├── README.md                           # This file
├── workflow/
│   ├── n8n-workflow.json              # Complete n8n workflow (importable)
│   └── workflow-architecture.md        # High-level architecture diagram
├── docs/
│   ├── node-breakdown.md              # Detailed node-by-node guide
│   ├── implementation-guide.md        # Step-by-step setup instructions
│   ├── api-integrations.md            # API keys and configurations
│   └── best-practices.md              # Scaling, compliance, and optimization
└── config/
    ├── form-template.json             # Example form configuration
    └── sheets-template.json           # Google Sheets schema
```

## Quick Start

1. **Import Workflow**: Import `workflow/n8n-workflow.json` into your n8n instance
2. **Configure APIs**: Set up credentials for Google Maps, LinkedIn, and Google Sheets
3. **Customize Parameters**: Edit the "Set Input Parameters" node with your search criteria
4. **Test Run**: Click "Execute Workflow" button to run with your parameters (start with small lead limit like 5)
5. **Optional - Schedule**: Add a Schedule Trigger node if you want automated daily execution

## Workflow Architecture

### High-Level Flow

```
┌─────────────────┐
│ Manual Trigger  │ (Click "Execute Workflow")
│  - Click to run │
└────────┬────────┘
         │
         v
┌─────────────────┐
│ Set Input Params│ (Configurable Node)
│  - Business Type│
│  - Location(s)  │
│  - Keywords     │
│  - Lead Limit   │
└────────┬────────┘
         │
         v
┌─────────────────┐
│   Validation    │ (Function Node)
│  - Input check  │
│  - Format data  │
└────────┬────────┘
         │
         v
┌─────────────────┐
│  Google Maps    │ (HTTP Request + Loop)
│    Scraper      │
│  - Search query │
│  - Pagination   │
│  - Extract data │
└────────┬────────┘
         │
         v
┌─────────────────┐
│  Enrichment     │ (Multiple HTTP + MCP)
│  - Website scan │
│  - Email finder │
│  - LinkedIn     │
└────────┬────────┘
         │
         v
┌─────────────────┐
│ Deduplication   │ (Function + Compare)
│  - Name match   │
│  - Phone match  │
│  - URL match    │
└────────┬────────┘
         │
         v
┌─────────────────┐
│ Google Sheets   │ (Append to Sheet)
│  - Store leads  │
│  - Log metadata │
└────────┬────────┘
         │
         v
┌─────────────────┐
│ Error Handler   │ (Error Trigger)
│  - Log failures │
│  - Send alerts  │
└─────────────────┘
```

## Target Metrics

- **Daily Leads**: ~100 qualified leads
- **Success Rate**: >95% data extraction
- **Enrichment Rate**: >80% with email/owner info
- **Duplicate Rate**: <5%
- **Execution Time**: 30-45 minutes per batch

## Prerequisites

### n8n Setup
- Self-hosted n8n instance (VPS recommended)
- n8n version: 1.0.0 or higher
- n8n MCP support enabled

### API Access
- Google Maps API (or scraping proxy service)
- Hunter.io or similar email finder API (optional)
- LinkedIn Scraper API (RapidAPI or similar)
- Google Sheets API credentials

### Resources
- VPS with 2GB+ RAM
- Stable internet connection
- Proxy rotation service (recommended for scale)

## Key Components

### 1. Form Intake Node
- **Type**: Webhook / n8n Form Trigger
- **Purpose**: Collect search parameters
- **Fields**: Business type, location, keywords, daily limit

### 2. Google Maps Scraper
- **Type**: HTTP Request + Loop Node
- **Purpose**: Search and extract business listings
- **Rate Limit**: 2-5 second delays between requests

### 3. Enrichment Pipeline
- **Components**: Website parser, Email finder, LinkedIn matcher
- **MCP Usage**: Intelligent parsing of unstructured data
- **Fallbacks**: Multiple data sources per field

### 4. Deduplication Logic
- **Method**: Hash-based comparison
- **Fields**: Company name, phone, website
- **Storage**: In-memory or Redis cache

### 5. Data Storage
- **Primary**: Google Sheets (append mode)
- **Backup**: CSV export option
- **Schema**: 10 columns with metadata

## Security & Compliance

- **Rate Limiting**: Respectful delays to avoid ToS violations
- **Data Privacy**: GDPR-compliant data handling
- **API Security**: Environment variables for credentials
- **Error Logging**: No sensitive data in logs

## Monitoring & Maintenance

- **Daily Health Checks**: Automated workflow validation
- **Error Notifications**: Email/Slack alerts on failures
- **Performance Metrics**: Track success rate and timing
- **Monthly Review**: Adjust scraping patterns and sources

## Support & Documentation

Detailed implementation guides are available in the `docs/` directory:
- [Node-by-Node Breakdown](docs/node-breakdown.md)
- [Implementation Guide](docs/implementation-guide.md)
- [API Integrations](docs/api-integrations.md)
- [Best Practices](docs/best-practices.md)

## License

MIT License - See LICENSE file for details

## Author

Jelvin Kiteete - HVAC Automation Specialist

---

**Ready to deploy?** Start with the [Implementation Guide](docs/implementation-guide.md)
