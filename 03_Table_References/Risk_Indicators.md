# Risk Assessment Tables Documentation

This document provides detailed information about the risk assessment-related tables used by the Credit Risk team. These tables are central to evaluating merchant risk, making reserve recommendations, and monitoring high-risk activity.

## Table of Contents

- [Overview](#overview)
- [Key Tables](#key-tables)
  - [HVCR Predictions](#hvcr-predictions)
  - [Trust Platform Shop Risk Attributes](#trust-platform-shop-risk-attributes)
  - [Shopify Payments Reserve Configurations](#shopify-payments-reserve-configurations)
  - [Shop Insights](#shop-insights)
- [Common Metrics](#common-metrics)
- [Sample Queries](#sample-queries)
- [Table Relationships](#table-relationships)
- [Best Practices](#best-practices)

## Overview

Risk assessment data is used to evaluate merchant behavior, predict potential issues, and implement appropriate safeguards. These tables provide:

1. Predictive risk scores from machine learning models
2. Daily shop risk attributes like unfulfilled orders
3. Reserve configuration information
4. Trust Battery and other merchant trust indicators

## Key Tables

### HVCR Predictions

**Full Path:** `sdp-prd-cti-data.intermediate.hvcr_predictions_v2`

**Description:** Contains High Value Credit Risk (HVCR) model predictions for merchants. The model assesses the likelihood that a merchant will generate significant credit risk.

**Primary Keys:** shop_id, processed_at

**Refresh Frequency:** Daily

**Partitioning:** Table is partitioned by `observed_at` field - all queries must include a filter on this column

**Schema:**
- `remote_account_id` - Unique identifier for the remote account
- `provider_name` - Name of the payment provider
- `shop_id` - Unique identifier of the Shopify shop
- `model_id` - Value identifying the model that produced this row
- `uuid` - Unique identifier of a score
- `processed_at` - Time when the prediction was computed
- `observed_at` - Time when the features were observed
- `score` - The predicted probability of the positive class (High Value Credit Risk), ranging from 0 to 1
- `features` - The feature values used by model to produce score (JSON format)
- `shap_values` - The SHAP values for the model prediction (JSON format)

**Notes:**
- The table is partitioned by `observed_at` date and clustered by `shop_id`
- All queries must include a filter on the `observed_at` column
- Key features used in risk assessment include dispute counts, fulfillment times, and complaint metrics
- The `features` JSON contains all inputs used for model scoring
- `score` values closer to 1 indicate higher predicted risk
- SHAP values show the contribution of each feature to the final prediction

### Trust Platform Shop Risk Attributes

**Full Path:** `sdp-prd-cti-data.intermediate.trust_platform_shop_risk_attributes_daily`

**Description:** Daily aggregated risk attributes for shops, used for monitoring and case management in the Trust Platform.

**Primary Keys:** shop_id, date

**Refresh Frequency:** Daily

**Schema:**
- `shop_id` - Unique identifier of Shopify shop
- `date` - Date when the attributes were observed
- `week_start_date` - First date of the week (Shopify considers weeks to start on Sunday)
- `month_start_date` - First date of the month
- `order_count` - Number of orders created on the shop on the date
- `sp_order_count` - Number of orders processed through Shopify Payments
- `fulfilled_order_count` - Number of fulfilled orders
- `unfulfilled_order_count` - Number of unfulfilled orders
- `net_unfulfilled_sp_sales_usd` - SP Sales amount in USD of unfulfilled orders, net of chargebacks/refunds
- `chargeback_count` - Number of chargebacks issued against the shop on the date
- Many additional risk metrics including fraud complaints and risk scores

**Notes:**
- Only includes 2 years of historical data
- `net_unfulfilled_sp_sales_usd` is a key metric for reserve recommendations
- The table is clustered by `shop_id` for efficient querying
- Designed to be aggregated to weekly or monthly levels for operational performance
- Used in the Trust Platform Case Management System (Monpliance)

### Shopify Payments Reserve Configurations

**Full Path:** `shopify-dw.money_products.shopify_payments_reserve_configurations_v1`

**Description:** Information about merchant reserves set up through Shopify Payments, including both fixed amount and rolling percentage reserves.

**Primary Keys:** shopify_payments_reserve_configuration_id, provider_account_type, fixed_reserve_currency_code

**Refresh Frequency:** Daily

**Schema:**
- `shopify_payments_reserve_configuration_id` - Unique identifier for the reserve configuration record
- `shop_id` - Shop ID associated with the reserve configuration
- `account_type` - The account type associated with the Shopify Payments Account ID
- `is_active` - Boolean flag indicating if the reserve configuration is active
- `rolling_reserve_percentage` - The percentage of the reserve balance to be held
- `hold_duration` - Length of time in days that the reserve balance will be held
- `created_at` - The timestamp when the reserve configuration was created
- `expires_at` - The timestamp when the reserve configuration will expire
- `fixed_reserve_status` - Current status of the fixed reserve (active, pending, failure, ended)
- `rolling_reserve_status` - Current status of the rolling reserve (active, pending, failure, ended)
- `fixed_reserve_amount` - The fixed amount to be held in reserve
- `fixed_reserve_currency_code` - The currency code for the fixed reserve amount

**Notes:**
- Supports both fixed reserves (specific amount) and rolling reserves (percentage)
- Known caveats include discrepancies between `fixed_reserve_status` and overall is_active status
- In some cases `expires_at` and `remote_expires_at` can be months apart
- Reserve status values include: active, pending, failure, ended
- Used for reserve management and monitoring

### Shop Insights

**Full Path:** `sdp-prd-cti-data.intermediate.shop_insights_shops`

**Description:** Comprehensive shop information including Trust Battery scores and merchant risk indicators.

**Primary Keys:** shop_id

**Refresh Frequency:** Daily

**Schema:**
- `shop_id` - Unique identifier of Shopify Shop
- `shop_name` - Name of shop
- `shop_domain` - Shop storefront domain
- `shop_created_at` - Shop created time
- `shop_country` - Shop country
- `trust_battery_score` - Trust Battery risk score (numeric value)
- `trust_battery` - Trust Battery in six levels: max, high, medium, low, none, unspecified
- `trust_battery_published_at` - Published at timestamp for trust battery
- Plus numerous fields for merchant behavior, financial metrics, and risk signals

**Notes:**
- Contains sensitive information (PII)
- Trust Battery categories include: max, high, medium, low, none, unspecified
- The `trust_battery_score` is a numeric value, while `trust_battery` is the categorical interpretation
- Used extensively in risk assessment and reserve recommendations
- Contains fields tracking shop financial activity, login patterns, and user information

## Common Metrics

When working with risk assessment data, the following metrics are commonly used:

1. **HVCR Score**: Probability of a merchant generating significant credit risk
   ```sql
   -- Gets the most recent HVCR score for a specific shop
   SELECT 
     shop_id, 
     score 
   FROM `sdp-prd-cti-data.intermediate.hvcr_predictions_v2`
   WHERE observed_at >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 30 DAY)
   AND processed_at = (
     SELECT MAX(processed_at) 
     FROM `sdp-prd-cti-data.intermediate.hvcr_predictions_v2` 
     WHERE shop_id = 12345678
     AND observed_at >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 30 DAY)
   )
   ```

2. **Unfulfilled Exposure**: Amount of money at risk due to unfulfilled orders
   ```sql
   -- Calculates the total unfulfilled order exposure for a shop over the past 120 days
   SELECT 
     SUM(net_unfulfilled_sp_sales_usd) as unfulfilled_exposure
   FROM `sdp-prd-cti-data.intermediate.trust_platform_shop_risk_attributes_daily`
   WHERE shop_id = 12345678 
   AND date >= DATE_SUB(CURRENT_DATE(), INTERVAL 120 DAY)
   ```

3. **Trust Battery Category Distribution**:
   ```sql
   -- Counts shops by Trust Battery category, with a meaningful sort order
   SELECT 
     trust_battery, 
     COUNT(*) as shop_count
   FROM `sdp-prd-cti-data.intermediate.shop_insights_shops`
   GROUP BY trust_battery
   ORDER BY CASE 
     WHEN trust_battery = 'max' THEN 1
     WHEN trust_battery = 'high' THEN 2
     WHEN trust_battery = 'medium' THEN 3
     WHEN trust_battery = 'low' THEN 4
     WHEN trust_battery = 'none' THEN 5
     ELSE 6
   END
   ```

## Sample Queries

### Query 1: Identify High-Risk Shops with Unfulfilled Orders

```sql
-- Purpose: Find high-risk merchants with significant unfulfilled order exposure
-- This query combines HVCR model predictions with unfulfilled order data to identify shops
-- that may require reserves or other risk mitigation measures

WITH high_risk_predictions AS (
  -- Identify shops with high HVCR scores from the most recent predictions
  SELECT 
    shop_id, 
    score
  FROM `sdp-prd-cti-data.intermediate.hvcr_predictions_v2`
  WHERE processed_at = (
    SELECT MAX(processed_at)
    FROM `sdp-prd-cti-data.intermediate.hvcr_predictions_v2`
    WHERE observed_at >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 30 DAY)
  )
  AND observed_at >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 30 DAY)
  AND score > 0.7  -- Only consider shops with high risk scores
),

unfulfilled_exposure AS (
  -- Calculate total unfulfilled order exposure for each shop
  SELECT
    shop_id,
    SUM(net_unfulfilled_sp_sales_usd) as unfulfilled_exposure_usd
  FROM `sdp-prd-cti-data.intermediate.trust_platform_shop_risk_attributes_daily`
  WHERE date >= DATE_SUB(CURRENT_DATE(), INTERVAL 30 DAY)
  GROUP BY shop_id
  HAVING SUM(net_unfulfilled_sp_sales_usd) > 5000  -- Only shops with significant exposure
)

-- Join all data together to get a complete risk profile
SELECT 
  h.shop_id,
  s.shop_name,
  s.shop_domain,
  s.trust_battery,
  h.score as hvcr_score,
  u.unfulfilled_exposure_usd
FROM high_risk_predictions h
JOIN unfulfilled_exposure u ON h.shop_id = u.shop_id
JOIN `sdp-prd-cti-data.intermediate.shop_insights_shops` s ON h.shop_id = s.shop_id
ORDER BY h.score DESC, u.unfulfilled_exposure_usd DESC
LIMIT 100
```

### Query 2: Check for Expiring Reserves

```sql
-- Purpose: Identify shops with reserves that are about to expire
-- This query helps risk analysts proactively manage reserve expirations
-- and determine if reserves need to be extended or modified

SELECT
  c.shop_id,
  s.shop_name,
  s.shop_domain,
  s.trust_battery,                                                          -- Trust level of the shop
  c.created_at as reserve_created_at,
  c.expires_at as reserve_expires_at,
  DATE_DIFF(c.expires_at, CURRENT_TIMESTAMP(), DAY) as days_until_expiry,   -- Days remaining until reserve expires
  CASE
    WHEN fixed_reserve_status = 'active' AND rolling_reserve_status = 'active' THEN 'Both'
    WHEN fixed_reserve_status = 'active' THEN 'Fixed'
    WHEN rolling_reserve_status = 'active' THEN 'Rolling'
  END as reserve_type,                                                      -- Type of reserve applied
  c.fixed_reserve_amount,
  c.fixed_reserve_currency_code,
  c.rolling_reserve_percentage
FROM `shopify-dw.money_products.shopify_payments_reserve_configurations_v1` c
JOIN `sdp-prd-cti-data.intermediate.shop_insights_shops` s ON c.shop_id = s.shop_id
WHERE c.is_active = TRUE
  AND c.expires_at IS NOT NULL
  AND DATE_DIFF(c.expires_at, CURRENT_TIMESTAMP(), DAY) BETWEEN 0 AND 30    -- Focus on reserves expiring within 30 days
  AND (c.fixed_reserve_status = 'active' OR c.rolling_reserve_status = 'active')
ORDER BY days_until_expiry ASC
```

## Table Relationships

Risk assessment tables relate to other tables in the following ways:

1. `hvcr_predictions_v2` links to shop tables through `shop_id`
2. `trust_platform_shop_risk_attributes_daily` aggregates data from order and transaction tables
3. `shopify_payments_reserve_configurations_v1` connects to shop-level data via `shop_id`
4. `shop_insights_shops` acts as a hub table connecting to various other shop-related tables

For a visual representation of these relationships, see the [Table Relationships](./Table_Relationships.md) document.

## Best Practices

When working with risk assessment data, follow these best practices:

1. **For HVCR predictions**:
   - Always use the most recent prediction for a shop
   - Always include a filter on the `observed_at` partition column
   - Consider both the raw score and the feature contributions (SHAP values)
   - Combine with other risk signals for a comprehensive assessment

2. **For unfulfilled exposure**:
   - Look at both count and monetary value of unfulfilled orders
   - Consider the time dimension (how long orders have been unfulfilled)
   - Check if the unfulfilled amount is significant relative to the shop's total GPV

3. **For reserve configurations**:
   - Verify both the `is_active` flag and the specific reserve status fields
   - Consider the remaining time until reserve expiration
   - Check both fixed and rolling reserves as they may coexist

4. **For Trust Battery**:
   - Use the categorical value (`trust_battery`) for high-level segmentation
   - Use the numeric score (`trust_battery_score`) for more granular analysis
   - Consider when the Trust Battery was last updated (`trust_battery_published_at`)

### Temporal Fields

| Field Name | Data Type | Comparison Function | Example |
|------------|-----------|---------------------|---------|
| observed_at | TIMESTAMP | TIMESTAMP_SUB() | `WHERE observed_at >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 7 DAY)` |
| processed_at | TIMESTAMP | TIMESTAMP_SUB() | `WHERE processed_at >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 7 DAY)` |
| date | DATE | DATE_SUB() | `WHERE date >= DATE_SUB(CURRENT_DATE(), INTERVAL 30 DAY)` |
| created_at | TIMESTAMP | TIMESTAMP_SUB() | `WHERE created_at >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 30 DAY)` |
| expires_at | TIMESTAMP | TIMESTAMP_SUB() | `WHERE expires_at >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 30 DAY)` |

## Related Resources

- [Chargeback Tables](./Chargeback_Tables.md)
- [Shop Tables](./Shop_Tables.md)
- [Reserve Management](../04_Domain_Knowledge/Reserves.md)

---