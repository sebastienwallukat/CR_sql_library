# Financial Tables

This document provides comprehensive information about financial tables used for credit risk analysis at Shopify.

## Overview

Financial tables contain transaction data, GMV/GPV metrics, merchant balances, reserve configurations, and other financial indicators crucial for credit risk assessment. These tables form the foundation for analyzing merchant financial activity and risk exposure.

## Primary Tables

### shopify-dw.finance.shop_gmv_current

**Description**:  
This table provides current Gross Merchandise Value (GMV) and Gross Payment Volume (GPV) metrics for shops across different time windows (1-day, 7-day, 28-day). It's used to assess the recent volume and financial activity of merchants.

**Update Frequency**: Daily

**Primary Keys**: `shop_id`, `currency`

**Schema**:

| Field Name | Type | Description |
|------------|------|-------------|
| `shop_id` | INTEGER | Unique identifier for a shop |
| `currency` | STRING | Currency of the financial metrics |
| `gmv_1d` | FLOAT | Gross Merchandise Value over the past 1 day |
| `gmv_7d` | FLOAT | Gross Merchandise Value over the past 7 days |
| `gmv_28d` | FLOAT | Gross Merchandise Value over the past 28 days |
| `gpv_1d` | FLOAT | Gross Payment Volume over the past 1 day |
| `gpv_7d` | FLOAT | Gross Payment Volume over the past 7 days |
| `gpv_28d` | FLOAT | Gross Payment Volume over the past 28 days |
| `order_count_1d` | INTEGER | Number of orders over the past 1 day |
| `order_count_7d` | INTEGER | Number of orders over the past 7 days |
| `order_count_28d` | INTEGER | Number of orders over the past 28 days |
| `last_updated_at` | TIMESTAMP | When this record was last updated |

**Usage Notes**:
- Primary table for understanding current merchant processing volume
- GPV represents Shopify Payments volume and direct exposure
- GMV represents total volume (including other payment methods)
- Different time windows (1d, 7d, 28d) provide insights into volume trends
- Always join with `currency` when comparing amounts

**Example Query**:
```sql
-- Find high-volume merchants with recent activity
SELECT 
  shop_id,
  currency,
  gpv_28d,
  gpv_7d / gpv_28d AS recent_activity_ratio
FROM `shopify-dw.finance.shop_gmv_current`
WHERE currency = 'USD'
  AND gpv_28d > 10000
ORDER BY gpv_28d DESC
LIMIT 100
```

### shopify-dw.finance.shop_gmv_daily_summary_v1_1

**Description**:  
This table provides daily GMV and GPV data for each shop, allowing for detailed time series analysis. It contains daily snapshots of financial activity, enabling trend analysis and volatility assessment.

**Update Frequency**: Daily

**Primary Keys**: `shop_id`, `date`, `currency`

**Schema**:

| Field Name | Type | Description |
|------------|------|-------------|
| `shop_id` | INTEGER | Unique identifier for a shop |
| `date` | DATE | The date of the metrics |
| `currency` | STRING | Currency of the financial metrics |
| `gmv` | FLOAT | Gross Merchandise Value for the day |
| `gpv` | FLOAT | Gross Payment Volume for the day |
| `order_count` | INTEGER | Number of orders for the day |
| `transaction_count` | INTEGER | Number of transactions for the day |
| `refund_amount` | FLOAT | Amount refunded on this day |
| `refund_count` | INTEGER | Number of refunds on this day |
| `last_updated_at` | TIMESTAMP | When this record was last updated |

**Usage Notes**:
- Essential for time series analysis of merchant activity
- Useful for detecting unusual patterns or spikes in volume
- Data is aggregated by day, shop_id, and currency
- Refund metrics provide additional risk indicators
- Always include date filters for performance optimization

**Example Query**:
```sql
-- Identify merchants with unusual activity spikes
WITH daily_metrics AS (
  SELECT 
    shop_id,
    date,
    gpv,
    AVG(gpv) OVER(
      PARTITION BY shop_id
      ORDER BY date
      ROWS BETWEEN 13 PRECEDING AND 1 PRECEDING
    ) AS avg_gpv_14d
  FROM `shopify-dw.finance.shop_gmv_daily_summary_v1_1`
  WHERE date >= DATE_SUB(CURRENT_DATE(), INTERVAL 30 DAY)
    AND currency = 'USD'
)

SELECT 
  shop_id,
  date,
  gpv,
  avg_gpv_14d,
  gpv / NULLIF(avg_gpv_14d, 0) AS volume_ratio
FROM daily_metrics
WHERE gpv > 1000
  AND gpv > 3 * avg_gpv_14d  -- 3x increase over 14-day average
ORDER BY volume_ratio DESC
```

