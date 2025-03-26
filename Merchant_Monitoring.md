# Merchant Monitoring Queries 👁️

Welcome to the Merchant Monitoring section of our SQL Query Resource Center. This page contains datasets and queries for tracking merchant health and activity.

## Datasets 📁

### `sdp-prd-cti-data.merchant.shop_risk_indicators`

This dataset provides daily snapshots of key risk indicators for each merchant.

🔗 [Access on Dataplex](https://console.cloud.google.com/dataplex/projects/sdp-prd-cti-data/locations/us/entryGroups/@bigquery/entries/path_to_dataset)

### `sdp-prd-cti-data.merchant.shop_activity_events`

This dataset contains significant merchant activity events that may indicate changes in risk profile.

🔗 [Access on Dataplex](https://console.cloud.google.com/dataplex/projects/sdp-prd-cti-data/locations/us/entryGroups/@bigquery/entries/path_to_dataset)

## Common Queries 💻

### High-Risk Merchant Detection

```sql
SELECT
  shop_id,
  risk_score,
  chargeback_rate_30d,
  fraud_rate_30d,
  customer_complaint_count_30d,
  avg_transaction_amount,
  total_processing_volume_30d,
  industry_category,
  country_code,
  days_since_first_transaction
FROM
  `sdp-prd-cti-data.merchant.shop_risk_indicators`
WHERE
  snapshot_date = CURRENT_DATE()
  AND (
    risk_score > 0.8
    OR chargeback_rate_30d > 0.01
    OR fraud_rate_30d > 0.005
    OR customer_complaint_count_30d > 10
  )
ORDER BY
  risk_score DESC
```

### Merchant Activity Pattern Changes

```sql
WITH daily_metrics AS (
  SELECT
    shop_id,
    snapshot_date,
    avg_transaction_amount,
    transaction_count_daily,
    LAG(avg_transaction_amount) OVER (PARTITION BY shop_id ORDER BY snapshot_date) as prev_avg_amount,
    LAG(transaction_count_daily) OVER (PARTITION BY shop_id ORDER BY snapshot_date) as prev_count
  FROM
    `sdp-prd-cti-data.merchant.shop_risk_indicators`
  WHERE
    snapshot_date >= DATE_SUB(CURRENT_DATE(), INTERVAL 30 DAY)
),

changes AS (
  SELECT
    shop_id,
    snapshot_date,
    avg_transaction_amount,
    transaction_count_daily,
    prev_avg_amount,
    prev_count,
    SAFE_DIVIDE(avg_transaction_amount - prev_avg_amount, prev_avg_amount) as amount_pct_change,
    SAFE_DIVIDE(transaction_count_daily - prev_count, prev_count) as count_pct_change
  FROM
    daily_metrics
  WHERE
    prev_avg_amount IS NOT NULL
    AND prev_count IS NOT NULL
)

SELECT
  shop_id,
  snapshot_date,
  avg_transaction_amount,
  transaction_count_daily,
  amount_pct_change,
  count_pct_change
FROM
  changes
WHERE
  ABS(amount_pct_change) > 0.5
  OR ABS(count_pct_change) > 0.5
ORDER BY
  snapshot_date DESC,
  ABS(amount_pct_change) + ABS(count_pct_change) DESC
```

### Significant Merchant Events

```sql
SELECT
  shop_id,
  event_type,
  event_timestamp,
  event_details,
  risk_impact_score
FROM
  `sdp-prd-cti-data.merchant.shop_activity_events`
WHERE
  event_timestamp >= DATE_SUB(CURRENT_DATE(), INTERVAL 7 DAY)
  AND risk_impact_score > 0.5
ORDER BY
  event_timestamp DESC
```

### Merchant Industry Risk Comparison

```sql
WITH industry_stats AS (
  SELECT
    industry_category,
    COUNT(DISTINCT shop_id) as shop_count,
    AVG(risk_score) as avg_risk_score,
    AVG(chargeback_rate_30d) as avg_chargeback_rate,
    AVG(fraud_rate_30d) as avg_fraud_rate,
    APPROX_QUANTILES(risk_score, 100)[OFFSET(75)] as p75_risk_score,
    APPROX_QUANTILES(chargeback_rate_30d, 100)[OFFSET(75)] as p75_chargeback_rate,
    APPROX_QUANTILES(fraud_rate_30d, 100)[OFFSET(75)] as p75_fraud_rate
  FROM
    `sdp-prd-cti-data.merchant.shop_risk_indicators`
  WHERE
    snapshot_date = CURRENT_DATE()
  GROUP BY
    industry_category
)

SELECT
  s.shop_id,
  s.industry_category,
  s.risk_score,
  i.avg_risk_score as industry_avg_risk_score,
  s.risk_score - i.avg_risk_score as risk_score_delta,
  s.chargeback_rate_30d,
  i.avg_chargeback_rate as industry_avg_chargeback_rate,
  s.chargeback_rate_30d - i.avg_chargeback_rate as chargeback_rate_delta,
  s.fraud_rate_30d,
  i.avg_fraud_rate as industry_avg_fraud_rate,
  s.fraud_rate_30d - i.avg_fraud_rate as fraud_rate_delta
FROM
  `sdp-prd-cti-data.merchant.shop_risk_indicators` s
  JOIN industry_stats i ON s.industry_category = i.industry_category
WHERE
  s.snapshot_date = CURRENT_DATE()
  AND (
    s.risk_score > i.p75_risk_score
    OR s.chargeback_rate_30d > i.p75_chargeback_rate
    OR s.fraud_rate_30d > i.p75_fraud_rate
  )
ORDER BY
  (s.risk_score - i.avg_risk_score) +
  (s.chargeback_rate_30d - i.avg_chargeback_rate) * 100 +
  (s.fraud_rate_30d - i.avg_fraud_rate) * 100 DESC
```

## Event Types Reference 📋

Common merchant activity event types include:

1. **Business Model Change** - Significant change in product offerings or business model
2. **Velocity Spike** - Unusual increase in transaction volume or amount
3. **Product Category Shift** - Change in product categories being sold
4. **Geographical Expansion** - New customer locations or shipping destinations
5. **Customer Demographics Shift** - Change in customer age, spending patterns, etc.

## Notes and Best Practices 📝

- Combine multiple risk indicators for more accurate merchant risk assessment
- Consider both absolute values and changes over time when evaluating risk
- Compare merchants against appropriate benchmarks (industry, size, region)
- Set up alerts for significant pattern changes to enable early intervention

Feel free to contribute new queries or improve the existing ones. Together, we can build a comprehensive resource for merchant monitoring.

Happy merchant monitoring! 🔍 