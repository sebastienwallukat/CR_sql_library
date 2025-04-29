# Credit Risk SQL Query Resource Center 📊

Welcome to the **Credit Risk SQL Query Resource Center**! This repository serves as the central knowledge hub for our team's most valuable SQL queries, datasets, and analytics resources.

## Purpose

This repository is designed to:
- Maintain consistency in our data analysis approach
- Provide quick access to verified, tested SQL queries
- Document our data sources and calculation methodologies
- Serve as an onboarding resource for new team members

## Resource Categories

### Risk Assessment Tools
- [Chargeback Analysis](./Chargeback.md) - Queries for chargeback rates, trends, and risk indicators
- [GPV (Gross Processing Volume)](./GPV.md) - Processing volume metrics and growth patterns
- [Transaction Risk Scoring](./Risk_Scoring.md) - Models and queries for transaction risk evaluation

### Financial Metrics
- [Shopify Payment Balance](./Shopify_Payment_Balance.md) - Balance tracking and reconciliation
- [Booked Losses](./Booked_Losses.md) - Loss tracking, classification, and trend analysis
- [Revenue Metrics](./Revenue.md) - Revenue tracking and performance indicators
- [Reserves Management](./Reserves.md) - Reserve configurations and balance tracking

### Operational Resources
- [TrustPlatform Tickets](./TrustPlatform_Tickets.md) - Ticket analytics and case management using:
  - `shopify-dw.risk.trust_platform_tickets_summary_v1`
  - `shopify-dw.risk.trust_platform_actions_summary_v1`
  - `shopify-dw.risk.trust_platform_ticket_escalations_v1`
- [Merchant Monitoring](./Merchant_Monitoring.md) - Merchant health and activity tracking
- [Fraud Detection](./Fraud_Detection.md) - Fraud patterns and detection mechanisms

### Comprehensive Queries
- [Full Query - Shop](./Full_Query_Shop.md) - End-to-end shop analysis queries
- [Full Query - TrustPlatform](./Full_Query_TrustPlatform.md) - Complete TrustPlatform data analysis

### Reserves Management
- [Reserve Configurations](./Reserve_Configurations.md) - Merchant reserve setup and management using:
  - `sdp-prd-cti-data.base.base__shopify_payments_reserve_configurations`
- [Reserve Analysis](./Reserve_Analysis.md) - Analysis of reserve impact and effectiveness

## Access Requirements & Billing 🔑

### Essential Access
- **BigQuery Billing Project** - ⚠️ Important: All queries and reports must use `shopify-dw` as the billing project
- **PII Data Access** - For sensitive data access, claim the `sdp-pii` Clouddo permit
- **Service Accounts** - For Looker Studio integration, submit a Helpdesk request for a Looker Studio Service Account (requires Trust approval)

### Permission Management
- **Clouddo** - For managing data access permissions, including `sdp-pii`
- **Looker Studio Support** - Available at #help-looker-studio-access Slack channel

## Frequently Used Tables 📋

These tables are most commonly used in Credit Risk queries and reports:

| Table Name | Description |
|------------|-------------|
| `shopify-dw.money_products.chargebacks_summary` | Comprehensive chargebacks summary data |
| `sdp-prd-cti-data.intermediate.shop_chargeback_rates_daily_snapshot` | Chargeback count/rate at any given date |
| `sdp-prd-cti-data.intermediate.shop_chargeback_rates_current` | Current chargeback rates |
| `shopify-dw.risk.trust_platform_tickets_summary_v1` | Tickets summary with CR ticket filtering capability |
| `shopify-dw.finance.shop_gmv_current` | GMV/GPV data (aggregated per day and timeframes) |
| `shopify-dw.money_products.shopify_payments_balance_account_daily_cumulative_summary` | GPV balance at current moment or any given date |
| `shopify-dw.raw_shopify.payments_refunds` | Comprehensive refunds data |
| `sdp-prd-cti-data.intermediate.shop_current_shopify_payments_status` | Current Shopify Payments status |
| `shopify-dw.money_products.order_transactions_payments_summary` | Detailed order information |
| `sdp-prd-cti-data.base.base__shopify_payments_reserve_configurations` | Merchant reserve setup and configuration |

## Data Discovery Resources 🔍

### Google Cloud Dataplex
The primary tool for exploring and discovering datasets in Shopify's data platform. Dataplex provides metadata, schema information, and lineage for all datasets.

🔗 [Access Dataplex Console](https://console.cloud.google.com/dataplex/search?project=shopify-dw)

Key Features:
- Search across all Shopify data projects
- View dataset schemas and metadata
- Understand data lineage and relationships
- Check data freshness and quality metrics

### Shopify Metadata Portal
Internal metadata portal for discovering analytically modeled data in the Shopify data platform.

🔗 [Access Metadata Portal](https://data.shopify.io/search/result?s=gcp&sys=bigquery&p=shopify-dw&c=analytically_modeled)

Key Features:
- Focused on analytically modeled data
- Provides business context and definitions
- Shows dataset ownership and support contacts
- Links to documentation and example queries

### Looker Studio
For building interactive dashboards and reports from SQL queries.

🔗 [Looker Studio Homepage](https://lookerstudio.google.com/)

Key Features:
- Create interactive data visualizations
- Schedule automated report refresh
- Share dashboards with stakeholders
- Integrate with BigQuery data sources

When searching for data:
1. Start with the Metadata Portal for business-context and well-documented datasets
2. Use Dataplex for technical exploration and schema information
3. Always verify the data freshness and quality before use

## AI Productivity Tools 🤖

### Cursor
AI-powered code editor that enhances productivity when writing SQL queries.

Key Features:
- Code understanding: Analyzes and understands your entire codebase
- AI-assisted editing: Edit code using natural language commands
- Intelligent autocomplete: Predicts your next edits with context-aware suggestions
- Chat interface: Built-in AI assistant for coding questions and problem-solving

🔗 Resources:
- [Getting Started with Cursor](https://cursor.sh/docs/getting-started)
- [Shopify Cursor Service Page](https://kepler.shopify.io/services/cursor)
- Slack support: #cursor channel

### Other AI Resources
- **SQL Helper Tool** - Query assistance and optimization
- **Hemingway** - Writing and documentation assistance
- **Perplexity** - Research assistant for financial risk analysis

## How to Use This Repository

1. **Navigate to your topic of interest** - Each file focuses on a specific data domain
2. **Review the available queries** - Each file contains tested, documented SQL queries
3. **Check source tables** - Each topic includes the recommended data sources
4. **Contribute your knowledge** - Add new queries or improve existing ones

## Best Practices

- Always use the recommended data tables for consistency
- Document any data limitations or caveats
- Include comments in complex queries
- Test queries before adding them to the repository
- Use `shopify-dw` as the billing project for all queries
- Follow Shopify data handling guidelines for PII data

---

**Happy data exploring!** 📈 💻

*For questions or suggestions about this repository, please contact the Credit Risk Data Team.*

*Last Updated: April 2025*