### shopify-dw.money_products.order_transactions_payments_summary

**Description**:  
Comprehensive table with detailed information about each payment transaction. This is one of the most important tables for transaction-level analysis, containing order details, payment method, status, and amount information.

**Update Frequency**: Near real-time (hourly updates)

**Primary Keys**: `transaction_id`

**Foreign Keys**: 
- `shop_id` references `shopify-dw.shopify.shops.id`
- `order_id` references `shopify-dw.shopify.orders.id`

**Schema**:

| Field Name | Type | Description |
|------------|------|-------------|
| `transaction_id` | STRING | Unique identifier for the transaction |
| `shop_id` | INTEGER | Unique identifier for a shop |
| `order_id` | INTEGER | Order ID associated with this transaction |
| `created_at` | TIMESTAMP | When the transaction was created |
| `processed_at` | TIMESTAMP | When the transaction was processed |
| `status` | STRING | Transaction status (success, failure, pending, etc.) |
| `payment_method` | STRING | Payment method used (credit_card, paypal, etc.) |
| `gateway` | STRING | Payment gateway used |
| `amount` | FLOAT | Transaction amount |
| `currency` | STRING | Transaction currency |
| `authorization` | STRING | Authorization code (if applicable) |
| `avs_result_code` | STRING | Address Verification System result |
| `cvv_result_code` | STRING | Card Verification Value result |
| `error_code` | STRING | Error code (if transaction failed) |
| `source_name` | STRING | Source of the transaction |
| `card_brand` | STRING | Card brand (Visa, Mastercard, etc.) |
| `payment_details` | RECORD | Nested record with additional payment details |
| `risk_level` | STRING | Risk level assigned by payment processor |

**Usage Notes**:
- This table is partitioned by `created_at` - always include date filters
- Critical table for analyzing transaction patterns and risk indicators
- `status` field can be used to calculate approval/decline rates
- `risk_level` provides insight into gateway risk assessment
- AVS/CVV results are important fraud indicators

**Example Query**:
```sql
-- Calculate transaction approval rates by shop
SELECT 
  shop_id,
  COUNT(*) AS transaction_count,
  COUNTIF(status = 'success') AS approved_count,
  COUNTIF(status = 'failure') AS declined_count,
  COUNTIF(status = 'success') / COUNT(*) AS approval_rate
FROM `shopify-dw.money_products.order_transactions_payments_summary`
WHERE created_at >= DATE_SUB(CURRENT_DATE(), INTERVAL 30 DAY)
GROUP BY shop_id
HAVING COUNT(*) >= 50  -- Only include shops with sufficient volume
ORDER BY approval_rate ASC
LIMIT 100
```

### shopify-dw.money_products.shopify_payments_balance_account_daily_cumulative_summary

**Description**:  
This table provides daily snapshots of merchant balance accounts, showing the cumulative balance for each shop at the end of each day. It's essential for understanding merchant liquidity and financial exposure.

**Update Frequency**: Daily

**Primary Keys**: `shop_id`, `date`, `currency`

**Schema**:

| Field Name | Type | Description |
|------------|------|-------------|
| `shop_id` | INTEGER | Unique identifier for a shop |
| `date` | DATE | The date of the balance snapshot |
| `currency` | STRING | Currency of the balance |
| `balance` | FLOAT | Cumulative balance amount |
| `reserved_balance` | FLOAT | Portion of balance currently reserved |
| `pending_balance` | FLOAT | Portion of balance pending release |
| `available_balance` | FLOAT | Portion of balance available for payout |
| `last_updated_at` | TIMESTAMP | When this record was last updated |

**Usage Notes**:
- Use this table to monitor merchant balances over time
- Understand the impact of reserves on merchant liquidity
- Track changes in financial position
- Balances are cumulative as of the end of each day
- Always join with both `date` and `currency` when analyzing

