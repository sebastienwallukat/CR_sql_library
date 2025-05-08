# Booked Losses Analysis

## Introduction
Booked losses represent financial impacts from various risk events, including chargebacks, fraud, and merchant bankruptcies. This page provides SQL queries to analyze losses, identify trends, and categorize losses for better risk management.

## Datasets
The following datasets contain information related to booked losses:

- `shopify-dw.money_products.shopify_payments_estimated_booked_losses` - Primary source for Shopify Payments booked loss data
- `shopify-dw.mart_cti_data.shop_loss_metrics__wide` - Merchant-level loss metrics and associated attributes
- `shopify-dw.money_products.banking_balance_exacerbated_shopify_payments_exposure` - Balance-related payment exposure data
- `shopify-dw.risk.shop_terminations_summary` - Merchant termination records
- `shopify-dw.risk.shop_terminations_history` - Detailed history of merchant terminations

## Common Queries

### Monthly Loss Trend by Provider
```sql
SELECT
  DATE_TRUNC(booked_date, MONTH) AS month,
  provider_name,
  COUNT(*) AS loss_count,
  SUM(booked_loss_usd) AS total_loss_usd,
  AVG(booked_loss_usd) AS avg_loss_usd
FROM
  `shopify-dw.money_products.shopify_payments_estimated_booked_losses`
WHERE
  booked_date >= DATE_SUB(CURRENT_DATE(), INTERVAL 12 MONTH)
GROUP BY
  month, provider_name
ORDER BY
  month DESC, total_loss_usd DESC
```

### Loss by Merchant Category
```sql
SELECT
  s.shop_primary_predicted_product_category AS merchant_category,
  COUNT(DISTINCT s.shop_id) AS merchant_count,
  SUM(s.net_sp_booked_loss_usd) AS total_loss_usd,
  AVG(s.net_sp_booked_loss_usd) AS avg_loss_usd
FROM
  `shopify-dw.mart_cti_data.shop_loss_metrics__wide` s
WHERE
  s.date >= DATE_SUB(CURRENT_DATE(), INTERVAL 6 MONTH)
GROUP BY
  merchant_category
HAVING
  merchant_count > 10
ORDER BY
  total_loss_usd DESC
```

### High-Value Loss Merchants
```sql
SELECT
  shop_id,
  COUNT(*) AS loss_count,
  SUM(net_sp_booked_loss_usd) AS total_loss_usd,
  MAX(net_sp_booked_loss_usd) AS max_single_loss_usd,
  MIN(date) AS first_loss_date,
  MAX(date) AS last_loss_date
FROM
  `shopify-dw.mart_cti_data.shop_loss_metrics__wide`
WHERE
  date >= DATE_SUB(CURRENT_DATE(), INTERVAL 12 MONTH)
GROUP BY
  shop_id
HAVING
  total_loss_usd > 10000  -- Focus on high-value losses
ORDER BY
  total_loss_usd DESC
LIMIT 100
```

### Loss-to-GPV Ratio by Merchant Category
```sql
WITH merchant_gpv AS (
  SELECT
    m.shop_id,
    m.shop_primary_predicted_product_category AS merchant_category,
    SUM(m.gpv_usd) AS total_gpv_usd
  FROM
    `sdp-prd-cti-data.intermediate.shop_trust_daily_snapshot` m
  WHERE
    m.date >= DATE_SUB(CURRENT_DATE(), INTERVAL 6 MONTH)
  GROUP BY
    m.shop_id, m.shop_primary_predicted_product_category
),

merchant_losses AS (
  SELECT
    l.shop_id,
    SUM(l.net_sp_booked_loss_usd) AS total_loss_usd
  FROM
    `shopify-dw.mart_cti_data.shop_loss_metrics__wide` l
  WHERE
    l.date >= DATE_SUB(CURRENT_DATE(), INTERVAL 6 MONTH)
  GROUP BY
    l.shop_id
)

SELECT
  g.merchant_category,
  COUNT(DISTINCT g.shop_id) AS merchant_count,
  SUM(g.total_gpv_usd) AS category_gpv_usd,
  SUM(COALESCE(l.total_loss_usd, 0)) AS category_loss_usd,
  ROUND(SUM(COALESCE(l.total_loss_usd, 0)) / NULLIF(SUM(g.total_gpv_usd), 0) * 100, 4) AS loss_to_gpv_ratio_pct
FROM
  merchant_gpv g
LEFT JOIN
  merchant_losses l
  ON g.shop_id = l.shop_id
GROUP BY
  g.merchant_category
HAVING
  category_gpv_usd > 1000000  -- Only include categories with meaningful GPV
ORDER BY
  loss_to_gpv_ratio_pct DESC
```

