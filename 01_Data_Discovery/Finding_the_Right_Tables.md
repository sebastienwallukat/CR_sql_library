# Finding the Right Tables for Credit Risk Analysis

This guide provides a practical overview of the tools and processes for finding, understanding, and using data tables effectively for Credit Risk analysis at Shopify.

## Table of Contents

- [Overview](#overview)
- [Finding Tables](#finding-tables)
  - [Using Plex](#using-plex)
  - [Using Metadata Portal](#using-metadata-portal)
- [Reviewing Table Information](#reviewing-table-information)
- [Understanding Data Lineage](#understanding-data-lineage)
- [Key Tables for Credit Risk](#key-tables-for-credit-risk)
- [Best Practices](#best-practices)
- [Troubleshooting](#troubleshooting)

## Overview

Finding the right tables is the first step in any successful data analysis. Shopify has several tools for discovering and understanding data tables:

- **Plex** - Shopify's data catalog and discovery platform
- **Metadata Portal** - Resource for detailed BigQuery table information
- **Credit Risk Data Catalog** - Our team's curated list of essential tables

This guide will help you leverage these tools efficiently to find the exact data you need for your Credit Risk analysis tasks.

## Finding Tables

### Using Plex

Plex is Shopify's primary data catalog and discovery platform that helps you:

1. **Access Plex**
   - Navigate to [plex](https://console.cloud.google.com/dataplex/search?project=shopify-dw)
   - Sign in with your Shopify credentials

2. **Search for Data**
   - From the search bar, enter terms related to credit risk analysis:
     - `chargeback` - Find chargeback-related tables
     - `risk` - Discover risk assessment tables
     - `shop payments` - Find Shopify Payments tables
     - `reserve` - Locate reserve-related data

3. **Use Filters to Narrow Results**
   - **Type**: Tables, Views, Dashboards
   - **Data Source**: BigQuery
   - **Team**: Money, Data Platform, Trust
   - **Database**: shopify-dw, sdp-prd-cti-data

4. **Review Results**
   - Click on a result to view detailed information

**Example Search Strategy**:

For finding chargeback data tables:
- Search for `chargeback`
- Filter by Type: Tables
- Filter by Database: shopify-dw
- Look for tables in the money_products dataset

### Using Metadata Portal

The [Metadata Portal](https://data.shopify.io/search/result?s=gcp&sys=bigquery&p=shopify-dw&c=analytically_modeled) provides detailed information about BigQuery tables and views.

1. **Access Metadata Portal**
   - Navigate to [data.shopify.io](https://data.shopify.io)
   - Sign in with your Shopify credentials

2. **Search for Tables**
   - Use search terms like "chargeback", "risk", "shop", or "payment"
   - Look for tables in the "analytically_modeled" category for analysis-ready data

3. **Review Results**
   - Click on a table to view its detailed information

**Example Workflow**:
1. Search for "chargeback" in the portal
2. Review schema details for relevant tables
4. Check lineage to understand data sources
5. Verify update frequency matches analysis needs
6. Use field descriptions to build accurate queries

## Reviewing Table Information

Both Plex and Metadata Portal provide detailed information about tables:

1. **Overview Information**:
   - Description of the table's purpose
   - Update frequency
   - Tags and classifications
   - Data owners

2. **Schema Details**:
   - Complete field listings with data types
   - Field descriptions and business context
   - Primary key information
   - Sample values

3. **Data Quality Metrics**:
   - Row counts
   - Last update timestamp
   - Data distribution for key fields
   - Data quality scores

## Understanding Data Lineage

Data lineage helps you understand how tables are connected:

1. **Lineage Diagrams**:
   - Shows upstream sources and downstream consumers
   - Click on connected tables to navigate the data ecosystem
   - Understand data transformation steps

2. **Transformation Logic**:
   - See how data is processed and transformed
   - Understand calculation methodologies
   - Identify potential data quality issues

## Key Tables for Credit Risk

For a comprehensive list of the most important tables for Credit Risk analysis, refer to the [Data Tables Catalog](./Data_Tables_Catalog.md#key-tables-for-credit-risk-analytics).

Here are some of the most frequently used tables:

| Table Name | Description | What to Look For |
|------------|-------------|------------------|
| `shopify-dw.money_products.chargebacks_summary` | Comprehensive chargebacks data | Schema details, update frequency, lineage |
| `sdp-prd-cti-data.intermediate.shop_chargeback_rates_current` | Current chargeback rates | Schema documentation, calculation methodology |
| `shopify-dw.finance.shop_gmv_daily_summary_v1_1` | Daily GMV data | Partition details, schema, data freshness |
| `shopify-dw.risk.trust_platform_tickets_summary_v1` | Trust Platform tickets | Schema, field descriptions, related tables |
| `sdp-prd-cti-data.base.base__shopify_payments_reserve_configurations` | Reserve configurations | Schema, update frequency, lineage |

## Best Practices

### 1. Check Both Tools

- Start with Plex for initial discovery
- Use Metadata Portal for detailed schema information
- Cross-reference information between both tools

### 2. Bookmark Key Tables

Create bookmarks for frequently accessed tables:
1. Click the bookmark icon on any table page
2. Access bookmarks from your profile menu

### 3. Validate Data Quality

Before using a table for critical analysis:
1. Check the Statistics/Data Quality tab for data freshness
2. Review row counts to ensure completeness
3. Look for any data quality warnings or issues

### 4. Contact Data Owners

When you have questions about a dataset:
1. Find the owner in the table's Overview section
2. Note their team and contact information
3. Reach out via Slack or email with specific questions

### 5. Understand Data Lineage

For important tables:
1. Review the lineage diagram to understand data sources
2. Check transformation steps to understand calculation logic
3. Identify related tables that might provide additional context

### 6. Document Your Discoveries

When you learn something useful about a table:
1. Consider adding comments to share knowledge
2. Update team documentation with insights
3. Share findings in the Credit Risk team channels

## Common Table Search Tasks

### Finding Chargeback Rate Calculation Logic

1. Search for `shop_chargeback_rates_current`
2. Navigate to the Lineage tab
3. Identify upstream transformation steps
4. Review calculation methodology

### Identifying Financial Tables with GPV Data

1. Search for `gpv` or `gmv`
2. Review schema to confirm GPV-related fields
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
3. Check if the table has a different name convention

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



--- 