**Example Query**:
```sql
-- Calculate balance-to-GPV ratios to assess financial cushion
WITH daily_balances AS (
  SELECT 
    b.shop_id,
    b.date,
    b.currency,
    b.balance,
    b.reserved_balance,
    g.gpv
  FROM `shopify-dw.money_products.shopify_payments_balance_account_daily_cumulative_summary` b
  LEFT JOIN `shopify-dw.finance.shop_gmv_daily_summary_v1_1` g
    ON b.shop_id = g.shop_id
    AND b.date = g.date
    AND b.currency = g.currency
  WHERE b.date >= DATE_SUB(CURRENT_DATE(), INTERVAL 30 DAY)
    AND b.currency = 'USD'
)

SELECT 
  shop_id,
  date,
  balance,
  gpv,
  SAFE_DIVIDE(balance, gpv) AS balance_to_gpv_ratio
FROM daily_balances
WHERE gpv > 0
ORDER BY balance_to_gpv_ratio ASC
LIMIT 100
```

### sdp-prd-cti-data.base.base__shopify_payments_reserve_configurations

**Description**:  
Contains information about reserve configurations applied to merchant accounts. Reserves are a critical risk control mechanism that holds a portion of merchant funds to mitigate potential chargeback or fraud exposure.

**Update Frequency**: Near real-time

**Primary Keys**: `shop_id`, `id`

**Schema**:

| Field Name | Type | Description |
|------------|------|-------------|
| `id` | STRING | Unique identifier for the reserve configuration |
| `shop_id` | INTEGER | Unique identifier for a shop |
| `reserve_type` | STRING | Type of reserve (percentage, fixed, etc.) |
| `percentage` | FLOAT | Reserve percentage (for percentage reserves) |
| `amount` | FLOAT | Fixed reserve amount (for fixed reserves) |
| `period_in_days` | INTEGER | Number of days funds are held in reserve |
| `created_at` | TIMESTAMP | When the reserve was created |
| `updated_at` | TIMESTAMP | When the reserve was last updated |
| `is_active` | BOOLEAN | Whether the reserve is currently active |
| `currency` | STRING | Currency of the reserve |
| `reason` | STRING | Reason for applying the reserve |

**Usage Notes**:
- Critical table for understanding risk controls applied to merchants
- `reserve_type` indicates how funds are reserved (usually 'percentage')
- `percentage` typically ranges from 5% to 100% of transaction value
- `period_in_days` indicates how long funds are held (typically 7-120 days)
- `is_active` should be used to filter for current reserves only

**Example Query**:
```sql
-- Find active reserves and calculate average hold percentage
SELECT 
  reserve_type,
  COUNT(*) AS reserve_count,
  AVG(percentage) AS avg_percentage,
  AVG(period_in_days) AS avg_hold_period
FROM `sdp-prd-cti-data.base.base__shopify_payments_reserve_configurations`
WHERE is_active = TRUE
GROUP BY reserve_type
ORDER BY reserve_count DESC
```

### shopify-dw.raw_shopify.payments_refunds

**Description**:  
Contains detailed information about refund transactions, providing insights into refund patterns and potential risk indicators.

**Update Frequency**: Daily

**Primary Keys**: `id`

**Foreign Keys**:
- `shop_id` references `shopify-dw.shopify.shops.id` 
- `order_id` references `shopify-dw.shopify.orders.id`
- `transaction_id` references transaction records

**Schema**:

| Field Name | Type | Description |
|------------|------|-------------|
| `id` | INTEGER | Unique identifier for the refund |
| `shop_id` | INTEGER | Unique identifier for a shop |
| `order_id` | INTEGER | Order ID associated with this refund |
| `transaction_id` | STRING | Transaction ID for the refund |
| `created_at` | TIMESTAMP | When the refund was created |
| `processed_at` | TIMESTAMP | When the refund was processed |
| `amount` | FLOAT | Refund amount |
| `currency` | STRING | Refund currency |
| `gateway` | STRING | Payment gateway used |
| `reason` | STRING | Reason for the refund |
| `status` | STRING | Refund status |
| `restock` | BOOLEAN | Whether items were restocked |

**Usage Notes**:
- High refund rates can be a risk indicator
- Use this table to calculate refund ratios and identify unusual patterns
- Join with order/transaction tables to correlate with other metrics
- Always include date filters for performance optimization

