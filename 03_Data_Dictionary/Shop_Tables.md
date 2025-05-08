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
| `provider_name_array` | STRING (REPEATED) | Array containing names of all the providers which support accounts for the shop, currently can be Stripe or PayPal |
| `latest_created_provider_name` | STRING | Name of the provider which support the most recently created account, currently can be Stripe or PayPal |
| `earliest_provider_account_created_at` | TIMESTAMP | When considering the latest account for each provider, this is the timestamp when the earliest such Shopify Payments account was created |
| `latest_provider_account_created_at` | TIMESTAMP | When considering the latest account for each provider, this is the timestamp when the latest such Shopify Payments account was created |
| `is_transfer_enabled` | BOOLEAN | Whether transfers are enabled for ANY account associated with a shop across providers |
| `is_charge_enabled` | BOOLEAN | Whether charges are enabled for ANY account associated with a shop across providers |
| `rejected_reason` | STRING | Reason of rejection for the most recently created provider account, when there exists such a rejected account |
| `verification_disabled_reason` | STRING | Verification disabled reasons for the most recently created provider account, when there exists such a rejected account |
| `latest_created_shopify_payments_status` | STRING | CTI-specific status of the latest Shopify Payments account created for a shop |
| `shopify_payments_status` | STRING | CTI-specific status of the Shopify Payments account across providers |

**Usage Notes**:
- This table is essential for filtering analyses to only include active shops
- The `shopify_payments_status` field is the primary indicator of account status
- Common values for `shopify_payments_status` include:
  - `active`: Shop can process payments
  - `transfers paused`: The account's transfers were disabled after previously being enabled
  - `rejected`: Application was rejected
  - `setup incomplete`: Account has not been set up yet

**Example Query**:
```sql
-- Find all active shops with Shopify Payments enabled
SELECT 
  shop_id,
  shopify_payments_status,
  latest_provider_account_created_at AS created_at
FROM `sdp-prd-cti-data.intermediate.shop_current_shopify_payments_status`
WHERE is_charge_enabled = TRUE
  AND shopify_payments_status = 'active'
```

### shopify-dw.accounts_and_administration.shop_profile_current

**Description**:  
The master table containing core information about Shopify merchants. Contains shop creation dates, plan information, and other shop-level attributes.

**Update Frequency**: Daily

**Primary Keys**: `shop_id`

**Schema**:

| Field Name | Type | Description |
|------------|------|-------------|
| `shop_id` | INTEGER | Unique identifier for a shop |
| `organization_id` | INTEGER | The natural key identifier of the Organization that the shop belongs to |
| `billing_account_id` | INTEGER | Natural key identifier for the billing account |
| `created_at` | TIMESTAMP | When the shop was created |
| `domain` | STRING | The domain at which the shop is hosted |
| `permanent_domain` | STRING | The Shopify domain with which the shop is permanently assigned |
| `name` | STRING | Name of the shop |
| `company_name` | STRING | The company name of the shop |
| `country_code` | STRING | Two-letter country code for the shop's location |
| `province_state` | STRING | The province or state of the shop's location |
| `primary_timezone` | STRING | The primary timezone of the shop |
| `primary_locale` | STRING | The locale setting of the shop |
| `city` | STRING | The user-submitted value for the city that the shop is located |
| `shop_currency` | STRING | The currency that is set for the shop |
| `is_password_enabled` | BOOLEAN | Boolean indicating if the storefront has a password enabled for viewing |
| `is_manual_tax` | BOOLEAN | Boolean indicating if the storefront has manual tax |

**Usage Notes**:
- Essential for enriching risk analyses with shop age and plan information
- The `shop_id` field matches `shop_id` in payment and transaction tables
- The column `city` is not standardized and should be used with caution

**Example Query**:
```sql
-- Get shop information for analysis
SELECT 
  s.shop_id,
  s.created_at,
  DATE_DIFF(CURRENT_DATE(), DATE(s.created_at), DAY) AS shop_age_days,
  s.country_code,
  s.primary_locale
FROM shopify-dw.accounts_and_administration.shop_profile_current s
WHERE s.country_code = 'US'
ORDER BY s.created_at DESC
LIMIT 100
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

### shopify-dw.finance.shop_gmv_current

**Description**:  
Contains information about a shop's Gross Merchandise Value (GMV), which is useful for understanding a shop's sales volume.

**Update Frequency**: Daily

**Primary Keys**: `shop_id`

**Schema**:

| Field Name | Type | Description |
|------------|------|-------------|
| `shop_id` | INTEGER | Unique identifier for a shop |
| `gmv_usd_l365d` | NUMERIC | GMV in USD for the last 365 days |
| `cumulative_gmv_usd` | NUMERIC | Total cumulative GMV in USD |
| `first_order_with_gmv_date` | DATE | Date of first order with GMV |
| `date` | DATE | Date of the GMV snapshot |

**Usage Notes**:
- Helpful for understanding a shop's sales volume and history
- Can be correlated with risk metrics to assess risk relative to GMV
- Only includes shops with at least one transaction with non-zero GMV in the previous 365 days

**Example Query**:
```sql
-- Find shops with high GMV
SELECT 
  shop_id,
  gmv_usd_l365d,
  cumulative_gmv_usd,
  first_order_with_gmv_date
