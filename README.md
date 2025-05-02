# Credit Risk SQL Query Resource Center 📊

Welcome to the **Credit Risk SQL Query Resource Center**! This repository serves as the central knowledge hub for our team's most valuable SQL queries, datasets, and analytics resources.

## Purpose

This repository is designed to:
- Maintain consistency in our data analysis approach
- Provide quick access to verified, tested SQL queries
- Document our data sources and calculation methodologies
- Serve as an onboarding resource for new team members

## Getting Started 🚀

### Access Requirements & Billing

- **BigQuery Billing Project** - ⚠️ Important: All queries and reports must use `shopify-dw` as the billing project
- **PII Data Access** - For sensitive data access, claim the `sdp-pii` Clouddo permit
- **Service Accounts** - For Looker Studio integration, submit a Helpdesk request for a Looker Studio Service Account (requires Trust approval)

### Permission Management
- **Clouddo** - For managing data access permissions, including `sdp-pii`
- **Looker Studio Support** - Available at #help-looker-studio-access Slack channel

### How to Use This Repository

1. **Navigate to your topic of interest** - Each file focuses on a specific data domain
2. **Review the available queries** - Each file contains tested, documented SQL queries
3. **Check source tables** - Each topic includes the recommended data sources
4. **Contribute your knowledge** - Add new queries or improve existing ones

## Repository Structure 📂

### Risk Assessment Tools
- [Chargeback Analysis](./Chargeback.md) - Queries for chargeback rates, trends, and risk indicators
- [GPV (Gross Processing Volume)](./GPV.md) - Processing volume metrics and growth patterns
- [Transaction Risk Scoring](./Risk_Scoring.md) - Models and queries for transaction risk evaluation

### Financial Metrics
- [Shopify Payment Balance](./Shopify_Payment_Balance.md) - Balance tracking and reconciliation
- [Booked Losses](./Booked_Losses.md) - Loss tracking, classification, and trend analysis
- [Revenue Metrics](./Revenue.md) - Revenue tracking and performance indicators
- [Reserves](./Reserves.md) - Reserve configurations and balance tracking
- [Capital Recovery](./Capital_Recovery.md) - Analysis of capital recovery activities and effectiveness

### Operational Resources
- [TrustPlatform Tickets](./TrustPlatform_Tickets.md) - Ticket analytics and case management
- [Merchant Monitoring](./Merchant_Monitoring.md) - Merchant health and activity tracking
- [Fraud Detection](./Fraud_Detection.md) - Fraud patterns and detection mechanisms

### Comprehensive Queries
- [Full Query - Shop](./Full_Query_Shop.md) - End-to-end shop analysis queries
- [Full Query - TrustPlatform](./Full_Query_TrustPlatform.md) - Complete TrustPlatform data analysis

## Data Resources 📊

### Key Tables and Datasets

These tables are most commonly used in Credit Risk queries and reports:

| Table Name | Description |
|------------|-------------|
| `shopify-dw.money_products.chargebacks_summary` | Comprehensive chargebacks summary data |
| `sdp-prd-cti-data.intermediate.shop_chargeback_rates_daily_snapshot` | Chargeback count/rate at any given date |
| `sdp-prd-cti-data.intermediate.shop_chargeback_rates_current` | Current chargeback rates |
| `shopify-dw.risk.trust_platform_tickets_summary_v1` | Tickets summary with CR ticket filtering capability |
| `shopify-dw.finance.shop_gmv_current` | GMV/GPV data (aggregated per day and timeframes) |
| `shopify-dw.finance.shop_gmv_daily_summary_v1_1` | Detailed daily GMV data for in-depth time series analysis |
| `shopify-dw.money_products.shopify_payments_balance_account_daily_cumulative_summary` | GPV balance at current moment or any given date |
| `shopify-dw.raw_shopify.payments_refunds` | Comprehensive refunds data |
| `sdp-prd-cti-data.intermediate.shop_current_shopify_payments_status` | Current Shopify Payments status |
| `shopify-dw.money_products.order_transactions_payments_summary` | Detailed order information |
| `sdp-prd-cti-data.base.base__shopify_payments_reserve_configurations` | Merchant reserve setup and configuration |

### Data Discovery Methods

#### Google Cloud BigQuery Console
The primary tool for exploring and discovering datasets in Shopify's data platform.

