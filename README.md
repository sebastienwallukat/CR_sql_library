# Credit Risk Team Knowledge Base 📊

Welcome to the **Credit Risk Team Knowledge Base**! This repository serves as the central hub for our team's most valuable SQL queries, data resources, tools, and analytics knowledge.

## Purpose

This knowledge base is designed to:
- Provide a comprehensive onboarding resource for new team members
- Maintain consistency in our data analysis approach
- Document our tools, data sources, and methodologies
- Serve as a reference for common queries and analysis patterns
- Promote collaboration and knowledge sharing across the team

## Repository Navigation 🧭

### Getting Started
- [Onboarding Checklist](./01_Getting_Started/Onboarding_Checklist.md) - Step-by-step guide for new team members
- [Repository Navigation](./01_Getting_Started/Repository_Navigation.md) - How to find information in this knowledge base
- [Access and Permissions](./01_Getting_Started/Access_and_Permissions.md) - Guide to required access rights and how to get them

### Tools Documentation
- [BigQuery Guide](./02_Tools/BigQuery_Guide.md) - Complete guide to using BigQuery for credit risk analysis
- [Plex Guide](./02_Tools/Plex_Guide.md) - How to use Plex for data discovery and metadata
- [SQL Helper Guide](./02_Tools/SQL_Helper_Guide.md) - Guide to SQL Helper tool for query assistance
- [AI Chat Tools](./02_Tools/AI_Chat_Tools.md) - When and how to use AI tools for data analysis
- [Looker Guide](./02_Tools/Looker_Guide.md) - Creating visualizations and dashboards with Looker

### SQL Resources
- [SQL Basics](./03_SQL_Guide/SQL_Basics.md) - Introduction to SQL structure and syntax
- [CTE Guide](./03_SQL_Guide/CTE_Guide.md) - Working with Common Table Expressions
- [SQL Best Practices](./03_SQL_Guide/SQL_Best_Practices.md) - Writing efficient and maintainable SQL
- [Common SQL Patterns](./03_SQL_Guide/Common_SQL_Patterns.md) - Reusable patterns for credit risk analysis
- [SQL Troubleshooting](./03_SQL_Guide/SQL_Troubleshooting.md) - Fixing common SQL errors and issues

### Data Dictionary
- [Table Overview](./04_Data_Dictionary/Table_Overview.md) - Comprehensive list of tables used by our team
- [Table Relationships](./04_Data_Dictionary/Table_Relationships.md) - How tables connect to each other
- [Chargeback Tables](./04_Data_Dictionary/Chargeback_Tables.md) - Tables related to chargebacks
- [Shop Tables](./04_Data_Dictionary/Shop_Tables.md) - Tables related to shop data
- [Risk Tables](./04_Data_Dictionary/Risk_Tables.md) - Tables related to risk indicators
- [Financial Tables](./04_Data_Dictionary/Financial_Tables.md) - Tables related to financial data
- [Trust Platform Tables](./04_Data_Dictionary/Trust_Platform_Tables.md) - Tables related to Trust Platform

### Domain Knowledge
- [Chargeback Analysis](./05_Domain_Knowledge/Chargeback.md) - Queries for chargeback rates, trends, and risk indicators
- [GPV Analysis](./05_Domain_Knowledge/GPV.md) - Processing volume metrics and growth patterns
- [Risk Scoring](./05_Domain_Knowledge/Risk_Scoring.md) - Models and queries for transaction risk evaluation
- [Reserves Management](./05_Domain_Knowledge/Reserves.md) - Reserve configurations and balance tracking
- [Capital Recovery](./05_Domain_Knowledge/Capital_Recovery.md) - Analysis of capital recovery activities
- [Revenue Metrics](./05_Domain_Knowledge/Revenue.md) - Revenue tracking and performance indicators
- [Fraud Detection](./05_Domain_Knowledge/Fraud_Detection.md) - Fraud patterns and detection mechanisms
- [Merchant Monitoring](./05_Domain_Knowledge/Merchant_Monitoring.md) - Merchant health and activity tracking
- [TrustPlatform Tickets](./05_Domain_Knowledge/TrustPlatform_Tickets.md) - Ticket analytics and case management
- [Booked Losses](./05_Domain_Knowledge/Booked_Losses.md) - Loss tracking, classification, and trend analysis
- [Shopify Payment Balance](./05_Domain_Knowledge/Shopify_Payment_Balance.md) - Balance tracking and reconciliation