FROM `shopify-dw.finance.shop_gmv_current`
WHERE gmv_usd_l365d > 1000000
ORDER BY gmv_usd_l365d DESC
LIMIT 100
```

## How to Join Shop Tables

### Joining with Transaction Data

```sql
-- Join shop status with transaction data
SELECT 
  s.shop_id,
  s.shopify_payments_status,
  s.is_charge_enabled,
  COUNT(t.transaction_id) AS transaction_count,
  SUM(t.amount) AS total_amount
FROM `sdp-prd-cti-data.intermediate.shop_current_shopify_payments_status` s
JOIN `shopify-dw.money_products.order_transactions_payments_summary` t
  ON s.shop_id = t.shop_id
WHERE s.is_charge_enabled = TRUE
  AND t.created_at >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 30 DAY)
GROUP BY 
  s.shop_id,
  s.shopify_payments_status,
  s.is_charge_enabled
```

### Joining with Chargeback Data

```sql
-- Join shop information with chargeback data
SELECT 
  s.shop_id,
  s.name,
  s.created_at AS shop_created_at,
  c.chargeback_rate,
  c.transaction_count
FROM `shopify-dw.accounts_and_administration.shop_profile_current` s
JOIN `sdp-prd-cti-data.intermediate.shop_chargeback_rates_current` c
  ON s.shop_id = c.shop_id
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
WHERE s.is_charge_enabled = TRUE
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

3. **Temporal Fields Documentation**:

| Field Name | Data Type | Comparison Function | Example |
|------------|-----------|---------------------|---------|
| created_at | TIMESTAMP | TIMESTAMP_SUB() | `WHERE created_at >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 7 DAY)` |
| last_updated_at | TIMESTAMP | TIMESTAMP_SUB() | `WHERE last_updated_at >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 1 DAY)` |
| first_order_with_gmv_date | DATE | DATE_SUB() | `WHERE first_order_with_gmv_date >= DATE_SUB(CURRENT_DATE(), INTERVAL 30 DAY)` |

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
    shop_id,
    DATE_DIFF(CURRENT_DATE(), DATE(created_at), DAY) AS age_days,
    CASE
      WHEN DATE_DIFF(CURRENT_DATE(), DATE(created_at), DAY) < 30 THEN 'New (< 30 days)'
      WHEN DATE_DIFF(CURRENT_DATE(), DATE(created_at), DAY) < 90 THEN 'Recent (30-90 days)'
      WHEN DATE_DIFF(CURRENT_DATE(), DATE(created_at), DAY) < 365 THEN 'Established (90-365 days)'
      ELSE 'Mature (365+ days)'
    END AS age_category
  FROM `shopify-dw.accounts_and_administration.shop_profile_current`
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
WHERE s.is_charge_enabled = TRUE
  AND c.transaction_count >= 100
GROUP BY b.business_model
ORDER BY avg_chargeback_rate DESC
```

### GMV Range Analysis

```sql
-- Analyze risk metrics by GMV range
SELECT 
  CASE
    WHEN g.gmv_usd_l365d < 10000 THEN 'Under $10K'
    WHEN g.gmv_usd_l365d < 100000 THEN '$10K - $100K'
    WHEN g.gmv_usd_l365d < 1000000 THEN '$100K - $1M'
    ELSE 'Over $1M'
  END AS gmv_category,
  COUNT(DISTINCT g.shop_id) AS shop_count,
  AVG(c.chargeback_rate) AS avg_chargeback_rate,
  APPROX_QUANTILES(c.chargeback_rate, 100)[OFFSET(50)] AS median_chargeback_rate
FROM `shopify-dw.finance.shop_gmv_current` g
JOIN `sdp-prd-cti-data.intermediate.shop_current_shopify_payments_status` s
  ON g.shop_id = s.shop_id
JOIN `sdp-prd-cti-data.intermediate.shop_chargeback_rates_current` c
  ON g.shop_id = c.shop_id
WHERE s.is_charge_enabled = TRUE
  AND c.transaction_count >= 100
GROUP BY gmv_category
ORDER BY 
  CASE
    WHEN gmv_category = 'Under $10K' THEN 1
    WHEN gmv_category = '$10K - $100K' THEN 2
    WHEN gmv_category = '$100K - $1M' THEN 3
    WHEN gmv_category = 'Over $1M' THEN 4
    ELSE 5
  END
```

---