# Shop Tables

This document provides comprehensive information about shop-related tables used for credit risk analysis at Shopify.

## Overview

Shop tables contain information about merchant accounts, their status, and key attributes. These tables are essential for identifying merchants, understanding their account status, and correlating risk metrics with shop-level data.

## Primary Tables

### sdp-prd-cti-data.intermediate.shop_current_shopify_payments_status

**Description**:  
This table provides the current Shopify Payments activation status for each shop. It's used to determine whether a shop can process payments through Shopify Payments and their current account status.

**Update Frequency**: Daily

**Primary Keys**: `shop_id`

**Schema**:

| Field Name | Type | Description |
|------------|------|-------------|
| `shop_id` | INTEGER | Unique identifier for a shop |
| `shopify_payments_status` | STRING | Current status of the Shopify Payments account (e.g., 'active', 'pending', 'rejected', 'disabled') |
| `account_active` | BOOLEAN | Whether the Shopify Payments account is currently active |
| `disabled_reason` | STRING | Reason for disablement, if the account is disabled |
| `last_updated_at` | TIMESTAMP | When this record was last updated |
| `created_at` | TIMESTAMP | When the Shopify Payments account was created |

**Usage Notes**:
- This table is essential for filtering analyses to only include active shops
- The `shopify_payments_status` field is the primary indicator of account status
- Common values for `shopify_payments_status` include:
  - `active`: Shop can process payments
  - `pending`: Application in progress
  - `disabled`: Account has been disabled (check `disabled_reason`)
  - `rejected`: Application was rejected

**Example Query**:
```sql
-- Find all active shops with Shopify Payments enabled
SELECT 
  shop_id,
  shopify_payments_status,
  created_at
FROM `sdp-prd-cti-data.intermediate.shop_current_shopify_payments_status`
WHERE account_active = TRUE
  AND shopify_payments_status = 'active'
```

### shopify-dw.shopify.shops

**Description**:  
The master table containing core information about Shopify merchants. Contains shop creation dates, plan information, and other shop-level attributes.

**Update Frequency**: Daily

**Primary Keys**: `id`

**Schema**:

| Field Name | Type | Description |
|------------|------|-------------|
| `id` | INTEGER | Unique identifier for a shop (same as shop_id in other tables) |
| `shop_name` | STRING | Name of the shop |
| `myshopify_domain` | STRING | The shop's myshopify.com domain |
| `created_at` | TIMESTAMP | When the shop was created |
| `updated_at` | TIMESTAMP | When the shop record was last updated |
| `plan_name` | STRING | The shop's subscription plan (e.g., 'basic', 'shopify', 'advanced') |
| `has_discounts` | BOOLEAN | Whether the shop has discounts enabled |
| `has_gift_cards` | BOOLEAN | Whether the shop has gift cards enabled |
| `country_code` | STRING | Two-letter country code for the shop's location |
| `currency` | STRING | Default currency used by the shop |
| `customer_email` | STRING | Shop owner's email (PII) |
| `shop_owner` | STRING | Name of the shop owner (PII) |
| `timezone` | STRING | The shop's timezone |

**Usage Notes**:
- Contains PII data (`customer_email`, `shop_owner`) - requires proper permissions
- Essential for enriching risk analyses with shop age and plan information
- The `id` field matches `shop_id` in payment and transaction tables

**Example Query**:
```sql
-- Get shop age information for correlation with risk metrics
WITH risk_metrics AS (
  SELECT 
    shop_id,
    chargeback_rate
  FROM `sdp-prd-cti-data.intermediate.shop_chargeback_rates_current`
  WHERE transaction_count >= 100
)

SELECT 
  r.shop_id,
  s.created_at,
  DATE_DIFF(CURRENT_DATE(), DATE(s.created_at), DAY) AS shop_age_days,
  s.plan_name,
  s.country_code,
  r.chargeback_rate
FROM risk_metrics r
JOIN `shopify-dw.shopify.shops` s ON r.shop_id = s.id
ORDER BY r.chargeback_rate DESC
```

### sdp-prd-cti-data.intermediate.shop_current_business_model

**Description**:  
Contains information about the merchant's business model classification, which is relevant for risk assessment.

**Update Frequency**: Daily

**Primary Keys**: `shop_id`

**Schema**:

