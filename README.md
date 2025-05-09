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

### Tools Documentation
- [BigQuery Guide](./01_Tools/BigQuery_Guide.md) - Complete guide to using BigQuery for credit risk analysis
- [Table Search and Review Guide](./01_Tools/Table_Search_and_Review_Guide.md) - How to find and review tables using Plex and Metadata Portal
- [SQL Helper Guide](./01_Tools/SQL_Helper_Guide.md) - Guide to SQL Helper tool for query assistance
- [AI Chat Tools](./01_Tools/AI_Chat_Tools.md) - When to use AI assistants and how to craft effective prompts for SQL optimization and data analysis

### SQL Resources
- [SQL Basics](./02_SQL_Guide/SQL_Basics.md) - Introduction to SQL structure and syntax
- [CTE Guide](./02_SQL_Guide/CTE_Guide.md) - Working with Common Table Expressions
- [SQL Best Practices](./02_SQL_Guide/SQL_Best_Practices.md) - Writing efficient and maintainable SQL
- [Common SQL Patterns](./02_SQL_Guide/Common_SQL_Patterns.md) - Reusable patterns for credit risk analysis
- [SQL Troubleshooting](./02_SQL_Guide/SQL_Troubleshooting.md) - Fixing common SQL errors and issues
- [Date vs Timestamp Guide](./02_SQL_Guide/Date_vs_Timestamp_Guide.md) - Proper handling of temporal fields in BigQuery
- [Common Query Examples](./02_SQL_Guide/Common_Query_Examples.md) - Validated SQL query examples for common tasks

### Data Dictionary
- [Table Overview](./03_Data_Dictionary/Table_Overview.md) - Comprehensive list of tables used by our team
- [Table Relationships](./03_Data_Dictionary/Table_Relationships.md) - How tables connect to each other
- [Chargeback Tables](./03_Data_Dictionary/Chargeback_Tables.md) - Tables related to chargebacks
- [Shop Tables](./03_Data_Dictionary/Shop_Tables.md) - Tables related to shop data
- [Risk Assessment Tables](./03_Data_Dictionary/Risk_Assessment_Tables.md) - Tables related to risk indicators
- [Financial Tables](./03_Data_Dictionary/Financial_Tables.md) - Tables related to financial data
- [Trust Platform Tables](./03_Data_Dictionary/Trust_Platform_Tables.md) - Tables related to Trust Platform

### Domain Knowledge
- [Chargeback Analysis](./04_Domain_Knowledge/Chargeback.md) - Queries for chargeback rates, trends, and risk indicators
- [GPV Analysis](./04_Domain_Knowledge/GPV.md) - Processing volume metrics and growth patterns
- [Reserves Management](./04_Domain_Knowledge/Reserves.md) - Reserve configurations and balance tracking
- [Capital Recovery](./04_Domain_Knowledge/Capital_Recovery.md) - Analysis of capital recovery activities
- [Fraud Detection](./04_Domain_Knowledge/Fraud_Detection.md) - Fraud patterns and detection mechanisms
- [TrustPlatform Tickets](./04_Domain_Knowledge/TrustPlatform_Tickets.md) - Ticket analytics and case management
- [Booked Losses](./04_Domain_Knowledge/Booked_Losses.md) - Loss tracking, classification, and trend analysis
- [Shopify Payment Balance](./04_Domain_Knowledge/Shopify_Payment_Balance.md) - Balance tracking and reconciliation

### Example Queries
- [Full Query - Shop](./05_Example_Queries/Full_Query_Shop.md) - End-to-end shop analysis queries
- [Full Query - TrustPlatform](./05_Example_Queries/Full_Query_TrustPlatform.md) - Complete TrustPlatform data analysis

## Key Tables and Datasets 📊

These tables are most commonly used in Credit Risk queries and reports:

