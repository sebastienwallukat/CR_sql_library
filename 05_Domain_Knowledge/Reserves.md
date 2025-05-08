# Reserve Management Queries 🔒

Welcome to the Reserves section of our SQL Query Resource Center. This page contains datasets and queries for analyzing merchant reserve configurations and impact.

## Primary Datasets 📁

### `sdp-prd-cti-data.base.base__shopify_payments_reserve_configurations`

This is the primary recommended dataset for all reserve analysis. It contains comprehensive information about merchant reserve configurations, including rates, rolling windows, and effective dates.

## Common Queries 💻

### Current Reserve Configurations by Merchant

```sql
SELECT
  merchant_id,
  reserve_rate,
  reserve_type,
  rolling_window_days,
  minimum_reserve_amount_usd,
  effective_from_date,
  effective_to_date,
  last_updated_at
FROM
  `sdp-prd-cti-data.base.base__shopify_payments_reserve_configurations`
WHERE
  CURRENT_DATE() BETWEEN effective_from_date AND COALESCE(effective_to_date, '9999-12-31')
ORDER BY
  reserve_rate DESC
```

### Reserve Configuration Changes Over Time

```sql
WITH reserve_changes AS (
  SELECT
    merchant_id,
    effective_from_date,
    effective_to_date,
    reserve_rate,
    reserve_type,
    LAG(reserve_rate) OVER (PARTITION BY merchant_id ORDER BY effective_from_date) AS previous_reserve_rate,
    LAG(reserve_type) OVER (PARTITION BY merchant_id ORDER BY effective_from_date) AS previous_reserve_type
  FROM
    `sdp-prd-cti-data.base.base__shopify_payments_reserve_configurations`
)

SELECT
  merchant_id,
  effective_from_date,
  previous_reserve_rate,
  reserve_rate,
  reserve_rate - COALESCE(previous_reserve_rate, 0) AS reserve_rate_change,
  previous_reserve_type,
  reserve_type
FROM
  reserve_changes
WHERE
  reserve_rate != COALESCE(previous_reserve_rate, -1)
  OR reserve_type != COALESCE(previous_reserve_type, '')
ORDER BY
  effective_from_date DESC
```

### Reserve Rate Distribution Analysis

```sql
SELECT
  CASE
    WHEN reserve_rate = 0 THEN '0% (No Reserve)'
    WHEN reserve_rate BETWEEN 0.01 AND 0.05 THEN '1-5%'
    WHEN reserve_rate BETWEEN 0.06 AND 0.10 THEN '6-10%'
    WHEN reserve_rate BETWEEN 0.11 AND 0.15 THEN '11-15%'
    WHEN reserve_rate BETWEEN 0.16 AND 0.20 THEN '16-20%'
    WHEN reserve_rate BETWEEN 0.21 AND 0.25 THEN '21-25%'
    ELSE '26% or higher'
  END AS reserve_rate_bucket,
  COUNT(*) AS merchant_count,
  COUNT(*) / SUM(COUNT(*)) OVER () AS percentage
FROM
  `sdp-prd-cti-data.base.base__shopify_payments_reserve_configurations`
WHERE
  CURRENT_DATE() BETWEEN effective_from_date AND COALESCE(effective_to_date, '9999-12-31')
GROUP BY
  reserve_rate_bucket
ORDER BY
  MIN(reserve_rate)
```

### Correlating Reserves with Risk Indicators