| Field Name | Type | Description |
|------------|------|-------------|
| `shop_id` | INTEGER | Unique identifier for a shop |
| `business_model` | STRING | The business model classification |
| `business_model_source` | STRING | Source of the business model classification |
| `last_updated_at` | TIMESTAMP | When this record was last updated |

**Usage Notes**:
- Business model is a key factor in risk assessment
- Common business model values include:
  - 'Dropshipping'
  - 'Pre-order/Made to order/Limited release'
  - 'Retail/Domestic'
  - 'Replica'

**Example Query**:
```sql
-- Analyze chargeback rates by business model
SELECT 
  b.business_model,
  COUNT(DISTINCT c.shop_id) AS shop_count,
  AVG(c.chargeback_rate) AS avg_chargeback_rate,
  APPROX_QUANTILES(c.chargeback_rate, 100)[OFFSET(50)] AS median_chargeback_rate,
  MAX(c.chargeback_rate) AS max_chargeback_rate
FROM `sdp-prd-cti-data.intermediate.shop_current_business_model` b
JOIN `sdp-prd-cti-data.intermediate.shop_chargeback_rates_current` c
  ON b.shop_id = c.shop_id
WHERE c.transaction_count >= 100
GROUP BY b.business_model
ORDER BY avg_chargeback_rate DESC
```

### shopify-dw.shopify.shop_metafields

**Description**:  
Contains custom metadata associated with shops, including risk-related annotations.

**Update Frequency**: Daily

**Primary Keys**: `id`

**Foreign Keys**: `shop_id` references `shopify-dw.shopify.shops.id`

**Schema**:

| Field Name | Type | Description |
|------------|------|-------------|
| `id` | INTEGER | Unique identifier for the metafield |
| `shop_id` | INTEGER | Unique identifier for a shop |
| `namespace` | STRING | Namespace of the metafield |
| `key` | STRING | Key of the metafield |
| `value` | STRING | Value of the metafield |
| `value_type` | STRING | Data type of the value |
| `created_at` | TIMESTAMP | When the metafield was created |
| `updated_at` | TIMESTAMP | When the metafield was last updated |

**Usage Notes**:
- Risk-related metafields typically use specific namespaces
- Common risk-related namespaces include:
  - `risk`
  - `payments`
  - `trust`
- Useful for identifying shops with special risk designations

**Example Query**:
```sql
-- Find shops with risk-related metafields
SELECT 
  shop_id,
  namespace,
  key,
  value
FROM `shopify-dw.shopify.shop_metafields`
WHERE namespace IN ('risk', 'payments', 'trust')
  AND created_at >= DATE_SUB(CURRENT_DATE(), INTERVAL 90 DAY)
```

## Secondary Tables

### sdp-prd-cti-data.intermediate.shop_metadata

**Description**:  
Consolidated shop metadata from various sources, providing a comprehensive view of shop attributes.

**Update Frequency**: Daily

**Primary Keys**: `shop_id`

**Schema**:

| Field Name | Type | Description |
|------------|------|-------------|
| `shop_id` | INTEGER | Unique identifier for a shop |
| `created_at` | TIMESTAMP | When the shop was created |
| `first_transaction_at` | TIMESTAMP | Timestamp of first transaction |
| `first_order_at` | TIMESTAMP | Timestamp of first order |
| `country_code` | STRING | Two-letter country code |
| `plan_name` | STRING | Subscription plan name |
| `has_plus` | BOOLEAN | Whether the shop is on Shopify Plus |
| `industry` | STRING | Shop's industry category |
| `marketing_activity_score` | FLOAT | Measure of marketing activity |
| `last_updated_at` | TIMESTAMP | When this record was last updated |

**Usage Notes**:
- Provides a consolidated view of important shop attributes
- Useful for enriching risk analyses with shop metadata
- The `marketing_activity_score` can be an indicator of certain risk patterns

**Example Query**:
```sql
-- Compare risk metrics across different shop attributes
SELECT 
  CASE
    WHEN DATE_DIFF(CURRENT_DATE(), DATE(s.created_at), DAY) < 30 THEN 'New (< 30 days)'
    WHEN DATE_DIFF(CURRENT_DATE(), DATE(s.created_at), DAY) < 90 THEN 'Recent (30-90 days)'
    ELSE 'Established (90+ days)'
  END AS shop_age_category,
  s.has_plus,
  COUNT(DISTINCT s.shop_id) AS shop_count,
  AVG(c.chargeback_rate) AS avg_chargeback_rate
FROM `sdp-prd-cti-data.intermediate.shop_metadata` s
JOIN `sdp-prd-cti-data.intermediate.shop_chargeback_rates_current` c
  ON s.shop_id = c.shop_id
WHERE c.transaction_count >= 100
GROUP BY 
  shop_age_category,
  s.has_plus
ORDER BY 
  shop_age_category,
  s.has_plus
```

