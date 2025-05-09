# Shop Tables

## Introduction
This document provides a comprehensive reference for shop-related tables used in credit risk analysis at Shopify. These tables contain vital information about merchant accounts, their status, and key attributes essential for identifying merchants, understanding their account status, and correlating risk metrics with shop-level data.

## Table of Contents
- [Primary Tables](#primary-tables)
  - [shop_current_shopify_payments_status](#sdp-prd-cti-dataintermediateshop_current_shopify_payments_status)
  - [shop_profile_current](#shopify-dwaccounts_and_administrationshop_profile_current)
  - [shop_current_business_model](#sdp-prd-cti-dataintermediateshop_current_business_model)
- [Secondary Tables](#secondary-tables)
  - [shop_metadata](#sdp-prd-cti-dataintermediateshop_metadata)
  - [shop_gmv_current](#shopify-dwfinanceshop_gmv_current)
- [How to Join Shop Tables](#how-to-join-shop-tables)
- [Data Quality Considerations](#data-quality-considerations)
- [Common Analysis Patterns](#common-analysis-patterns)

## Primary Tables

### sdp-prd-cti-data.intermediate.shop_current_shopify_payments_status

**Description**:  
This table provides the current Shopify Payments activation status for each shop. It's used to determine whether a shop can process payments through Shopify Payments and their current account status.

**Update Frequency**: Daily

**Primary Keys**: `shop_id`

**SQL Query**:
```sql
-- Find all active shops with Shopify Payments enabled
SELECT 
  shop_id,                                           -- Unique identifier for the shop
  shopify_payments_status,                           -- Current payments status (active, rejected, etc.)
  latest_provider_account_created_at AS created_at   -- When the account was created
FROM `sdp-prd-cti-data.intermediate.shop_current_shopify_payments_status`
WHERE is_charge_enabled = TRUE                       -- Only shops that can process payments
  AND shopify_payments_status = 'active'             -- Only shops with active status
```

**Description**:
This query retrieves all shops that currently have an active Shopify Payments account and are able to process charges. The `is_charge_enabled` field indicates whether the shop can accept payments, while the `shopify_payments_status` field indicates the current status of their account.

The table contains a comprehensive view of shops' payment processing capabilities, with status values including:
- `active`: Shop can process payments normally
- `transfers paused`: The account's transfers were disabled after previously being enabled
- `rejected`: Application was rejected for Shopify Payments
- `setup incomplete`: Account has not been fully set up yet

**Example**:
A typical row might show a shop with ID 59428569294 having "active" status, created on 2024-09-08, indicating this merchant can currently process payments through Shopify Payments.

### shopify-dw.accounts_and_administration.shop_profile_current

**Description**:  
The master table containing core information about Shopify merchants. Contains shop creation dates, plan information, and other shop-level attributes.

**Update Frequency**: Daily

**Primary Keys**: `shop_id`

**SQL Query**:
```sql
-- Get shop information for analysis
SELECT 
  s.shop_id,                                                   -- Unique identifier for the shop
  s.created_at,                                                -- When the shop was created
  DATE_DIFF(CURRENT_DATE(), DATE(s.created_at), DAY) AS shop_age_days,  -- Shop age in days
  s.country_code,                                              -- Country code for the shop
  s.primary_locale                                             -- Primary language for the shop
FROM `shopify-dw.accounts_and_administration.shop_profile_current` s
WHERE s.country_code = 'US'                                    -- Filter for US shops only
ORDER BY s.created_at DESC                                     -- Sort by newest shops first
```

**Description**:
This query retrieves basic profile information for shops located in the United States, calculating their age in days. It's useful for analyzing shop attributes based on location and age.

The `shop_profile_current` table contains essential merchant information including:
- Shop identification (name, domain, ID)
- Geographic data (country, province/state, city)
- Temporal data (creation date)
- Localization settings (timezone, locale)
- Plan type information

This table serves as the foundation for most shop-related analyses and can be joined with other tables to enrich risk profiles.

**Example**:
Results would include newer US-based shops with their shop ID, creation date, age in days, and localization preferences, enabling analysis of shop characteristics by country and age cohort.

### sdp-prd-cti-data.intermediate.shop_current_business_model

**Description**:  
Contains information about the merchant's business model classification, which is relevant for risk assessment.

**Update Frequency**: Daily

**Primary Keys**: `shop_id`

**SQL Query**:
```sql
-- Analyze chargeback rates by business model
SELECT 
  b.business_model,                                           -- Business model classification 
  COUNT(DISTINCT c.shop_id) AS shop_count,                    -- Number of unique shops
  AVG(c.chargeback_rate) AS avg_chargeback_rate,              -- Average chargeback rate
  APPROX_QUANTILES(c.chargeback_rate, 100)[OFFSET(50)] AS median_chargeback_rate,  -- Median chargeback rate
  MAX(c.chargeback_rate) AS max_chargeback_rate               -- Maximum chargeback rate
FROM `sdp-prd-cti-data.intermediate.shop_current_business_model` b
JOIN `sdp-prd-cti-data.intermediate.shop_chargeback_rates_current` c
  ON b.shop_id = c.shop_id
WHERE c.transaction_count >= 100                              -- Only shops with sufficient transaction volume
GROUP BY b.business_model
ORDER BY avg_chargeback_rate DESC                             -- Sort by highest average chargeback rate first
```

**Description**:
This query performs an analysis of chargeback rates across different business models, showing the average, median, and maximum rates for each model. It only includes shops with at least 100 transactions to ensure statistical significance.

Business model is a key factor in risk assessment as certain models have inherently different risk profiles. Common business model values include:
- 'Dropshipping' - Merchants that don't hold inventory and ship directly from suppliers
- 'Pre-order/Made to order/Limited release' - Merchants that take payment before fulfillment
- 'Retail/Domestic' - Traditional retail businesses with inventory
- 'Replica' - Businesses selling replica products

Understanding these distinctions helps in tailoring risk mitigation strategies to specific business types.

**Example**:
The results might show that 'Dropshipping' business models have an average chargeback rate of 0.023 (2.3%) across 1,253 shops, with a median of 0.018 and a maximum of 0.157, while 'Retail/Domestic' models have lower average rates of 0.009 (0.9%).

## Secondary Tables

### sdp-prd-cti-data.intermediate.shop_metadata

**Description**:  
Consolidated shop metadata from various sources, providing a comprehensive view of shop attributes.

**Update Frequency**: Daily

**Primary Keys**: `shop_id`

**SQL Query**:
```sql
-- Compare risk metrics across different shop attributes
SELECT 
  CASE
    WHEN DATE_DIFF(CURRENT_DATE(), DATE(s.created_at), DAY) < 30 THEN 'New (< 30 days)'
    WHEN DATE_DIFF(CURRENT_DATE(), DATE(s.created_at), DAY) < 90 THEN 'Recent (30-90 days)'
    ELSE 'Established (90+ days)'
  END AS shop_age_category,                                    -- Shop age grouping
  s.has_plus,                                                  -- Whether shop has Shopify Plus
  COUNT(DISTINCT s.shop_id) AS shop_count,                     -- Number of shops in segment
  AVG(c.chargeback_rate) AS avg_chargeback_rate                -- Average chargeback rate
FROM `sdp-prd-cti-data.intermediate.shop_metadata` s
JOIN `sdp-prd-cti-data.intermediate.shop_chargeback_rates_current` c
  ON s.shop_id = c.shop_id
WHERE c.transaction_count >= 100                               -- Only shops with sufficient transaction volume
GROUP BY 
  shop_age_category,
  s.has_plus
ORDER BY 
  shop_age_category,
  s.has_plus
```

**Description**:
This query segments shops by age category and Shopify Plus status, then calculates the average chargeback rate for each segment. It provides insights into how shop age and premium plan status correlate with risk metrics.

The `shop_metadata` table consolidates various important attributes about shops from different sources, including:
- Plan details (Plus status, plan type)
- Shop age and activity metrics
- Marketing and sales performance indicators
- Technical configuration details

This consolidated view makes it easier to perform multifaceted analyses without needing to join multiple tables.

**Example**:
Results might show that new shops (less than 30 days old) without Shopify Plus have an average chargeback rate of 0.032 (3.2%), while established shops (90+ days) with Shopify Plus have a much lower rate of 0.007 (0.7%).

### shopify-dw.finance.shop_gmv_current

**Description**:  
Contains information about a shop's Gross Merchandise Value (GMV), which is useful for understanding a shop's sales volume.

**Update Frequency**: Daily

**Primary Keys**: `shop_id`

**SQL Query**:
```sql
-- Find shops with high GMV
SELECT 
  shop_id,                                                    -- Unique identifier for the shop
  gmv_usd_l365d,                                              -- GMV in USD for last 365 days
  cumulative_gmv_usd,                                         -- Total lifetime GMV in USD
  first_order_with_gmv_date                                   -- Date of first order with GMV
FROM `shopify-dw.finance.shop_gmv_current`
WHERE gmv_usd_l365d > 1000000                                 -- Only shops with over $1M in annual GMV
ORDER BY gmv_usd_l365d DESC                                   -- Sort by highest GMV first
```

**Description**:
This query identifies high-volume merchants with over $1 million in Gross Merchandise Value (GMV) in the past year. It provides their annual GMV, lifetime GMV, and the date of their first order with GMV.

The `shop_gmv_current` table contains key sales metrics including:
- GMV over various time periods (7 days, 30 days, 90 days, 365 days)
- Cumulative (lifetime) GMV
- First order date information
- Transaction counts

This information is valuable for understanding a shop's size, growth trajectory, and sales history when assessing risk.

**Example**:
Results might show a shop with ID 12345 having $4.2 million in GMV over the past year, $12.7 million in cumulative GMV, with their first order occurring on 2019-04-15.

## How to Join Shop Tables

### Joining with Transaction Data

```sql
-- Join shop status with transaction data
SELECT 
  s.shop_id,
  s.shopify_payments_status,                                  -- Current payments status
  s.is_charge_enabled,                                        -- Whether charges are enabled
  COUNT(t.transaction_id) AS transaction_count,               -- Count of transactions
  SUM(t.amount) AS total_amount                               -- Sum of transaction amounts
FROM `sdp-prd-cti-data.intermediate.shop_current_shopify_payments_status` s
JOIN `shopify-dw.money_products.order_transactions_payments_summary` t
  ON s.shop_id = t.shop_id
WHERE s.is_charge_enabled = TRUE                              -- Only shops that can process payments
  AND t.created_at >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 30 DAY)  -- Only recent transactions
GROUP BY 
  s.shop_id,
  s.shopify_payments_status,
  s.is_charge_enabled
```

**Description**:
This query demonstrates how to join shop payment status information with transaction data to analyze recent processing volumes for active shops. It counts transactions and sums transaction amounts for each shop over the past 30 days.

### Joining with Chargeback Data

```sql
-- Join shop information with chargeback data
SELECT 
  s.shop_id,
  s.name,                                                     -- Shop name
  s.created_at AS shop_created_at,                            -- Shop creation date
  c.chargeback_rate,                                          -- Chargeback rate
  c.transaction_count                                         -- Number of transactions
FROM `shopify-dw.accounts_and_administration.shop_profile_current` s
JOIN `sdp-prd-cti-data.intermediate.shop_chargeback_rates_current` c
  ON s.shop_id = c.shop_id
WHERE c.transaction_count >= 100                              -- Only shops with sufficient transaction volume
  AND c.chargeback_rate > 0.01                                -- Only shops with chargeback rate > 1%
ORDER BY c.chargeback_rate DESC                               -- Sort by highest chargeback rate first
```

**Description**:
This query joins shop profile information with chargeback data to identify shops with elevated chargeback rates (above 1%). It includes basic shop information along with their chargeback metrics, focusing on shops with sufficient transaction volume for statistical significance.

### Joining with Business Model

```sql
-- Join shop status, business model, and chargeback data
SELECT 
  s.shop_id,
  s.shopify_payments_status,                                  -- Current payments status
  b.business_model,                                           -- Business model classification
  c.chargeback_rate,                                          -- Chargeback rate
  c.transaction_count                                         -- Number of transactions
FROM `sdp-prd-cti-data.intermediate.shop_current_shopify_payments_status` s
JOIN `sdp-prd-cti-data.intermediate.shop_current_business_model` b
  ON s.shop_id = b.shop_id
JOIN `sdp-prd-cti-data.intermediate.shop_chargeback_rates_current` c
  ON s.shop_id = c.shop_id
WHERE s.is_charge_enabled = TRUE                              -- Only shops that can process payments
  AND c.transaction_count >= 100                              -- Only shops with sufficient transaction volume
ORDER BY c.chargeback_rate DESC                               -- Sort by highest chargeback rate first
```

**Description**:
This query demonstrates a three-way join between shop payment status, business model, and chargeback data. It's useful for comprehensive risk analysis that considers both the business model and payment status along with chargeback metrics.

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
    DATE_DIFF(CURRENT_DATE(), DATE(created_at), DAY) AS age_days,   -- Calculate shop age in days
    CASE
      WHEN DATE_DIFF(CURRENT_DATE(), DATE(created_at), DAY) < 30 THEN 'New (< 30 days)'
      WHEN DATE_DIFF(CURRENT_DATE(), DATE(created_at), DAY) < 90 THEN 'Recent (30-90 days)'
      WHEN DATE_DIFF(CURRENT_DATE(), DATE(created_at), DAY) < 365 THEN 'Established (90-365 days)'
      ELSE 'Mature (365+ days)'
    END AS age_category                                              -- Group shops by age category
  FROM `shopify-dw.accounts_and_administration.shop_profile_current`
)

SELECT 
  a.age_category,
  COUNT(DISTINCT a.shop_id) AS shop_count,                           -- Count of shops in each age category
  AVG(c.chargeback_rate) AS avg_chargeback_rate,                     -- Average chargeback rate
  APPROX_QUANTILES(c.chargeback_rate, 100)[OFFSET(50)] AS median_chargeback_rate,  -- Median chargeback rate
  APPROX_QUANTILES(c.chargeback_rate, 100)[OFFSET(75)] AS p75_chargeback_rate      -- 75th percentile
FROM shop_age a
JOIN `sdp-prd-cti-data.intermediate.shop_chargeback_rates_current` c
  ON a.shop_id = c.shop_id
WHERE c.transaction_count >= 100                                     -- Only shops with sufficient transaction volume
GROUP BY a.age_category
ORDER BY 
  CASE
    WHEN a.age_category = 'New (< 30 days)' THEN 1
    WHEN a.age_category = 'Recent (30-90 days)' THEN 2
    WHEN a.age_category = 'Established (90-365 days)' THEN 3
    WHEN a.age_category = 'Mature (365+ days)' THEN 4
    ELSE 5
  END                                                                -- Order results by age category
```

**Description**:
This query analyzes how chargeback risk correlates with shop age by segmenting shops into age categories and calculating chargeback statistics for each segment. It uses a Common Table Expression (CTE) to first categorize shops by age, then joins with chargeback data to calculate risk metrics.

The results provide insights into the relationship between merchant tenure and risk, which is a critical factor in credit risk assessment.

### Business Model Risk Comparison

```sql
-- Compare risk metrics across business models
SELECT 
  b.business_model,                                                  -- Business model classification
  COUNT(DISTINCT b.shop_id) AS shop_count,                           -- Count of shops per business model
  AVG(c.chargeback_rate) AS avg_chargeback_rate,                     -- Average chargeback rate
  APPROX_QUANTILES(c.chargeback_rate, 100)[OFFSET(50)] AS median_chargeback_rate,  -- Median chargeback rate
  AVG(r.percentage) AS avg_reserve_percentage,                       -- Average reserve percentage
  COUNT(DISTINCT r.shop_id) / COUNT(DISTINCT b.shop_id) AS pct_with_reserve  -- Percentage with reserves
FROM `sdp-prd-cti-data.intermediate.shop_current_business_model` b
JOIN `sdp-prd-cti-data.intermediate.shop_current_shopify_payments_status` s
  ON b.shop_id = s.shop_id
LEFT JOIN `sdp-prd-cti-data.intermediate.shop_chargeback_rates_current` c
  ON b.shop_id = c.shop_id
LEFT JOIN `sdp-prd-cti-data.base.base__shopify_payments_reserve_configurations` r
  ON b.shop_id = r.shop_id AND r.is_active = TRUE
WHERE s.is_charge_enabled = TRUE                                     -- Only shops that can process payments
  AND c.transaction_count >= 100                                     -- Only shops with sufficient transaction volume
GROUP BY b.business_model
ORDER BY avg_chargeback_rate DESC                                    -- Sort by highest average chargeback rate first
```

**Description**:
This query provides a comprehensive comparison of risk metrics across different business models, including chargeback rates and reserve configurations. It calculates both the average reserve percentage and the proportion of shops with active reserves for each business model.

These insights help in understanding which business models present higher risk and how that correlates with the application of risk mitigation measures like reserves.

### GMV Range Analysis

```sql
-- Analyze risk metrics by GMV range
SELECT 
  CASE
    WHEN g.gmv_usd_l365d < 10000 THEN 'Under $10K'
    WHEN g.gmv_usd_l365d < 100000 THEN '$10K - $100K'
    WHEN g.gmv_usd_l365d < 1000000 THEN '$100K - $1M'
    ELSE 'Over $1M'
  END AS gmv_category,                                               -- GMV range category
  COUNT(DISTINCT g.shop_id) AS shop_count,                           -- Count of shops in each GMV category
  AVG(c.chargeback_rate) AS avg_chargeback_rate,                     -- Average chargeback rate
  APPROX_QUANTILES(c.chargeback_rate, 100)[OFFSET(50)] AS median_chargeback_rate  -- Median chargeback rate
FROM `shopify-dw.finance.shop_gmv_current` g
JOIN `sdp-prd-cti-data.intermediate.shop_current_shopify_payments_status` s
  ON g.shop_id = s.shop_id
JOIN `sdp-prd-cti-data.intermediate.shop_chargeback_rates_current` c
  ON g.shop_id = c.shop_id
WHERE s.is_charge_enabled = TRUE                                     -- Only shops that can process payments
  AND c.transaction_count >= 100                                     -- Only shops with sufficient transaction volume
GROUP BY gmv_category
ORDER BY 
  CASE
    WHEN gmv_category = 'Under $10K' THEN 1
    WHEN gmv_category = '$10K - $100K' THEN 2
    WHEN gmv_category = '$100K - $1M' THEN 3
    WHEN gmv_category = 'Over $1M' THEN 4
    ELSE 5
  END                                                                -- Order results by GMV category
```

**Description**:
This query examines the relationship between shop size (measured by GMV) and chargeback risk by categorizing shops into GMV ranges and calculating risk metrics for each segment. It helps identify whether larger or smaller merchants tend to have different risk profiles.

The results can inform risk assessment and underwriting strategies based on merchant size, allowing for more tailored risk management approaches.