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

| Column Name | Data Type | Description | Example |
|-------------|-----------|-------------|---------|
| remote_account_id | STRING | Identifier for the remote account | acc_1a2b3c4d |
| provider_name | STRING | Payment provider name | stripe |
| shop_id | INTEGER | Unique identifier for the shop | 12345678 |
| model_id | STRING | Identifier for the model version used | hvcr_v2_20230401 |
| uuid | STRING | Unique identifier for the prediction | pred_1a2b3c4d |
| processed_at | TIMESTAMP | When the prediction was computed | 2023-04-15 12:00:00 UTC |
| observed_at | TIMESTAMP | When features were observed | 2023-04-15 10:00:00 UTC |
| score | FLOAT | Probability of high credit risk (0-1) | 0.78 |
| features | JSON | Feature values used by the model | {"dispute_count_30d": "5", "avg_fulfillment_time_m_30d": "4320"} |
| shap_values | JSON | SHAP values for model explainability | {"dispute_count_30d": 0.25, "avg_fulfillment_time_m_30d": 0.15} |

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

| Column Name | Data Type | Description | Example |
|-------------|-----------|-------------|---------|
| shop_id | INTEGER | Unique identifier for the shop | 12345678 |
| date | DATE | Date of observation | 2023-04-15 |
| week_start_date | DATE | First date of the reporting week | 2023-04-09 |
| month_start_date | DATE | First date of the month | 2023-04-01 |
| order_count | INTEGER | Number of orders created that day | 45 |
| sp_order_count | INTEGER | Number of Shopify Payments orders created | 38 |
| fulfilled_order_count | INTEGER | Number of fulfilled orders | 30 |
| unfulfilled_order_count | INTEGER | Number of unfulfilled orders | 15 |
| unfulfilled_sp_order_count | INTEGER | Number of unfulfilled Shopify Payments orders | 12 |
| net_unfulfilled_sp_order_count | INTEGER | Net unfulfilled SP orders | 10 |
| fulfilled_gpv_usd | NUMERIC | Fulfilled GPV in USD | 3500.50 |
| unfulfilled_gpv_usd | NUMERIC | Unfulfilled GPV in USD | 2250.75 |
| fulfilled_sp_sales_usd | NUMERIC | Fulfilled SP sales in USD | 3200.25 |
| unfulfilled_sp_sales_usd | NUMERIC | Unfulfilled SP sales in USD | 1980.50 |
| net_unfulfilled_sp_sales_usd | NUMERIC | Net unfulfilled SP sales in USD | 1800.00 |
| sp_disputable_transaction_count | INTEGER | Number of SP transactions excluding certain payment methods | 35 |
| sp_transaction_count | INTEGER | Number of SP transactions | 40 |
| chargeback_count | INTEGER | Number of chargebacks issued that day | 2 |
| fraudulent_chargeback_count | INTEGER | Number of fraudulent chargebacks | 1 |
| product_not_received_chargeback_count | INTEGER | Number of product not received chargebacks | 1 |
| other_chargeback_count | INTEGER | Number of other chargebacks | 0 |
| risky_order_count | INTEGER | Number of orders classified as high-risk | 3 |
| risky_order_gmv_usd | NUMERIC | GMV amount of orders classified as high-risk | 450.00 |
| buyer_complaint_count | INTEGER | Number of buyer complaints | 1 |

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

| Column Name | Data Type | Description | Example |
|-------------|-----------|-------------|---------|
| shopify_payments_reserve_configuration_id | INTEGER | Unique identifier for the configuration | 12345 |
| shop_id | INTEGER | Unique identifier for the shop | 12345678 |
| account_type | STRING | Account type associated with the SPay Account | standard |
| account_id | INTEGER | Shopify Payments Account ID | 5678901 |
| is_active | BOOLEAN | Whether the configuration is active | TRUE |
| rolling_reserve_percentage | NUMERIC | Percentage of the reserve to be held | 0.15 |
| hold_duration | NUMERIC | Days the reserve balance will be held | 30 |
| ended_reason | STRING | Why the reserve was ended | expired |
| created_at | TIMESTAMP | When the configuration was created | 2023-01-15 10:30:00 UTC |
| expires_at | TIMESTAMP | When the configuration will expire | 2023-07-15 10:30:00 UTC |
| cancelled_at | TIMESTAMP | When the configuration was cancelled | NULL |
| updated_at | TIMESTAMP | When the record was last updated | 2023-02-15 14:20:00 UTC |
| provider_account_id | INTEGER | ID of the provider account | 123456 |
| provider_account_type | STRING | Type of the provider account | ShopifyPayments::Provider::Stripe::Account |
| remote_created_at | TIMESTAMP | When created in provider system | 2023-01-15 10:35:00 UTC |
| remote_expires_at | TIMESTAMP | When expires in provider system | 2023-07-15 10:35:00 UTC |
| fixed_reserve_remote_id | STRING | Provider's identifier for fixed reserve | rsv_123456 |
| rolling_reserve_remote_id | STRING | Provider's identifier for rolling reserve | rsv_234567 |
| fixed_reserve_status | STRING | Status of the fixed reserve | active |
| rolling_reserve_status | STRING | Status of the rolling reserve | active |
| fixed_reserve_failure_reason | STRING | Reason for fixed reserve failure | NULL |
| rolling_reserve_failure_reason | STRING | Reason for rolling reserve failure | NULL |
| fixed_reserve_amount | NUMERIC | Fixed amount to be held in reserve | 1000.00 |
| fixed_reserve_currency_code | STRING | Currency code for the fixed amount | USD |
| fixed_reserve_idempotency_key | STRING | Idempotency key for fixed reserve | key_12345 |
| rolling_reserve_idempotency_key | STRING | Idempotency key for rolling reserve | key_67890 |

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