### Balance-Exacerbated Exposure Analysis
```sql
SELECT
  DATE_TRUNC(negative_balance_created_on, MONTH) AS month,
  COUNT(*) AS exposure_events,
  SUM(balance_exacerbated_sp_exposure_usd) AS balance_exacerbated_exposure_usd,
  AVG(balance_exacerbated_sp_exposure_usd) AS avg_exposure_per_event_usd,
  SUM(balance_exacerbated_sp_exposure_usd) / COUNT(*) AS mean_exposure_per_event_usd
FROM
  `shopify-dw.money_products.banking_balance_exacerbated_shopify_payments_exposure`
WHERE
  negative_balance_created_on >= DATE_SUB(CURRENT_DATE(), INTERVAL 12 MONTH)
GROUP BY
  month
ORDER BY
  month DESC
```

### Termination Analysis for Fraud Shops with Losses
```sql
SELECT
  t.latest_termination_reason_category,
  COUNT(DISTINCT l.shop_id) AS merchant_count,
  SUM(l.net_sp_booked_loss_usd) AS total_loss_usd,
  AVG(l.net_sp_booked_loss_usd) AS avg_loss_per_merchant_usd,
  MIN(TIMESTAMP_DIFF(l.date, CAST(t.first_termination_at AS DATE), DAY)) AS min_days_to_first_loss,
  AVG(TIMESTAMP_DIFF(l.date, CAST(t.first_termination_at AS DATE), DAY)) AS avg_days_to_first_loss,
  MAX(TIMESTAMP_DIFF(l.date, CAST(t.first_termination_at AS DATE), DAY)) AS max_days_to_first_loss
FROM
  `shopify-dw.mart_cti_data.shop_loss_metrics__wide` l
JOIN
  `shopify-dw.risk.shop_terminations_summary` t
  ON l.shop_id = t.shop_id
WHERE
  l.is_shop_still_terminated_for_fraud = TRUE
  AND l.date >= DATE_SUB(CURRENT_DATE(), INTERVAL 12 MONTH)
GROUP BY
  t.latest_termination_reason_category
ORDER BY
  total_loss_usd DESC
```

## Best Practices

- Segment loss analysis by merchant category, size, and tenure to identify risk patterns
- Review recovery rates by loss type to optimize collection strategies
- Monitor the time gap between termination events and losses to improve early warning systems
- Correlate losses with specific risk indicators to enhance predictive models
- Establish loss reserves based on historical patterns by merchant category
- Conduct quarterly reviews of high-value loss merchants for risk mitigation
- Track loss-to-GPV ratios to establish healthy benchmarks by merchant category
- Document root causes for significant losses to prevent similar events

## Temporal Fields Documentation

### Key Temporal Fields

| Field Name | Data Type | Comparison Function | Example |
|------------|-----------|---------------------|---------|
| booked_date | DATE | DATE_SUB() | `WHERE booked_date >= DATE_SUB(CURRENT_DATE(), INTERVAL 7 DAY)` |
| date | DATE | DATE_SUB() | `WHERE date >= DATE_SUB(CURRENT_DATE(), INTERVAL 7 DAY)` |
| negative_balance_created_on | DATE | DATE_SUB() | `WHERE negative_balance_created_on >= DATE_SUB(CURRENT_DATE(), INTERVAL 7 DAY)` |
| shop_created_at | TIMESTAMP | TIMESTAMP_SUB() | `WHERE shop_created_at >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 7 DAY)` |
| first_termination_at | TIMESTAMP | TIMESTAMP_SUB() | `WHERE first_termination_at >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 7 DAY)` |

