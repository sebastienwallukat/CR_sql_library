# Shopify Payment Balance Queries 💰

Welcome to the Shopify Payment Balance section of our SQL Query Resource Center. This page contains datasets and queries for tracking and reconciling payment balances.

## Datasets 📁

### `sdp-prd-cti-data.finance.payment_balance_daily`

This dataset provides daily snapshots of merchant payment balances, perfect for tracking balance changes over time.

### `sdp-prd-cti-data.finance.payment_balance_transactions`

This dataset contains individual transactions affecting payment balances, including charges, payouts, and adjustments.

## Common Queries 💻

### Balance Reconciliation by Shop

```sql
WITH daily_balance AS (
  SELECT
    shop_id,
    balance_date,
    ending_balance_usd
  FROM
    `sdp-prd-cti-data.finance.payment_balance_daily`
  WHERE
    balance_date BETWEEN DATE_SUB(CURRENT_DATE(), INTERVAL 30 DAY) AND CURRENT_DATE()
),

daily_transactions AS (
  SELECT
    shop_id,
    transaction_date,
    SUM(CASE WHEN transaction_type = 'charge' THEN amount_usd ELSE 0 END) as charges,
    SUM(CASE WHEN transaction_type = 'payout' THEN amount_usd ELSE 0 END) as payouts,
    SUM(CASE WHEN transaction_type = 'adjustment' THEN amount_usd ELSE 0 END) as adjustments,
    SUM(amount_usd) as net_change
  FROM
    `sdp-prd-cti-data.finance.payment_balance_transactions`
  WHERE
    transaction_date BETWEEN DATE_SUB(CURRENT_DATE(), INTERVAL 30 DAY) AND CURRENT_DATE()
  GROUP BY
    shop_id, transaction_date
)

SELECT
  b.shop_id,
  b.balance_date,
  b.ending_balance_usd,
  t.charges,
  t.payouts,
  t.adjustments,
  t.net_change,
  LAG(b.ending_balance_usd) OVER (PARTITION BY b.shop_id ORDER BY b.balance_date) + t.net_change as calculated_balance,
  b.ending_balance_usd - (LAG(b.ending_balance_usd) OVER (PARTITION BY b.shop_id ORDER BY b.balance_date) + t.net_change) as discrepancy
FROM
  daily_balance b
  JOIN daily_transactions t ON b.shop_id = t.shop_id AND b.balance_date = t.transaction_date
ORDER BY
  ABS(b.ending_balance_usd - (LAG(b.ending_balance_usd) OVER (PARTITION BY b.shop_id ORDER BY b.balance_date) + t.net_change)) DESC
```

### Merchants with Negative Balances

```sql
SELECT
  shop_id,
  ending_balance_usd,
  balance_date,
  DATEDIFF(CURRENT_DATE(), balance_date) as days_negative
FROM
  `sdp-prd-cti-data.finance.payment_balance_daily`
WHERE
  balance_date = CURRENT_DATE()
  AND ending_balance_usd < 0
ORDER BY
  ending_balance_usd ASC
```

### Balance Trend Analysis

```sql
WITH monthly_balance AS (
  SELECT
    shop_id,
    DATE_TRUNC(balance_date, MONTH) as month,
    AVG(ending_balance_usd) as avg_balance,
    MAX(ending_balance_usd) as max_balance,
    MIN(ending_balance_usd) as min_balance
  FROM
    `sdp-prd-cti-data.finance.payment_balance_daily`
  WHERE
    balance_date >= DATE_SUB(CURRENT_DATE(), INTERVAL 6 MONTH)
  GROUP BY
    shop_id, month
)

SELECT
  shop_id,
  month,
  avg_balance,
  max_balance,
  min_balance,
  LAG(avg_balance) OVER (PARTITION BY shop_id ORDER BY month) as prev_month_avg,
  avg_balance - LAG(avg_balance) OVER (PARTITION BY shop_id ORDER BY month) as avg_balance_change
FROM
  monthly_balance
ORDER BY
  avg_balance_change DESC
```

## Notes and Best Practices 📝

- Always verify balance calculations against official records
- Be aware of currency conversion impacts on balance reconciliation
- Consider timezone differences when analyzing transaction timing
- For large merchants, high transaction volumes may cause temporary discrepancies

Feel free to contribute new queries or improve the existing ones. Together, we can build a comprehensive resource for payment balance analysis.

Happy balance reconciling! 📊 