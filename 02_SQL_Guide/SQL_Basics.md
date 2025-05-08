# SQL Basics: A Guide for Credit Risk Analysis

## Introduction

This guide provides a comprehensive introduction to SQL (Structured Query Language) specifically tailored for credit risk analysis at Shopify. SQL is the primary language used to query and analyze data in our BigQuery environment.

## Basic SQL Structure

All SQL queries follow a logical structure that mirrors how we think about data analysis:

```sql
-- Basic query structure
SELECT column1, column2     -- What data do I want to see?
FROM table_name             -- Where is the data stored?
WHERE condition             -- What specific data am I interested in?
GROUP BY column1            -- How should I organize/aggregate the data?
ORDER BY column2            -- How should I sort the results?
LIMIT 100                   -- How many results do I want to see?
```

### The SELECT Clause

The SELECT clause specifies which columns you want to retrieve from the database.

```sql
-- Simple SELECT
SELECT 
  shop_id,
  transaction_date,
  amount_usd
FROM 
  `shopify-dw.money_products.banking_balance_transactions`
LIMIT 10
```

You can also perform calculations or transformations in your SELECT clause:

```sql
-- SELECT with calculations
SELECT
  shop_id,
  posted_at AS transaction_date,
  amount_usd,
  amount_usd * 0.029 AS processing_fee,
  DATE_DIFF(CURRENT_DATE(), DATE(posted_at), DAY) AS days_since_transaction
FROM
  `shopify-dw.money_products.banking_balance_transactions`
LIMIT 10
```

### The FROM Clause

The FROM clause specifies the table(s) you're querying. In BigQuery, tables are referenced using a three-part name: `project.dataset.table`.

For Credit Risk analysis, we primarily use tables from these projects:
- `shopify-dw` - The main Shopify data warehouse
- `sdp-prd-cti-data` - Credit Risk specific data models

```sql
-- FROM clause example
SELECT 
  shop_id, 
  chargeback_rate_30d
FROM
  `sdp-prd-cti-data.intermediate.shop_chargeback_rates_current`
LIMIT 10
```

### The WHERE Clause

The WHERE clause filters your results based on specific conditions. This is crucial for limiting your data to relevant subsets.

```sql
-- Basic WHERE clause
SELECT 
  shop_id,
  chargeback_rate_30d
FROM
  `sdp-prd-cti-data.intermediate.shop_chargeback_rates_current`
WHERE
  chargeback_rate_30d > 0.01  -- Only shops with >1% chargeback rate
  AND sp_transaction_amount_usd_30d > 10000  -- Only shops with meaningful volume
LIMIT 100
```

#### Common Operators in WHERE Clauses

| Operator | Description | Example |
|----------|-------------|---------|
| = | Equal to | `WHERE shop_id = 12345` |
| <> or != | Not equal to | `WHERE shop_country <> 'US'` |
| > | Greater than | `WHERE chargeback_rate_30d > 0.01` |
| < | Less than | `WHERE risk_score < 0.7` |
| >= | Greater than or equal to | `WHERE transaction_amount >= 1000` |
| <= | Less than or equal to | `WHERE days_since_creation <= 30` |
| IN | Matches any value in a list | `WHERE shop_id IN (1, 2, 3)` |
| BETWEEN | Within a range (inclusive) | `WHERE transaction_date BETWEEN '2023-01-01' AND '2023-12-31'` |
| IS NULL | Is a null value | `WHERE closed_at IS NULL` |
| IS NOT NULL | Is not a null value | `WHERE dispute_reason IS NOT NULL` |
| AND | Both conditions must be true | `WHERE chargeback_rate > 0.01 AND sp_transaction_amount_usd_30d > 10000` |
| OR | Either condition can be true | `WHERE is_flagged = TRUE OR risk_score > 0.9` |

#### Date Filtering (Important for Performance)

Always include date filters to limit the amount of data processed:

```sql
-- Date filtering with BigQuery date functions
SELECT 
  shop_id,
  posted_at AS transaction_date,
  amount_usd
FROM
  `shopify-dw.money_products.banking_balance_transactions`
WHERE
  -- Last 90 days of data
  posted_at >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 90 DAY)
  AND shop_id = 12345
```

### The GROUP BY Clause