## References
- [Financial Loss Classification Guide](https://shopify.dev/docs)
- [Risk Mitigation Strategies](https://shopify.dev/api)
- [Loss Recovery Best Practices](https://shopify.dev/docs) 

### Basic Booked Loss Query

Use this query to get basic information about booked losses:

```sql
SELECT
  bl.shop_id,
  bl.transaction_id,
  bl.shop_currency_code,
  bl.booked_loss_amount_usd,
  bl.booked_loss_amount_shop_currency,
  bl.created_at as booked_loss_created_at,
  bl.updated_at as booked_loss_updated_at, 
  bl.booked_loss_category,
  bl.booked_loss_reason,
  bl.booked_loss_description,
  o.order_name,
  o.payment_method,
  o.processor_type
FROM `shopify-dw.money_products.booked_losses` bl
JOIN `shopify-dw.money_products.order_transactions_payments_summary` o
  ON bl.transaction_id = o.transaction_id
WHERE bl.shop_id = 12345678
ORDER BY bl.created_at DESC
LIMIT 100
```

### Aggregate Loss Analysis

This query helps analyze booked losses by category and reason:

```sql
WITH loss_data AS (
  SELECT
    DATE_TRUNC(created_at, MONTH) as month,
    booked_loss_category,
    booked_loss_reason,
    SUM(booked_loss_amount_usd) as total_loss_usd,
    COUNT(*) as loss_count
  FROM `shopify-dw.money_products.booked_losses`
  WHERE created_at >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 12 MONTH)
  GROUP BY month, booked_loss_category, booked_loss_reason
)

SELECT
  month,
  booked_loss_category,
  booked_loss_reason,
  total_loss_usd,
  loss_count,
  ROUND(total_loss_usd / loss_count, 2) as avg_loss_per_incident
FROM loss_data
ORDER BY month DESC, total_loss_usd DESC
```

### Merchants with High Losses

Identify merchants with the highest booked losses:

```sql
SELECT
  bl.shop_id,
  s.shop_name,
  s.shop_domain,
  s.trust_battery,
  COUNT(*) as loss_count,
  SUM(bl.booked_loss_amount_usd) as total_loss_usd,
  MIN(bl.created_at) as first_loss_date,
  MAX(bl.created_at) as latest_loss_date,
  ARRAY_AGG(DISTINCT bl.booked_loss_category ORDER BY bl.booked_loss_category) as loss_categories
FROM `shopify-dw.money_products.booked_losses` bl
JOIN sdp-prd-cti-data.intermediate.shop_insights_shops s ON bl.shop_id = s.shop_id
WHERE bl.created_at >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 180 DAY)
GROUP BY bl.shop_id, s.shop_name, s.shop_domain, s.trust_battery
HAVING total_loss_usd > 1000
ORDER BY total_loss_usd DESC
LIMIT 100
```

### Timeframe Comparison

Compare booked losses across different time periods:

```sql
WITH monthly_losses AS (
  SELECT
    DATE_TRUNC(created_at, MONTH) as month,
    SUM(booked_loss_amount_usd) as monthly_loss_usd,
    COUNT(*) as loss_count,
    COUNT(DISTINCT shop_id) as affected_merchant_count
  FROM `shopify-dw.money_products.booked_losses`
  WHERE created_at >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 24 MONTH)
  GROUP BY month
)

SELECT
  month,
  monthly_loss_usd,
  loss_count,
  affected_merchant_count,
  ROUND(monthly_loss_usd / loss_count, 2) as avg_loss_per_incident,
  ROUND(monthly_loss_usd / affected_merchant_count, 2) as avg_loss_per_merchant,
  ROUND(loss_count / affected_merchant_count, 2) as avg_incidents_per_merchant
FROM monthly_losses
ORDER BY month DESC
```

### Loss and Chargeback Relationship

Analyze relationship between booked losses and chargebacks:

```sql
WITH loss_data AS (
  SELECT
    shop_id,
    SUM(booked_loss_amount_usd) as total_loss_usd,
    COUNT(*) as loss_count
  FROM `shopify-dw.money_products.booked_losses`
  WHERE created_at >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 90 DAY)
  GROUP BY shop_id
),

chargeback_data AS (
  SELECT
    shop_id,
    COUNT(*) as chargeback_count,
    SUM(CASE WHEN dispute_reason LIKE '%fraud%' THEN 1 ELSE 0 END) as fraud_chargeback_count,
    SUM(CASE WHEN dispute_reason NOT LIKE '%fraud%' THEN 1 ELSE 0 END) as non_fraud_chargeback_count
  FROM `shopify-dw.money_products.chargebacks_summary`
  WHERE created_at >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 180 DAY)
  GROUP BY shop_id
)

SELECT
  l.shop_id,
  s.shop_name,
  s.shop_domain,
  l.total_loss_usd,
  l.loss_count,
  c.chargeback_count,
  c.fraud_chargeback_count,
  c.non_fraud_chargeback_count,
  SAFE_DIVIDE(c.chargeback_count, l.loss_count) as chargebacks_per_loss
FROM loss_data l
JOIN chargeback_data c ON l.shop_id = c.shop_id
JOIN sdp-prd-cti-data.intermediate.shop_insights_shops s ON l.shop_id = s.shop_id
WHERE l.total_loss_usd > 500
  AND c.chargeback_count > 10
ORDER BY l.total_loss_usd DESC
LIMIT 100
``` 