### Example Queries
- [Full Query - Shop](./06_Example_Queries/Full_Query_Shop.md) - End-to-end shop analysis queries
- [Full Query - TrustPlatform](./06_Example_Queries/Full_Query_TrustPlatform.md) - Complete TrustPlatform data analysis
- [Common Query Patterns](./06_Example_Queries/Common_Query_Patterns.md) - Frequently used query structures

### Case Studies
- [Case Study 1: Fraud Detection](./07_Case_Studies/Case_Study_1_Fraud_Detection.md) - End-to-end fraud detection workflow
- [Case Study 2: Reserve Management](./07_Case_Studies/Case_Study_2_Reserve_Management.md) - Reserve optimization analysis
- [Case Study 3: Chargeback Analysis](./07_Case_Studies/Case_Study_3_Chargeback_Analysis.md) - Detailed chargeback investigation
- [Case Study 4: Risk Monitoring](./07_Case_Studies/Case_Study_4_Risk_Monitoring.md) - Ongoing risk monitoring process

### Team Resources
- [Collaboration Guide](./08_Team_Resources/Collaboration_Guide.md) - How we work together
- [Slack Channels](./08_Team_Resources/Slack_Channels.md) - Relevant Slack channels and their purposes
- [Code Review Practices](./08_Team_Resources/Code_Review_Practices.md) - How we review and maintain query quality
- [Troubleshooting Guide](./08_Team_Resources/Troubleshooting_Guide.md) - Solving common issues with tools and data

## Key Tables and Datasets 📊

These tables are most commonly used in Credit Risk queries and reports:

