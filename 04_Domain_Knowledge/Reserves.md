# Reserve Management Queries 🔒

Welcome to the Reserves section of our SQL Query Resource Center. This page contains datasets and queries for analyzing merchant reserve configurations and impact.

## Primary Datasets 📁

### `shopify-dw.money_products.shopify_payments_reserve_configurations_v1`

This is the primary recommended dataset for all reserve analysis. It combines data from Shopify Payments reserve configurations with their associated fixed and rolling reserves. It provides a unified view of all reserve types (fixed and rolling) that are mostly placed on merchant accounts via the Trust Platform.

#### Temporal Fields

| Field Name | Data Type | Comparison Function | Example |
|------------|-----------|---------------------|---------|
| created_at | TIMESTAMP | TIMESTAMP_SUB() | `WHERE created_at >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 7 DAY)` |
| expires_at | TIMESTAMP | TIMESTAMP_SUB() | `WHERE expires_at >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 7 DAY)` |
| cancelled_at | TIMESTAMP | TIMESTAMP_SUB() | `WHERE cancelled_at >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 7 DAY)` |

## Common Queries 💻

### Current Reserve Configurations by Merchant

```sql
SELECT
  shop_id as merchant_id,
  rolling_reserve_percentage as reserve_rate,
  CASE
    WHEN fixed_reserve_status = 'active' AND rolling_reserve_status = 'active' THEN 'Both'
    WHEN fixed_reserve_status = 'active' THEN 'Fixed'
    WHEN rolling_reserve_status = 'active' THEN 'Rolling'
    ELSE 'Unknown'
  END as reserve_type,
  hold_duration as rolling_window_days,
  fixed_reserve_amount as minimum_reserve_amount_usd,
  created_at as effective_from_date,
  expires_at as effective_to_date,
  updated_at as last_updated_at
FROM
  `shopify-dw.money_products.shopify_payments_reserve_configurations_v1`
WHERE
  CURRENT_DATE() BETWEEN DATE(created_at) AND COALESCE(DATE(expires_at), DATE('9999-12-31'))
ORDER BY
  rolling_reserve_percentage DESC
```

### Reserve Configuration Changes Over Time

```sql
WITH reserve_changes AS (
  SELECT
    shop_id as merchant_id,
    created_at as effective_from_date,
    expires_at as effective_to_date,
    rolling_reserve_percentage as reserve_rate,
    CASE
      WHEN fixed_reserve_status = 'active' AND rolling_reserve_status = 'active' THEN 'Both'
      WHEN fixed_reserve_status = 'active' THEN 'Fixed'
      WHEN rolling_reserve_status = 'active' THEN 'Rolling'
      ELSE 'Unknown'
    END as reserve_type,
    LAG(rolling_reserve_percentage) OVER (PARTITION BY shop_id ORDER BY created_at) AS previous_reserve_rate,
    LAG(CASE
      WHEN fixed_reserve_status = 'active' AND rolling_reserve_status = 'active' THEN 'Both'
      WHEN fixed_reserve_status = 'active' THEN 'Fixed'
      WHEN rolling_reserve_status = 'active' THEN 'Rolling'
      ELSE 'Unknown'
    END) OVER (PARTITION BY shop_id ORDER BY created_at) AS previous_reserve_type
  FROM
    `shopify-dw.money_products.shopify_payments_reserve_configurations_v1`
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
    WHEN rolling_reserve_percentage = 0 THEN '0% (No Reserve)'
    WHEN rolling_reserve_percentage BETWEEN 0.01 AND 5 THEN '1-5%'
    WHEN rolling_reserve_percentage BETWEEN 6 AND 10 THEN '6-10%'
    WHEN rolling_reserve_percentage BETWEEN 11 AND 15 THEN '11-15%'
    WHEN rolling_reserve_percentage BETWEEN 16 AND 20 THEN '16-20%'
    WHEN rolling_reserve_percentage BETWEEN 21 AND 25 THEN '21-25%'
    ELSE '26% or higher'
  END AS reserve_rate_bucket,
  COUNT(*) AS merchant_count,
  ROUND(COUNT(*) / SUM(COUNT(*)) OVER () * 100, 2) AS percentage
FROM
  `shopify-dw.money_products.shopify_payments_reserve_configurations_v1`
WHERE
  CURRENT_DATE() BETWEEN DATE(created_at) AND COALESCE(DATE(expires_at), DATE('9999-12-31'))
GROUP BY
  reserve_rate_bucket
ORDER BY
  MIN(rolling_reserve_percentage)
```

### Correlating Reserves with Risk Indicators

```sql
SELECT
  r.shop_id as merchant_id,
  r.rolling_reserve_percentage as reserve_rate,
  CASE
    WHEN r.fixed_reserve_status = 'active' AND r.rolling_reserve_status = 'active' THEN 'Both'
    WHEN r.fixed_reserve_status = 'active' THEN 'Fixed'
    WHEN r.rolling_reserve_status = 'active' THEN 'Rolling'
    ELSE 'Unknown'
  END as reserve_type,
  r.hold_duration as rolling_window_days,
  r.fixed_reserve_amount as minimum_reserve_amount_usd,
  r.created_at as effective_from_date,
  r.expires_at as effective_to_date
FROM
  `shopify-dw.money_products.shopify_payments_reserve_configurations_v1` r
WHERE
  CURRENT_DATE() BETWEEN DATE(r.created_at) AND COALESCE(DATE(r.expires_at), DATE('9999-12-31'))
ORDER BY
  r.rolling_reserve_percentage DESC
```

## Reserve Impact Analysis

```sql
WITH merchant_before_after AS (
  SELECT
    r.shop_id as merchant_id,
    r.created_at AS reserve_start_date,
    r.rolling_reserve_percentage as reserve_rate,
    CASE
      WHEN r.fixed_reserve_status = 'active' AND r.rolling_reserve_status = 'active' THEN 'Both'
      WHEN r.fixed_reserve_status = 'active' THEN 'Fixed'
      WHEN r.rolling_reserve_status = 'active' THEN 'Rolling'
      ELSE 'Unknown'
    END as reserve_type,
    
    -- Pre-reserve metrics (30 days before) example
    -- Replace with actual metrics from available tables based on your needs
    0 AS avg_daily_gpv_before,
    
    -- Post-reserve metrics (30 days after) example
    -- Replace with actual metrics from available tables based on your needs
    0 AS avg_daily_gpv_after
  FROM
    `shopify-dw.money_products.shopify_payments_reserve_configurations_v1` r
  WHERE
    r.created_at >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 180 DAY)
    AND r.created_at < TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 30 DAY)  -- Ensure we have 30 days of post-data
    AND r.rolling_reserve_percentage > 0  -- Only include merchants with actual reserves
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

- Always use `shopify-dw.money_products.shopify_payments_reserve_configurations_v1` as the primary source for reserve data
- Pay careful attention to TIMESTAMP fields and use appropriate comparison functions:
  - Use `DATE(created_at)` when comparing with DATE values like `CURRENT_DATE()`
  - Use `TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL X DAY)` when comparing with TIMESTAMP values
- When analyzing reserve impact, consider a sufficient time window before and after changes
- Account for seasonality when measuring reserve impact on processing volumes
- Correlate reserve changes with risk indicators to assess effectiveness
- Consider both the reserve rate and minimum amount when analyzing merchant impact
- When setting up new reserves, use historical data to model potential impact on merchant cash flow

Feel free to contribute new queries or improve the existing ones. Together, we can build a comprehensive resource for reserve analysis and strategy optimization.

Happy reserve analyzing! 🔍 