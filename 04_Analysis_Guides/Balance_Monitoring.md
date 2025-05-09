# Shopify Payment Balance Queries 💰

Welcome to the Shopify Payment Balance section of our SQL Query Resource Center. This page contains datasets and queries for tracking and analyzing financial transactions and payment data.

## Table of Contents
- [Introduction](#introduction)
- [Datasets](#datasets)
  - [gross_merchandise_volume](#shopify-dwfinancegross_merchandise_volume)
  - [tender_transactions](#shopify-dwfinancetender_transactions)
  - [payable_details](#shopify-dwmart_finance_datapayable_details)
- [Common Queries](#common-queries)
  - [Transaction Analysis by Shop](#transaction-analysis-by-shop)
  - [Shops with Highest Transaction Volumes](#shops-with-highest-transaction-volumes)
  - [Payment Transaction Trend Analysis](#payment-transaction-trend-analysis)
  - [Payable Status Analysis](#payable-status-analysis)
- [Notes and Best Practices](#notes-and-best-practices)

## Introduction

Payment balance queries help analysts track and analyze financial transactions, payment flows, and financial status across the Shopify platform. These queries are essential for reconciling transactions, understanding payment patterns, and identifying trends in merchant financial activity.

## Datasets 📁

### `shopify-dw.finance.gross_merchandise_volume`

This dataset provides information on transactions with GMV values, essential for tracking payments and sales volumes across Shopify. It enables analysis of payment flow by shop, transaction type, and time period.

The table contains a mix of data from different GMV pipeline versions, identified by the `gmv_version` column. It has one row per transaction with GMV values identified when the order is trusted, the transaction is trusted, and the order/transaction is monetized.

### `shopify-dw.finance.tender_transactions`

This dataset contains individual payment transactions, including details about payment methods, amounts, and timestamps. Tender transactions are order transactions that succeed and have actual transfer of money (excluding gift cards). The dataset includes currency conversion to USD and basic trustworthiness criteria.

Key characteristics:
- Append-only and immutable data
- Backfilled from 2018 onwards
- Use `reported_at` to find when the transaction occurred
- May contain duplicates (filtered out in base models)

### `shopify-dw.mart_finance_data.payable_details`

This dataset provides enhanced information about payables with invoice context, enabling analysis of payment status, dates, and timing-related calculations. The model adds invoice metadata (paid_at, billed_at, failed_at dates), boolean flags for payment status, and time-related calculations to the billing_payables model.

## Common Queries 💻

### Transaction Analysis by Shop

```sql
WITH daily_transactions AS (
  SELECT
    DATE(reported_at) AS transaction_date,
    shop_id,
    SUM(gmv_usd) AS gmv_usd,
    SUM(CASE WHEN shopify_payments_fees_usd > 0 THEN shopify_payments_fees_usd ELSE 0 END) AS fees_usd
  FROM
    `shopify-dw.finance.gross_merchandise_volume`
  WHERE
    reported_at >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 30 DAY)
    AND shop_id IS NOT NULL
  GROUP BY
    transaction_date, shop_id
)

SELECT
  shop_id,
  transaction_date,
  gmv_usd,
  fees_usd,
  gmv_usd - fees_usd AS net_amount,
  SUM(gmv_usd) OVER (PARTITION BY shop_id ORDER BY transaction_date) AS cumulative_gmv,
  SUM(fees_usd) OVER (PARTITION BY shop_id ORDER BY transaction_date) AS cumulative_fees
FROM
  daily_transactions
ORDER BY
  shop_id, transaction_date
```

#### Description
This query analyzes transaction data by shop over the past 30 days. It calculates the daily GMV, fees, and net amount for each shop, along with cumulative totals over time. The window functions provide a running total of GMV and fees per shop, allowing for trend analysis over the 30-day period.

#### Example
The query results show a breakdown of daily GMV, fees, net amount, and cumulative totals for each shop. For instance, Shop ID 12345 might show:
| shop_id | transaction_date | gmv_usd | fees_usd | net_amount | cumulative_gmv | cumulative_fees |
|---------|------------------|---------|----------|------------|----------------|-----------------|
| 12345   | 2023-06-01       | 1500.00 | 45.00    | 1455.00    | 1500.00        | 45.00           |
| 12345   | 2023-06-02       | 2000.00 | 60.00    | 1940.00    | 3500.00        | 105.00          |

### Shops with Highest Transaction Volumes

```sql
SELECT
  shop_id,
  COUNT(*) AS transaction_count,
  SUM(gmv_usd) AS total_gmv_usd,
  AVG(gmv_usd) AS avg_transaction_value_usd,
  MIN(reported_at) AS earliest_transaction,
  MAX(reported_at) AS latest_transaction
FROM
  `shopify-dw.finance.gross_merchandise_volume`
WHERE
  reported_at >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 30 DAY)
  AND shop_id IS NOT NULL
  AND gmv_usd > 0
GROUP BY
  shop_id
ORDER BY
  total_gmv_usd DESC
LIMIT 100
```

#### Description
This query identifies the top 100 shops by GMV over the past 30 days. It provides key metrics including transaction count, total GMV, average transaction value, and the time span of transaction activity. This analysis helps identify high-value merchants and understand their transaction patterns.

#### Example
The query results identify shops with the highest transaction volumes, showing metrics like:
| shop_id | transaction_count | total_gmv_usd | avg_transaction_value_usd | earliest_transaction | latest_transaction |
|---------|-------------------|---------------|---------------------------|----------------------|-------------------|
| 98765   | 1245              | 125000.00     | 100.40                    | 2023-06-01 00:05:12  | 2023-06-30 23:45:55 |
| 54321   | 867               | 98500.00      | 113.61                    | 2023-06-01 08:15:33  | 2023-06-30 21:12:08 |

### Payment Transaction Trend Analysis

```sql
WITH monthly_transactions AS (
  SELECT
    shop_id,
    DATE_TRUNC(reported_at, MONTH) AS month,
    SUM(gmv_usd) AS monthly_gmv,
    COUNT(*) AS transaction_count,
    COUNT(DISTINCT DATE(reported_at)) AS active_days
  FROM
    `shopify-dw.finance.gross_merchandise_volume`
  WHERE
    reported_at >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 6 MONTH)
    AND shop_id IS NOT NULL
  GROUP BY
    shop_id, month
)

SELECT
  shop_id,
  month,
  monthly_gmv,
  transaction_count,
  active_days,
  monthly_gmv / active_days AS avg_daily_gmv,
  LAG(monthly_gmv) OVER (PARTITION BY shop_id ORDER BY month) AS prev_month_gmv,
  monthly_gmv - LAG(monthly_gmv) OVER (PARTITION BY shop_id ORDER BY month) AS gmv_change,
  CASE 
    WHEN LAG(monthly_gmv) OVER (PARTITION BY shop_id ORDER BY month) > 0 
    THEN (monthly_gmv - LAG(monthly_gmv) OVER (PARTITION BY shop_id ORDER BY month)) / LAG(monthly_gmv) OVER (PARTITION BY shop_id ORDER BY month) 
    ELSE NULL 
  END AS gmv_change_pct
FROM
  monthly_transactions
ORDER BY
  shop_id, month
```

#### Description
This query analyzes monthly transaction trends for shops over the past 6 months. It calculates key metrics including monthly GMV, transaction count, active days, and average daily GMV. The query also includes month-over-month comparison, showing absolute and percentage changes in GMV. This helps identify growth patterns and seasonal variations in transaction activity.

#### Example
The results provide monthly transaction metrics for each shop, showing trends like:
| shop_id | month      | monthly_gmv | transaction_count | active_days | avg_daily_gmv | prev_month_gmv | gmv_change | gmv_change_pct |
|---------|------------|-------------|-------------------|-------------|---------------|----------------|------------|----------------|
| 12345   | 2023-01-01 | 45000.00    | 450               | 28          | 1607.14       | null           | null       | null           |
| 12345   | 2023-02-01 | 48500.00    | 485               | 28          | 1732.14       | 45000.00       | 3500.00    | 0.0778         |

### Payable Status Analysis

```sql
SELECT
  DATE_TRUNC(created_at, DAY) AS day,
  SUM(invoice_amount_usd) AS total_invoice_amount_usd,
  SUM(CASE WHEN is_paid THEN invoice_amount_usd ELSE 0 END) AS paid_amount_usd,
  SUM(CASE WHEN is_billed AND NOT is_paid AND NOT is_failed THEN invoice_amount_usd ELSE 0 END) AS pending_amount_usd,
  SUM(CASE WHEN is_failed THEN invoice_amount_usd ELSE 0 END) AS failed_amount_usd,
  ROUND(SUM(CASE WHEN is_paid THEN invoice_amount_usd ELSE 0 END) / NULLIF(SUM(invoice_amount_usd), 0) * 100, 2) AS payment_success_rate
FROM
  `shopify-dw.mart_finance_data.payable_details`
WHERE
  created_at >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 30 DAY)
GROUP BY
  day
ORDER BY
  day
```

#### Description
This query analyzes the payment status of invoices created in the last 30 days. It segments invoice amounts by payment status (paid, pending, failed) and calculates a payment success rate. The daily breakdown helps identify patterns or issues in the payment process and track payment performance over time.

#### Example
The results show daily breakdown of payment statuses:
| day        | total_invoice_amount_usd | paid_amount_usd | pending_amount_usd | failed_amount_usd | payment_success_rate |
|------------|--------------------------|-----------------|--------------------|--------------------|----------------------|
| 2023-06-01 | 125000.00                | 115000.00       | 8000.00            | 2000.00            | 92.00                |
| 2023-06-02 | 118500.00                | 105000.00       | 12000.00           | 1500.00            | 88.61                |

## Notes and Best Practices 📝

- Always include time-based filters when querying large transaction tables to improve query performance
- Use `TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL X DAY)` when comparing with timestamp fields
- Consider currency conversion impacts when analyzing financial data across different regions
- The `gross_merchandise_volume` table has a mix of data from different GMV pipeline versions; check the `gmv_version` column if needed
- For payment transaction analysis, focus on the `reported_at` field as the primary timestamp
- Always verify significant financial discrepancies against other data sources or operational systems
- For currency conversion, join with `shopify-dw.finance.currency_rate_daily_snapshot` on currency_code and date

Feel free to contribute new queries or improve the existing ones. Together, we can build a comprehensive resource for payment and financial analysis.

Happy balance reconciling! 📊 