GROUP BY aggregates rows that have the same values in specified columns into summary rows.

```sql
-- Basic GROUP BY
SELECT
  shop_id,
  COUNT(*) AS transaction_count,
  SUM(amount_usd) AS total_amount,
  AVG(amount_usd) AS average_transaction_size,
  MAX(posted_at) AS most_recent_transaction
FROM
  `shopify-dw.money_products.banking_balance_transactions`
WHERE
  posted_at >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 90 DAY)
GROUP BY
  shop_id
LIMIT 100
```

You can group by multiple columns:

```sql
-- Multiple GROUP BY columns
SELECT
  shop_id,
  DATE_TRUNC(DATE(posted_at), MONTH) AS month,
  network AS payment_method,
  COUNT(*) AS transaction_count,
  SUM(amount_usd) AS total_amount
FROM
  `shopify-dw.money_products.banking_balance_transactions`
WHERE
  posted_at >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 180 DAY)
GROUP BY
  shop_id, month, payment_method
ORDER BY
  shop_id, month
LIMIT 100
```

### The HAVING Clause

HAVING filters the results of GROUP BY aggregations.

```sql
-- HAVING clause example
SELECT
  shop_id,
  COUNT(*) AS chargeback_count,
  SUM(chargeback_amount_usd) AS chargeback_amount
FROM
  `shopify-dw.money_products.chargebacks_summary`
WHERE
  provider_chargeback_created_at >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 90 DAY)
GROUP BY
  shop_id
HAVING
  chargeback_count >= 5  -- Only shops with at least 5 chargebacks
  AND chargeback_amount > 1000  -- And at least $1000 in chargeback amount
ORDER BY
  chargeback_amount DESC
LIMIT 100
```

### The ORDER BY Clause

ORDER BY sorts your results based on specified columns.

```sql
-- ORDER BY example
SELECT
  shop_id,
  chargeback_rate_30d,
  sp_transaction_amount_usd_30d
FROM
  `sdp-prd-cti-data.intermediate.shop_chargeback_rates_current`
WHERE
  sp_transaction_amount_usd_30d > 10000
ORDER BY
  chargeback_rate_30d DESC  -- Highest chargeback rates first
LIMIT 20
```

You can sort by multiple columns:

```sql
-- Multiple ORDER BY columns
SELECT
  s.shop_country,
  s.shop_id,
  c.chargeback_rate_30d,
  c.sp_transaction_amount_usd_30d
FROM
  `sdp-prd-cti-data.intermediate.shop_chargeback_rates_current` c
JOIN
  `shopify-dw.accounts_and_administration.shops` s ON s.shop_id = c.shop_id
WHERE
  c.sp_transaction_amount_usd_30d > 10000
ORDER BY
  s.shop_country ASC,  -- First alphabetically by country
  c.chargeback_rate_30d DESC  -- Then by highest chargeback rate
LIMIT 100
```

### Joining Tables

Joins allow you to combine data from multiple tables based on related columns.

#### INNER JOIN

Returns rows when there is a match in both tables.

```sql
-- INNER JOIN example
SELECT
  s.shop_id,
  s.shop_name,
  s.industry,
  c.chargeback_rate_30d,
  c.sp_transaction_amount_usd_30d
FROM
  `shopify-dw.accounts_and_administration.shops` s
INNER JOIN
  `sdp-prd-cti-data.intermediate.shop_chargeback_rates_current` c ON s.shop_id = c.shop_id
WHERE
  c.sp_transaction_amount_usd_30d > 10000
  AND s.created_at >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 180 DAY)
ORDER BY
  c.chargeback_rate_30d DESC
LIMIT 10
```

#### LEFT JOIN

Returns all rows from the left table and matching rows from the right table.

```sql
-- LEFT JOIN example (finding shops without chargebacks)
SELECT
  s.shop_id,
  s.shop_name,
  s.industry,
  COALESCE(c.chargeback_rate_30d, 0) AS chargeback_rate_30d,  -- Default to 0 if null
  COALESCE(c.sp_transaction_amount_usd_30d, 0) AS sp_transaction_amount_usd_30d
FROM
  `shopify-dw.accounts_and_administration.shops` s
LEFT JOIN
  `sdp-prd-cti-data.intermediate.shop_chargeback_rates_current` c ON s.shop_id = c.shop_id
WHERE
  s.created_at >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 90 DAY)
  AND (c.shop_id IS NULL OR c.chargeback_rate_30d = 0)  -- Only shops with no chargebacks
LIMIT 100
```