**Example Query**:
```sql
-- Calculate refund rates by shop
WITH transactions AS (
  SELECT 
    shop_id,
    SUM(amount) AS total_amount,
    COUNT(*) AS transaction_count
  FROM `shopify-dw.money_products.order_transactions_payments_summary`
  WHERE created_at >= DATE_SUB(CURRENT_DATE(), INTERVAL 30 DAY)
    AND status = 'success'
  GROUP BY shop_id
),

refunds AS (
  SELECT 
    shop_id,
    SUM(amount) AS refund_amount,
    COUNT(*) AS refund_count
  FROM `shopify-dw.raw_shopify.payments_refunds`
  WHERE created_at >= DATE_SUB(CURRENT_DATE(), INTERVAL 30 DAY)
  GROUP BY shop_id
)

SELECT 
  t.shop_id,
  t.total_amount,
  t.transaction_count,
  r.refund_amount,
  r.refund_count,
  SAFE_DIVIDE(r.refund_amount, t.total_amount) AS refund_amount_ratio,
  SAFE_DIVIDE(r.refund_count, t.transaction_count) AS refund_count_ratio
FROM transactions t
LEFT JOIN refunds r ON t.shop_id = r.shop_id
WHERE t.transaction_count >= 50
ORDER BY refund_amount_ratio DESC
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

**Schema**:

| Field Name | Type | Description |
|------------|------|-------------|
| `id` | STRING | Unique identifier for the dispute |
| `shop_id` | INTEGER | Unique identifier for a shop |
| `order_id` | INTEGER | Order ID associated with this dispute |
| `amount` | FLOAT | Disputed amount |
| `currency` | STRING | Currency of the dispute |
| `reason` | STRING | Reason for the dispute |
| `status` | STRING | Current status of the dispute |
| `evidence_due_by` | TIMESTAMP | Deadline for evidence submission |
| `evidence_sent_on` | TIMESTAMP | When evidence was submitted |
| `finalized_on` | TIMESTAMP | When the dispute was finalized |
| `created_at` | TIMESTAMP | When the dispute was created |
| `updated_at` | TIMESTAMP | When the dispute was last updated |

**Usage Notes**:
- Provides more detailed dispute information than the chargebacks summary
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
WHERE created_at >= DATE_SUB(CURRENT_DATE(), INTERVAL 180 DAY)
GROUP BY reason
ORDER BY dispute_count DESC
```

### shopify-dw.finance.shop_gmv_weekly_summary_v1

**Description**:  
Provides weekly aggregated GMV and GPV data, useful for trend analysis at a less granular level than daily data.

**Update Frequency**: Weekly

**Primary Keys**: `shop_id`, `start_date`, `currency`

**Schema**:

| Field Name | Type | Description |
|------------|------|-------------|
| `shop_id` | INTEGER | Unique identifier for a shop |
| `start_date` | DATE | Start date of the week |
| `end_date` | DATE | End date of the week |
| `currency` | STRING | Currency of the financial metrics |
| `gmv` | FLOAT | Gross Merchandise Value for the week |
| `gpv` | FLOAT | Gross Payment Volume for the week |
| `order_count` | INTEGER | Number of orders for the week |
| `last_updated_at` | TIMESTAMP | When this record was last updated |

**Usage Notes**:
- Provides a weekly view of merchant activity
- Useful for identifying week-over-week trends
- Less granular than daily data, but good for longer-term analysis
- Always join with `currency` when comparing amounts

**Example Query**:
```sql
-- Analyze week-over-week growth trends
WITH weekly_growth AS (
  SELECT 
    shop_id,
    start_date,
    gpv,
    LAG(gpv) OVER(PARTITION BY shop_id, currency ORDER BY start_date) AS prev_week_gpv
  FROM `shopify-dw.finance.shop_gmv_weekly_summary_v1`
  WHERE start_date >= DATE_SUB(CURRENT_DATE(), INTERVAL 12 WEEK)
    AND currency = 'USD'
)

SELECT 
  shop_id,
  start_date,
  gpv,
  prev_week_gpv,
  (gpv - prev_week_gpv) / NULLIF(prev_week_gpv, 0) AS wow_growth
FROM weekly_growth
WHERE prev_week_gpv IS NOT NULL
  AND gpv > 1000
ORDER BY wow_growth DESC
LIMIT 100
```

### shopify-dw.money_products.shopify_payments_transfers

**Description**:  
Contains information about money transfers (payouts) to merchants, tracking when funds move from Shopify to merchant bank accounts.

**Update Frequency**: Daily

**Primary Keys**: `id`

**Foreign Keys**: `shop_id` references `shopify-dw.shopify.shops.id`

**Schema**:

