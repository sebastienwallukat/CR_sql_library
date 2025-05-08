# Fraud Detection Analysis

## Introduction
Fraud detection is a critical component of credit risk management. This page provides SQL queries to identify fraud patterns, analyze fraud indicators, and measure the effectiveness of detection mechanisms.

## Datasets
The following datasets contain fraud-related information:

- `shopify-dw.risk.fraud_cases` - Confirmed fraud cases and details
- `shopify-dw.merchant_trust.risk_signals` - Signals indicating potential fraud
- `shopify-dw.money_products.disputes` - Disputes with fraud-related reason codes
- `shopify-dw.risk.risk_assessments` - Transaction risk scores and assessments
- `shopify-dw.merchant_trust.fraud_rules` - Fraud detection rules and their performance

## Common Queries

### Fraud by Transaction Type
```sql
SELECT
  transaction_type,
  payment_method,
  COUNT(*) AS fraud_count,
  SUM(amount_usd) AS fraud_amount_usd,
  AVG(amount_usd) AS avg_fraud_amount
FROM
  `shopify-dw.risk.fraud_cases`
WHERE
  detection_date >= DATE_SUB(CURRENT_DATE(), INTERVAL 90 DAY)
  AND status = 'CONFIRMED'
GROUP BY
  transaction_type, payment_method
ORDER BY
  fraud_amount_usd DESC
```

### Fraud Detection Time Analysis
```sql
SELECT
  DATE_TRUNC(transaction_date, MONTH) AS month,
  COUNT(*) AS fraud_count,
  AVG(TIMESTAMP_DIFF(detection_date, transaction_date, HOUR)) AS avg_hours_to_detection,
  MIN(TIMESTAMP_DIFF(detection_date, transaction_date, HOUR)) AS min_hours_to_detection,
  MAX(TIMESTAMP_DIFF(detection_date, transaction_date, HOUR)) AS max_hours_to_detection,
  APPROX_QUANTILES(TIMESTAMP_DIFF(detection_date, transaction_date, HOUR), 100)[OFFSET(50)] AS median_hours_to_detection
FROM
  `shopify-dw.risk.fraud_cases`
WHERE
  transaction_date >= DATE_SUB(CURRENT_DATE(), INTERVAL 12 MONTH)
  AND status = 'CONFIRMED'
  AND detection_date IS NOT NULL
GROUP BY
  month
ORDER BY
  month
```

### Risk Signal Effectiveness for Fraud Detection
```sql
WITH fraud_transactions AS (
  SELECT
    transaction_id,
    detection_date
  FROM
    `shopify-dw.risk.fraud_cases`
  WHERE
    detection_date >= DATE_SUB(CURRENT_DATE(), INTERVAL 180 DAY)
    AND status = 'CONFIRMED'
)

SELECT
  rs.signal_type,
  COUNT(DISTINCT rs.transaction_id) AS flagged_transactions,
  COUNT(DISTINCT ft.transaction_id) AS fraud_transactions,
  COUNT(DISTINCT ft.transaction_id) / COUNT(DISTINCT rs.transaction_id) AS fraud_rate,
  AVG(CASE WHEN ft.transaction_id IS NOT NULL 
           THEN TIMESTAMP_DIFF(ft.detection_date, rs.detection_date, HOUR) 
           ELSE NULL END) AS avg_hours_from_signal_to_confirmation
FROM
  `shopify-dw.merchant_trust.risk_signals` rs
LEFT JOIN
  fraud_transactions ft
  ON rs.transaction_id = ft.transaction_id
WHERE
  rs.detection_date >= DATE_SUB(CURRENT_DATE(), INTERVAL 180 DAY)
GROUP BY
  rs.signal_type
HAVING
  COUNT(DISTINCT rs.transaction_id) >= 10  -- Only include signals with sufficient volume
ORDER BY
  fraud_rate DESC
```

### Fraud Rate by Merchant Category
```sql
WITH merchant_transactions AS (
  SELECT
    m.merchant_id,
    m.merchant_category,
    t.transaction_id,
    t.amount_usd
  FROM
    `shopify-dw.money_products.payment_transactions` t
  JOIN
    `shopify-dw.accounts_and_administration.merchants` m
    ON t.merchant_id = m.merchant_id
  WHERE
    t.transaction_date >= DATE_SUB(CURRENT_DATE(), INTERVAL 90 DAY)
    AND t.status = 'SUCCESS'
),

fraud_cases AS (
  SELECT
    transaction_id
  FROM
    `shopify-dw.risk.fraud_cases`
  WHERE
    detection_date >= DATE_SUB(CURRENT_DATE(), INTERVAL 90 DAY)
    AND status = 'CONFIRMED'
)

SELECT
  mt.merchant_category,
  COUNT(DISTINCT mt.merchant_id) AS merchant_count,
  COUNT(DISTINCT mt.transaction_id) AS transaction_count,
  SUM(mt.amount_usd) AS total_amount_usd,
  COUNT(DISTINCT fc.transaction_id) AS fraud_count,
  SUM(CASE WHEN fc.transaction_id IS NOT NULL THEN mt.amount_usd ELSE 0 END) AS fraud_amount_usd,
  ROUND(COUNT(DISTINCT fc.transaction_id) / COUNT(DISTINCT mt.transaction_id) * 100, 4) AS fraud_rate_pct,
  ROUND(SUM(CASE WHEN fc.transaction_id IS NOT NULL THEN mt.amount_usd ELSE 0 END) / SUM(mt.amount_usd) * 100, 4) AS fraud_amount_rate_pct
FROM
  merchant_transactions mt
LEFT JOIN
  fraud_cases fc
  ON mt.transaction_id = fc.transaction_id
GROUP BY
  mt.merchant_category
HAVING
  transaction_count >= 1000  -- Only include categories with sufficient volume
ORDER BY
  fraud_rate_pct DESC
```

### Rule Effectiveness Analysis
```sql
SELECT
  rule_id,
  rule_name,
  rule_category,
  COUNT(*) AS trigger_count,
  COUNT(DISTINCT transaction_id) AS unique_transactions,
  COUNT(DISTINCT merchant_id) AS unique_merchants,
  COUNT(CASE WHEN outcome = 'TRUE_POSITIVE' THEN 1 END) AS true_positives,
  COUNT(CASE WHEN outcome = 'FALSE_POSITIVE' THEN 1 END) AS false_positives,
  ROUND(COUNT(CASE WHEN outcome = 'TRUE_POSITIVE' THEN 1 END) / 
        NULLIF(COUNT(*), 0) * 100, 2) AS precision_rate,
  AVG(CASE WHEN outcome = 'TRUE_POSITIVE' THEN amount_usd ELSE NULL END) AS avg_true_positive_amount
FROM
  `shopify-dw.merchant_trust.fraud_rules`
WHERE
  trigger_date >= DATE_SUB(CURRENT_DATE(), INTERVAL 90 DAY)
GROUP BY
  rule_id, rule_name, rule_category
HAVING
  trigger_count >= 10  -- Only include rules with sufficient triggers
ORDER BY
  precision_rate DESC
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