| Table Name | Description | Location | Documentation |
|------------|-------------|----------|--------------|
| `shopify-dw.money_products.chargebacks_summary` | Comprehensive chargebacks summary data | [Chargeback Tables](./04_Data_Dictionary/Chargeback_Tables.md) | [Schema](./04_Data_Dictionary/Chargeback_Tables.md#chargebacks_summary) |
| `sdp-prd-cti-data.intermediate.shop_chargeback_rates_daily_snapshot` | Chargeback count/rate at any given date | [Chargeback Tables](./04_Data_Dictionary/Chargeback_Tables.md) | [Schema](./04_Data_Dictionary/Chargeback_Tables.md#shop_chargeback_rates_daily_snapshot) |
| `sdp-prd-cti-data.intermediate.shop_chargeback_rates_current` | Current chargeback rates | [Chargeback Tables](./04_Data_Dictionary/Chargeback_Tables.md) | [Schema](./04_Data_Dictionary/Chargeback_Tables.md#shop_chargeback_rates_current) |
| `shopify-dw.risk.trust_platform_tickets_summary_v1` | Tickets summary with CR ticket filtering capability | [Trust Platform Tables](./04_Data_Dictionary/Trust_Platform_Tables.md) | [Schema](./04_Data_Dictionary/Trust_Platform_Tables.md#trust_platform_tickets_summary_v1) |
| `shopify-dw.finance.shop_gmv_current` | GMV/GPV data (aggregated per day and timeframes) | [Financial Tables](./04_Data_Dictionary/Financial_Tables.md) | [Schema](./04_Data_Dictionary/Financial_Tables.md#shop_gmv_current) |
| `shopify-dw.finance.shop_gmv_daily_summary_v1_1` | Detailed daily GMV data for in-depth time series analysis | [Financial Tables](./04_Data_Dictionary/Financial_Tables.md) | [Schema](./04_Data_Dictionary/Financial_Tables.md#shop_gmv_daily_summary_v1_1) |
| `shopify-dw.money_products.shopify_payments_balance_account_daily_cumulative_summary` | GPV balance at current moment or any given date | [Financial Tables](./04_Data_Dictionary/Financial_Tables.md) | [Schema](./04_Data_Dictionary/Financial_Tables.md#shopify_payments_balance_account_daily_cumulative_summary) |
| `shopify-dw.raw_shopify.payments_refunds` | Comprehensive refunds data | [Financial Tables](./04_Data_Dictionary/Financial_Tables.md) | [Schema](./04_Data_Dictionary/Financial_Tables.md#payments_refunds) |
| `sdp-prd-cti-data.intermediate.shop_current_shopify_payments_status` | Current Shopify Payments status | [Shop Tables](./04_Data_Dictionary/Shop_Tables.md) | [Schema](./04_Data_Dictionary/Shop_Tables.md#shop_current_shopify_payments_status) |
| `shopify-dw.money_products.order_transactions_payments_summary` | Detailed order information | [Financial Tables](./04_Data_Dictionary/Financial_Tables.md) | [Schema](./04_Data_Dictionary/Financial_Tables.md#order_transactions_payments_summary) |
| `sdp-prd-cti-data.base.base__shopify_payments_reserve_configurations` | Merchant reserve setup and configuration | [Financial Tables](./04_Data_Dictionary/Financial_Tables.md) | [Schema](./04_Data_Dictionary/Financial_Tables.md#base__shopify_payments_reserve_configurations) |

For a complete list of tables with detailed descriptions, see the [Data Dictionary](./04_Data_Dictionary/Table_Overview.md).

## Getting Started with This Repository 🚀

### Access Requirements

- **BigQuery Billing Project** - ⚠️ Important: All queries and reports must use `shopify-dw` as the billing project
- **PII Data Access** - For sensitive data access, claim the `sdp-pii` Clouddo permit
- **Service Accounts** - For Looker integration, submit a Helpdesk request for a Looker Service Account (requires Trust approval)

See the [Access and Permissions](./01_Getting_Started/Access_and_Permissions.md) guide for detailed instructions.

### How to Use This Knowledge Base

1. **Start with onboarding** - New team members should follow the [Onboarding Checklist](./01_Getting_Started/Onboarding_Checklist.md)
2. **Learn the tools** - Familiarize yourself with our [Tools Documentation](./02_Tools/)
3. **Explore SQL resources** - Review the [SQL Guide](./03_SQL_Guide/) for query development best practices
4. **Understand the data** - Browse the [Data Dictionary](./04_Data_Dictionary/) to learn about available tables
5. **Study domain knowledge** - Explore domain-specific analysis in the [Domain Knowledge](./05_Domain_Knowledge/) section
6. **See practical examples** - Check [Example Queries](./06_Example_Queries/) and [Case Studies](./07_Case_Studies/) for practical applications
7. **Get support** - Use the [Team Resources](./08_Team_Resources/) section when you need help

### Contributing to the Knowledge Base

We welcome and encourage contributions from all team members! To contribute:

1. **Clone the repository** - Create a local copy using Cursor or your preferred Git client
2. **Create a branch** - Make your changes in a dedicated branch
3. **Follow style guidelines** - Maintain consistent formatting and documentation patterns
4. **Make your edits** - Add new content or improve existing documentation
5. **Submit a pull request** - Request a review of your changes
6. **Respond to feedback** - Make any requested adjustments

See the [Collaboration Guide](./08_Team_Resources/Collaboration_Guide.md) for detailed contribution instructions.

## SQL Best Practices 📝

For detailed SQL best practices, see the [SQL Best Practices](./03_SQL_Guide/SQL_Best_Practices.md) guide. Key highlights:

- Always use the recommended data tables for consistency
- Document any data limitations or caveats
- Include comments in complex queries
- Test queries before adding them to the repository
- Use `shopify-dw` as the billing project for all queries
- Follow Shopify data handling guidelines for PII data

## Frequently Asked Questions ❓

For a complete list of FAQs, see the [Troubleshooting Guide](./08_Team_Resources/Troubleshooting_Guide.md). Common questions:

**Q: How do I get access to the tables mentioned in this repository?**  
A: Access to BigQuery tables requires appropriate permissions. Use Clouddo to request access to the `shopify-dw` project and specific datasets like `sdp-pii` for sensitive data.

**Q: How often are these queries updated?**  
A: We aim to validate and update queries quarterly. Each query includes a "Last validated" date to indicate when it was last checked.

**Q: How can I verify my chargeback rate calculations are correct?**  
A: Compare your results with the standard tables like `sdp-prd-cti-data.intermediate.shop_chargeback_rates_current`. Chargeback rates should divide the count of chargebacks by the count of transactions (not the monetary amounts).

**Q: What's the difference between GMV and GPV in the context of risk analysis?**  
A: Gross Merchandise Value (GMV) represents the total sales value of merchandise sold, while Gross Payment Volume (GPV) represents the portion processed through Shopify Payments. For risk analysis, GPV is often more relevant as it represents financial exposure.

---

**Happy data exploring!** 📈 💻

*For questions or suggestions about this repository, please contact the Credit Risk Data Team.*

*Last Updated: May 2024*