## Aliases

Aliases make your queries more readable by providing shorthand names for tables and columns.

```sql
-- Table and column aliases
SELECT
  s.shop_id,
  s.shop_name,
  c.chargeback_rate_30d AS cb_rate,
  r.risk_score,
  CASE
    WHEN c.chargeback_rate_30d > 0.01 OR r.risk_score > 0.8 THEN 'High Risk'
    WHEN c.chargeback_rate_30d > 0.005 OR r.risk_score > 0.6 THEN 'Medium Risk'
    ELSE 'Low Risk'
  END AS risk_category
FROM
  `shopify-dw.accounts_and_administration.shops` s
JOIN
  `sdp-prd-cti-data.intermediate.shop_chargeback_rates_current` c ON s.shop_id = c.shop_id
JOIN
  `sdp-prd-cti-data.merchant.shop_risk_indicators` r ON s.shop_id = r.shop_id
WHERE
  r.snapshot_date = CURRENT_DATE()
LIMIT 100
```

## Aggregate Functions

Aggregate functions perform calculations on a set of values and return a single value.

| Function | Description | Example |
|----------|-------------|---------|
| COUNT() | Counts rows | `COUNT(transaction_id)` |
| SUM() | Adds values | `SUM(amount_usd)` |
| AVG() | Calculates average | `AVG(risk_score)` |
| MIN() | Finds minimum value | `MIN(posted_at)` |
| MAX() | Finds maximum value | `MAX(amount_usd)` |
| APPROX_COUNT_DISTINCT() | Approximates count of distinct values (more efficient) | `APPROX_COUNT_DISTINCT(shop_id)` |

```sql
-- Aggregate functions example
SELECT
  COUNT(*) AS total_transactions,
  COUNT(DISTINCT shop_id) AS unique_shops,
  SUM(amount_usd) AS total_amount,
  AVG(amount_usd) AS average_amount,
  MIN(amount_usd) AS smallest_transaction,
  MAX(amount_usd) AS largest_transaction
FROM
  `shopify-dw.money_products.banking_balance_transactions`
WHERE
  posted_at >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 30 DAY)
```

## CASE Statements

CASE statements allow you to implement conditional logic in your queries.

```sql
-- CASE statement example for risk categorization
SELECT
  shop_id,
  chargeback_rate_30d,
  CASE
    WHEN chargeback_rate_30d >= 0.01 THEN 'High Risk (>1%)'
    WHEN chargeback_rate_30d >= 0.005 THEN 'Medium Risk (0.5-1%)'
    WHEN chargeback_rate_30d > 0 THEN 'Low Risk (<0.5%)'
    ELSE 'No Chargebacks'
  END AS risk_category,
  COUNT(*) OVER (PARTITION BY 
    CASE
      WHEN chargeback_rate_30d >= 0.01 THEN 'High Risk (>1%)'
      WHEN chargeback_rate_30d >= 0.005 THEN 'Medium Risk (0.5-1%)'
      WHEN chargeback_rate_30d > 0 THEN 'Low Risk (<0.5%)'
      ELSE 'No Chargebacks'
    END
  ) AS merchants_in_category
FROM
  `sdp-prd-cti-data.intermediate.shop_chargeback_rates_current`
WHERE
  sp_transaction_amount_usd_30d > 5000  -- Only include merchants with meaningful volume
ORDER BY
  chargeback_rate_30d DESC
LIMIT 100
```

## Common SQL Patterns for Credit Risk Analysis

### 1. Finding Merchants with Elevated Risk

