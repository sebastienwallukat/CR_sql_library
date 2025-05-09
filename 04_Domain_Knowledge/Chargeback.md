# Chargeback Analysis

This guide provides essential queries and techniques for analyzing chargebacks in Shopify's data platform. 

## Key Chargeback Tables

| Table Name | Description | Temporal Field | Data Type |
|------------|-------------|----------------|-----------|
| `shopify-dw.money_products.chargebacks_summary` | Comprehensive chargebacks data | `provider_chargeback_created_at` | TIMESTAMP |
| `sdp-prd-cti-data.intermediate.shop_chargeback_rates_daily_snapshot` | Daily snapshots of chargeback rates | `date` | DATE |
| `sdp-prd-cti-data.intermediate.shop_chargeback_rates_current` | Current chargeback rates | N/A (snapshot) | N/A |

## Basic Chargeback Queries

### Recent Chargebacks for a Specific Shop

```sql
SELECT
  chargeback_id,
  provider_chargeback_created_at,
  reason,
  status,
  chargeback_amount_usd,
  order_id
FROM `shopify-dw.money_products.chargebacks_summary`
WHERE shop_id = 12345678
  AND provider_chargeback_created_at >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 90 DAY)  -- TIMESTAMP field
ORDER BY provider_chargeback_created_at DESC
```

### Chargeback Rate Trends

```sql

SELECT
  date,
  chargeback_rate_90d,
  sp_transaction_count_90d,
  chargeback_count_90d
FROM `sdp-prd-cti-data.intermediate.shop_chargeback_rates_daily_snapshot`
WHERE shop_id = 12345678
  AND date >= DATE_SUB(CURRENT_DATE(), INTERVAL 180 DAY)  -- DATE field
ORDER BY date DESC
```

### Shops with Highest Chargeback Rates

```sql

SELECT
  shop_id,
  chargeback_rate_90d,
  chargeback_count_90d,
  sp_transaction_count_90d,
  sp_transaction_amount_usd_90d
FROM `sdp-prd-cti-data.intermediate.shop_chargeback_rates_current`
WHERE sp_transaction_amount_usd_90d > 10000  -- Only include shops with at least $10k in transactions
  AND chargeback_count_90d >= 5  -- Only include shops with at least 5 chargebacks
ORDER BY chargeback_rate_90d DESC
LIMIT 100
```

## Advanced Chargeback Analysis

### Chargeback Reasons Analysis

```sql
WITH transaction_totals AS (
  SELECT
    shop_id AS merchant_id,
    COUNT(*) AS total_transactions,
    SUM(amount_usd) AS total_amount_usd
  FROM
    `shopify-dw.money_products.payment_transactions`
  WHERE
    status = 'SUCCESS'
    AND transaction_date >= DATE_SUB(CURRENT_DATE(), INTERVAL 90 DAY)
  GROUP BY
    shop_id
),

SELECT
  reason,
  COUNT(*) AS chargeback_count,
  SUM(chargeback_amount_usd) AS total_amount_usd,
  AVG(chargeback_amount_usd) AS avg_amount_usd,
  COUNT(DISTINCT shop_id) AS shop_count
FROM `shopify-dw.money_products.chargebacks_summary`
WHERE provider_chargeback_created_at >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 180 DAY)  -- TIMESTAMP field
GROUP BY reason
ORDER BY chargeback_count DESC
```

### Win Rate Analysis by Reason

```sql
SELECT
  reason,
  COUNT(*) AS total_chargebacks,
  SUM(CASE WHEN status = 'won' THEN 1 ELSE 0 END) AS won_count,
  ROUND(SUM(CASE WHEN status = 'won' THEN 1 ELSE 0 END) / COUNT(*) * 100, 2) AS win_rate_percentage,
  SUM(chargeback_amount_usd) AS total_disputed_usd
FROM `shopify-dw.money_products.chargebacks_summary`
WHERE provider_chargeback_created_at >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 180 DAY)  -- TIMESTAMP field
  AND status IN ('won', 'lost')  -- Only include resolved chargebacks
GROUP BY reason
HAVING total_chargebacks > 10  -- Only include reasons with sufficient data
ORDER BY win_rate_percentage DESC
```

