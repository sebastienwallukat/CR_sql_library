# Financial Tables

This document provides comprehensive information about financial tables used for credit risk analysis at Shopify.

## Table of Contents
- [Overview](#overview)
- [Primary Tables](#primary-tables)
  - [shop_gmv_current](#shopify-dwfinanceshop_gmv_current)
  - [shop_gmv_daily_summary_v1](#shopify-dwfinanceshop_gmv_daily_summary_v1)
  - [shop_gmv_daily_summary_v1_1](#shopify-dwintermediateshop_gmv_daily_summary_v1_1)
  - [order_transactions_payments_summary](#shopify-dwmoney_productsorder_transactions_payments_summary)
  - [shopify_payments_balance_account_daily_cumulative_summary](#shopify-dwmoney_productsshopify_payments_balance_account_daily_cumulative_summary)
  - [base__shopify_payments_reserve_configurations](#sdp-prd-cti-databasebase__shopify_payments_reserve_configurations)
  - [base__payments_refunds](#shopify-dwbasebase__payments_refunds)
- [Secondary Tables](#secondary-tables)
  - [shopify_payments_disputes](#shopify-dwmoney_productsshopify_payments_disputes)
  - [shop_gpv_daily_summary_v1](#shopify-dwfinanceshop_gpv_daily_summary_v1)
  - [shopify_payments_transfers](#shopify-dwmoney_productsshopify_payments_transfers)
- [How to Join Financial Tables](#how-to-join-financial-tables)
- [Data Quality Considerations](#data-quality-considerations)
- [Common Analysis Patterns](#common-analysis-patterns)

## Overview

Financial tables contain transaction data, GMV/GPV metrics, merchant balances, reserve configurations, and other financial indicators crucial for credit risk assessment. These tables form the foundation for analyzing merchant financial activity and risk exposure.

## Primary Tables

### shopify-dw.finance.shop_gmv_current

**Description**:  
This table provides current Gross Merchandise Value (GMV) and Gross Payment Volume (GPV) metrics for shops across different time windows (1-day, 7-day, 28-day). It's used to assess the recent volume and financial activity of merchants.

**Update Frequency**: Daily

**Primary Keys**: `shop_id`


**Usage Notes**:
- Primary table for understanding current merchant processing volume
- GPV represents Shopify Payments volume and direct exposure
- GMV represents total volume (including other payment methods)
- Different time windows (7d, 28d, etc.) provide insights into volume trends
- The table is clustered by `shop_id` for query performance

**Example Query**:
```sql
-- Find high-volume merchants with recent activity
SELECT 
  shop_id,
  gmv_usd_l28d,
  gpv_usd_l7d / NULLIF(gpv_usd_l28d, 0) AS recent_activity_ratio
FROM `shopify-dw.finance.shop_gmv_current`
WHERE gmv_usd_l28d > 10000
ORDER BY gmv_usd_l28d DESC
LIMIT 100
```

### shopify-dw.finance.shop_gmv_daily_summary_v1

**Description**:  
This table provides daily GMV and order data for each shop, allowing for detailed time series analysis. It contains daily snapshots of financial activity, enabling trend analysis and volatility assessment.

**Update Frequency**: Daily

**Primary Keys**: `shop_id`, `date`


**Usage Notes**:
- Essential for time series analysis of merchant activity
- Useful for detecting unusual patterns or spikes in volume
- Only contains records for dates on which a shop has 1+ transaction in the previous 365 days with non-zero GMV
- Data is aggregated by day and shop_id
- Always include date filters for performance optimization

**Example Query**:
```sql
-- Identify merchants with unusual activity spikes
WITH daily_metrics AS (
  SELECT 
    shop_id,
    date,
    gmv_usd,
    AVG(gmv_usd) OVER(
      PARTITION BY shop_id
      ORDER BY date
      ROWS BETWEEN 13 PRECEDING AND 1 PRECEDING
    ) AS avg_gmv_14d
  FROM `shopify-dw.finance.shop_gmv_daily_summary_v1`
  WHERE date >= DATE_SUB(CURRENT_DATE(), INTERVAL 30 DAY)
    AND gmv_usd > 0
)

SELECT 
  shop_id,
  date,
  gmv_usd,
  avg_gmv_14d,
  gmv_usd / NULLIF(avg_gmv_14d, 0) AS volume_ratio
FROM daily_metrics
WHERE gmv_usd > 1000
  AND gmv_usd > 3 * avg_gmv_14d  -- 3x increase over 14-day average
ORDER BY volume_ratio DESC
```

### shopify-dw.intermediate.shop_gmv_daily_summary_v1_1

**Description**:  
This table contains a shop-level daily rollup of GMV (in USD), gross orders with GMV, and net orders with GMV. It provides detailed time series data for monitoring merchant sales activity and trends.

**Update Frequency**: Daily

**Primary Keys**: `shop_id`, `date`


**Usage Notes**:
- Only contains records for dates on which a shop has 1+ transaction in the previous 365 days with non-zero GMV
- Attempts to only publish data for "complete" days where both upstreams have reported a complete day of transactions
- For the most recent data by shop, check out the `shop_gmv_current` model
- The table is partitioned by `date` and clustered by `shop_id` for query performance

**Example Query**:
```sql
-- Get monthly GMV for a specific shop
SELECT
  DATE_TRUNC(date, MONTH) as reported_month,
  SUM(gmv_usd) AS gmv_usd
FROM
  `shopify-dw.intermediate.shop_gmv_daily_summary_v1_1`
WHERE
  shop_id = 2939277
  AND date >= DATE_SUB(CURRENT_DATE(), INTERVAL 180 DAY)
GROUP BY 1
ORDER BY 1 DESC
```

### shopify-dw.money_products.order_transactions_payments_summary

**Description**:  
Comprehensive table with detailed information about each payment transaction. This is one of the most important tables for transaction-level analysis, containing order details, payment method, status, and amount information.

**Update Frequency**: Near real-time (hourly updates)

**Primary Keys**: `order_transaction_id`

**Foreign Keys**: 
- `shop_id` references `shopify-dw.shopify.shops.id`
- `order_id` references `shopify-dw.shopify.orders.id`


**Usage Notes**:
- This table has data from 2018-01-01 onwards
- Critical table for analyzing transaction patterns and risk indicators
- `order_transaction_status` field can be used to calculate approval/decline rates
- Always use `order_transaction_created_at` for date filtering
- Pay attention to the `is_shopify_payments_gateway` field to identify Shopify Payments transactions

**Example Query**:
```sql
-- Calculate transaction approval rates by shop
SELECT 
  shop_id,
  COUNT(*) AS transaction_count,
  COUNTIF(order_transaction_status = 'success') AS approved_count,
  COUNTIF(order_transaction_status = 'failure') AS declined_count,
  COUNTIF(order_transaction_status = 'success') / COUNT(*) AS approval_rate
FROM `shopify-dw.money_products.order_transactions_payments_summary`
WHERE order_transaction_created_at >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 30 DAY)
GROUP BY shop_id
HAVING COUNT(*) >= 50  -- Only include shops with sufficient volume
ORDER BY approval_rate ASC
LIMIT 100
```

### shopify-dw.money_products.shopify_payments_balance_account_daily_cumulative_summary

**Description**:  
This table provides daily snapshots of merchant balance accounts, showing the cumulative balance for each shop at the end of each day. It's essential for understanding merchant liquidity and financial exposure.

**Update Frequency**: Daily

**Primary Keys**: `remote_account_id`, `currency_code`, `date`


**Usage Notes**:
- Use this table to monitor merchant balances over time
- Understand the impact of reserves on merchant liquidity
- Track changes in financial position
- Balances are cumulative as of the end of each day
- Join with `money_products.shopify_payments_provider_accounts` to get additional shop information
- Use `is_latest_date = TRUE` to get the most recent snapshot

**Example Query**:
```sql
-- Calculate balance-to-GPV ratios to assess financial cushion
WITH daily_balances AS (
  SELECT 
    sppa.shop_id,
    s.date,
    s.currency_code,
    s.cumulative_net_local_sum AS balance,
    s.day_reserve_local AS reserved_balance,
    g.gmv_usd
  FROM `shopify-dw.money_products.shopify_payments_balance_account_daily_cumulative_summary` s
  LEFT JOIN `shopify-dw.money_products.shopify_payments_provider_accounts` sppa
    ON s.remote_account_id = sppa.remote_account_id
  LEFT JOIN `shopify-dw.finance.shop_gmv_daily_summary_v1` g
    ON sppa.shop_id = g.shop_id
    AND s.date = g.date
  WHERE s.date >= DATE_SUB(CURRENT_DATE(), INTERVAL 30 DAY)
    AND s.currency_code = 'USD'
)

SELECT 
  shop_id,
  date,
  balance,
  reserved_balance,
  gmv_usd,
  SAFE_DIVIDE(balance, gmv_usd) AS balance_to_gmv_ratio
FROM daily_balances
WHERE gmv_usd > 0
ORDER BY balance_to_gmv_ratio ASC
LIMIT 100
```

### sdp-prd-cti-data.base.base__shopify_payments_reserve_configurations

**Description**:  
Contains information about reserve configurations applied to merchant accounts. Reserves are a critical risk control mechanism that holds a portion of merchant funds to mitigate potential chargeback or fraud exposure.

**Update Frequency**: Near real-time

**Primary Keys**: `shop_id`, `account_id`


**Usage Notes**:
- Critical table for understanding risk controls applied to merchants
- Reserves typically hold a percentage of transaction value for a specified period
- `is_active` should be used to filter for current reserves only
- `reserve_hold_duration` indicates how long funds are held (typically 7-120 days)
- `reserve_percentage` typically ranges from 5% to 100% of transaction value

**Example Query**:
```sql
-- Find active reserves and calculate average hold percentage
SELECT 
  account_type,
  COUNT(*) AS reserve_count,
  AVG(reserve_percentage) AS avg_percentage,
  AVG(reserve_hold_duration) AS avg_hold_period
FROM `sdp-prd-cti-data.base.base__shopify_payments_reserve_configurations`
WHERE is_active = TRUE
GROUP BY account_type
ORDER BY reserve_count DESC
```

### shopify-dw.base.base__payments_refunds

**Description**:  
Contains detailed information about payment refund transactions, providing insights into refund patterns and potential risk indicators.

**Update Frequency**: Daily

**Primary Keys**: `payments_refund_id`

**Foreign Keys**:
- `shop_id` references `shopify-dw.shopify.shops.id` 
- `order_transaction_id` references order transaction records


**Usage Notes**:
- High refund rates can be a risk indicator
- Use this table to calculate refund ratios and identify unusual patterns
- Join with order/transaction tables to correlate with other metrics
- Excludes deleted and test payments refunds
- The table contains one row per payment refund transaction

**Example Query**:
```sql
-- Calculate refund rates by shop
WITH transactions AS (
  SELECT 
    shop_id,
    SUM(amount_local) AS total_amount,
    COUNT(*) AS transaction_count
  FROM `shopify-dw.money_products.order_transactions_payments_summary`
  WHERE order_transaction_created_at >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 30 DAY)
    AND order_transaction_status = 'success'
  GROUP BY shop_id
),

refunds AS (
  SELECT 
    shop_id,
    COUNT(*) AS refund_count
  FROM `shopify-dw.base.base__payments_refunds`
  WHERE created_at >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 30 DAY)
    AND status = 'success'
  GROUP BY shop_id
)

SELECT 
  t.shop_id,
  t.transaction_count,
  r.refund_count,
  SAFE_DIVIDE(r.refund_count, t.transaction_count) AS refund_count_ratio
FROM transactions t
LEFT JOIN refunds r ON t.shop_id = r.shop_id
WHERE t.transaction_count >= 50
ORDER BY refund_count_ratio DESC
LIMIT 100
```

## Secondary Tables

### shopify-dw.money_products.shopify_payments_disputes

**Description**:  
Contains information about payment disputes, including chargebacks and inquiries. This table is useful for understanding dispute patterns and resolution outcomes.

**Update Frequency**: Daily

**Primary Keys**: `id`

**Foreign Keys**:
- `shop_id` references `shopify-dw.shopify.shops.id`
- `order_id` references `shopify-dw.shopify.orders.id`


**Usage Notes**:
- Provides detailed dispute information
- Track dispute status and resolution outcomes
- Monitor evidence submission and deadlines
- Calculate win/loss rates for disputes

**Example Query**:
```sql
-- Analyze dispute outcomes by reason
SELECT 
  reason,
  COUNT(*) AS dispute_count,
  COUNTIF(status = 'won') AS won_count,
  COUNTIF(status = 'lost') AS lost_count,
  COUNTIF(status = 'under_review') AS pending_count,
  COUNTIF(status = 'won') / COUNTIF(status IN ('won', 'lost')) AS win_rate
FROM `shopify-dw.money_products.shopify_payments_disputes`
WHERE created_at >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 180 DAY)
GROUP BY reason
ORDER BY dispute_count DESC
```

### shopify-dw.finance.shop_gpv_daily_summary_v1

**Description**:  
Provides daily aggregated GPV data, providing detailed time series analysis of payment volume.

**Update Frequency**: Daily

**Primary Keys**: `shop_id`, `date`


**Usage Notes**:
- Provides a daily view of payment processing activity
- Useful for identifying day-over-day trends
- Only contains records for dates on which a shop has transactions with non-zero GPV
- Always join with `date` field when analyzing

**Example Query**:
```sql
-- Analyze day-over-day growth trends
WITH daily_growth AS (
  SELECT 
    shop_id,
    date,
    gpv_usd,
    LAG(gpv_usd) OVER(PARTITION BY shop_id ORDER BY date) AS prev_day_gpv
  FROM `shopify-dw.finance.shop_gpv_daily_summary_v1`
  WHERE date >= DATE_SUB(CURRENT_DATE(), INTERVAL 30 DAY)
)

SELECT 
  shop_id,
  date,
  gpv_usd,
  prev_day_gpv,
  (gpv_usd - prev_day_gpv) / NULLIF(prev_day_gpv, 0) AS daily_growth
FROM daily_growth
WHERE prev_day_gpv IS NOT NULL
  AND gpv_usd > 1000
ORDER BY daily_growth DESC
LIMIT 100
```

### shopify-dw.money_products.shopify_payments_transfers

**Description**:  
Contains information about money transfers (payouts) to merchants, tracking when funds move from Shopify to merchant bank accounts.

**Update Frequency**: Daily

**Primary Keys**: `id`

**Foreign Keys**: `shop_id` references `shopify-dw.shopify.shops.id`


**Usage Notes**:
- Track merchant payout frequency and amounts
- Identify failed transfers which may indicate banking issues
- Monitor payout patterns relative to transaction volume
- `type` is usually "payout" for standard transfers to merchants

**Example Query**:
```sql
-- Calculate average days between payouts
WITH transfers AS (
  SELECT 
    shop_id,
    created_at,
    LAG(created_at) OVER(
      PARTITION BY shop_id
      ORDER BY created_at
    ) AS prev_transfer_date
  FROM `shopify-dw.money_products.shopify_payments_transfers`
  WHERE type = 'payout'
    AND status = 'paid'
    AND created_at >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 90 DAY)
)

SELECT 
  shop_id,
  COUNT(*) AS transfer_count,
  AVG(TIMESTAMP_DIFF(created_at, prev_transfer_date, DAY)) AS avg_days_between_payouts
FROM transfers
WHERE prev_transfer_date IS NOT NULL
GROUP BY shop_id
HAVING COUNT(*) >= 5
ORDER BY avg_days_between_payouts DESC
```

## How to Join Financial Tables

### Joining Financial Tables with Shop Data

```sql
-- Join GPV data with shop information
SELECT 
  s.name AS shop_name,
  s.country_code,
  s.created_at AS shop_created_at,
  g.gmv_usd_l28d,
  DATE_DIFF(CURRENT_DATE(), DATE(s.created_at), DAY) AS shop_age_days
FROM `shopify-dw.finance.shop_gmv_current` g
JOIN `shopify-dw.shopify.shops` s ON g.shop_id = s.id
WHERE g.gmv_usd_l28d > 10000
ORDER BY g.gmv_usd_l28d DESC
```

### Joining Transaction Data with Refunds

```sql
-- Calculate refund rates for transactions
WITH transactions AS (
  SELECT 
    shop_id,
    order_id,
    order_transaction_id,
    amount_local,
    currency_code_local,
    order_transaction_created_at
  FROM `shopify-dw.money_products.order_transactions_payments_summary`
  WHERE order_transaction_created_at >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 30 DAY)
    AND order_transaction_status = 'success'
),

refunds AS (
  SELECT 
    shop_id,
    order_id,
    amount AS refund_amount,
    created_at AS refund_date
  FROM `shopify-dw.base.base__payments_refunds`
  WHERE created_at >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 30 DAY)
)

SELECT 
  t.shop_id,
  COUNT(DISTINCT t.order_id) AS order_count,
  SUM(t.amount_local) AS total_amount,
  COUNT(DISTINCT r.order_id) AS refunded_order_count,
  SUM(r.refund_amount) AS total_refund_amount,
  COUNT(DISTINCT r.order_id) / COUNT(DISTINCT t.order_id) AS order_refund_rate,
  SUM(r.refund_amount) / SUM(t.amount_local) AS amount_refund_rate
FROM transactions t
LEFT JOIN refunds r ON t.shop_id = r.shop_id AND t.order_id = r.order_id
GROUP BY t.shop_id
HAVING COUNT(DISTINCT t.order_id) >= 10
ORDER BY amount_refund_rate DESC
```

### Joining Balance Data with Reserves

```sql
-- Analyze reserve impact on merchant balances
SELECT 
  sppa.shop_id,
  s.date,
  s.cumulative_net_local_sum AS balance,
  s.day_reserve_local AS reserved_balance,
  r.account_type,
  r.reserve_percentage,
  r.reserve_hold_duration,
  s.day_reserve_local / NULLIF(s.cumulative_net_local_sum, 0) AS reserved_ratio
FROM `shopify-dw.money_products.shopify_payments_balance_account_daily_cumulative_summary` s
JOIN `shopify-dw.money_products.shopify_payments_provider_accounts` sppa
  ON s.remote_account_id = sppa.remote_account_id
JOIN `sdp-prd-cti-data.base.base__shopify_payments_reserve_configurations` r
  ON sppa.shop_id = r.shop_id
WHERE s.date = DATE_SUB(CURRENT_DATE(), INTERVAL 1 DAY)
  AND r.is_active = TRUE
  AND s.cumulative_net_local_sum > 0
ORDER BY reserved_ratio DESC
```

## Data Quality Considerations

1. **Currency Consistency**:
   - Always join financial tables using appropriate keys (shop_id, date, currency_code)
   - Be careful when aggregating amounts across different currencies
   - Use currency conversion when necessary for cross-currency analysis

2. **Temporal Field Types**:
   - Pay close attention to DATE vs TIMESTAMP fields
   - Use DATE functions (DATE_SUB, DATE_DIFF) for DATE fields
   - Use TIMESTAMP functions (TIMESTAMP_SUB, TIMESTAMP_DIFF) for TIMESTAMP fields

3. **Balance Calculations**:
   - Balance tables show cumulative amounts at the end of each day
   - Reserved amounts are a subset of the total balance
   - Use appropriate joins when working with balance-related tables

4. **Volume Aggregation**:
   - GPV represents Shopify Payments volume only
   - GMV includes all payment methods (higher than GPV)
   - Different time windows (7d, 28d, etc.) are pre-calculated for convenience

5. **Data Freshness**:
   - Transaction data typically updates hourly
   - Balance and GMV/GPV data typically updates daily
   - Always check for the most recent data

## Common Analysis Patterns

### Financial Health Assessment

```sql
-- Comprehensive financial health metrics
WITH volume_metrics AS (
  SELECT 
    shop_id,
    gmv_usd_l28d,
    gpv_usd_l7d / NULLIF(gpv_usd_l28d, 0) AS recent_activity_ratio
  FROM `shopify-dw.finance.shop_gmv_current`
  WHERE gmv_usd_l28d > 0
),

balance_metrics AS (
  SELECT 
    sppa.shop_id,
    s.currency_code,
    s.cumulative_net_local_sum AS balance,
    s.day_reserve_local AS reserved_balance,
    s.cumulative_net_local_sum - s.day_reserve_local AS available_balance,
    s.day_reserve_local / NULLIF(s.cumulative_net_local_sum, 0) AS reserve_ratio
  FROM `shopify-dw.money_products.shopify_payments_balance_account_daily_cumulative_summary` s
  JOIN `shopify-dw.money_products.shopify_payments_provider_accounts` sppa
    ON s.remote_account_id = sppa.remote_account_id
  WHERE s.date = DATE_SUB(CURRENT_DATE(), INTERVAL 1 DAY)
    AND s.currency_code = 'USD'
),

reserve_configs AS (
  SELECT 
    shop_id,
    account_type AS reserve_type,
    reserve_percentage AS percentage,
    reserve_hold_duration AS period_in_days
  FROM `sdp-prd-cti-data.base.base__shopify_payments_reserve_configurations`
  WHERE is_active = TRUE
)

SELECT 
  v.shop_id,
  v.gmv_usd_l28d,
  v.recent_activity_ratio,
  b.balance,
  b.available_balance,
  b.reserve_ratio,
  r.percentage AS reserve_percentage,
  r.period_in_days AS reserve_period,
  b.balance / NULLIF(v.gmv_usd_l28d, 0) AS balance_to_monthly_gmv_ratio
FROM volume_metrics v
LEFT JOIN balance_metrics b ON v.shop_id = b.shop_id
LEFT JOIN reserve_configs r ON v.shop_id = r.shop_id
ORDER BY balance_to_monthly_gmv_ratio ASC
```

### Transaction Quality Analysis

```sql
-- Analyze transaction quality indicators
SELECT 
  shop_id,
  COUNT(*) AS transaction_count,
  
  -- Success rates
  COUNTIF(order_transaction_status = 'success') / COUNT(*) AS success_rate,
  COUNTIF(order_transaction_status = 'failure') / COUNT(*) AS failure_rate,
  
  -- Card details
  COUNTIF(card_country_code = card_country_code) / COUNTIF(card_country_code IS NOT NULL) AS card_country_match_rate,
  
  -- Amount distribution
  AVG(amount_local) AS avg_amount,
  APPROX_QUANTILES(amount_local, 100)[OFFSET(50)] AS median_amount,
  STDDEV(amount_local) AS amount_stddev
  
FROM `shopify-dw.money_products.order_transactions_payments_summary`
WHERE order_transaction_created_at >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 30 DAY)
  AND payment_details_type = 'credit_card'
GROUP BY shop_id
HAVING COUNT(*) >= 50
ORDER BY success_rate ASC
LIMIT 100
```

### Reserve Coverage Analysis

```sql
-- Analyze if reserves adequately cover risk exposure
WITH chargeback_exposure AS (
  SELECT 
    shop_id,
    COUNT(*) AS chargeback_count,
    SUM(amount) AS chargeback_amount
  FROM `shopify-dw.money_products.shopify_payments_disputes`
  WHERE created_at >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 90 DAY)
    AND status IN ('lost', 'under_review')
  GROUP BY shop_id
),

gpv_data AS (
  SELECT 
    shop_id,
    gpv_usd_l28d
  FROM `shopify-dw.finance.shop_gmv_current`
  WHERE gpv_usd_l28d > 0
),

reserve_data AS (
  SELECT 
    r.shop_id,
    r.reserve_percentage AS percentage,
    r.reserve_hold_duration AS period_in_days,
    b.day_reserve_local AS reserved_balance,
    g.gpv_usd_l28d * (r.reserve_percentage / 100) AS theoretical_reserve
  FROM `sdp-prd-cti-data.base.base__shopify_payments_reserve_configurations` r
  JOIN gpv_data g ON r.shop_id = g.shop_id
  LEFT JOIN (
    SELECT 
      sppa.shop_id,
      s.day_reserve_local
    FROM `shopify-dw.money_products.shopify_payments_balance_account_daily_cumulative_summary` s
    JOIN `shopify-dw.money_products.shopify_payments_provider_accounts` sppa
      ON s.remote_account_id = sppa.remote_account_id
    WHERE s.date = DATE_SUB(CURRENT_DATE(), INTERVAL 1 DAY)
      AND s.currency_code = 'USD'
  ) b ON r.shop_id = b.shop_id
  WHERE r.is_active = TRUE
)

SELECT 
  r.shop_id,
  r.percentage,
  r.period_in_days,
  r.reserved_balance,
  r.theoretical_reserve,
  c.chargeback_amount,
  c.chargeback_amount / NULLIF(r.reserved_balance, 0) AS exposure_to_reserve_ratio
FROM reserve_data r
LEFT JOIN chargeback_exposure c ON r.shop_id = c.shop_id
WHERE c.chargeback_amount > 0
ORDER BY exposure_to_reserve_ratio DESC
```