```sql
-- Identifying high-risk merchants
SELECT
  s.shop_id,
  s.shop_name,
  s.created_at AS shop_created_at,
  DATE_DIFF(CURRENT_DATE(), DATE(s.created_at), DAY) AS shop_age_days,
  c.chargeback_rate_30d,
  c.chargeback_count_30d,
  c.sp_transaction_amount_usd_30d,
  r.risk_score
FROM
  `shopify-dw.accounts_and_administration.shops` s
JOIN
  `sdp-prd-cti-data.intermediate.shop_chargeback_rates_current` c ON s.shop_id = c.shop_id
JOIN
  `sdp-prd-cti-data.merchant.shop_risk_indicators` r ON s.shop_id = r.shop_id
WHERE
  c.sp_transaction_amount_usd_30d > 10000  -- Only merchants with meaningful volume
  AND (
    c.chargeback_rate_30d > 0.01  -- Over 1% chargeback rate
    OR r.risk_score > 0.8  -- Or high risk score
  )
  AND r.snapshot_date = CURRENT_DATE()  -- Most recent risk scores
ORDER BY
  c.chargeback_rate_30d DESC
LIMIT 100
```

### 2. Time Series Analysis of Chargeback Rates

```sql
-- Analyzing chargeback trends over time
SELECT
  shop_id,
  snapshot_date,
  chargeback_rate_30d,
  sp_transaction_amount_usd_30d,
  -- Calculate the 7-day moving average of chargeback rate
  AVG(chargeback_rate_30d) OVER (
    PARTITION BY shop_id
    ORDER BY snapshot_date
    ROWS BETWEEN 6 PRECEDING AND CURRENT ROW
  ) AS chargeback_rate_7day_avg,
  -- Compare to previous day's rate
  LAG(chargeback_rate_30d) OVER (
    PARTITION BY shop_id
    ORDER BY snapshot_date
  ) AS previous_day_rate,
  -- Calculate day-over-day change
  chargeback_rate_30d - LAG(chargeback_rate_30d) OVER (
    PARTITION BY shop_id
    ORDER BY snapshot_date
  ) AS day_over_day_change
FROM
  `sdp-prd-cti-data.intermediate.shop_chargeback_rates_daily_snapshot`
WHERE
  shop_id = 12345  -- Specific shop analysis
  AND snapshot_date >= DATE_SUB(CURRENT_DATE(), INTERVAL 90 DAY)
ORDER BY
  snapshot_date DESC
```

### 3. Comparing Metrics Across Merchant Segments

```sql
-- Comparing risk metrics across industry categories
SELECT
  s.industry,
  COUNT(DISTINCT s.shop_id) AS merchant_count,
  AVG(c.chargeback_rate_30d) AS avg_chargeback_rate,
  APPROX_QUANTILES(c.chargeback_rate_30d, 100)[OFFSET(50)] AS median_chargeback_rate,
  APPROX_QUANTILES(c.chargeback_rate_30d, 100)[OFFSET(75)] AS p75_chargeback_rate,
  APPROX_QUANTILES(c.chargeback_rate_30d, 100)[OFFSET(90)] AS p90_chargeback_rate,
  SUM(c.chargeback_count_30d) AS total_chargebacks,
  SUM(c.sp_transaction_amount_usd_30d) AS total_sp_transaction_amount_30d
FROM
  `shopify-dw.accounts_and_administration.shops` s
JOIN
  `sdp-prd-cti-data.intermediate.shop_chargeback_rates_current` c ON s.shop_id = c.shop_id
WHERE
  c.sp_transaction_amount_usd_30d > 5000  -- Only include merchants with meaningful volume
GROUP BY
  s.industry
HAVING
  COUNT(DISTINCT s.shop_id) >= 10  -- Only include categories with enough merchants
ORDER BY
  avg_chargeback_rate DESC
```

## Next Steps

Now that you're familiar with basic SQL, proceed to these more advanced topics:

- [Common Table Expressions (CTEs)](./CTE_Guide.md) - For more readable, modular queries
- [SQL Best Practices](./SQL_Best_Practices.md) - For writing efficient and maintainable SQL
- [Common SQL Patterns](./Common_SQL_Patterns.md) - For reusable analysis patterns
- [SQL Troubleshooting](./SQL_Troubleshooting.md) - For fixing common SQL errors

## Resources

- [BigQuery SQL Reference](https://cloud.google.com/bigquery/docs/reference/standard-sql/query-syntax)
- [SQL Style Guide](https://github.com/mattm/sql-style-guide)
- [BigQuery Performance Optimization](https://cloud.google.com/bigquery/docs/best-practices-performance-overview) 