### Chargeback Time-to-Resolution Analysis

```sql
SELECT
  EXTRACT(MONTH FROM provider_chargeback_created_at) AS chargeback_month,
  EXTRACT(YEAR FROM provider_chargeback_created_at) AS chargeback_year,
  COUNT(*) AS chargeback_count,
  AVG(TIMESTAMP_DIFF(finalized_at, provider_chargeback_created_at, DAY)) AS avg_days_to_resolution,
  APPROX_QUANTILES(TIMESTAMP_DIFF(finalized_at, provider_chargeback_created_at, DAY), 100)[OFFSET(50)] AS median_days_to_resolution
FROM `shopify-dw.money_products.chargebacks_summary`
WHERE provider_chargeback_created_at >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 365 DAY)  -- TIMESTAMP field
  AND finalized_at IS NOT NULL
GROUP BY chargeback_month, chargeback_year
ORDER BY chargeback_year DESC, chargeback_month DESC
```

## Combined Analysis with Other Data Sources

### Chargeback and Shop Payments Status

```sql
SELECT
  spps.shopify_payments_status,
  COUNT(DISTINCT cb.shop_id) AS shop_count,
  AVG(cb.chargeback_rate_90d) AS avg_chargeback_rate_90d,
  APPROX_QUANTILES(cb.chargeback_rate_90d, 100)[OFFSET(50)] AS median_chargeback_rate_90d,
  APPROX_QUANTILES(cb.chargeback_rate_90d, 100)[OFFSET(75)] AS p75_chargeback_rate_90d,
  APPROX_QUANTILES(cb.chargeback_rate_90d, 100)[OFFSET(90)] AS p90_chargeback_rate_90d,
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
SELECT
  cb.shop_id,
  cb.chargeback_rate_90d,
  cb.chargeback_count_90d,
  cb.sp_transaction_amount_usd_90d,
  COUNT(DISTINCT tp.trust_platform_ticket_id) AS ticket_count,
  STRING_AGG(DISTINCT tp.rule_group, ', ') AS rule_groups
FROM `sdp-prd-cti-data.intermediate.shop_chargeback_rates_current` cb
JOIN `shopify-dw.risk.trust_platform_tickets_summary_v1` tp
  ON cb.shop_id = tp.subjectable_id
WHERE cb.chargeback_rate_90d > 0.01  -- 1%+ chargeback rate
  AND cb.sp_transaction_amount_usd_90d > 10000  -- $10k+ in transactions
  AND tp.created_at >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 90 DAY)  -- TIMESTAMP field
  AND tp.subjectable_type = 'Shop'
GROUP BY cb.shop_id, cb.chargeback_rate_90d, cb.chargeback_count_90d, cb.sp_transaction_amount_usd_90d
ORDER BY cb.chargeback_rate_90d DESC
```

## Important Notes

### DATE vs TIMESTAMP Fields

- `provider_chargeback_created_at` in `shopify-dw.money_products.chargebacks_summary` is a **TIMESTAMP** field
  - Use `TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL X DAY)` for filtering
- `date` in `sdp-prd-cti-data.intermediate.shop_chargeback_rates_daily_snapshot` is a **DATE** field
  - Use `DATE_SUB(CURRENT_DATE(), INTERVAL X DAY)` for filtering

### Performance Tips

1. Always filter on partitioned fields:
   - Filter on `provider_chargeback_created_at` for `chargebacks_summary`
   - Filter on `date` for `shop_chargeback_rates_daily_snapshot`

2. Use clustering columns in your filters when possible:
   - Filter on `shop_id` for better performance in both tables

For more information on handling temporal fields, see the [Date vs Timestamp Guide](../02_SQL_Guide/Date_vs_Timestamp_Guide.md).

---

## References
- [Dispute Resolution Guide](https://shopify.dev/docs)
- [Chargeback Reason Codes](https://shopify.dev/api)
- [Risk Management Documentation](https://shopify.dev/docs)