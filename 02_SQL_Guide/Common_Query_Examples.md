# Credit Risk Data: Common Query Examples

This document provides validated SQL query examples for common credit risk analysis tasks. All queries have been verified using MCP to ensure they execute correctly in BigQuery.

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

```sql
SELECT 
  shop_id,
  COUNT(chargeback_id) AS chargeback_count,
  SUM(chargeback_amount_usd) AS total_chargeback_amount_usd
FROM 
  `shopify-dw.money_products.chargebacks_summary`
WHERE 
  provider_chargeback_created_at >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 90 DAY)
GROUP BY 
  shop_id
ORDER BY 
  total_chargeback_amount_usd DESC
LIMIT 
  100
```

### Calculate Chargeback Rate By Shop

```sql
SELECT
  shop_id,
  chargeback_count_30d,
  sp_transaction_amount_usd_30d,
  SAFE_DIVIDE(chargeback_count_30d, sp_transaction_count_30d) * 100 AS chargeback_rate_percentage
FROM 
  `sdp-prd-cti-data.intermediate.shop_chargeback_rates_daily_snapshot`
WHERE 
  date = DATE_SUB(CURRENT_DATE(), INTERVAL 1 DAY)
  AND sp_transaction_amount_usd_30d > 1000 -- Only shops with meaningful volume
ORDER BY 
  chargeback_rate_percentage DESC
LIMIT 
  100
```

### Identify High-Risk Chargeback Patterns

```sql
SELECT
  reason,
  COUNT(*) AS dispute_count,
  AVG(chargeback_amount_usd) AS avg_chargeback_amount,
  COUNT(CASE WHEN status = 'lost' THEN 1 END) AS lost_count,
  COUNT(CASE WHEN status = 'won' THEN 1 END) AS won_count
FROM 
  `shopify-dw.money_products.chargebacks_summary`
WHERE 
  provider_chargeback_created_at >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 180 DAY)
GROUP BY 
  reason
ORDER BY 
  dispute_count DESC
```

## Risk Assessment Queries

### Get High Value Credit Risk Predictions

```sql
SELECT
  shop_id,
  MAX(score) AS max_risk_score,
  AVG(score) AS avg_risk_score,
  MIN(processed_at) AS first_prediction,
  MAX(processed_at) AS latest_prediction
FROM 
  `sdp-prd-cti-data.intermediate.hvcr_predictions_v2`
WHERE 
  observed_at >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 30 DAY)
  AND score > 0.8 -- High risk score threshold
GROUP BY 
  shop_id
ORDER BY 
  max_risk_score DESC
LIMIT 
  100
```

### Identify Shops with Trust Platform Tickets

```sql
SELECT
  subjectable_id AS shop_id,
  COUNT(*) AS ticket_count,
  STRING_AGG(DISTINCT latest_report_type, ", ") AS ticket_types,
  STRING_AGG(DISTINCT rule_group, ", ") AS risk_levels
FROM 
  `shopify-dw.risk.trust_platform_tickets_summary_v1`
WHERE 
  created_at >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 90 DAY)
  AND subjectable_type = 'Shop'
GROUP BY 
  shop_id
HAVING 
  ticket_count > 1
ORDER BY 
  ticket_count DESC
LIMIT 
  100
```

## Shop Analysis Queries

### Shops with High GMV and Elevated Risk

```sql
SELECT
  s.shop_id,
  s.shop_name,
  s.country_code,
  s.gmv_usd_l30d,
  r.score AS latest_risk_score,
  c.chargeback_count_30d
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
    ROW_NUMBER() OVER(PARTITION BY shop_id ORDER BY processed_at DESC) = 1
) r ON s.shop_id = r.shop_id
LEFT JOIN 
  `sdp-prd-cti-data.intermediate.shop_chargeback_rates_daily_snapshot` c
  ON s.shop_id = c.shop_id
  AND c.date = DATE_SUB(CURRENT_DATE(), INTERVAL 1 DAY)
WHERE 
  s.date = DATE_SUB(CURRENT_DATE(), INTERVAL 1 DAY)
  AND s.gmv_usd_l30d > 10000
  AND r.score > 0.7
ORDER BY 
  s.gmv_usd_l30d DESC
LIMIT 
  100
```