| Column Name | Data Type | Description | Example |
|-------------|-----------|-------------|---------|
| shop_id | INTEGER | Unique identifier for the shop | 12345678 |
| shop_name | STRING | Name of the shop | Trending Fashion |
| shop_created_at | TIMESTAMP | When the shop was created | 2022-08-10 15:20:10 UTC |
| shop_country | STRING | Shop country | US |
| shop_domain | STRING | Shop storefront domain | trending-fashion.myshopify.com |
| shop_email | STRING | Email address on shop | contact@trendingfashion.com |
| shop_plan | STRING | Current plan name | shopify_plus |
| trust_battery_version | FLOAT | Version of Trust Battery logic | 2.0 |
| trust_battery_score | NUMERIC | Trust Battery risk score (0-100) | 85 |
| trust_battery | STRING | Trust Battery category | high |
| trust_battery_published_at | TIMESTAMP | When Trust Battery was published | 2023-04-10 08:15:30 UTC |
| signup_score | FLOAT | Signup Model risk score | 0.23 |
| billing_score | FLOAT | Billing Model risk score | 0.15 |
| idv_status | STRING | Identity verification status | completed |
| shop_gmv | NUMERIC | Shop GMV (approximate) | 250000.00 |
| shop_gpv | NUMERIC | Shop GPV (approximate) | 225000.00 |
| shop_order_count | INTEGER | Number of orders brokered by Shopify | 5000 |
| shop_sp_order_count | INTEGER | Number of SP orders | 4500 |
| partner_id | INTEGER | Partner ID of shop | 123456 |
| partner_name | STRING | Name of partner | Partner Agency |
| payment_type | STRING | Billing payment type | credit_card |
| payment_details | STRING | Billing payment information | 4123************5678 |

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
   SELECT shop_id, score 
   FROM sdp-prd-cti-data.intermediate.hvcr_predictions_v2
   WHERE observed_at >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 30 DAY)
   AND processed_at = (
     SELECT MAX(processed_at) 
     FROM sdp-prd-cti-data.intermediate.hvcr_predictions_v2 
     WHERE shop_id = 12345678
     AND observed_at >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 30 DAY)
   )
   ```

2. **Unfulfilled Exposure**: Amount of money at risk due to unfulfilled orders
   ```sql
   SELECT SUM(net_unfulfilled_sp_sales_usd) as unfulfilled_exposure
   FROM sdp-prd-cti-data.intermediate.trust_platform_shop_risk_attributes_daily
   WHERE shop_id = 12345678 AND date >= DATE_SUB(CURRENT_DATE(), INTERVAL 120 DAY)
   ```

3. **Trust Battery Category Distribution**:
   ```sql
   SELECT trust_battery, COUNT(*) as shop_count
   FROM sdp-prd-cti-data.intermediate.shop_insights_shops
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
WITH high_risk_predictions AS (
  SELECT 
    shop_id, 
    score
  FROM sdp-prd-cti-data.intermediate.hvcr_predictions_v2
  WHERE processed_at = (
    SELECT MAX(processed_at)
    FROM sdp-prd-cti-data.intermediate.hvcr_predictions_v2
    WHERE observed_at >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 30 DAY)
  )
  AND observed_at >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 30 DAY)
  AND score > 0.7
),

unfulfilled_exposure AS (
  SELECT
    shop_id,
    SUM(net_unfulfilled_sp_sales_usd) as unfulfilled_exposure_usd
  FROM sdp-prd-cti-data.intermediate.trust_platform_shop_risk_attributes_daily
  WHERE date >= DATE_SUB(CURRENT_DATE(), INTERVAL 30 DAY)
  GROUP BY shop_id
  HAVING SUM(net_unfulfilled_sp_sales_usd) > 5000
)

SELECT 
  h.shop_id,
  s.shop_name,
  s.shop_domain,
  s.trust_battery,
  h.score as hvcr_score,
  u.unfulfilled_exposure_usd
FROM high_risk_predictions h
JOIN unfulfilled_exposure u ON h.shop_id = u.shop_id
JOIN sdp-prd-cti-data.intermediate.shop_insights_shops s ON h.shop_id = s.shop_id
ORDER BY h.score DESC, u.unfulfilled_exposure_usd DESC
LIMIT 100
```

### Query 2: Check for Expiring Reserves

```sql
SELECT
  c.shop_id,
  s.shop_name,
  s.shop_domain,
  s.trust_battery,
  c.created_at as reserve_created_at,
  c.expires_at as reserve_expires_at,
  DATE_DIFF(c.expires_at, CURRENT_TIMESTAMP(), DAY) as days_until_expiry,
  CASE
    WHEN fixed_reserve_status = 'active' AND rolling_reserve_status = 'active' THEN 'Both'
    WHEN fixed_reserve_status = 'active' THEN 'Fixed'
    WHEN rolling_reserve_status = 'active' THEN 'Rolling'
  END as reserve_type,
  c.fixed_reserve_amount,
  c.fixed_reserve_currency_code,
  c.rolling_reserve_percentage
FROM shopify-dw.money_products.shopify_payments_reserve_configurations_v1 c
JOIN sdp-prd-cti-data.intermediate.shop_insights_shops s ON c.shop_id = s.shop_id
WHERE c.is_active = TRUE
  AND c.expires_at IS NOT NULL
  AND DATE_DIFF(c.expires_at, CURRENT_TIMESTAMP(), DAY) BETWEEN 0 AND 30
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