| Table Name | Description | Location | Documentation |
|------------|-------------|----------|--------------|
| `shopify-dw.money_products.chargebacks_summary` | Comprehensive chargebacks summary data | [Chargeback Tables](./03_Data_Dictionary/Chargeback_Tables.md) | [Schema](./03_Data_Dictionary/Chargeback_Tables.md#chargebacks_summary) |
| `sdp-prd-cti-data.intermediate.shop_chargeback_rates_daily_snapshot` | Chargeback count/rate at any given date | [Chargeback Tables](./03_Data_Dictionary/Chargeback_Tables.md) | [Schema](./03_Data_Dictionary/Chargeback_Tables.md#shop_chargeback_rates_daily_snapshot) |
| `sdp-prd-cti-data.intermediate.shop_chargeback_rates_current` | Current chargeback rates | [Chargeback Tables](./03_Data_Dictionary/Chargeback_Tables.md) | [Schema](./03_Data_Dictionary/Chargeback_Tables.md#shop_chargeback_rates_current) |
| `shopify-dw.risk.trust_platform_tickets_summary_v1` | Tickets summary with CR ticket filtering capability | [Trust Platform Tables](./03_Data_Dictionary/Trust_Platform_Tables.md) | [Schema](./03_Data_Dictionary/Trust_Platform_Tables.md#trust_platform_tickets_summary_v1) |
| `shopify-dw.finance.shop_gmv_current` | GMV/GPV data (aggregated per day and timeframes) | [Financial Tables](./03_Data_Dictionary/Financial_Tables.md) | [Schema](./03_Data_Dictionary/Financial_Tables.md#shop_gmv_current) |
| `shopify-dw.finance.shop_gmv_daily_summary_v1_1` | Detailed daily GMV data for in-depth time series analysis | [Financial Tables](./03_Data_Dictionary/Financial_Tables.md) | [Schema](./03_Data_Dictionary/Financial_Tables.md#shop_gmv_daily_summary_v1_1) |
| `shopify-dw.money_products.shopify_payments_balance_account_daily_cumulative_summary` | GPV balance at current moment or any given date | [Financial Tables](./03_Data_Dictionary/Financial_Tables.md) | [Schema](./03_Data_Dictionary/Financial_Tables.md#shopify_payments_balance_account_daily_cumulative_summary) |
| `shopify-dw.raw_shopify.payments_refunds` | Comprehensive refunds data | [Financial Tables](./03_Data_Dictionary/Financial_Tables.md) | [Schema](./03_Data_Dictionary/Financial_Tables.md#payments_refunds) |
| `sdp-prd-cti-data.intermediate.shop_current_shopify_payments_status` | Current Shopify Payments status | [Shop Tables](./03_Data_Dictionary/Shop_Tables.md) | [Schema](./03_Data_Dictionary/Shop_Tables.md#shop_current_shopify_payments_status) |
| `shopify-dw.money_products.order_transactions_payments_summary` | Detailed order information | [Financial Tables](./03_Data_Dictionary/Financial_Tables.md) | [Schema](./03_Data_Dictionary/Financial_Tables.md#order_transactions_payments_summary) |
| `sdp-prd-cti-data.base.base__shopify_payments_reserve_configurations` | Merchant reserve setup and configuration | [Financial Tables](./03_Data_Dictionary/Financial_Tables.md) | [Schema](./03_Data_Dictionary/Financial_Tables.md#base__shopify_payments_reserve_configurations) |

For a complete list of tables with detailed descriptions, see the [Data Dictionary](./03_Data_Dictionary/Table_Overview.md).

## SQL Best Practices 📝

For detailed SQL best practices, see the [SQL Best Practices](./02_SQL_Guide/SQL_Best_Practices.md) guide. Key highlights:

- Document any data limitations or caveats
- Include comments in complex queries
- Test queries before adding them to the repository
- Use `shopify-dw` as the billing project for all queries
- Follow Shopify data handling guidelines for PII data

---

**Happy data exploring!** 📈 💻

*For questions or suggestions about this repository, please contact the Credit Risk Team.*