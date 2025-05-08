# Plex Guide for Credit Risk Analysis

This guide provides a practical overview of Plex, Shopify's data discovery and metadata tool, with specific focus on how Credit Risk analysts can use it effectively.

## What is Plex?

Plex is Shopify's data catalog and discovery platform that helps you:

- Search for and discover relevant datasets
- Understand data lineage and relationships
- Review table metadata and schema information
- Explore data quality metrics
- Find data owners and experts

## When to Use Plex

Use Plex when you need to:

- Discover tables relevant to credit risk analysis
- Understand table schema, fields, and relationships
- Find documentation about specific datasets
- Determine data freshness and quality
- Identify data owners for questions
- Track data lineage to understand sources
- Explore related tables in the same domain

## Accessing Plex

1. Navigate to [plex.shopify.io](https://plex.shopify.io)
2. Sign in with your Shopify credentials
3. You'll land on the main search page

## Key Features for Credit Risk Analysis

### 1. Data Search

The search functionality is the main entry point to finding relevant data:

1. From the search bar, enter terms related to credit risk analysis:
   - `chargeback` - Find chargeback-related tables
   - `risk` - Discover risk assessment tables
   - `shop payments` - Find Shopify Payments tables
   - `reserve` - Locate reserve-related data

2. Use filters to narrow results:
   - **Type**: Tables, Views, Dashboards
   - **Data Source**: BigQuery
   - **Team**: Money, Data Platform, Trust
   - **Database**: shopify-dw, sdp-prd-cti-data

3. Click on a result to view detailed information

**Example Search Strategy**:

For finding chargeback data tables:
- Search for `chargeback`
- Filter by Type: Tables
- Filter by Database: shopify-dw
- Look for tables in the money_products dataset

### 2. Table Exploration

Once you've found a table, Plex provides detailed information:

1. **Overview tab**:
   - Description of the table's purpose
   - Update frequency
   - Tags and classifications
   - Data owners

2. **Schema tab**:
   - Full list of columns with descriptions
   - Data types for each field
   - Primary keys and partition fields

3. **Lineage tab**:
   - Visual representation of data sources
   - Downstream dependencies
   - Process flows showing how data is transformed

4. **Statistics tab**:
   - Row counts
   - Last update timestamp
   - Data distribution for key fields

### 3. Finding Related Tables

Plex helps you discover related tables through:

1. **Lineage diagrams**:
   - Shows upstream sources and downstream consumers
   - Click on connected tables to navigate the data ecosystem

2. **Tags and categories**:
   - Tables with similar tags are likely related
   - Look for domain-specific categorizations

3. **"Similar tables" section**:
   - Recommendations based on content and usage patterns

### 4. Important Tables for Credit Risk

Key tables you'll want to locate and explore in Plex:

| Table Name | Description | What to Look for in Plex |
|------------|-------------|--------------------------|
| `shopify-dw.money_products.chargebacks_summary` | Comprehensive chargebacks data | Schema details, update frequency, lineage |
| `sdp-prd-cti-data.intermediate.shop_chargeback_rates_current` | Current chargeback rates | Schema documentation, calculation methodology |
| `shopify-dw.finance.shop_gmv_daily_summary_v1_1` | Daily GMV data | Partition details, schema, data freshness |
| `shopify-dw.risk.trust_platform_tickets_summary_v1` | Trust Platform tickets | Schema, field descriptions, related tables |
| `sdp-prd-cti-data.base.base__shopify_payments_reserve_configurations` | Reserve configurations | Schema, update frequency, lineage |

## Plex Integration with BigQuery

Plex integrates with BigQuery in several ways:

1. **Direct links to BigQuery**:
   - Each table in Plex has a "Query in BigQuery" link
   - Click to open the table in the BigQuery console

2. **Schema reference**:
   - Use Plex to understand table structure before writing queries
   - Copy field names directly from Plex

3. **Query examples**:
   - Some tables include sample queries
   - Learn from existing usage patterns

4. **Data preview**:
   - View sample data without running queries
   - Understand data distribution before analysis

## Finding Table Documentation

Plex serves as the central documentation source for data tables:

1. **Table descriptions**:
   - Overview of table purpose and contents
   - Update frequency and process
   - Data sources and transformations

2. **Column descriptions**:
   - Field-level documentation
   - Data types and constraints
   - Business meaning of each field

3. **Tags and classifications**:
   - PII indicators
   - Sensitive data markers
   - Business domain categorization

## Best Practices for Credit Risk Analysts

### 1. Bookmark Key Tables

Create bookmarks for frequently accessed tables:
1. Click the bookmark icon on any table page
2. Access bookmarks from your profile menu

### 2. Validate Data Quality

Before using a table for critical analysis:
1. Check the **Statistics** tab for data freshness
2. Review row counts to ensure completeness
3. Look for any data quality warnings or issues

### 3. Contact Data Owners

When you have questions about a dataset:
1. Find the owner in the table's Overview section
2. Note their team and contact information
3. Reach out via Slack or email with specific questions

### 4. Understand Data Lineage

For important tables:
1. Review the lineage diagram to understand data sources
2. Check transformation steps to understand calculation logic
3. Identify related tables that might provide additional context

### 5. Document Your Discoveries

When you learn something useful about a table:
1. Consider adding comments to share knowledge
2. Update team documentation with insights
3. Share findings in the Credit Risk team channels

## Common Plex Tasks for Credit Risk Analysis

### Finding Chargeback Rate Calculation Logic

1. Search for `shop_chargeback_rates_current`
2. Navigate to the Lineage tab
3. Identify upstream transformation steps
4. Review calculation methodology

### Identifying Financial Tables with GPV Data

1. Search for `gpv` or `gmv`
2. Filter for tables in the `finance` dataset
3. Review schema to confirm GPV-related fields
4. Check descriptions to understand calculation methods

### Exploring Trust Platform Data Structure

1. Search for `trust_platform`
2. Look for the `trust_platform_tickets_summary_v1` table
3. Review the schema to understand ticket structure
4. Find related tables through lineage

### Locating Reserve Configuration Data

1. Search for `reserve configurations`
2. Find the base table for reserve settings
3. Review schema to understand how reserves are defined
4. Check lineage to find related tables

## Troubleshooting

### Can't Find an Expected Table

1. Try alternative search terms
2. Remove filters that might be too restrictive
3. Ask in the `#credit-risk-data` Slack channel
4. Check if the table has a different name convention

### Missing Metadata or Documentation

1. Look for related tables that might be better documented
2. Check lineage to understand the data flow
3. Contact the data owner listed in the Overview
4. Ask in the data team's Slack channel

### Data Freshness Concerns

1. Check the Statistics tab for last update time
2. Review any documentation about update frequency
3. Look at lineage to understand the data pipeline
4. Contact the data owner with specific concerns

## Getting Help with Plex

If you need assistance with Plex:

1. Check the internal Plex documentation
2. Ask in the `#help-data-discovery` Slack channel
3. Contact the Data Platform team
4. Attend Plex training sessions when available

---

*Last Updated: May 2024* 