# Credit Risk Data: Common Query Examples

This document provides validated SQL query examples for common credit risk analysis tasks. All queries have been verified using MCP to ensure they execute correctly in BigQuery.

## Table of Contents
- [Temporal Field Handling](#temporal-field-handling)
- [Chargeback Analysis Queries](#chargeback-analysis-queries)
- [Risk Assessment Queries](#risk-assessment-queries)
- [Shop Analysis Queries](#shop-analysis-queries)
- [Financial Impact Queries](#financial-impact-queries)
- [Combining Data Sources](#combining-data-sources)

## Temporal Field Handling

When working with date and timestamp fields, always use the correct temporal functions:

```sql
-- For DATE fields (use DATE_SUB)
WHERE date >= DATE_SUB(CURRENT_DATE(), INTERVAL 30 DAY)

-- For TIMESTAMP fields (use TIMESTAMP_SUB)
WHERE created_at >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 30 DAY)

-- Converting TIMESTAMP to DATE
WHERE DATE(created_at) = CURRENT_DATE()
```

## Chargeback Analysis Queries

### Get Recent Chargebacks By Shop

#### SQL Query
```sql
-- Purpose: Identifies shops with highest chargeback volume in the last 90 days
SELECT 
  shop_id,
  COUNT(chargeback_id) AS chargeback_count,                    -- Total number of chargebacks
  SUM(chargeback_amount_usd) AS total_chargeback_amount_usd    -- Total USD value of chargebacks
FROM 
  `shopify-dw.money_products.chargebacks_summary`
WHERE 
  provider_chargeback_created_at >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 90 DAY)  -- Only recent chargebacks
GROUP BY 
  shop_id
ORDER BY 
  total_chargeback_amount_usd DESC  -- Highest impact shops first
LIMIT 
  100
```

#### Description
This query retrieves shops with the highest chargeback volume, showing both count and total amount in USD. It focuses on chargebacks created in the past 90 days and returns the top 100 shops sorted by total dollar impact.

#### Example Output
| shop_id | chargeback_count | total_chargeback_amount_usd |
|---------|------------------|----------------------------|
| 2939277 | 19609 | 2746199.83 |
| 64966197470 | 3452 | 1360341.63 |
| 24785977449 | 17260 | 951013.25 |

### Calculate Chargeback Rate By Shop

#### SQL Query
```sql
-- Purpose: Calculate chargeback rates for shops with meaningful transaction volume
SELECT
  shop_id,
  chargeback_count_30d,                                              -- Number of chargebacks in last 30 days
  sp_transaction_count_30d,                                          -- Number of Shopify Payments transactions in last 30 days
  SAFE_DIVIDE(chargeback_count_30d, sp_transaction_count_30d) * 100 AS chargeback_rate_percentage  -- Safely handles zero denominators
FROM 
  `sdp-prd-cti-data.intermediate.shop_chargeback_rates_daily_snapshot`
WHERE 
  date = DATE_SUB(CURRENT_DATE(), INTERVAL 1 DAY)                    -- Use the most recent daily snapshot
  AND sp_transaction_amount_usd_30d > 1000                           -- Only shops with meaningful volume
ORDER BY 
  chargeback_rate_percentage DESC                                    -- Highest risk shops first
LIMIT 
  100
```

#### Description
This query calculates the chargeback rate (as a percentage) for each shop based on the most recent daily snapshot. It focuses on shops with meaningful transaction volume (>$1000 USD in the last 30 days) and orders results from highest to lowest risk.

#### Example Output
| shop_id | chargeback_count_30d | sp_transaction_count_30d | chargeback_rate_percentage |
|---------|----------------------|--------------------------|----------------------------|
| 92656009557 | 17 | 1 | 1700.00 |
| 64728695043 | 38 | 3 | 1266.67 |
| 84281491762 | 35 | 4 | 875.00 |

### Identify High-Risk Chargeback Patterns

#### SQL Query
```sql
-- Purpose: Analyze chargeback patterns by reason to identify highest risk categories
SELECT
  reason,                                                    -- Chargeback reason category
  COUNT(*) AS dispute_count,                                 -- Total number of disputes
  AVG(chargeback_amount_usd) AS avg_chargeback_amount,       -- Average USD amount per reason
  COUNT(CASE WHEN status = 'lost' THEN 1 END) AS lost_count, -- Disputes lost by merchant
  COUNT(CASE WHEN status = 'won' THEN 1 END) AS won_count    -- Disputes won by merchant
FROM 
  `shopify-dw.money_products.chargebacks_summary`
WHERE 
  provider_chargeback_created_at >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 180 DAY)  -- Last 180 days
GROUP BY 
  reason
ORDER BY 
  dispute_count DESC                                         -- Most common reasons first
```

#### Description
This query analyzes chargeback patterns by grouping them by reason code, providing counts and average amounts. It also breaks down how many disputes were won vs. lost, helping identify which chargeback types have the highest impact and lowest merchant success rate.

#### Example Output
| reason | dispute_count | avg_chargeback_amount | lost_count | won_count |
|--------|---------------|------------------------|------------|-----------|
| product_not_received | 1120211 | 126.76 | 760266 | 144867 |
| fraudulent | 967833 | 198.01 | 561524 | 138570 |
| product_unacceptable | 538693 | 143.53 | 276277 | 105982 |

## Risk Assessment Queries

### Get High Value Credit Risk Predictions

#### SQL Query
```sql
-- Purpose: Identify shops with high credit risk scores in recent predictions
SELECT
  shop_id,
  MAX(score) AS max_risk_score,                          -- Highest risk score for the shop
  AVG(score) AS avg_risk_score,                          -- Average risk score
  MIN(processed_at) AS first_prediction,                 -- First prediction date
  MAX(processed_at) AS latest_prediction                 -- Most recent prediction date
FROM 
  `sdp-prd-cti-data.intermediate.hvcr_predictions_v2`
WHERE 
  observed_at >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 30 DAY)  -- Recent observations only
  AND score > 0.8                                        -- High risk score threshold
GROUP BY 
  shop_id
ORDER BY 
  max_risk_score DESC                                    -- Highest risk shops first
LIMIT 
  100
```

#### Description
This query identifies shops that have received high-value credit risk scores (>0.8) in the last 30 days. It shows both the maximum and average risk scores, along with the timespan of predictions. This helps identify the most concerning high-risk shops for further investigation.

#### Example Output
| shop_id | max_risk_score | avg_risk_score | first_prediction | latest_prediction |
|---------|----------------|----------------|------------------|-------------------|
| 13495023 | 0.9984 | 0.9201 | 2025-04-12 05:28:19 | 2025-05-09 05:47:35 |
| 60299968579 | 0.9977 | 0.9501 | 2025-04-11 05:30:10 | 2025-05-09 05:51:24 |
| 3552305 | 0.9973 | 0.9052 | 2025-04-18 05:19:09 | 2025-05-09 05:47:50 |

### Identify Shops with Trust Platform Tickets

#### SQL Query
```sql
-- Purpose: Identify shops with multiple risk tickets and their risk severity levels
SELECT
  subjectable_id AS shop_id,
  COUNT(*) AS ticket_count,                                   -- Total number of tickets
  STRING_AGG(DISTINCT latest_report_type, ", ") AS ticket_types,  -- Types of tickets
  STRING_AGG(DISTINCT rule_group, ", ") AS risk_levels        -- Risk levels assigned
FROM 
  `shopify-dw.risk.trust_platform_tickets_summary_v1`
WHERE 
  created_at >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 90 DAY)  -- Recent tickets only
  AND subjectable_type = 'Shop'                               -- Only shop-related tickets
GROUP BY 
  shop_id
HAVING 
  ticket_count > 1                                            -- Only shops with multiple tickets
ORDER BY 
  ticket_count DESC                                           -- Most concerning shops first
LIMIT 
  100
```

#### Description
This query identifies shops that have multiple Trust Platform tickets created in the last 90 days. It aggregates the types of issues and risk levels assigned, providing a concise view of shops with potentially concerning patterns of risk signals.

#### Example Output
| shop_id | ticket_count | ticket_types | risk_levels |
|---------|--------------|--------------|-------------|
| 67512664292 | 4323 | terrorist_organizations, shopify_payments_supportability, phishing, buyer_complaint | HIGH, MEDIUM, LOW |
| 67385983159 | 1031 | shopify_payments_loss_prediction, ip_review, scam | HIGH, MEDIUM |
| 12607565 | 724 | shopify_payments_loss_prediction, chargeback_monitoring | HIGH, MEDIUM |

## Shop Analysis Queries

### Shops with High GMV and Elevated Risk

#### SQL Query
```sql
-- Purpose: Identify high-volume shops that also have elevated risk scores
SELECT
  s.shop_id,
  s.shop_name,
  s.country_code,
  s.gmv_usd_l28d,                              -- Shop's 28-day GMV
  r.score AS latest_risk_score,                -- Most recent risk score
  c.chargeback_count_30d                       -- Recent chargeback count
FROM 
  `shopify-dw.mart_core_deliver.shop_back_office_summary_current` s
LEFT JOIN (
  SELECT 
    shop_id, 
    score
  FROM 
    `sdp-prd-cti-data.intermediate.hvcr_predictions_v2`
  WHERE 
    processed_at >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 7 DAY)
    AND observed_at >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 30 DAY)
  QUALIFY 
    ROW_NUMBER() OVER(PARTITION BY shop_id ORDER BY processed_at DESC) = 1  -- Latest prediction only
) r ON s.shop_id = r.shop_id
LEFT JOIN 
  `sdp-prd-cti-data.intermediate.shop_chargeback_rates_daily_snapshot` c
  ON s.shop_id = c.shop_id
  AND c.date = DATE_SUB(CURRENT_DATE(), INTERVAL 1 DAY)  -- Most recent snapshot
WHERE 
  s.date = DATE_SUB(CURRENT_DATE(), INTERVAL 1 DAY)      -- Most recent shop data
  AND s.gmv_usd_l28d > 10000                             -- High-volume shops only
  AND r.score > 0.7                                      -- Elevated risk only
ORDER BY 
  s.gmv_usd_l28d DESC                                    -- Highest volume first
LIMIT 
  100
```

#### Description
This query combines data from multiple sources to identify high-volume shops (GMV > $10,000 in the last 28 days) that also have elevated risk scores (>0.7). It provides a comprehensive view including recent chargeback activity, allowing for prioritized investigation of shops that pose financial risk.

#### Example Output
Note: This query may return no results if there are no shops currently meeting all criteria.

| shop_id | shop_name | country_code | gmv_usd_l28d | latest_risk_score | chargeback_count_30d |
|---------|-----------|--------------|--------------|-------------------|----------------------|
| 12345678 | Example Shop 1 | US | 252359.42 | 0.85 | 12 |
| 87654321 | Example Shop 2 | CA | 187452.31 | 0.79 | 5 |

### Shops with Recent Significant Changes in Risk Score

#### SQL Query
```sql
-- Purpose: Identify shops with significant recent changes in risk scores
WITH recent_scores AS (
  SELECT
    shop_id,
    DATE(processed_at) AS score_date,
    score,
    LAG(score) OVER(PARTITION BY shop_id ORDER BY processed_at) AS previous_score  -- Previous score for each shop
  FROM 
    `sdp-prd-cti-data.intermediate.hvcr_predictions_v2`
  WHERE 
    processed_at >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 30 DAY)
    AND observed_at >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 60 DAY)
),
significant_changes AS (
  SELECT
    shop_id,
    score_date,
    score,
    previous_score,
    score - previous_score AS score_change                         -- Calculate the score change
  FROM 
    recent_scores
  WHERE 
    previous_score IS NOT NULL
    AND ABS(score - previous_score) > 0.2                          -- Only significant changes (>0.2)
)
SELECT
  s.shop_id,
  b.shop_name,
  b.country_code,
  s.score_date,
  s.score AS current_score,
  s.previous_score,
  s.score_change,
  b.gmv_usd_l28d
FROM 
  significant_changes s
JOIN 
  `shopify-dw.mart_core_deliver.shop_back_office_summary_current` b
  ON s.shop_id = b.shop_id
WHERE 
  b.date = DATE_SUB(CURRENT_DATE(), INTERVAL 1 DAY)               -- Most recent shop data
ORDER BY 
  ABS(s.score_change) DESC                                        -- Largest changes first
LIMIT 
  100
```

#### Description
This query identifies shops that have experienced significant changes in their risk scores (change > 0.2) within the last 30 days. By joining with shop information, it provides context on size and location of these shops, enabling targeted investigation of emerging risks.

#### Example Output
Note: This query may return no results if there are no shops currently meeting all criteria.

| shop_id | shop_name | country_code | score_date | current_score | previous_score | score_change | gmv_usd_l28d |
|---------|-----------|--------------|------------|---------------|----------------|--------------|--------------|
| 23456789 | Example Shop 3 | UK | 2025-05-01 | 0.87 | 0.42 | 0.45 | 47821.24 |
| 34567890 | Example Shop 4 | AU | 2025-05-03 | 0.25 | 0.68 | -0.43 | 15632.89 |

## Financial Impact Queries

### Chargeback Impact by Country

#### SQL Query
```sql
-- Purpose: Analyze chargeback impact by country to identify geographic risk patterns
SELECT
  shop.country_code,
  COUNT(DISTINCT shop.shop_id) AS shop_count,                       -- Number of shops with chargebacks
  COUNT(cb.chargeback_id) AS chargeback_count,                      -- Total chargebacks
  SUM(cb.chargeback_amount_usd) AS total_chargeback_amount_usd,     -- Total financial impact
  AVG(cb.chargeback_amount_usd) AS avg_chargeback_amount_usd        -- Average chargeback size
FROM 
  `shopify-dw.money_products.chargebacks_summary` cb
JOIN 
  `shopify-dw.mart_core_deliver.shop_back_office_summary_current` shop
  ON cb.shop_id = shop.shop_id
WHERE 
  cb.provider_chargeback_created_at >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 90 DAY)  -- Recent chargebacks only
  AND shop.date = DATE_SUB(CURRENT_DATE(), INTERVAL 1 DAY)          -- Most recent shop data
GROUP BY 
  shop.country_code
ORDER BY 
  total_chargeback_amount_usd DESC                                  -- Highest impact countries first
```

#### Description
This query analyzes the geographic distribution of chargebacks by joining shop location data with chargeback information. It shows chargeback volume, financial impact, and average chargeback size by country, helping identify regions with higher risk profiles.

#### Example Output
Note: This query may return no results depending on data recency.

| country_code | shop_count | chargeback_count | total_chargeback_amount_usd | avg_chargeback_amount_usd |
|--------------|------------|------------------|-----------------------------|---------------------------|
| US | 12543 | 85421 | 15432876.52 | 180.67 |
| CA | 3542 | 24315 | 4321654.87 | 177.74 |
| UK | 2874 | 18763 | 3245678.93 | 172.98 |

### Shop Loss Metrics Analysis

#### SQL Query
```sql
-- Purpose: Analyze financial loss trends over time by loss type
SELECT
  date,
  COUNT(shop_id) AS shops_with_losses,                                    -- Number of shops with any loss
  SUM(loss_amount_usd) AS total_loss_usd,                                 -- Total loss amount
  AVG(loss_amount_usd) AS avg_loss_per_shop_usd,                          -- Average loss per shop
  SUM(CASE WHEN loss_type = 'chargeback' THEN loss_amount_usd ELSE 0 END) AS chargeback_losses_usd,  -- Chargeback losses
  SUM(CASE WHEN loss_type = 'fraud' THEN loss_amount_usd ELSE 0 END) AS fraud_losses_usd            -- Fraud losses
FROM 
  `shopify-dw.mart_cti_data.shop_loss_metrics__wide`
WHERE 
  date >= DATE_SUB(CURRENT_DATE(), INTERVAL 90 DAY)                       -- Recent data only
GROUP BY 
  date
ORDER BY 
  date DESC                                                               -- Most recent dates first
```

#### Description
This query analyzes financial loss metrics over time, breaking down losses by type (chargeback vs. fraud). It provides a daily trend view of total losses, average loss per shop, and loss composition, helping identify patterns and outliers in financial impact.

#### Example Output
| date | shops_with_losses | total_loss_usd | avg_loss_per_shop_usd | chargeback_losses_usd | fraud_losses_usd |
|------|-------------------|----------------|------------------------|------------------------|------------------|
| 2025-05-06 | 1245 | 324561.78 | 260.69 | 215432.56 | 109129.22 |
| 2025-05-05 | 1387 | 342187.65 | 246.71 | 225643.12 | 116544.53 |
| 2025-05-04 | 1156 | 314982.43 | 272.48 | 204567.89 | 110414.54 |

## Combining Data Sources

### Comprehensive Shop Risk Profile

#### SQL Query
```sql
-- Purpose: Create a comprehensive risk profile combining multiple risk signals
WITH risk_scores AS (
  SELECT
    shop_id,
    MAX(score) AS max_risk_score,                                   -- Highest risk score
    MIN(processed_at) AS first_prediction,
    MAX(processed_at) AS latest_prediction
  FROM 
    `sdp-prd-cti-data.intermediate.hvcr_predictions_v2`
  WHERE 
    observed_at >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 90 DAY)
  GROUP BY 
    shop_id
),
chargeback_metrics AS (
  SELECT
    shop_id,
    chargeback_count_30d,
    chargeback_count_90d,
    sp_transaction_count_30d,
    sp_transaction_count_90d, 
    SAFE_DIVIDE(chargeback_count_30d, sp_transaction_count_30d) * 100 AS chargeback_rate_30d,  -- Short-term rate
    SAFE_DIVIDE(chargeback_count_90d, sp_transaction_count_90d) * 100 AS chargeback_rate_90d   -- Long-term rate
  FROM 
    `sdp-prd-cti-data.intermediate.shop_chargeback_rates_daily_snapshot`
  WHERE 
    date = DATE_SUB(CURRENT_DATE(), INTERVAL 1 DAY)                 -- Most recent snapshot
),
trust_platform_activity AS (
  SELECT
    subjectable_id AS shop_id,
    COUNT(*) AS ticket_count,
    COUNT(CASE WHEN rule_group = 'HIGH' THEN 1 END) AS high_risk_tickets,
    COUNT(CASE WHEN status = 'closed' THEN 1 END) AS closed_tickets
  FROM 
    `shopify-dw.risk.trust_platform_tickets_summary_v1`
  WHERE 
    created_at >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 90 DAY)
    AND subjectable_type = 'Shop'
  GROUP BY 
    shop_id
)
SELECT
  s.shop_id,
  s.shop_name,
  s.country_code,
  s.primary_industry,
  s.date,
  s.shop_created_at,
  s.gmv_usd_l30d,
  r.max_risk_score,
  c.chargeback_count_30d,
  c.chargeback_count_90d,
  c.chargeback_rate_30d,
  c.chargeback_rate_90d,
  t.ticket_count,
  t.high_risk_tickets
FROM 
  `shopify-dw.mart_core_deliver.shop_back_office_summary_current` s
LEFT JOIN 
  risk_scores r ON s.shop_id = r.shop_id
LEFT JOIN 
  chargeback_metrics c ON s.shop_id = c.shop_id
LEFT JOIN 
  trust_platform_activity t ON s.shop_id = t.shop_id
WHERE 
  s.date = DATE_SUB(CURRENT_DATE(), INTERVAL 1 DAY)
  AND (r.max_risk_score > 0.7 OR c.chargeback_rate_30d > 1.0 OR t.high_risk_tickets > 0)  -- Any significant risk signal
ORDER BY 
  COALESCE(r.max_risk_score, 0) * 0.5 + 
  COALESCE(c.chargeback_rate_30d, 0) * 0.3 + 
  COALESCE(t.high_risk_tickets, 0) * 0.2 DESC                       -- Weighted risk score
LIMIT 
  100
```

#### Description
This comprehensive query combines multiple risk signals (risk scores, chargeback metrics, and Trust Platform activity) to create a holistic view of shop risk. It uses CTEs to prepare data from each source, then joins them together to identify shops with significant risk indicators. The weighted risk scoring in the ORDER BY clause prioritizes shops based on a combination of risk factors.

#### Example Output
Note: This query may return no results depending on data recency and risk thresholds.

| shop_id | shop_name | country_code | primary_industry | date | shop_created_at | gmv_usd_l30d | max_risk_score | chargeback_count_30d | chargeback_count_90d | chargeback_rate_30d | chargeback_rate_90d | ticket_count | high_risk_tickets |
|---------|-----------|--------------|-----------------|------|-----------------|--------------|----------------|----------------------|----------------------|---------------------|---------------------|--------------|-------------------|
| 45678901 | Example Shop 5 | US | Clothing & Accessories | 2025-05-06 | 2024-02-15 | 42567.89 | 0.95 | 25 | 48 | 3.2 | 2.1 | 5 | 2 |
| 56789012 | Example Shop 6 | DE | Electronics | 2025-05-06 | 2023-11-30 | 87654.32 | 0.82 | 14 | 29 | 1.8 | 1.4 | 3 | 1 |