Key Features:
- Search across all Shopify data projects
- View dataset schemas and metadata
- Run queries directly against datasets
- Check data freshness metrics

#### Shopify Metadata Portal
Internal metadata portal for discovering analytically modeled data in the Shopify data platform.

Key Features:
- Focused on analytically modeled data
- Provides business context and definitions
- Shows dataset ownership and support contacts
- Links to documentation and example queries

#### Looker Studio
For building interactive dashboards and reports from SQL queries.

Key Features:
- Create interactive data visualizations
- Schedule automated report refresh
- Share dashboards with stakeholders
- Integrate with BigQuery data sources

When searching for data:
1. Start with the Metadata Portal for business-context and well-documented datasets
2. Use BigQuery Console for technical exploration and schema information
3. Always verify the data freshness and quality before use

## Working with This Repository 🔄

This repository is hosted on GitHub, a platform for version control and collaboration. Here's how you can work with it using Cursor:

### Getting Started with GitHub

GitHub is a platform that hosts Git repositories, allowing teams to collaborate on code. Key concepts:
- **Repository (Repo)**: A storage location for your project containing all files and revision history
- **Clone**: Creating a local copy of a repository on your computer
- **Commit**: Saving changes to files in your local repository
- **Push**: Uploading committed changes to the remote repository
- **Pull**: Downloading changes from the remote repository to your local copy
- **Branch**: A parallel version of the repository for making changes without affecting the main project

### Cloning the Repository with Cursor