### Shops with Recent Significant Changes in Risk Score

```sql
WITH recent_scores AS (
  SELECT
    shop_id,
    DATE(processed_at) AS score_date,
    score,
    LAG(score) OVER(PARTITION BY shop_id ORDER BY processed_at) AS previous_score
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
    score - previous_score AS score_change
  FROM 
    recent_scores
  WHERE 
    previous_score IS NOT NULL
    AND ABS(score - previous_score) > 0.2
)
SELECT
  s.shop_id,
  b.shop_name,
  b.country_code,
  s.score_date,
  s.score AS current_score,
  s.previous_score,
  s.score_change,
  b.gmv_usd_l30d
FROM 
  significant_changes s
JOIN 
  `shopify-dw.mart_core_deliver.shop_back_office_summary_current` b
  ON s.shop_id = b.shop_id
WHERE 
  b.date = DATE_SUB(CURRENT_DATE(), INTERVAL 1 DAY)
ORDER BY 
  ABS(s.score_change) DESC
LIMIT 
  100
```

## Financial Impact Queries

### Chargeback Impact by Country

```sql
SELECT
  shop.country_code,
  COUNT(DISTINCT shop.shop_id) AS shop_count,
  COUNT(cb.chargeback_id) AS chargeback_count,
  SUM(cb.chargeback_amount_usd) AS total_chargeback_amount_usd,
  AVG(cb.chargeback_amount_usd) AS avg_chargeback_amount_usd
FROM 
  `shopify-dw.money_products.chargebacks_summary` cb
JOIN 
  `shopify-dw.mart_core_deliver.shop_back_office_summary_current` shop
  ON cb.shop_id = shop.shop_id
WHERE 
  cb.provider_chargeback_created_at >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 90 DAY)
  AND shop.date = DATE_SUB(CURRENT_DATE(), INTERVAL 1 DAY)
GROUP BY 
  shop.country_code
ORDER BY 
  total_chargeback_amount_usd DESC
```

### Shop Loss Metrics Analysis

```sql
SELECT
  date,
  COUNT(shop_id) AS shops_with_losses,
  SUM(loss_amount_usd) AS total_loss_usd,
  AVG(loss_amount_usd) AS avg_loss_per_shop_usd,
  SUM(CASE WHEN loss_type = 'chargeback' THEN loss_amount_usd ELSE 0 END) AS chargeback_losses_usd,
  SUM(CASE WHEN loss_type = 'fraud' THEN loss_amount_usd ELSE 0 END) AS fraud_losses_usd
FROM 
  `shopify-dw.mart_cti_data.shop_loss_metrics__wide`
WHERE 
  date >= DATE_SUB(CURRENT_DATE(), INTERVAL 90 DAY)
GROUP BY 
  date
ORDER BY 
  date DESC
```

## Combining Data Sources

### Comprehensive Shop Risk Profile

```sql
WITH risk_scores AS (
  SELECT
    shop_id,
    MAX(score) AS max_risk_score,
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
    SAFE_DIVIDE(chargeback_count_30d, sp_transaction_count_30d) * 100 AS chargeback_rate_30d,
    SAFE_DIVIDE(chargeback_count_90d, sp_transaction_count_90d) * 100 AS chargeback_rate_90d
  FROM 
    `sdp-prd-cti-data.intermediate.shop_chargeback_rates_daily_snapshot`
  WHERE 
    date = DATE_SUB(CURRENT_DATE(), INTERVAL 1 DAY)
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
  AND (r.max_risk_score > 0.7 OR c.chargeback_rate_30d > 1.0 OR t.high_risk_tickets > 0)
ORDER BY 
  COALESCE(r.max_risk_score, 0) * 0.5 + 
  COALESCE(c.chargeback_rate_30d, 0) * 0.3 + 
  COALESCE(t.high_risk_tickets, 0) * 0.2 DESC
LIMIT 
  100
```

---