### shopify-dw.payments_fraud.shopify_payments_risk_settings

**Description**:  
Contains information about risk settings configured for shops using Shopify Payments.

**Update Frequency**: Daily

**Primary Keys**: `shop_id`

**Schema**:

| Field Name | Type | Description |
|------------|------|-------------|
| `shop_id` | INTEGER | Unique identifier for a shop |
| `authorization_reason` | STRING | Reason for authorization (if applicable) |
| `decline_on_avs_failure` | BOOLEAN | Whether to decline if AVS check fails |
| `decline_on_cvc_failure` | BOOLEAN | Whether to decline if CVC check fails |
| `created_at` | TIMESTAMP | When these settings were created |
| `updated_at` | TIMESTAMP | When these settings were last updated |

**Usage Notes**:
- Helpful for understanding a shop's risk tolerance settings
- Can be correlated with chargeback rates to assess effectiveness
- Changes in these settings might indicate risk management actions

**Example Query**:
```sql
-- Compare chargeback rates based on risk settings
SELECT 
  r.decline_on_avs_failure,
  r.decline_on_cvc_failure,
  COUNT(DISTINCT r.shop_id) AS shop_count,
  AVG(c.chargeback_rate) AS avg_chargeback_rate
FROM `shopify-dw.payments_fraud.shopify_payments_risk_settings` r
JOIN `sdp-prd-cti-data.intermediate.shop_chargeback_rates_current` c
  ON r.shop_id = c.shop_id
WHERE c.transaction_count >= 100
GROUP BY 
  r.decline_on_avs_failure,
  r.decline_on_cvc_failure
ORDER BY 
  avg_chargeback_rate DESC
```

## How to Join Shop Tables

### Joining with Transaction Data

```sql
-- Join shop status with transaction data
SELECT 
  s.shop_id,
  s.shopify_payments_status,
  s.account_active,
  COUNT(t.transaction_id) AS transaction_count,
  SUM(t.amount) AS total_amount
FROM `sdp-prd-cti-data.intermediate.shop_current_shopify_payments_status` s
JOIN `shopify-dw.money_products.order_transactions_payments_summary` t
  ON s.shop_id = t.shop_id
WHERE s.account_active = TRUE
  AND t.created_at >= DATE_SUB(CURRENT_DATE(), INTERVAL 30 DAY)
GROUP BY 
  s.shop_id,
  s.shopify_payments_status,
  s.account_active
```

### Joining with Chargeback Data

```sql
-- Join shop information with chargeback data
SELECT 
  s.id AS shop_id,
  s.shop_name,
  s.created_at AS shop_created_at,
  s.plan_name,
  c.chargeback_rate,
  c.transaction_count
FROM `shopify-dw.shopify.shops` s
JOIN `sdp-prd-cti-data.intermediate.shop_chargeback_rates_current` c
  ON s.id = c.shop_id
WHERE c.transaction_count >= 100
  AND c.chargeback_rate > 0.01
ORDER BY c.chargeback_rate DESC
```

### Joining with Business Model

```sql
-- Join shop status, business model, and chargeback data
SELECT 
  s.shop_id,
  s.shopify_payments_status,
  b.business_model,
  c.chargeback_rate,
  c.transaction_count
FROM `sdp-prd-cti-data.intermediate.shop_current_shopify_payments_status` s
JOIN `sdp-prd-cti-data.intermediate.shop_current_business_model` b
  ON s.shop_id = b.shop_id
JOIN `sdp-prd-cti-data.intermediate.shop_chargeback_rates_current` c
  ON s.shop_id = c.shop_id
WHERE s.account_active = TRUE
  AND c.transaction_count >= 100
ORDER BY c.chargeback_rate DESC
```

## Data Quality Considerations

1. **Shop Status Changes**:
   - Shop status can change over time
   - Always use the most recent status from `shop_current_shopify_payments_status`
   - Consider historical status changes for trend analysis

2. **Missing Data**:
   - New shops may not have entries in all tables
   - Business model may be missing for some shops
   - Use appropriate joins (LEFT JOIN) and NULL handling