| Field Name | Type | Description |
|------------|------|-------------|
| `id` | STRING | Unique identifier for the transfer |
| `shop_id` | INTEGER | Unique identifier for a shop |
| `amount` | FLOAT | Transfer amount |
| `currency` | STRING | Transfer currency |
| `type` | STRING | Type of transfer (payout, charge, etc.) |
| `status` | STRING | Status of the transfer |
| `created_at` | TIMESTAMP | When the transfer was created |
| `processed_at` | TIMESTAMP | When the transfer was processed |
| `failure_code` | STRING | Reason for failure (if applicable) |
| `arrival_date` | DATE | Expected arrival date |

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
    processed_at,
    LAG(processed_at) OVER(
      PARTITION BY shop_id
      ORDER BY processed_at
    ) AS prev_transfer_date
  FROM `shopify-dw.money_products.shopify_payments_transfers`
  WHERE type = 'payout'
    AND status = 'paid'
    AND created_at >= DATE_SUB(CURRENT_DATE(), INTERVAL 90 DAY)
)

SELECT 
  shop_id,
  COUNT(*) AS transfer_count,
  AVG(DATE_DIFF(processed_at, prev_transfer_date, DAY)) AS avg_days_between_payouts
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
  s.shop_name,
  s.country_code,
  s.created_at AS shop_created_at,
  g.currency,
  g.gpv_28d,
  DATE_DIFF(CURRENT_DATE(), DATE(s.created_at), DAY) AS shop_age_days
FROM `shopify-dw.finance.shop_gmv_current` g
JOIN `shopify-dw.shopify.shops` s ON g.shop_id = s.id
WHERE g.gpv_28d > 10000
  AND g.currency = 'USD'
ORDER BY g.gpv_28d DESC
```

### Joining Transaction Data with Refunds

```sql
-- Calculate refund rates for transactions
WITH transactions AS (
  SELECT 
    shop_id,
    order_id,
    transaction_id,
    amount,
    currency,
    created_at
  FROM `shopify-dw.money_products.order_transactions_payments_summary`
  WHERE created_at >= DATE_SUB(CURRENT_DATE(), INTERVAL 30 DAY)
    AND status = 'success'
),

refunds AS (
  SELECT 
    shop_id,
    order_id,
    amount AS refund_amount,
    created_at AS refund_date
  FROM `shopify-dw.raw_shopify.payments_refunds`
  WHERE created_at >= DATE_SUB(CURRENT_DATE(), INTERVAL 30 DAY)
)

SELECT 
  t.shop_id,
  COUNT(DISTINCT t.order_id) AS order_count,
  SUM(t.amount) AS total_amount,
  COUNT(DISTINCT r.order_id) AS refunded_order_count,
  SUM(r.refund_amount) AS total_refund_amount,
  COUNT(DISTINCT r.order_id) / COUNT(DISTINCT t.order_id) AS order_refund_rate,
  SUM(r.refund_amount) / SUM(t.amount) AS amount_refund_rate
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
  b.shop_id,
  b.date,
  b.balance,
  b.reserved_balance,
  r.reserve_type,
  r.percentage,
  r.period_in_days,
  b.reserved_balance / NULLIF(b.balance, 0) AS reserved_ratio
FROM `shopify-dw.money_products.shopify_payments_balance_account_daily_cumulative_summary` b
JOIN `sdp-prd-cti-data.base.base__shopify_payments_reserve_configurations` r
  ON b.shop_id = r.shop_id 
  AND b.currency = r.currency
WHERE b.date = DATE_SUB(CURRENT_DATE(), INTERVAL 1 DAY)
  AND r.is_active = TRUE
  AND b.balance > 0
ORDER BY reserved_ratio DESC
```

## Data Quality Considerations

1. **Currency Consistency**:
   - Always join financial tables using both `shop_id` and `currency`
   - Be careful when aggregating amounts across different currencies
   - Use currency conversion when necessary for cross-currency analysis

2. **Timing Considerations**:
   - Most financial tables have a `created_at` and/or `date` field
   - Align date ranges when joining multiple tables
   - Be aware of timezone differences in timestamp fields

3. **Balance Calculations**:
   - Balance tables show cumulative amounts at the end of each day
   - Reserve amounts are a subset of the total balance
   - Available balance = total balance - reserved balance - pending balance

4. **Volume Aggregation**:
   - GPV represents Shopify Payments volume only
   - GMV includes all payment methods (higher than GPV)
   - Different time windows (1d, 7d, 28d) are pre-calculated for convenience

5. **Data Freshness**:
   - Transaction data typically updates hourly
   - Balance and GMV/GPV data typically updates daily
   - Check `last_updated_at` fields for recency information

## Common Analysis Patterns

### Financial Health Assessment

```sql
-- Comprehensive financial health metrics
WITH volume_metrics AS (
  SELECT 
    shop_id,
    currency,
    gpv_28d,
    gpv_7d / NULLIF(gpv_28d, 0) AS recent_activity_ratio
  FROM `shopify-dw.finance.shop_gmv_current`
  WHERE currency = 'USD'
    AND gpv_28d > 0
),

