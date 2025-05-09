# Reserve Management Queries 🔒

Welcome to the Reserves section of our SQL Query Resource Center. This page contains datasets and queries for analyzing merchant reserve configurations and impact.

## Table of Contents
- [Introduction](#introduction)
- [Primary Datasets](#primary-datasets)
- [Current Reserve Configurations](#current-reserve-configurations)
- [Reserve Configuration Changes](#reserve-configuration-changes)
- [Reserve Rate Distribution](#reserve-rate-distribution)
- [Correlating Reserves with Risk Indicators](#correlating-reserves-with-risk-indicators)
- [Reserve Impact Analysis](#reserve-impact-analysis)
- [Notes and Best Practices](#notes-and-best-practices)

## Introduction

Merchant reserves are financial safeguards placed on merchant accounts to mitigate risk. Shopify Payments uses two primary types of reserves:
- **Fixed reserves**: Hold a specific monetary amount from merchant payouts
- **Rolling reserves**: Hold a percentage of future transactions for a defined period

The queries in this document help analyze reserve configurations, changes over time, and their impact on merchants.

## Primary Datasets 📁

### `shopify-dw.money_products.shopify_payments_reserve_configurations_v1`

This is the primary recommended dataset for all reserve analysis. It combines data from Shopify Payments reserve configurations with their associated fixed and rolling reserves. It provides a unified view of all reserve types (fixed and rolling) that are mostly placed on merchant accounts via the Trust Platform.

#### Temporal Fields

| Field Name | Data Type | Comparison Function | Example |
|------------|-----------|---------------------|---------|
| created_at | TIMESTAMP | TIMESTAMP_SUB() | `WHERE created_at >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 7 DAY)` |
| expires_at | TIMESTAMP | TIMESTAMP_SUB() | `WHERE expires_at >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 7 DAY)` |
| cancelled_at | TIMESTAMP | TIMESTAMP_SUB() | `WHERE cancelled_at >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 7 DAY)` |

## Current Reserve Configurations

### SQL Query

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

### Description

This query identifies all current active reserve configurations across merchants. It retrieves:
- The reserve rate (percentage of funds held)
- Reserve type (fixed, rolling, or both)
- Duration for which funds are held (in days)
- Fixed reserve amount (if applicable)
- Effective dates for the reserve configuration

The query filters to only show currently active reserves by checking if the current date falls between the creation and expiration dates of the reserve.

### Example

The results show merchants with different reserve configurations. Each row represents a merchant with details about their reserve type, rate, and duration. This data can be used to understand the distribution of reserve rates across the merchant base and identify merchants with unusually high reserve rates.

## Reserve Configuration Changes

### SQL Query

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

### Description

This query tracks how merchant reserve configurations have changed over time. It uses the LAG window function to compare each reserve configuration with the previous one for the same merchant. Key metrics include:
- Changes in reserve rate (percentage increase or decrease)
- Changes in reserve type (e.g., switching from rolling to fixed)
- The timing of these changes

The query filters to only show merchants where either the reserve rate or type has changed.

### Example

The results show merchants where reserve configurations have changed. This allows for analysis of reserve adjustment patterns over time, identification of merchants with frequent changes, and understanding of whether reserves are typically increased or decreased.

## Reserve Rate Distribution

### SQL Query

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

### Description

This query analyzes the distribution of reserve rates across merchants by:
- Categorizing reserve rates into predefined buckets (e.g., 1-5%, 6-10%, etc.)
- Counting the number of merchants in each bucket
- Calculating the percentage of total merchants in each bucket

The query filters for currently active reserves and provides a clear view of how reserve rates are distributed.

### Example

The results show the distribution of reserve rates across all merchants with active reserves. This helps identify typical reserve rate ranges and outliers, which can inform reserve policy decisions and risk assessment strategies.

## Correlating Reserves with Risk Indicators

### SQL Query

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

### Description

This query is designed to be extended with joins to risk indicator tables. It retrieves merchant reserve configurations that can be correlated with various risk factors from other datasets. The base query includes:
- Reserve rate and type information
- Reserve duration and amount
- Effective dates

To make this query useful for risk analysis, join it with relevant risk indicator tables (e.g., disputes, chargebacks, or merchant risk scores).

### Example

The results provide a foundation for risk analysis by showing merchant reserve configurations. When joined with risk data, this can reveal correlations between reserve rates and risk factors, helping determine if appropriate reserves have been set for merchants with specific risk profiles.

## Reserve Impact Analysis

### SQL Query

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

### Description

This query template allows for analyzing the impact of reserves on merchant behavior and processing volumes. It:
- Creates a before-and-after comparison window around when reserves were implemented
- Calculates key metrics before and after reserve implementation
- Determines both absolute and percentage changes in metrics

The placeholder metrics (avg_daily_gpv_before/after) should be replaced with actual metrics from relevant tables, such as processing volume, transaction count, or dispute rates.

### Example

When properly configured with actual metrics, this query can reveal how merchant behavior changes after reserves are implemented. It can help answer questions like:
- Does GMV/GPV decrease after reserves are applied?
- Do merchants with higher reserve rates show different behavior patterns?
- What is the typical business impact of implementing different reserve types?

## Notes and Best Practices 📝

- Pay careful attention to TIMESTAMP fields and use appropriate comparison functions:
  - Use `DATE(created_at)` when comparing with DATE values like `CURRENT_DATE()`
  - Use `TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL X DAY)` when comparing with TIMESTAMP values
- When analyzing reserve impact, consider a sufficient time window before and after changes
- Account for seasonality when measuring reserve impact on processing volumes
- Correlate reserve changes with risk indicators to assess effectiveness
- Consider both the reserve rate and minimum amount when analyzing merchant impact

Happy reserve analyzing! 🔍 