```sql
SELECT
  r.merchant_id,
  r.reserve_rate,
  r.reserve_type,
  r.rolling_window_days,
  rm.risk_score,
  cb.chargeback_rate,
  gpv.monthly_gpv_usd
FROM
  `sdp-prd-cti-data.base.base__shopify_payments_reserve_configurations` r
LEFT JOIN (
  -- Join to risk metrics
  SELECT
    merchant_id,
    risk_score,
    snapshot_date
  FROM
    `sdp-prd-cti-data.merchant.risk_metrics`
  WHERE
    snapshot_date = DATE_SUB(CURRENT_DATE(), INTERVAL 1 DAY)
) rm ON r.merchant_id = rm.merchant_id
LEFT JOIN (
  -- Join to chargeback rates
  SELECT
    merchant_id,
    chargeback_count / NULLIF(transaction_count, 0) AS chargeback_rate
  FROM
    `sdp-prd-cti-data.merchant.chargeback_metrics`
  WHERE
    metric_date = DATE_SUB(CURRENT_DATE(), INTERVAL 1 DAY)
) cb ON r.merchant_id = cb.merchant_id
LEFT JOIN (
  -- Join to GPV
  SELECT
    merchant_id,
    SUM(amount_usd) AS monthly_gpv_usd
  FROM
    `sdp-prd-cti-data.finance.transactions`
  WHERE
    transaction_date >= DATE_SUB(CURRENT_DATE(), INTERVAL 30 DAY)
  GROUP BY
    merchant_id
) gpv ON r.merchant_id = gpv.merchant_id
WHERE
  CURRENT_DATE() BETWEEN r.effective_from_date AND COALESCE(r.effective_to_date, '9999-12-31')
ORDER BY
  r.reserve_rate DESC
```

## Reserve Impact Analysis

```sql
WITH merchant_before_after AS (
  SELECT
    m.merchant_id,
    r.effective_from_date AS reserve_start_date,
    r.reserve_rate,
    r.reserve_type,
    
    -- Pre-reserve metrics (30 days before)
    (SELECT AVG(daily_gpv_usd)
     FROM `sdp-prd-cti-data.finance.merchant_daily_gpv`
     WHERE merchant_id = m.merchant_id
       AND gpv_date BETWEEN DATE_SUB(r.effective_from_date, INTERVAL 30 DAY) 
                        AND DATE_SUB(r.effective_from_date, INTERVAL 1 DAY)
    ) AS avg_daily_gpv_before,
    
    -- Post-reserve metrics (30 days after)
    (SELECT AVG(daily_gpv_usd)
     FROM `sdp-prd-cti-data.finance.merchant_daily_gpv`
     WHERE merchant_id = m.merchant_id
       AND gpv_date BETWEEN r.effective_from_date
                        AND DATE_ADD(r.effective_from_date, INTERVAL 30 DAY)
    ) AS avg_daily_gpv_after
  FROM
    `sdp-prd-cti-data.base.base__shopify_payments_reserve_configurations` r
  JOIN
    `sdp-prd-cti-data.merchant.merchant_details` m
    ON r.merchant_id = m.merchant_id
  WHERE
    r.effective_from_date >= DATE_SUB(CURRENT_DATE(), INTERVAL 180 DAY)
    AND r.effective_from_date < DATE_SUB(CURRENT_DATE(), INTERVAL 30 DAY)  -- Ensure we have 30 days of post-data
    AND r.reserve_rate > 0  -- Only include merchants with actual reserves
)

SELECT
  merchant_id,
  reserve_start_date,
  reserve_rate,
  reserve_type,
  avg_daily_gpv_before,
  avg_daily_gpv_after,
  avg_daily_gpv_after - avg_daily_gpv_before AS absolute_gpv_change,
  SAFE_DIVIDE(avg_daily_gpv_after - avg_daily_gpv_before, avg_daily_gpv_before) AS percent_gpv_change
FROM
  merchant_before_after
WHERE
  avg_daily_gpv_before > 0  -- Avoid division by zero
ORDER BY
  percent_gpv_change
```

## Notes and Best Practices 📝

- Always use `sdp-prd-cti-data.base.base__shopify_payments_reserve_configurations` as the primary source for reserve data
- When analyzing reserve impact, consider a sufficient time window before and after changes
- Account for seasonality when measuring reserve impact on processing volumes
- Correlate reserve changes with risk indicators to assess effectiveness
- Consider both the reserve rate and minimum amount when analyzing merchant impact
- When setting up new reserves, use historical data to model potential impact on merchant cash flow

Feel free to contribute new queries or improve the existing ones. Together, we can build a comprehensive resource for reserve analysis and strategy optimization.

Happy reserve analyzing! 🔍 