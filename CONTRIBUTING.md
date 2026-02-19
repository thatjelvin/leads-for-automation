# Contributing to HVAC Lead Automation

Thank you for your interest in improving the HVAC lead automation workflow!

## How to Customize

### Changing Business Niche

While this workflow is designed for HVAC businesses, you can easily adapt it for other B2B niches:

1. **Update Search Keywords**: Modify the form template or hard-code keywords in the workflow
   - Example niches: Plumbing, Electrical, Roofing, Landscaping
2. **Adjust Category Filter**: Update the quality filter in the "Filter Qualified" node
3. **Modify Form Labels**: Change "HVAC" to your target industry

### Adding Custom Fields

To add additional data fields:

1. **Update Parser**: Modify the "Parse Maps Results" function to extract new fields
2. **Update Sheet Schema**: Add columns to `config/sheets-template.json`
3. **Update Append Node**: Add new column mappings in "Append to Leads Sheet"

### Custom Enrichment Sources

To add new data sources:

1. **Add HTTP Request Node**: After the Rate Limiter
2. **Parse Response**: Create function node to extract data
3. **Merge Data**: Update the enrichment merge logic

## Workflow Improvements

### Suggested Enhancements

- **Multiple Keywords**: Loop through all keywords, not just the first
- **Retry Logic**: Add exponential backoff for failed API calls
- **Database Storage**: Replace Google Sheets with PostgreSQL/MySQL for large scale
- **Webhook Notifications**: Send real-time updates via Slack/Discord
- **Lead Validation**: Add phone number verification API
- **Social Media**: Extract Twitter, Facebook, Instagram profiles
- **CRM Integration**: Auto-export to Salesforce, HubSpot, or Pipedrive

### Code Quality

When contributing:

- Follow existing code patterns
- Add comments for complex logic
- Test with small lead limits first
- Document any new API integrations
- Update relevant documentation

## Testing Your Changes

1. **Test with 5 leads** first
2. **Check all fields** are populated correctly
3. **Verify deduplication** works
4. **Test error handling** by simulating failures
5. **Monitor API usage** to stay within limits

## Reporting Issues

When reporting bugs:

- Describe the issue clearly
- Include workflow execution logs
- Specify which node failed
- Provide example input data (sanitized)
- List your n8n version and environment

## Feature Requests

For new features:

- Explain the use case
- Describe expected behavior
- Consider impact on existing workflow
- Suggest implementation approach

## License

By contributing, you agree that your contributions will be licensed under the MIT License.

## Questions?

Open an issue or discussion in the GitHub repository.

---

**Happy Automating!**