1. **Install Cursor**: Download and install Cursor from [cursor.sh](https://cursor.sh) if you haven't already
2. **Open Cursor**: Launch the Cursor application
3. **Clone Repository**:
   - Click on "File" → "Clone Repository" or use the keyboard shortcut (Cmd+Shift+P on Mac, Ctrl+Shift+P on Windows) and type "Clone Repository"
   - Enter the repository URL: `https://github.com/your-org/CR_sql_library.git`
   - Choose a location on your computer to save the repository

### Making Changes

1. **Open the Repository**:
   - In Cursor, navigate to "File" → "Open Folder" and select the cloned repository folder
   
2. **Make Changes**:
   - Navigate to the file you want to edit in the file explorer panel
   - Make your edits to the file
   - Save your changes (Cmd+S on Mac, Ctrl+S on Windows)

3. **Commit Your Changes**:
   - Click on the Source Control icon in the sidebar (or press Cmd+Shift+G on Mac, Ctrl+Shift+G on Windows)
   - Review your changes
   - Enter a meaningful commit message describing what you changed
   - Click "Commit" to save your changes locally

4. **Push Your Changes**:
   - After committing, click "Push" to send your changes to the GitHub repository
   - If this is your first time pushing, you may need to authenticate with GitHub

### Query Versioning Guidelines

When contributing new queries or updating existing ones, follow these versioning guidelines:

1. **Add a version comment at the top of each query:**
   ```sql
   -- Version: 1.2
   -- Last validated: 2023-07-15
   -- Updated by: [Your Name]
   ```

2. **Increment the version number when:**
   - Making significant changes to query logic
   - Updating to use newer table versions
   - Changing filter conditions or aggregation logic

3. **Document all changes in the query comments**

## Best Practices 📝

- Always use the recommended data tables for consistency
- Document any data limitations or caveats
- Include comments in complex queries
- Test queries before adding them to the repository
- Use `shopify-dw` as the billing project for all queries
- Follow Shopify data handling guidelines for PII data

## SQL Query Performance Optimization 🚀

Optimizing SQL queries is crucial for efficient analysis, especially when working with large datasets:

### General Optimization Tips

1. **Select Only Needed Columns**
   ```sql
   -- Good: Only select what you need
   SELECT shop_id, transaction_date, amount_usd
   FROM transactions
   
   -- Avoid: Selecting everything
   SELECT *
   FROM transactions
   ```

2. **Use Appropriate Filters**
   - Always add date filters to limit data volume
   - Filter on partitioned columns when possible
   - Push filters to the earliest stage of your query

3. **Optimize JOIN Operations**
   - Join on indexed/partitioned columns
   - Filter tables before joining them
   - Use appropriate join types (INNER vs LEFT)

4. **Use Partitioning Effectively**
   ```sql
   -- Efficient: Uses partition pruning
   SELECT *
   FROM `shopify-dw.money_products.chargebacks_summary`
   WHERE provider_chargeback_created_at >= DATE_SUB(CURRENT_DATE(), INTERVAL 90 DAY)
   ```

5. **Leverage CTEs for Complex Logic**
   - Break down complex queries into manageable CTEs
   - Avoid deeply nested subqueries
   - Use CTEs to improve readability and maintainability

### BigQuery-Specific Optimizations

1. **Use Approximate Aggregations for Large Datasets**
   ```sql
   -- Fast approximate count of distinct shops
   SELECT APPROX_COUNT_DISTINCT(shop_id) as shop_count
   FROM `shopify-dw.finance.shop_gmv_current`
   ```

2. **Leverage BigQuery's Cache**
   - Identical queries within 24 hours use cached results
   - Use query parameters to maximize cache hits

3. **Monitor Query Cost**
   - Check query size before running (in the BigQuery UI)
   - Use `--dry_run` flag to estimate bytes processed
   - Consider materializing frequently used subqueries

## Cross-Functional Analysis Templates 🔄

These templates demonstrate how to combine Credit Risk data with other business domains for comprehensive analysis:

### Risk and Revenue Analysis

```sql
-- Template: Combining risk signals with revenue metrics
-- Version: 1.0
-- Last validated: 2023-07-15

WITH shop_risk AS (
  SELECT
    shop_id,
    risk_score,
    chargeback_rate_30d,
    fraud_rate_30d
  FROM
    `sdp-prd-cti-data.merchant.shop_risk_indicators`
  WHERE
    snapshot_date = CURRENT_DATE()
),

shop_revenue AS (
  SELECT
    shop_id,
    SUM(amount_usd) as total_revenue_90d
  FROM
    `sdp-prd-cti-data.finance.revenue_transactions`
  WHERE
    transaction_date >= DATE_SUB(CURRENT_DATE(), INTERVAL 90 DAY)
  GROUP BY
    shop_id
)

SELECT
  r.shop_id,
  r.risk_score,
  r.chargeback_rate_30d,
  r.fraud_rate_30d,
  v.total_revenue_90d,
  SAFE_DIVIDE(r.chargeback_rate_30d * v.total_revenue_90d, 100) as estimated_chargeback_loss
FROM
  shop_risk r
JOIN
  shop_revenue v ON r.shop_id = v.shop_id
ORDER BY
  estimated_chargeback_loss DESC
```

### Merchant Growth and Risk Correlation

```sql
-- Template: Analyzing merchant growth patterns against risk indicators
-- Version: 1.0
-- Last validated: 2023-07-15

WITH merchant_gmv AS (
  SELECT
    shop_id,
    gmv_usd_l30d,
    gmv_usd_l90d,
    gmv_usd_l365d,
    SAFE_DIVIDE(gmv_usd_l30d, 30) as daily_gmv_current,
    SAFE_DIVIDE(gmv_usd_l90d - gmv_usd_l30d, 60) as daily_gmv_previous
  FROM
    `shopify-dw.finance.shop_gmv_current`
),

merchant_risk AS (
  SELECT
    shop_id,
    risk_score,
    chargeback_rate_30d
  FROM
    `sdp-prd-cti-data.merchant.shop_risk_indicators`
  WHERE
    snapshot_date = CURRENT_DATE()
)

SELECT
  g.shop_id,
  g.gmv_usd_l30d,
  g.gmv_usd_l90d,
  r.risk_score,
  r.chargeback_rate_30d,
  g.daily_gmv_current,
  g.daily_gmv_previous,
  SAFE_DIVIDE(g.daily_gmv_current - g.daily_gmv_previous, g.daily_gmv_previous) as gmv_growth_rate,
  CASE
    WHEN SAFE_DIVIDE(g.daily_gmv_current - g.daily_gmv_previous, g.daily_gmv_previous) > 0.5 AND r.risk_score > 0.7
    THEN 'High Growth, High Risk'
    WHEN SAFE_DIVIDE(g.daily_gmv_current - g.daily_gmv_previous, g.daily_gmv_previous) > 0.5
    THEN 'High Growth, Normal Risk'
    WHEN r.risk_score > 0.7
    THEN 'Low Growth, High Risk'
    ELSE 'Normal Profile'
  END as merchant_profile
FROM
  merchant_gmv g
JOIN
  merchant_risk r ON g.shop_id = r.shop_id
WHERE
  g.gmv_usd_l90d > 10000  -- Focus on merchants with meaningful GMV
ORDER BY
  gmv_growth_rate DESC
```

## Standard Query Template 📄

Use this template as a starting point for new queries to maintain consistency:

```sql
-- Title: [Brief Description of Query Purpose]
-- Version: 1.0
-- Last validated: YYYY-MM-DD
-- Author: [Your Name]
-- Description: [Detailed explanation of what this query analyzes and its business context]
-- Caveats: [Any limitations, edge cases, or considerations when using this query]

-- Constants or parameters (if applicable)
DECLARE shop_id_param INT64 DEFAULT 12345;
DECLARE lookback_days INT64 DEFAULT 90;

-- Main query
WITH base_data AS (
  -- First logical step of the analysis
  SELECT
    shop_id,
    transaction_date,
    amount_usd
  FROM
    `project.dataset.table`
  WHERE
    -- Always include date filters to limit data volume
    transaction_date >= DATE_SUB(CURRENT_DATE(), INTERVAL lookback_days DAY)
    -- Additional filters as needed
),

aggregated_data AS (
  -- Second logical step with aggregations
  SELECT
    shop_id,
    DATE_TRUNC(transaction_date, MONTH) as month,
    SUM(amount_usd) as monthly_total
  FROM
    base_data
  GROUP BY
    shop_id, month
)

-- Final output with business logic
SELECT
  a.shop_id,
  a.month,
  a.monthly_total,
  -- Additional calculated fields
  LAG(a.monthly_total) OVER (PARTITION BY a.shop_id ORDER BY a.month) as previous_month,
  SAFE_DIVIDE(
    a.monthly_total - LAG(a.monthly_total) OVER (PARTITION BY a.shop_id ORDER BY a.month),
    LAG(a.monthly_total) OVER (PARTITION BY a.shop_id ORDER BY a.month)
  ) as month_over_month_growth
FROM
  aggregated_data a
WHERE
  -- Additional filters on the aggregated data
  a.shop_id = shop_id_param
ORDER BY
  a.month DESC
```

## Frequently Asked Questions ❓

### General Questions

**Q: How do I get access to the tables mentioned in this repository?**  
A: Access to BigQuery tables requires appropriate permissions. Use Clouddo to request access to the `shopify-dw` project and specific datasets like `sdp-pii` for sensitive data.

**Q: How often are these queries updated?**  
A: We aim to validate and update queries quarterly. Each query includes a "Last validated" date to indicate when it was last checked.

**Q: Can I use these queries in production systems?**  
A: These queries are primarily designed for analysis and reporting. For production systems, additional optimization and error handling may be required.

### Technical Questions

**Q: Why do some of my queries timeout in BigQuery?**  
A: Queries processing large volumes of data may timeout. Try limiting your date range, adding more specific filters, or optimizing your query using the techniques in the "SQL Query Performance Optimization" section.

**Q: How do I handle NULL values in my analysis?**  
A: Use `COALESCE()` or `IFNULL()` functions to substitute NULL values. For calculations, utilize `SAFE_DIVIDE()` instead of the division operator to avoid errors.

**Q: How can I verify my chargeback rate calculations are correct?**  
A: Compare your results with the standard tables like `sdp-prd-cti-data.intermediate.shop_chargeback_rates_current`. Chargeback rates should divide the count of chargebacks by the count of transactions (not the monetary amounts).

### Business Questions

**Q: What's a normal chargeback rate for Shopify merchants?**  
A: Chargeback rates vary significantly by industry. Generally, rates below 0.5% are considered healthy, 0.5-1% warrant attention, and rates above 1% are concerning. Use industry benchmarks for more specific guidance.

**Q: How do I identify high-risk merchants most efficiently?**  
A: Combine multiple risk signals including chargeback rates, velocity changes, and Trust Battery scores. The `shop_risk_indicators` table provides consolidated risk metrics.

**Q: What's the difference between GMV and GPV in the context of risk analysis?**  
A: Gross Merchandise Value (GMV) represents the total sales value of merchandise sold, while Gross Payment Volume (GPV) represents the portion processed through Shopify Payments. For risk analysis, GPV is often more relevant as it represents financial exposure.

---

**Happy data exploring!** 📈 💻

*For questions or suggestions about this repository, please contact the Credit Risk Data Team.*

*Last Updated: July 2023*
