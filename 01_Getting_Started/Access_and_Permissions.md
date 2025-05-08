# Access and Permissions Guide

This guide provides detailed instructions for obtaining the necessary access and permissions required for credit risk analysis at Shopify.

## BigQuery Access

### Requesting BigQuery Access

1. Access the Clouddo portal at [clouddo.shopify.io](https://clouddo.shopify.io)
2. Select "Request Access" and choose "BigQuery"
3. Request access to the `shopify-dw` project
4. Provide a business justification explaining:
   - Your role on the Credit Risk team
   - Types of analyses you'll be performing
   - Why you need BigQuery access
5. Submit your request for manager approval

### Billing Project Configuration

⚠️ **Critical**: All queries and reports must use `shopify-dw` as the billing project.

To set the billing project in BigQuery:
1. In the BigQuery web interface, click on the project selector
2. Select `shopify-dw` from the dropdown
3. Verify the selected project shows in the upper left corner

In SQL queries via API, always include:
```python
client = bigquery.Client(project='shopify-dw')
```

### Access Levels

BigQuery access is role-based:
- **Viewer**: Can run queries but cannot create or modify resources
- **User**: Can run queries and create personal resources
- **Editor**: Can create and modify shared resources
- **Admin**: Full administrative control

Most Credit Risk team members require User-level access.

## PII Data Access

### Requesting PII Access

To access tables with personally identifiable information (PII):

1. Claim the `sdp-pii` Clouddo permit:
   - Navigate to [clouddo.shopify.io/permits](https://clouddo.shopify.io/permits)
   - Search for "sdp-pii"
   - Click "Claim Permit"
   - Provide justification for PII data access

2. Complete required PII training:
   - Access the training through Shopify's learning platform
   - Complete the "PII Data Handling" course
   - Upload your completion certificate to Clouddo

3. Review and acknowledge PII handling guidelines

### PII Handling Requirements

When working with PII data:
- Never export raw PII data to local systems
- Only include PII in analyses when absolutely necessary
- Use aggregation and anonymization when possible
- Document all instances of PII usage
- Limit sharing of query results containing PII
- Follow Shopify's data retention policies

## Service Account Setup

### Looker Service Account

For automated reporting or dashboards requiring Looker integration:

1. Submit a Helpdesk request for a Looker Service Account:
   - Go to [helpdesk.shopify.io](https://helpdesk.shopify.io)
   - Create a new ticket with category "Data & Reporting"
   - Specify "Looker Service Account Request" in the subject
   - Include your use case, query requirements, and data sensitivity

2. Trust approval process:
   - The request will be routed to the Trust team for review
   - Prepare to provide additional justification if requested
   - Typical approval time is 3-5 business days

3. Configuration after approval:
   - You'll receive credentials and configuration instructions
   - Store credentials securely according to Shopify standards
   - Update credentials every 90 days as required

## Dataset-Specific Access

### Key Datasets and Access Requirements

| Dataset | Access Requirement | Request Process | Approval Level |
|---------|-------------------|-----------------|----------------|
| `shopify-dw.money_products.*` | BigQuery + Money Domain | Clouddo BigQuery request | Manager |
| `shopify-dw.finance.*` | BigQuery + Finance Domain | Clouddo BigQuery request | Manager |
| `shopify-dw.risk.*` | BigQuery + Risk Domain | Clouddo BigQuery request | Manager |
| `shopify-dw.raw_shopify.*` | BigQuery + Raw Data | Clouddo BigQuery request | Manager |
| `sdp-prd-cti-data.*` | BigQuery + CTI Data | Clouddo BigQuery request | Manager |
| PII Datasets | `sdp-pii` permit | Clouddo Permit claim | Trust |

### Access Verification

To verify your access to specific datasets:
1. In BigQuery, navigate to the Explorer panel
2. Expand the project and look for the dataset
3. If you can't see a dataset, you don't have access
4. Try running a simple query like `SELECT 1 FROM dataset.table LIMIT 1`

## Troubleshooting Access Issues

### Common Issues and Solutions

**Issue**: "Access Denied" when running a query
- **Solution**: Verify you have access to all tables in the query
- **Action**: Request access to specific datasets via Clouddo

**Issue**: "Resource Exceeded" error
- **Solution**: Your query may be too resource-intensive
- **Action**: Optimize your query or request higher quotas

**Issue**: Can't see expected datasets
- **Solution**: You may need specific domain access
- **Action**: Request domain-specific access via Clouddo

### Escalation Process

If you encounter persistent access issues:
1. Check the [Troubleshooting Guide](../08_Team_Resources/Troubleshooting_Guide.md)
2. Ask in the `#credit-risk-data` Slack channel
3. Contact your team's data specialist
4. Submit a Helpdesk ticket if the issue remains unresolved

## Permissions Review and Renewal

- Access permissions are reviewed quarterly
- Inactive permissions may be automatically revoked
- Document your use cases to justify continued access
- Maintain a log of the datasets and queries you regularly use

---

*Last Updated: May 2024* 