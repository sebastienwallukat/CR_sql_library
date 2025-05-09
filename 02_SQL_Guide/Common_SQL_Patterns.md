# Common SQL Patterns for Credit Risk Analysis

This guide documents common SQL patterns specifically tailored for credit risk analysis at Shopify. These patterns provide reusable code templates for frequent analysis needs.

## Table of Contents

- [Time-Series Analysis Patterns](#time-series-analysis-patterns)
  - [Daily Trend Analysis](#1-daily-trend-analysis)
  - [Rolling Average Calculation](#2-rolling-average-calculation)
  - [Year-over-Year Comparison](#3-year-over-year-comparison)
- [Aggregation Patterns](#aggregation-patterns)
  - [Chargeback Rate Calculation](#1-chargeback-rate-calculation)
  - [Risk Score Aggregation](#2-risk-score-aggregation)
  - [Segmentation Analysis](#3-segmentation-analysis)
- [Filtering Patterns](#filtering-patterns)
  - [Dynamic Date Range Filtering](#1-dynamic-date-range-filtering)
  - [Nested Filtering](#2-nested-filtering)
  - [Exclusion Filtering](#3-exclusion-filtering)
- [Window Function Patterns](#window-function-patterns)
  - [Ranking Merchants by Risk](#1-ranking-merchants-by-risk)
  - [Detecting Anomalies with Window Functions](#2-detecting-anomalies-with-window-functions)
  - [Cumulative Metrics Calculation](#3-cumulative-metrics-calculation)
- [CTE Patterns for Complex Analysis](#cte-patterns-for-complex-analysis)
  - [Multi-Step Analysis Pattern](#1-multi-step-analysis-pattern)
  - [Cohort Analysis Pattern](#2-cohort-analysis-pattern)
  - [Funnel Analysis Pattern](#3-funnel-analysis-pattern)
- [Additional Patterns](#additional-patterns)
  - [Reserve Impact Analysis](#1-reserve-impact-analysis)
  - [Fraud Velocity Check](#2-fraud-velocity-check)
  - [Shop Performance Transition Matrix](#3-shop-performance-transition-matrix)
- [When to Use These Patterns](#when-to-use-these-patterns)
- [Best Practices When Using These Patterns](#best-practices-when-using-these-patterns)
- [Temporal Fields Documentation](#temporal-fields-documentation)

## Time-Series Analysis Patterns

### 1. Daily Trend Analysis

```sql
-- Purpose: Track daily transaction metrics over the past 90 days
-- This query calculates daily transaction counts, volumes, and merchant counts
-- with day-over-day changes to identify trends

WITH daily_metrics AS (
  SELECT 
    DATE(order_transaction_created_at) AS date,                 -- Group by transaction date
    COUNT(*) AS transaction_count,                              -- Daily transaction volume
    SUM(amount_local) AS total_amount,                          -- Daily monetary volume
    COUNT(DISTINCT shop_id) AS merchant_count                   -- Number of active merchants per day
  FROM `shopify-dw.money_products.order_transactions_payments_summary`
  WHERE order_transaction_created_at >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 90 DAY)
    AND order_transaction_status = 'success'                    -- Only count successful transactions
  GROUP BY DATE(order_transaction_created_at)
)

SELECT 
  date,
  transaction_count,
  total_amount,
  merchant_count,
  -- Calculate day-over-day changes to identify trends
  transaction_count - LAG(transaction_count) OVER(ORDER BY date) AS transaction_count_change,
  (transaction_count - LAG(transaction_count) OVER(ORDER BY date)) / 
    NULLIF(LAG(transaction_count) OVER(ORDER BY date), 0) AS transaction_count_pct_change
FROM daily_metrics
ORDER BY date
```

### 2. Rolling Average Calculation

```sql
-- Purpose: Calculate rolling averages for chargeback metrics
-- This query creates 7-day and 28-day rolling averages to smooth out daily fluctuations
-- and identify underlying trends in chargeback activity

WITH daily_chargebacks AS (
  SELECT 
    DATE(shopify_chargeback_created_at) AS date,                -- Group by chargeback date
    COUNT(*) AS chargeback_count,                               -- Number of chargebacks per day
    SUM(chargeback_amount_usd) AS chargeback_amount             -- Total chargeback amount in USD
  FROM `shopify-dw.money_products.chargebacks_summary`
  WHERE provider_chargeback_created_at >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 90 DAY)
  GROUP BY DATE(shopify_chargeback_created_at)
)

SELECT 
  date,
  chargeback_count,
  -- Calculate 7-day rolling average to smooth out weekly patterns
  AVG(chargeback_count) OVER(
    ORDER BY date 
    ROWS BETWEEN 6 PRECEDING AND CURRENT ROW
  ) AS chargeback_count_7day_avg,
  
  AVG(chargeback_amount) OVER(
    ORDER BY date 
    ROWS BETWEEN 6 PRECEDING AND CURRENT ROW
  ) AS chargeback_amount_7day_avg,
  
  -- Calculate 28-day rolling average for longer-term trend analysis
  AVG(chargeback_count) OVER(
    ORDER BY date 
    ROWS BETWEEN 27 PRECEDING AND CURRENT ROW
  ) AS chargeback_count_28day_avg
FROM daily_chargebacks
ORDER BY date
```

### 3. Year-over-Year Comparison

```sql
-- Year-over-year comparison by month

WITH monthly_metrics AS (
  SELECT 
    EXTRACT(YEAR FROM order_transaction_created_at) AS year,
    EXTRACT(MONTH FROM order_transaction_created_at) AS month,
    COUNT(*) AS transaction_count,
    SUM(amount_local) AS total_amount
  FROM `shopify-dw.money_products.order_transactions_payments_summary`
  WHERE order_transaction_created_at >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 24 MONTH)
    AND order_transaction_status = 'success'
  GROUP BY 
    EXTRACT(YEAR FROM order_transaction_created_at),
    EXTRACT(MONTH FROM order_transaction_created_at)
)

SELECT 
  current_year.month,
  current_year.transaction_count AS current_year_count,
  previous_year.transaction_count AS previous_year_count,
  (current_year.transaction_count - previous_year.transaction_count) / 
    NULLIF(previous_year.transaction_count, 0) AS yoy_growth
FROM 
  (SELECT * FROM monthly_metrics WHERE year = EXTRACT(YEAR FROM CURRENT_DATE())) current_year
LEFT JOIN 
  (SELECT * FROM monthly_metrics WHERE year = EXTRACT(YEAR FROM CURRENT_DATE()) - 1) previous_year
  ON current_year.month = previous_year.month
ORDER BY
  current_year.month
```

## Aggregation Patterns

### 1. Chargeback Rate Calculation

```sql
-- Chargeback rate calculation by shop

WITH transactions AS (
  SELECT 
    shop_id,
    COUNT(*) AS transaction_count
  FROM `shopify-dw.money_products.order_transactions_payments_summary`
  WHERE order_transaction_created_at >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 90 DAY)
    AND order_transaction_status = 'success'
  GROUP BY shop_id
  HAVING COUNT(*) >= 100  -- Only include merchants with sufficient volume
),

chargebacks AS (
  SELECT 
    shop_id,
    COUNT(*) AS chargeback_count
  FROM `shopify-dw.money_products.chargebacks_summary`
  WHERE provider_chargeback_created_at >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 90 DAY)
  GROUP BY shop_id
)

SELECT 
  t.shop_id,
  t.transaction_count,
  COALESCE(c.chargeback_count, 0) AS chargeback_count,
  SAFE_DIVIDE(COALESCE(c.chargeback_count, 0), t.transaction_count) AS chargeback_rate
FROM transactions t
LEFT JOIN chargebacks c ON t.shop_id = c.shop_id
ORDER BY chargeback_rate DESC
```

### 2. Risk Score Aggregation

```sql
-- Multi-factor risk score calculation

WITH transaction_metrics AS (
  SELECT 
    shop_id,
    COUNT(*) AS transaction_count,
    COUNTIF(order_transaction_status = 'failure') / NULLIF(COUNT(*), 0) AS decline_rate,
    SUM(amount_local) AS total_amount,
    SUM(amount_local) / NULLIF(COUNT(*), 0) AS avg_transaction_amount,
    STDDEV(amount_local) AS amount_stddev
  FROM `shopify-dw.money_products.order_transactions_payments_summary`
  WHERE order_transaction_created_at >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 90 DAY)
  GROUP BY shop_id
  HAVING COUNT(*) >= 100
),

chargeback_metrics AS (
  SELECT 
    shop_id,
    COUNT(*) AS chargeback_count,
    SUM(chargeback_amount_usd) AS chargeback_amount
  FROM `shopify-dw.money_products.chargebacks_summary`
  WHERE provider_chargeback_created_at >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 90 DAY)
  GROUP BY shop_id
),

refund_metrics AS (
  SELECT 
    shop_id,
    COUNT(*) AS refund_count,
    SUM(amount) AS refund_amount
  FROM `shopify-dw.money_products.payment_refunds_summary`
  WHERE created_at >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 90 DAY)
  GROUP BY shop_id
)

SELECT 
  t.shop_id,
  t.transaction_count,
  t.decline_rate,
  SAFE_DIVIDE(COALESCE(c.chargeback_count, 0), t.transaction_count) AS chargeback_rate,
  SAFE_DIVIDE(COALESCE(r.refund_count, 0), t.transaction_count) AS refund_rate,
  -- Composite risk score calculation
  (
    (SAFE_DIVIDE(COALESCE(c.chargeback_count, 0), t.transaction_count) * 0.6) + 
    (t.decline_rate * 0.3) + 
    (SAFE_DIVIDE(COALESCE(r.refund_count, 0), t.transaction_count) * 0.1)
  ) AS composite_risk_score
FROM transaction_metrics t
LEFT JOIN chargeback_metrics c ON t.shop_id = c.shop_id
LEFT JOIN refund_metrics r ON t.shop_id = r.shop_id
ORDER BY composite_risk_score DESC
```

### 3. Segmentation Analysis

```sql
-- Merchant segmentation based on risk profile

WITH risk_metrics AS (
  SELECT 
    t.shop_id,
    COUNT(*) AS transaction_count,
    SAFE_DIVIDE(
      COUNTIF(cb.chargeback_id IS NOT NULL), 
      COUNT(*)
    ) AS chargeback_rate,
    COUNTIF(t.order_transaction_status = 'failure') / COUNT(*) AS decline_rate,
    AVG(t.amount_local) AS avg_amount
  FROM `shopify-dw.money_products.order_transactions_payments_summary` t
  LEFT JOIN `shopify-dw.money_products.chargebacks_summary` cb
    ON t.order_transaction_id = cb.order_transaction_id
  WHERE t.order_transaction_created_at >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 90 DAY)
    AND (cb.provider_chargeback_created_at IS NULL OR cb.provider_chargeback_created_at >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 90 DAY))
  GROUP BY t.shop_id
  HAVING COUNT(*) >= 100
)

SELECT
  shop_id,
  transaction_count,
  chargeback_rate,
  decline_rate,
  avg_amount,
  CASE
    WHEN chargeback_rate > 0.01 THEN 'High Risk'
    WHEN chargeback_rate > 0.005 OR decline_rate > 0.2 THEN 'Medium Risk'
    WHEN transaction_count < 200 AND avg_amount > 1000 THEN 'Low Data, High Amount'
    ELSE 'Low Risk'
  END AS risk_segment
FROM risk_metrics
ORDER BY 
  CASE
    WHEN chargeback_rate > 0.01 THEN 1
    WHEN chargeback_rate > 0.005 OR decline_rate > 0.2 THEN 2
    WHEN transaction_count < 200 AND avg_amount > 1000 THEN 3
    ELSE 4
  END,
  chargeback_rate DESC
```

## Filtering Patterns

### 1. Dynamic Date Range Filtering

```sql
-- Dynamic date range filtering

-- Declare parameters
DECLARE start_date TIMESTAMP DEFAULT TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 90 DAY);
DECLARE end_date TIMESTAMP DEFAULT CURRENT_TIMESTAMP();

-- Use parameters in query
SELECT 
  DATE(order_transaction_created_at) AS date,
  COUNT(*) AS transaction_count
FROM `shopify-dw.money_products.order_transactions_payments_summary`
WHERE order_transaction_created_at >= start_date
  AND order_transaction_created_at < end_date
GROUP BY DATE(order_transaction_created_at)
ORDER BY date
```

### 2. Nested Filtering

```sql
-- Nested filtering for complex conditions

WITH high_value_transactions AS (
  SELECT 
    order_transaction_id,
    shop_id,
    amount_local,
    currency_code_local,
    order_transaction_created_at
  FROM `shopify-dw.money_products.order_transactions_payments_summary`
  WHERE order_transaction_created_at >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 90 DAY)
    AND amount_local > 1000
    AND order_transaction_status = 'success'
),

new_shops AS (
  SELECT
    shop_id,
    created_at AS shop_created_at
  FROM `shopify-dw.accounts_and_administration.shops`
  WHERE created_at >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 180 DAY)
)

SELECT 
  t.shop_id,
  COUNT(*) AS high_value_tx_count,
  SUM(amount_local) AS high_value_tx_amount,
  s.shop_created_at
FROM high_value_transactions t
JOIN new_shops s ON t.shop_id = s.shop_id
WHERE 
  -- Additional conditions on joined data
  TIMESTAMP_DIFF(t.order_transaction_created_at, s.shop_created_at, DAY) <= 30
GROUP BY 
  t.shop_id,
  s.shop_created_at
HAVING COUNT(*) >= 3  -- Shop has at least 3 high-value transactions
ORDER BY high_value_tx_amount DESC
```

### 3. Exclusion Filtering

```sql
-- Exclusion filtering pattern

WITH shops_with_reserves AS (
  SELECT DISTINCT shop_id
  FROM `sdp-prd-cti-data.base.base__shopify_payments_reserve_configurations`
  WHERE is_active = TRUE
)

SELECT 
  s.shop_id,
  s.shopify_payments_status,
  c.chargeback_rate_90d AS chargeback_rate
FROM `sdp-prd-cti-data.intermediate.shop_current_shopify_payments_status` s
LEFT JOIN `sdp-prd-cti-data.intermediate.shop_chargeback_rates_current` c
  ON s.shop_id = c.shop_id
-- Exclusion filter pattern
WHERE s.shop_id NOT IN (SELECT shop_id FROM shops_with_reserves)
  AND s.account_active = TRUE
  AND c.chargeback_rate_90d > 0.005
  AND c.sp_transaction_count_90d >= 100
ORDER BY c.chargeback_rate_90d DESC
```

## Window Function Patterns

### 1. Ranking Merchants by Risk

```sql
-- Ranking merchants by risk metrics

WITH merchant_metrics AS (
  SELECT 
    shop_id,
    chargeback_rate_90d AS chargeback_rate,
    sp_transaction_count_90d AS transaction_count
  FROM `sdp-prd-cti-data.intermediate.shop_chargeback_rates_current`
  WHERE sp_transaction_count_90d >= 100
)

SELECT 
  shop_id,
  chargeback_rate,
  transaction_count,
  -- Ranking functions
  RANK() OVER(ORDER BY chargeback_rate DESC) AS chargeback_rate_rank,
  PERCENT_RANK() OVER(ORDER BY chargeback_rate) AS chargeback_rate_percentile,
  NTILE(10) OVER(ORDER BY chargeback_rate) AS chargeback_rate_decile
FROM merchant_metrics
ORDER BY chargeback_rate_rank
LIMIT 100
```

### 2. Detecting Anomalies with Window Functions

```sql
-- Anomaly detection using window functions

WITH daily_chargebacks AS (
  SELECT 
    shop_id,
    DATE(shopify_chargeback_created_at) AS date,
    COUNT(*) AS chargeback_count
  FROM `shopify-dw.money_products.chargebacks_summary`
  WHERE provider_chargeback_created_at >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 90 DAY)
  GROUP BY shop_id, DATE(shopify_chargeback_created_at)
),

shop_stats AS (
  SELECT
    shop_id,
    date,
    chargeback_count,
    -- Calculate rolling stats
    AVG(chargeback_count) OVER(
      PARTITION BY shop_id 
      ORDER BY date 
      ROWS BETWEEN 14 PRECEDING AND 1 PRECEDING
    ) AS avg_chargeback_count,
    
    STDDEV(chargeback_count) OVER(
      PARTITION BY shop_id 
      ORDER BY date 
      ROWS BETWEEN 14 PRECEDING AND 1 PRECEDING
    ) AS stddev_chargeback_count
  FROM daily_chargebacks
)

SELECT
  shop_id,
  date,
  chargeback_count,
  avg_chargeback_count,
  stddev_chargeback_count,
  -- Flag anomalies (Z-score > 3)
  (chargeback_count - avg_chargeback_count) / NULLIF(stddev_chargeback_count, 0) AS z_score,
  CASE 
    WHEN (chargeback_count - avg_chargeback_count) / NULLIF(stddev_chargeback_count, 0) > 3 
    THEN TRUE 
    ELSE FALSE 
  END AS is_anomaly
FROM shop_stats
WHERE 
  -- Ensure we have enough data for baseline
  avg_chargeback_count IS NOT NULL
  AND stddev_chargeback_count IS NOT NULL
  -- Filter to anomalies only
  AND (chargeback_count - avg_chargeback_count) / NULLIF(stddev_chargeback_count, 0) > 3
ORDER BY 
  date DESC,
  z_score DESC
```

### 3. Cumulative Metrics Calculation

```sql
-- Cumulative metrics calculation

WITH daily_transactions AS (
  SELECT 
    DATE(order_transaction_created_at) AS date,
    COUNT(*) AS daily_tx_count,
    SUM(amount_local) AS daily_amount
  FROM `shopify-dw.money_products.order_transactions_payments_summary`
  WHERE order_transaction_created_at >= TIMESTAMP(DATE_TRUNC(DATE_SUB(CURRENT_DATE(), INTERVAL 1 MONTH), MONTH))
    AND order_transaction_status = 'success'
  GROUP BY DATE(order_transaction_created_at)
)

SELECT
  date,
  daily_tx_count,
  daily_amount,
  -- Calculate month-to-date metrics
  SUM(daily_tx_count) OVER(
    ORDER BY date
    ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
  ) AS mtd_tx_count,
  
  SUM(daily_amount) OVER(
    ORDER BY date
    ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
  ) AS mtd_amount
FROM daily_transactions
ORDER BY date
```

## CTE Patterns for Complex Analysis

### 1. Multi-Step Analysis Pattern

```sql
-- Multi-step analysis with progressive CTEs

-- Step 1: Extract base transaction data
WITH base_transactions AS (
  SELECT 
    shop_id,
    order_transaction_id,
    order_id,
    amount_local,
    currency_code_local,
    order_transaction_created_at,
    order_transaction_status
  FROM `shopify-dw.money_products.order_transactions_payments_summary`
  WHERE order_transaction_created_at >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 90 DAY)
    AND order_transaction_status = 'success'
),

-- Step 2: Aggregate by shop
shop_aggregates AS (
  SELECT
    shop_id,
    COUNT(*) AS transaction_count,
    SUM(amount_local) AS total_amount,
    MIN(order_transaction_created_at) AS first_transaction,
    MAX(order_transaction_created_at) AS last_transaction
  FROM base_transactions
  GROUP BY shop_id
),

-- Step 3: Join with chargeback data
shop_chargebacks AS (
  SELECT
    t.shop_id,
    COUNT(DISTINCT c.chargeback_id) AS chargeback_count,
    SUM(c.chargeback_amount_usd) AS chargeback_amount
  FROM base_transactions t
  LEFT JOIN `shopify-dw.money_products.chargebacks_summary` c
    ON t.order_transaction_id = c.order_transaction_id
    AND c.provider_chargeback_created_at >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 90 DAY)
  GROUP BY t.shop_id
),

-- Step 4: Calculate risk metrics
shop_risk AS (
  SELECT
    a.shop_id,
    a.transaction_count,
    a.total_amount,
    a.first_transaction,
    a.last_transaction,
    COALESCE(c.chargeback_count, 0) AS chargeback_count,
    COALESCE(c.chargeback_amount, 0) AS chargeback_amount,
    SAFE_DIVIDE(COALESCE(c.chargeback_count, 0), a.transaction_count) AS chargeback_rate,
    TIMESTAMP_DIFF(a.last_transaction, a.first_transaction, DAY) AS days_active
  FROM shop_aggregates a
  LEFT JOIN shop_chargebacks c ON a.shop_id = c.shop_id
)

-- Step 5: Final analysis with risk segmentation
SELECT
  shop_id,
  transaction_count,
  total_amount,
  chargeback_count,
  chargeback_rate,
  days_active,
  CASE
    WHEN days_active < 30 AND transaction_count > 100 THEN 'New High Volume'
    WHEN chargeback_rate > 0.01 THEN 'High Risk'
    WHEN chargeback_rate > 0.005 THEN 'Medium Risk'
    WHEN transaction_count < 50 THEN 'Low Data'
    ELSE 'Low Risk'
  END AS risk_segment
FROM shop_risk
WHERE transaction_count >= 10
ORDER BY 
  CASE
    WHEN days_active < 30 AND transaction_count > 100 THEN 1
    WHEN chargeback_rate > 0.01 THEN 2
    WHEN chargeback_rate > 0.005 THEN 3
    WHEN transaction_count < 50 THEN 4
    ELSE 5
  END,
  chargeback_rate DESC
```

### 2. Cohort Analysis Pattern

```sql
-- Merchant cohort analysis pattern

-- Step 1: Determine merchant cohorts by first transaction month
WITH merchant_cohorts AS (
  SELECT
    shop_id,
    DATE_TRUNC(MIN(DATE(order_transaction_created_at)), MONTH) AS cohort_month
  FROM `shopify-dw.money_products.order_transactions_payments_summary`
  WHERE order_transaction_created_at >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 12 MONTH)
  GROUP BY shop_id
),

-- Step 2: Calculate monthly activity
monthly_activity AS (
  SELECT
    shop_id,
    DATE_TRUNC(DATE(order_transaction_created_at), MONTH) AS activity_month,
    COUNT(*) AS transaction_count,
    SUM(amount_local) AS transaction_amount
  FROM `shopify-dw.money_products.order_transactions_payments_summary`
  WHERE order_transaction_created_at >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 12 MONTH)
    AND order_transaction_status = 'success'
  GROUP BY shop_id, DATE_TRUNC(DATE(order_transaction_created_at), MONTH)
),

-- Step 3: Join cohorts with activity
cohort_activity AS (
  SELECT
    c.cohort_month,
    a.activity_month,
    DATE_DIFF(a.activity_month, c.cohort_month, MONTH) AS months_since_first_transaction,
    COUNT(DISTINCT c.shop_id) AS active_merchants,
    SUM(a.transaction_count) AS total_transactions,
    SUM(a.transaction_amount) AS total_amount
  FROM merchant_cohorts c
  JOIN monthly_activity a ON c.shop_id = a.shop_id
  GROUP BY 
    c.cohort_month,
    a.activity_month,
    DATE_DIFF(a.activity_month, c.cohort_month, MONTH)
)

-- Step 4: Calculate cohort retention and performance
SELECT
  cohort_month,
  months_since_first_transaction,
  active_merchants,
  total_transactions,
  total_amount,
  -- Calculate the number of merchants in each cohort
  FIRST_VALUE(active_merchants) OVER(
    PARTITION BY cohort_month 
    ORDER BY months_since_first_transaction
    ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING
  ) AS original_cohort_size,
  
  -- Calculate retention rate
  active_merchants / FIRST_VALUE(active_merchants) OVER(
    PARTITION BY cohort_month 
    ORDER BY months_since_first_transaction
    ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING
  ) AS retention_rate
FROM cohort_activity
ORDER BY 
  cohort_month,
  months_since_first_transaction
```

### 3. Funnel Analysis Pattern

```sql
-- Risk funnel analysis pattern

-- Step 1: Identify all active merchants
WITH active_merchants AS (
  SELECT 
    shop_id
  FROM `sdp-prd-cti-data.intermediate.shop_current_shopify_payments_status`
  WHERE account_active = TRUE
  AND shopify_payments_status = 'active'
),

-- Step 2: Identify merchants with elevated chargeback rates
elevated_chargeback_merchants AS (
  SELECT 
    shop_id
  FROM `sdp-prd-cti-data.intermediate.shop_chargeback_rates_current`
  WHERE chargeback_rate_90d > 0.005
  AND sp_transaction_count_90d >= 100
),

-- Step 3: Identify merchants with active reserves
reserved_merchants AS (
  SELECT 
    shop_id
  FROM `sdp-prd-cti-data.base.base__shopify_payments_reserve_configurations`
  WHERE is_active = TRUE
),

-- Step 4: Identify merchants with open Trust tickets
ticketed_merchants AS (
  SELECT 
    subjectable_id AS shop_id
  FROM `shopify-dw.risk.trust_platform_tickets_summary_v1`
  WHERE status = 'OPEN'
  AND created_at >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 90 DAY)
)

-- Step 5: Build the risk funnel
SELECT
  'All Active Merchants' AS funnel_step,
  COUNT(*) AS merchant_count
FROM active_merchants

UNION ALL

SELECT
  'Elevated Chargeback Rate' AS funnel_step,
  COUNT(*) AS merchant_count
FROM active_merchants a
JOIN elevated_chargeback_merchants c ON a.shop_id = c.shop_id

UNION ALL

SELECT
  'With Active Reserve' AS funnel_step,
  COUNT(*) AS merchant_count
FROM active_merchants a
JOIN elevated_chargeback_merchants c ON a.shop_id = c.shop_id
JOIN reserved_merchants r ON a.shop_id = r.shop_id

UNION ALL

SELECT
  'With Open Trust Ticket' AS funnel_step,
  COUNT(*) AS merchant_count
FROM active_merchants a
JOIN elevated_chargeback_merchants c ON a.shop_id = c.shop_id
JOIN reserved_merchants r ON a.shop_id = r.shop_id
JOIN ticketed_merchants t ON a.shop_id = t.shop_id

ORDER BY
  CASE
    WHEN funnel_step = 'All Active Merchants' THEN 1
    WHEN funnel_step = 'Elevated Chargeback Rate' THEN 2
    WHEN funnel_step = 'With Active Reserve' THEN 3
    WHEN funnel_step = 'With Open Trust Ticket' THEN 4
    ELSE 5
  END
```

## Additional Patterns

### 1. Reserve Impact Analysis

```sql
-- Purpose: Analyze the impact of reserves on merchant cash flow
-- This query calculates the estimated reserve amount and its impact on 
-- merchant balance for different reserve types

WITH merchant_reserves AS (
  SELECT
    shop_id,
    reserve_type,                                               -- Type of reserve (percentage, fixed, etc.)
    percentage,                                                 -- Reserve percentage for percentage-based reserves
    period_in_days                                              -- Rolling reserve period in days
  FROM `sdp-prd-cti-data.base.base__shopify_payments_reserve_configurations`
  WHERE is_active = TRUE                                        -- Only consider active reserves
),

daily_processing AS (
  SELECT
    shop_id,
    date,
    gpv                                                         -- Gross payment volume for the day
  FROM `shopify-dw.intermediate.shop_gmv_daily_summary_v1_1`
  WHERE date >= DATE_SUB(CURRENT_DATE(), INTERVAL 30 DAY)
    AND currency = 'USD'                                        -- Limit to USD for consistency
),

daily_balances AS (
  SELECT
    shop_id,
    date,
    balance                                                     -- Daily balance in the merchant account
  FROM `shopify-dw.money_products.shopify_payments_balance_account_daily_cumulative_summary`
  WHERE date >= DATE_SUB(CURRENT_DATE(), INTERVAL 30 DAY)
    AND currency = 'USD'
)

SELECT
  p.shop_id,
  p.date,
  r.reserve_type,
  r.percentage,
  r.period_in_days,
  p.gpv,
  b.balance,
  -- Calculate estimated daily reserve amount based on reserve type
  CASE
    WHEN r.reserve_type = 'percentage' THEN p.gpv * r.percentage / 100
    ELSE NULL
  END AS estimated_daily_reserve,
  
  -- Calculate the impact of the reserve on the merchant's available balance
  CASE
    WHEN r.reserve_type = 'percentage' THEN 
      (p.gpv * r.percentage / 100) / NULLIF(b.balance, 0)
    ELSE NULL
  END AS reserve_to_balance_ratio
FROM daily_processing p
JOIN merchant_reserves r ON p.shop_id = r.shop_id
LEFT JOIN daily_balances b ON p.shop_id = b.shop_id AND p.date = b.date
ORDER BY 
  p.shop_id,
  p.date
```

### 2. Fraud Velocity Check

```sql
-- Fraud velocity check pattern

WITH hourly_transactions AS (
  SELECT
    shop_id,
    TIMESTAMP_TRUNC(order_transaction_created_at, HOUR) AS hour,
    COUNT(*) AS transaction_count,
    SUM(amount_local) AS transaction_amount,
    COUNT(DISTINCT order_id) AS order_count
  FROM `shopify-dw.money_products.order_transactions_payments_summary`
  WHERE order_transaction_created_at >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 7 DAY)
    AND order_transaction_status = 'success'
  GROUP BY 
    shop_id,
    TIMESTAMP_TRUNC(order_transaction_created_at, HOUR)
),

shop_hourly_stats AS (
  SELECT
    shop_id,
    AVG(transaction_count) AS avg_hourly_transactions,
    STDDEV(transaction_count) AS stddev_hourly_transactions,
    AVG(transaction_amount) AS avg_hourly_amount,
    STDDEV(transaction_amount) AS stddev_hourly_amount
  FROM hourly_transactions
  GROUP BY shop_id
  HAVING COUNT(*) >= 24  -- At least 24 hours of data
)

SELECT
  t.shop_id,
  t.hour,
  t.transaction_count,
  t.transaction_amount,
  t.order_count,
  s.avg_hourly_transactions,
  s.stddev_hourly_transactions,
  -- Calculate z-scores for velocity check
  (t.transaction_count - s.avg_hourly_transactions) / 
    NULLIF(s.stddev_hourly_transactions, 0) AS transaction_count_z_score,
    
  (t.transaction_amount - s.avg_hourly_amount) / 
    NULLIF(s.stddev_hourly_amount, 0) AS transaction_amount_z_score
FROM hourly_transactions t
JOIN shop_hourly_stats s ON t.shop_id = s.shop_id
WHERE 
  -- Flag hours with abnormally high activity
  (t.transaction_count - s.avg_hourly_transactions) / 
    NULLIF(s.stddev_hourly_transactions, 0) > 3
  OR
  (t.transaction_amount - s.avg_hourly_amount) / 
    NULLIF(s.stddev_hourly_amount, 0) > 3
ORDER BY
  t.hour DESC,
  transaction_count_z_score DESC
```

### 3. Shop Performance Transition Matrix

```sql
-- Shop performance transition matrix

WITH shop_current_period AS (
  SELECT
    shop_id,
    CASE
      WHEN chargeback_rate_90d >= 0.01 THEN 'High Risk'
      WHEN chargeback_rate_90d >= 0.005 THEN 'Medium Risk'
      ELSE 'Low Risk'
    END AS current_risk_level
  FROM `sdp-prd-cti-data.intermediate.shop_chargeback_rates_current`
  WHERE sp_transaction_count_90d >= 100
),

shop_previous_period AS (
  SELECT
    shop_id,
    CASE
      WHEN chargeback_rate_90d >= 0.01 THEN 'High Risk'
      WHEN chargeback_rate_90d >= 0.005 THEN 'Medium Risk'
      ELSE 'Low Risk'
    END AS previous_risk_level
  FROM `sdp-prd-cti-data.intermediate.shop_chargeback_rates_daily_snapshot`
  WHERE snapshot_date = DATE_SUB(CURRENT_DATE(), INTERVAL 30 DAY)
    AND sp_transaction_count_90d >= 100
)

SELECT
  p.previous_risk_level,
  c.current_risk_level,
  COUNT(*) AS shop_count,
  -- Calculate percentage within previous level
  COUNT(*) / SUM(COUNT(*)) OVER (PARTITION BY p.previous_risk_level) AS transition_pct
FROM shop_previous_period p
JOIN shop_current_period c ON p.shop_id = c.shop_id
GROUP BY 
  p.previous_risk_level, 
  c.current_risk_level
ORDER BY 
  CASE 
    WHEN p.previous_risk_level = 'High Risk' THEN 1
    WHEN p.previous_risk_level = 'Medium Risk' THEN 2
    ELSE 3
  END,
  CASE 
    WHEN c.current_risk_level = 'High Risk' THEN 1
    WHEN c.current_risk_level = 'Medium Risk' THEN 2
    ELSE 3
  END
```

## When to Use These Patterns

The patterns in this guide are designed to be starting points for credit risk analysis. Here's guidance on when to use each type:

1. **Time-Series Analysis Patterns**: Use when analyzing trends, seasonal patterns, or year-over-year comparisons of risk metrics.

2. **Aggregation Patterns**: Use when summarizing merchant performance, calculating risk scores, or classifying merchants by risk level.

3. **Filtering Patterns**: Use when you need to isolate specific subsets of data for analysis, such as high-risk merchants or recent activity.

4. **Window Function Patterns**: Use for ranking, calculating rolling averages, detecting anomalies, or performing time-based analysis.

5. **CTE Patterns**: Use for complex multi-step analyses that require progressive refinement of the data.

6. **Additional Patterns**: Use for specific credit risk use cases like reserve impact analysis or fraud detection.

## Best Practices When Using These Patterns

1. Always customize the pattern to your specific analysis needs
2. Update date ranges to reflect the appropriate timeframe for your analysis
3. Adjust thresholds (such as chargeback rate levels) based on current risk policies
4. Test on a limited dataset before running on full production data
5. Document any modifications you make to these patterns
7. For timestamp fields, always use TIMESTAMP_SUB() instead of DATE_SUB() when comparing with TIMESTAMP columns
8. Add necessary partition filters for partitioned tables
9. Verify table and column names before running queries

## Temporal Fields Documentation

| Field Name | Data Type | Comparison Function | Example |
|------------|-----------|---------------------|---------|
| order_transaction_created_at | TIMESTAMP | TIMESTAMP_SUB() | `WHERE order_transaction_created_at >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 7 DAY)` |
| shopify_chargeback_created_at | TIMESTAMP | TIMESTAMP_SUB() | `WHERE shopify_chargeback_created_at >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 30 DAY)` |
| provider_chargeback_created_at | TIMESTAMP | TIMESTAMP_SUB() | `WHERE provider_chargeback_created_at >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 90 DAY)` |
| date | DATE | DATE_SUB() | `WHERE date >= DATE_SUB(CURRENT_DATE(), INTERVAL 30 DAY)` |

---