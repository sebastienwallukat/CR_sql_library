# Chargeback Analysis

This guide provides essential queries and techniques for analyzing chargebacks in Shopify's data platform.

## Table of Contents
- [Key Chargeback Tables](#key-chargeback-tables)
- [Basic Chargeback Queries](#basic-chargeback-queries)
  - [Recent Chargebacks for a Specific Shop](#recent-chargebacks-for-a-specific-shop)
  - [Chargeback Rate Trends](#chargeback-rate-trends)
  - [Shops with Highest Chargeback Rates](#shops-with-highest-chargeback-rates)
- [Advanced Chargeback Analysis](#advanced-chargeback-analysis)
  - [Chargeback Reasons Analysis](#chargeback-reasons-analysis)
  - [Win Rate Analysis by Reason](#win-rate-analysis-by-reason)
  - [Chargeback Time-to-Resolution Analysis](#chargeback-time-to-resolution-analysis)
- [Combined Analysis with Other Data Sources](#combined-analysis-with-other-data-sources)
  - [Chargeback and Shop Payments Status](#chargeback-and-shop-payments-status)
  - [Chargebacks and Trust Platform Tickets](#chargebacks-and-trust-platform-tickets)
- [Important Notes](#important-notes)
  - [DATE vs TIMESTAMP Fields](#date-vs-timestamp-fields)
  - [Performance Tips](#performance-tips)
- [References](#references)

## Key Chargeback Tables

| Table Name | Description | Temporal Field | Data Type |
|------------|-------------|----------------|-----------|
| `shopify-dw.money_products.chargebacks_summary` | Comprehensive chargebacks data | `provider_chargeback_created_at` | TIMESTAMP |
| `sdp-prd-cti-data.intermediate.shop_chargeback_rates_daily_snapshot` | Daily snapshots of chargeback rates | `date` | DATE |
| `sdp-prd-cti-data.intermediate.shop_chargeback_rates_current` | Current chargeback rates | N/A (snapshot) | N/A |

## Basic Chargeback Queries

### Recent Chargebacks for a Specific Shop

```sql
-- Purpose: Retrieve recent chargebacks for a specific shop with key details
-- This query returns key chargeback details for a given shop over the last 90 days

SELECT
  chargeback_id,                          -- Unique identifier for the chargeback
  provider_chargeback_created_at,         -- Timestamp when the chargeback was created at the provider
  reason,                                 -- Reason code provided for the chargeback
  status,                                 -- Current status of the chargeback (e.g., 'won', 'lost', 'pending')
  chargeback_amount_usd,                  -- Amount of the chargeback in USD
  order_id                                -- Associated order identifier
FROM `shopify-dw.money_products.chargebacks_summary`
WHERE shop_id = 12345678                  -- Replace with actual shop ID
  AND provider_chargeback_created_at >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 90 DAY)  -- Filter for recent chargebacks
ORDER BY provider_chargeback_created_at DESC
```

### Chargeback Rate Trends

```sql
-- Purpose: Analyze chargeback rate trends over time for a specific shop
-- Shows 90-day rolling chargeback rate and relevant metrics

SELECT
  date,                                  -- Date of the snapshot
  chargeback_rate_90d,                   -- Chargeback rate over the prior 90 days
  sp_transaction_count_90d,              -- Count of Shopify Payments transactions in prior 90 days
  chargeback_count_90d                   -- Count of chargebacks in prior 90 days
FROM `sdp-prd-cti-data.intermediate.shop_chargeback_rates_daily_snapshot`
WHERE shop_id = 12345678                 -- Replace with actual shop ID
  AND date >= DATE_SUB(CURRENT_DATE(), INTERVAL 180 DAY)  -- Last 180 days of data
  AND date <= CURRENT_DATE()             -- Additional partition filter for best performance
ORDER BY date DESC
```

### Shops with Highest Chargeback Rates

```sql
-- Purpose: Identify shops with the highest chargeback rates
-- Filters for significant transaction volume and at least 5 chargebacks

SELECT
  shop_id,
  chargeback_rate_90d,                   -- Chargeback rate over the prior 90 days
  chargeback_count_90d,                  -- Count of chargebacks in prior 90 days
  sp_transaction_count_90d,              -- Count of Shopify Payments transactions in prior 90 days
  sp_transaction_amount_usd_90d          -- USD amount of Shopify Payments transactions in prior 90 days
FROM `sdp-prd-cti-data.intermediate.shop_chargeback_rates_current`
WHERE sp_transaction_amount_usd_90d > 10000  -- Only include shops with at least $10k in transactions
  AND chargeback_count_90d >= 5             -- Only include shops with at least 5 chargebacks
ORDER BY chargeback_rate_90d DESC
LIMIT 100
```

## Advanced Chargeback Analysis

### Chargeback Reasons Analysis

```sql
-- Purpose: Analyze distribution of chargeback reasons
-- Aggregates chargebacks by reason code to identify most common reasons

SELECT
  reason,                                -- The reason code for the chargeback
  COUNT(*) AS chargeback_count,          -- Total number of chargebacks with this reason
  SUM(chargeback_amount_usd) AS total_amount_usd,  -- Total USD value of chargebacks
  AVG(chargeback_amount_usd) AS avg_amount_usd,    -- Average USD value per chargeback
  COUNT(DISTINCT shop_id) AS shop_count  -- Number of unique shops affected
FROM `shopify-dw.money_products.chargebacks_summary`
WHERE provider_chargeback_created_at >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 180 DAY)
GROUP BY reason
ORDER BY chargeback_count DESC
```

### Win Rate Analysis by Reason

```sql
-- Purpose: Analyze chargeback win rates by reason code
-- Shows which chargeback reasons have the highest win rates

SELECT
  reason,                               -- The reason code for the chargeback
  COUNT(*) AS total_chargebacks,        -- Total chargebacks with this reason
  SUM(CASE WHEN status = 'won' THEN 1 ELSE 0 END) AS won_count,  -- Count of won chargebacks
  ROUND(SUM(CASE WHEN status = 'won' THEN 1 ELSE 0 END) / COUNT(*) * 100, 2) AS win_rate_percentage,  -- Win rate as percentage
  SUM(chargeback_amount_usd) AS total_disputed_usd  -- Total USD value disputed
FROM `shopify-dw.money_products.chargebacks_summary`
WHERE provider_chargeback_created_at >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 180 DAY)  
  AND status IN ('won', 'lost')  -- Only include resolved chargebacks
GROUP BY reason
HAVING total_chargebacks > 10    -- Only include reasons with sufficient data
ORDER BY win_rate_percentage DESC
```

### Chargeback Time-to-Resolution Analysis

```sql
-- Purpose: Analyze time-to-resolution trends for chargebacks
-- Shows how quickly chargebacks are being resolved over time

SELECT
  EXTRACT(MONTH FROM provider_chargeback_created_at) AS chargeback_month,
  EXTRACT(YEAR FROM provider_chargeback_created_at) AS chargeback_year,
  COUNT(*) AS chargeback_count,
  -- Average days from creation to resolution
  AVG(TIMESTAMP_DIFF(finalized_at, provider_chargeback_created_at, DAY)) AS avg_days_to_resolution,
  -- Median days from creation to resolution
  APPROX_QUANTILES(TIMESTAMP_DIFF(finalized_at, provider_chargeback_created_at, DAY), 100)[OFFSET(50)] AS median_days_to_resolution
FROM `shopify-dw.money_products.chargebacks_summary`
WHERE provider_chargeback_created_at >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 365 DAY)
  AND finalized_at IS NOT NULL  -- Only include chargebacks that have been finalized
GROUP BY chargeback_month, chargeback_year
ORDER BY chargeback_year DESC, chargeback_month DESC
```

## Combined Analysis with Other Data Sources

### Chargeback and Shop Payments Status

```sql
-- Purpose: Analyze chargeback rates across different Shopify Payments statuses
-- Shows how chargeback metrics vary by payments status

SELECT
  spps.shopify_payments_status,          -- Current status of shop's Shopify Payments account
  COUNT(DISTINCT cb.shop_id) AS shop_count,  -- Number of shops with this status
  -- Average chargeback rate metrics
  AVG(cb.chargeback_rate_90d) AS avg_chargeback_rate_90d,
  APPROX_QUANTILES(cb.chargeback_rate_90d, 100)[OFFSET(50)] AS median_chargeback_rate_90d,
  APPROX_QUANTILES(cb.chargeback_rate_90d, 100)[OFFSET(75)] AS p75_chargeback_rate_90d,
  APPROX_QUANTILES(cb.chargeback_rate_90d, 100)[OFFSET(90)] AS p90_chargeback_rate_90d,
  -- Total transaction volume across all shops with this status
  SUM(cb.sp_transaction_amount_usd_90d) AS total_transaction_amount_usd_90d
FROM `sdp-prd-cti-data.intermediate.shop_chargeback_rates_current` cb
JOIN `sdp-prd-cti-data.intermediate.shop_current_shopify_payments_status` spps
  ON cb.shop_id = spps.shop_id
WHERE cb.sp_transaction_amount_usd_90d > 1000  -- Only include shops with significant volume
GROUP BY spps.shopify_payments_status
ORDER BY avg_chargeback_rate_90d DESC
```

### Chargebacks and Trust Platform Tickets

```sql
-- Purpose: Identify high-chargeback shops with Trust Platform tickets
-- Shows correlation between chargebacks and risk indicators

SELECT
  cb.shop_id,
  cb.chargeback_rate_90d,
  cb.chargeback_count_90d,
  cb.sp_transaction_amount_usd_90d,
  -- Trust Platform ticket metrics
  COUNT(DISTINCT tp.trust_platform_ticket_id) AS ticket_count,
  STRING_AGG(DISTINCT tp.rule_group, ', ') AS rule_groups
FROM `sdp-prd-cti-data.intermediate.shop_chargeback_rates_current` cb
JOIN `shopify-dw.risk.trust_platform_tickets_summary_v1` tp
  ON cb.shop_id = tp.subjectable_id
WHERE cb.chargeback_rate_90d > 0.01  -- 1%+ chargeback rate
  AND cb.sp_transaction_amount_usd_90d > 10000  -- $10k+ in transactions
  AND tp.created_at >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 90 DAY)
  AND tp.subjectable_type = 'Shop'  -- Only include shop-level tickets
GROUP BY cb.shop_id, cb.chargeback_rate_90d, cb.chargeback_count_90d, cb.sp_transaction_amount_usd_90d
ORDER BY cb.chargeback_rate_90d DESC
```

## Important Notes

### DATE vs TIMESTAMP Fields

- `provider_chargeback_created_at` in `shopify-dw.money_products.chargebacks_summary` is a **TIMESTAMP** field
  - Use `TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL X DAY)` for filtering
- `date` in `sdp-prd-cti-data.intermediate.shop_chargeback_rates_daily_snapshot` is a **DATE** field
  - Use `DATE_SUB(CURRENT_DATE(), INTERVAL X DAY)` for filtering
  - Always include both lower and upper bounds (e.g., `date >= X AND date <= CURRENT_DATE()`)

### Performance Tips

1. Always filter on partitioned fields:
   - Filter on `provider_chargeback_created_at` for `chargebacks_summary`
   - Filter on `date` with both lower and upper bounds for `shop_chargeback_rates_daily_snapshot`

2. Use clustering columns in your filters when possible:
   - Filter on `shop_id` for better performance in both tables

For more information on handling temporal fields, see the [Date vs Timestamp Guide](../02_SQL_Guide/Date_vs_Timestamp_Guide.md).

## References
- [Dispute Resolution Guide](https://shopify.dev/docs)
- [Chargeback Reason Codes](https://shopify.dev/api)
- [Risk Management Documentation](https://shopify.dev/docs)