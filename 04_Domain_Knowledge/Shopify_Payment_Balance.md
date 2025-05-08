# Shopify Payment Balance Queries 💰

Welcome to the Shopify Payment Balance section of our SQL Query Resource Center. This page contains datasets and queries for tracking and analyzing financial transactions and payment data.

## Datasets 📁

### `shopify-dw.finance.gross_merchandise_volume`

This dataset provides information on transactions with GMV values, essential for tracking payments and sales volumes across Shopify. It enables analysis of payment flow by shop, transaction type, and time period.

### `shopify-dw.finance.tender_transactions`

This dataset contains individual payment transactions, including details about payment methods, amounts, and timestamps. It's ideal for analyzing specific payment events and reconciling transaction flows.

### `shopify-dw.mart_finance_data.payable_details`

This dataset provides enhanced information about payables with invoice context, enabling analysis of payment status, dates, and timing-related calculations.

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

## Notes and Best Practices 📝

- Always include time-based filters when querying large transaction tables to improve query performance
- Use `TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL X DAY)` when comparing with timestamp fields
- Consider currency conversion impacts when analyzing financial data across different regions
- The `gross_merchandise_volume` table has a mix of data from different GMV pipeline versions; check the `gmv_version` column if needed
- For payment transaction analysis, focus on the `reported_at` field as the primary timestamp
- Always verify significant financial discrepancies against other data sources or operational systems

Feel free to contribute new queries or improve the existing ones. Together, we can build a comprehensive resource for payment and financial analysis.

Happy balance reconciling! 📊 