3. **PII Considerations**:
   - Tables with email addresses and personal information require PII access
   - Aggregate or anonymize data when sharing results
   - Follow Shopify's data handling guidelines

4. **Data Freshness**:
   - Most shop tables update daily
   - Check `last_updated_at` fields for recency
   - Be aware of potential 1-day lag in some datasets

## Common Analysis Patterns

### Shop Age vs. Risk

```sql
-- Analyze relationship between shop age and risk
WITH shop_age AS (
  SELECT 
    id AS shop_id,
    DATE_DIFF(CURRENT_DATE(), DATE(created_at), DAY) AS age_days,
    CASE
      WHEN DATE_DIFF(CURRENT_DATE(), DATE(created_at), DAY) < 30 THEN 'New (< 30 days)'
      WHEN DATE_DIFF(CURRENT_DATE(), DATE(created_at), DAY) < 90 THEN 'Recent (30-90 days)'
      WHEN DATE_DIFF(CURRENT_DATE(), DATE(created_at), DAY) < 365 THEN 'Established (90-365 days)'
      ELSE 'Mature (365+ days)'
    END AS age_category
  FROM `shopify-dw.shopify.shops`
)

SELECT 
  a.age_category,
  COUNT(DISTINCT a.shop_id) AS shop_count,
  AVG(c.chargeback_rate) AS avg_chargeback_rate,
  APPROX_QUANTILES(c.chargeback_rate, 100)[OFFSET(50)] AS median_chargeback_rate,
  APPROX_QUANTILES(c.chargeback_rate, 100)[OFFSET(75)] AS p75_chargeback_rate
FROM shop_age a
JOIN `sdp-prd-cti-data.intermediate.shop_chargeback_rates_current` c
  ON a.shop_id = c.shop_id
WHERE c.transaction_count >= 100
GROUP BY a.age_category
ORDER BY 
  CASE
    WHEN a.age_category = 'New (< 30 days)' THEN 1
    WHEN a.age_category = 'Recent (30-90 days)' THEN 2
    WHEN a.age_category = 'Established (90-365 days)' THEN 3
    WHEN a.age_category = 'Mature (365+ days)' THEN 4
    ELSE 5
  END
```

### Business Model Risk Comparison

```sql
-- Compare risk metrics across business models
SELECT 
  b.business_model,
  COUNT(DISTINCT b.shop_id) AS shop_count,
  AVG(c.chargeback_rate) AS avg_chargeback_rate,
  APPROX_QUANTILES(c.chargeback_rate, 100)[OFFSET(50)] AS median_chargeback_rate,
  AVG(r.percentage) AS avg_reserve_percentage,
  COUNT(DISTINCT r.shop_id) / COUNT(DISTINCT b.shop_id) AS pct_with_reserve
FROM `sdp-prd-cti-data.intermediate.shop_current_business_model` b
JOIN `sdp-prd-cti-data.intermediate.shop_current_shopify_payments_status` s
  ON b.shop_id = s.shop_id
LEFT JOIN `sdp-prd-cti-data.intermediate.shop_chargeback_rates_current` c
  ON b.shop_id = c.shop_id
LEFT JOIN `sdp-prd-cti-data.base.base__shopify_payments_reserve_configurations` r
  ON b.shop_id = r.shop_id AND r.is_active = TRUE
WHERE s.account_active = TRUE
  AND c.transaction_count >= 100
GROUP BY b.business_model
ORDER BY avg_chargeback_rate DESC
```

### Plan Type Risk Analysis

```sql
-- Analyze risk metrics by plan type
SELECT 
  s.plan_name,
  COUNT(DISTINCT s.id) AS shop_count,
  AVG(c.chargeback_rate) AS avg_chargeback_rate,
  APPROX_QUANTILES(c.chargeback_rate, 100)[OFFSET(50)] AS median_chargeback_rate
FROM `shopify-dw.shopify.shops` s
JOIN `sdp-prd-cti-data.intermediate.shop_current_shopify_payments_status` status
  ON s.id = status.shop_id
JOIN `sdp-prd-cti-data.intermediate.shop_chargeback_rates_current` c
  ON s.id = c.shop_id
WHERE status.account_active = TRUE
  AND c.transaction_count >= 100
GROUP BY s.plan_name
ORDER BY avg_chargeback_rate DESC
```

---

*Last Updated: May 2024* 