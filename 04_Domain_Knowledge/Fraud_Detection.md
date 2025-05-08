# Fraud Detection Analysis

## Introduction
Fraud detection is a critical component of credit risk management. This page provides SQL queries to identify fraud patterns, analyze fraud indicators, and measure the effectiveness of detection mechanisms.

## Datasets
The following datasets contain fraud-related information:

- `shopify-dw.risk.trust_platform_tickets_summary` - Tickets for fraud cases and investigations
- `shopify-dw.risk.shop_terminations_history` - History of shop terminations with reasons
- `shopify-dw.money_products.chargebacks_summary` - Disputes and chargebacks with fraud-related reason codes
- `shopify-dw.risk.shop_loss_metrics_daily_summary` - Shop loss metrics for trust reporting
- `shopify-dw.risk.trust_platform_shopify_payments_account_activity` - Shopify Payments account activity related to fraud

## Common Queries

### Fraud Ticket Analysis by Type
```sql
SELECT
  latest_report_type AS fraud_type,
  source,
  COUNT(*) AS fraud_count,
  COUNT(DISTINCT subjectable_id) AS unique_entities,
  AVG(time_to_first_action) AS avg_time_to_action_minutes
FROM
  `shopify-dw.risk.trust_platform_tickets_summary`
WHERE
  created_at >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 90 DAY)
  AND latest_report_group = 'fraud'
  AND status = 'closed'
GROUP BY
  fraud_type, source
ORDER BY
  fraud_count DESC
```

### Fraud Detection Time Analysis
```sql
SELECT
  DATE_TRUNC(DATE(created_at), MONTH) AS month,
  COUNT(*) AS fraud_count,
  AVG(time_to_first_action) AS avg_minutes_to_detection,
  MIN(time_to_first_action) AS min_minutes_to_detection,
  MAX(time_to_first_action) AS max_minutes_to_detection,
  APPROX_QUANTILES(time_to_first_action, 100)[OFFSET(50)] AS median_minutes_to_detection
FROM
  `shopify-dw.risk.trust_platform_tickets_summary`
WHERE
  created_at >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 365 DAY)
  AND latest_report_group = 'fraud'
  AND first_action_at IS NOT NULL
GROUP BY
  month
ORDER BY
  month
```

### Fraud-Related Chargebacks Analysis
```sql
WITH fraud_chargebacks AS (
  SELECT
    chargeback_id,
    reason,
    status,
    shop_id,
    chargeback_amount_usd,
    provider_chargeback_created_at
  FROM
    `shopify-dw.money_products.chargebacks_summary`
  WHERE
    provider_chargeback_created_at >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 180 DAY)
    AND reason IN ('fraudulent', 'fraud_monitoring')
)

SELECT
  reason,
  COUNT(DISTINCT chargeback_id) AS chargeback_count,
  COUNT(DISTINCT shop_id) AS affected_shops,
  SUM(chargeback_amount_usd) AS total_chargeback_amount_usd,
  AVG(chargeback_amount_usd) AS avg_chargeback_amount_usd,
  COUNTIF(status = 'won') / COUNT(*) AS win_rate
FROM
  fraud_chargebacks
GROUP BY
  reason
ORDER BY
  chargeback_count DESC
```

### Shop Termination Analysis
```sql
SELECT
  termination_reason,
  COUNT(DISTINCT shop_id) AS terminated_shops_count,
  AVG(TIMESTAMP_DIFF(COALESCE(valid_to, CURRENT_TIMESTAMP()), valid_from, DAY)) AS avg_termination_duration_days
FROM
  `shopify-dw.risk.shop_terminations_history`
WHERE
  valid_from >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 90 DAY)
  AND termination_reason LIKE '%fraud%'
GROUP BY
  termination_reason
ORDER BY
  terminated_shops_count DESC
```

### Shop Loss Metrics Analysis
```sql
SELECT
  DATE_TRUNC(date, MONTH) AS month,
  COUNT(DISTINCT shop_id) AS shops_with_losses,
  SUM(net_sp_booked_loss_usd) AS total_loss_amount_usd,
  AVG(net_sp_booked_loss_usd) AS avg_loss_per_shop_usd
FROM
  `shopify-dw.risk.shop_loss_metrics_daily_summary`
WHERE
  date >= DATE_SUB(CURRENT_DATE(), INTERVAL 180 DAY)
GROUP BY
  month
ORDER BY
  month DESC
```

## Best Practices

- Monitor fraud patterns regularly to adapt detection rules
- Balance false positive and false negative rates for optimal protection
- Implement layered fraud detection with multiple signal types
- Analyze fraud by merchant category to develop targeted prevention strategies
- Track time-to-detection to measure effectiveness of real-time systems
- Review rule precision and recall regularly to improve fraud models
- Combine machine learning predictions with rule-based systems
- Maintain reference data for known fraud patterns and evolving techniques

## References
- [Fraud Detection Methodology](https://shopify.dev/docs)
- [Risk Signal Definitions](https://shopify.dev/api)
- [Transaction Monitoring Best Practices](https://shopify.dev/docs) 