balance_metrics AS (
  SELECT 
    shop_id,
    currency,
    balance,
    reserved_balance,
    available_balance,
    reserved_balance / NULLIF(balance, 0) AS reserve_ratio
  FROM `shopify-dw.money_products.shopify_payments_balance_account_daily_cumulative_summary`
  WHERE date = DATE_SUB(CURRENT_DATE(), INTERVAL 1 DAY)
    AND currency = 'USD'
),

reserve_configs AS (
  SELECT 
    shop_id,
    reserve_type,
    percentage,
    period_in_days
  FROM `sdp-prd-cti-data.base.base__shopify_payments_reserve_configurations`
  WHERE is_active = TRUE
    AND currency = 'USD'
)

SELECT 
  v.shop_id,
  v.gpv_28d,
  v.recent_activity_ratio,
  b.balance,
  b.available_balance,
  b.reserve_ratio,
  r.percentage AS reserve_percentage,
  r.period_in_days AS reserve_period,
  b.balance / NULLIF(v.gpv_28d, 0) AS balance_to_monthly_gpv_ratio
FROM volume_metrics v
LEFT JOIN balance_metrics b ON v.shop_id = b.shop_id AND v.currency = b.currency
LEFT JOIN reserve_configs r ON v.shop_id = r.shop_id
ORDER BY balance_to_monthly_gpv_ratio ASC
```

### Transaction Quality Analysis

```sql
-- Analyze transaction quality indicators
SELECT 
  shop_id,
  COUNT(*) AS transaction_count,
  
  -- Success rates
  COUNTIF(status = 'success') / COUNT(*) AS success_rate,
  COUNTIF(status = 'failure') / COUNT(*) AS failure_rate,
  
  -- AVS/CVV performance
  COUNTIF(avs_result_code = 'Y') / COUNTIF(avs_result_code IS NOT NULL) AS full_avs_match_rate,
  COUNTIF(cvv_result_code = 'M') / COUNTIF(cvv_result_code IS NOT NULL) AS cvv_match_rate,
  
  -- Risk indicators
  COUNTIF(risk_level = 'high') / COUNT(*) AS high_risk_rate,
  
  -- Amount distribution
  AVG(amount) AS avg_amount,
  APPROX_QUANTILES(amount, 100)[OFFSET(50)] AS median_amount,
  STDDEV(amount) AS amount_stddev
  
FROM `shopify-dw.money_products.order_transactions_payments_summary`
WHERE created_at >= DATE_SUB(CURRENT_DATE(), INTERVAL 30 DAY)
  AND payment_method = 'credit_card'
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
  FROM `shopify-dw.money_products.chargebacks_summary`
  WHERE created_at >= DATE_SUB(CURRENT_DATE(), INTERVAL 90 DAY)
    AND currency = 'USD'
  GROUP BY shop_id
),

gpv_data AS (
  SELECT 
    shop_id,
    gpv_28d
  FROM `shopify-dw.finance.shop_gmv_current`
  WHERE currency = 'USD'
    AND gpv_28d > 0
),

reserve_data AS (
  SELECT 
    r.shop_id,
    r.percentage,
    r.period_in_days,
    b.reserved_balance,
    g.gpv_28d * (r.percentage / 100) AS theoretical_reserve
  FROM `sdp-prd-cti-data.base.base__shopify_payments_reserve_configurations` r
  JOIN gpv_data g ON r.shop_id = g.shop_id
  LEFT JOIN `shopify-dw.money_products.shopify_payments_balance_account_daily_cumulative_summary` b
    ON r.shop_id = b.shop_id
    AND b.date = DATE_SUB(CURRENT_DATE(), INTERVAL 1 DAY)
    AND b.currency = 'USD'
  WHERE r.is_active = TRUE
    AND r.currency = 'USD'
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

---

*Last Updated: May 2024* 