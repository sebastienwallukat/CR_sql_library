# Fraud Detection Queries 🔍

Welcome to the Fraud Detection section of our SQL Query Resource Center. This page contains datasets and queries for identifying fraud patterns and implementing detection mechanisms.

## Datasets 📁

### `sdp-prd-cti-data.fraud.transaction_fraud_flags`

This dataset contains transaction-level fraud indicators and detection results.

🔗 [Access on Dataplex](https://console.cloud.google.com/dataplex/projects/sdp-prd-cti-data/locations/us/entryGroups/@bigquery/entries/path_to_dataset)

### `sdp-prd-cti-data.fraud.fraud_detection_rules`

This dataset contains the configuration and performance of fraud detection rules.

🔗 [Access on Dataplex](https://console.cloud.google.com/dataplex/projects/sdp-prd-cti-data/locations/us/entryGroups/@bigquery/entries/path_to_dataset)

## Common Queries 💻

### Fraud Detection Performance

```sql
SELECT
  rule_id,
  rule_name,
  rule_category,
  COUNT(*) as total_transactions,
  COUNTIF(is_fraud_confirmed = TRUE) as confirmed_fraud,
  COUNTIF(is_rule_triggered = TRUE) as rule_triggers,
  COUNTIF(is_fraud_confirmed = TRUE AND is_rule_triggered = TRUE) as true_positives,
  COUNTIF(is_fraud_confirmed = FALSE AND is_rule_triggered = TRUE) as false_positives,
  COUNTIF(is_fraud_confirmed = TRUE AND is_rule_triggered = FALSE) as false_negatives,
  SAFE_DIVIDE(
    COUNTIF(is_fraud_confirmed = TRUE AND is_rule_triggered = TRUE),
    COUNTIF(is_rule_triggered = TRUE)
  ) as precision,
  SAFE_DIVIDE(
    COUNTIF(is_fraud_confirmed = TRUE AND is_rule_triggered = TRUE),
    COUNTIF(is_fraud_confirmed = TRUE)
  ) as recall
FROM
  `sdp-prd-cti-data.fraud.transaction_fraud_flags` t
  JOIN `sdp-prd-cti-data.fraud.fraud_detection_rules` r
    ON t.rule_id = r.rule_id
WHERE
  transaction_date >= DATE_SUB(CURRENT_DATE(), INTERVAL 30 DAY)
GROUP BY
  rule_id, rule_name, rule_category
HAVING
  total_transactions > 100
ORDER BY
  precision * recall DESC  -- F1 score approximation
```

### Emerging Fraud Patterns

```sql
WITH fraud_transactions AS (
  SELECT
    transaction_id,
    shop_id,
    customer_id,
    payment_method,
    device_fingerprint,
    billing_country,
    shipping_country,
    transaction_amount,
    industry_category,
    fraud_score
  FROM
    `sdp-prd-cti-data.fraud.transaction_fraud_flags`
  WHERE
    transaction_date >= DATE_SUB(CURRENT_DATE(), INTERVAL 90 DAY)
    AND is_fraud_confirmed = TRUE
),

non_fraud_transactions AS (
  SELECT
    transaction_id,
    shop_id,
    customer_id,
    payment_method,
    device_fingerprint,
    billing_country,
    shipping_country,
    transaction_amount,
    industry_category,
    fraud_score
  FROM
    `sdp-prd-cti-data.fraud.transaction_fraud_flags`
  WHERE
    transaction_date >= DATE_SUB(CURRENT_DATE(), INTERVAL 90 DAY)
    AND is_fraud_confirmed = FALSE
    AND RAND() < 0.01  -- Sample for efficiency
)

SELECT
  'payment_method' as dimension,
  payment_method as value,
  COUNT(DISTINCT ft.transaction_id) as fraud_count,
  COUNT(DISTINCT nft.transaction_id) as non_fraud_count,
  SAFE_DIVIDE(
    COUNT(DISTINCT ft.transaction_id),
    COUNT(DISTINCT ft.transaction_id) + COUNT(DISTINCT nft.transaction_id)
  ) as fraud_rate,
  AVG(ft.fraud_score) as avg_fraud_score
FROM
  fraud_transactions ft
  LEFT JOIN non_fraud_transactions nft
    ON ft.payment_method = nft.payment_method
GROUP BY
  dimension, value
HAVING
  fraud_count > 10

UNION ALL

SELECT
  'billing_shipping_mismatch' as dimension,
  CASE WHEN ft.billing_country != ft.shipping_country THEN 'mismatch' ELSE 'match' END as value,
  COUNT(DISTINCT ft.transaction_id) as fraud_count,
  COUNT(DISTINCT nft.transaction_id) as non_fraud_count,
  SAFE_DIVIDE(
    COUNT(DISTINCT ft.transaction_id),
    COUNT(DISTINCT ft.transaction_id) + COUNT(DISTINCT nft.transaction_id)
  ) as fraud_rate,
  AVG(ft.fraud_score) as avg_fraud_score
FROM
  fraud_transactions ft
  LEFT JOIN non_fraud_transactions nft
    ON (ft.billing_country != ft.shipping_country) = (nft.billing_country != nft.shipping_country)
GROUP BY
  dimension, value
HAVING
  fraud_count > 10

ORDER BY
  fraud_rate DESC
```

### Device Fingerprint Analysis

```sql
WITH device_stats AS (
  SELECT
    device_fingerprint,
    COUNT(DISTINCT transaction_id) as transaction_count,
    COUNT(DISTINCT customer_id) as unique_customers,
    COUNT(DISTINCT shop_id) as unique_shops,
    COUNTIF(is_fraud_confirmed = TRUE) as fraud_count,
    MIN(transaction_date) as first_seen_date,
    MAX(transaction_date) as last_seen_date
  FROM
    `sdp-prd-cti-data.fraud.transaction_fraud_flags`
  WHERE
    transaction_date >= DATE_SUB(CURRENT_DATE(), INTERVAL 90 DAY)
    AND device_fingerprint IS NOT NULL
  GROUP BY
    device_fingerprint
)

SELECT
  device_fingerprint,
  transaction_count,
  unique_customers,
  unique_shops,
  fraud_count,
  SAFE_DIVIDE(fraud_count, transaction_count) as fraud_rate,
  SAFE_DIVIDE(unique_customers, transaction_count) as customers_per_transaction_ratio,
  DATE_DIFF(last_seen_date, first_seen_date, DAY) as days_active
FROM
  device_stats
WHERE
  transaction_count > 5
  AND unique_customers > 1
ORDER BY
  fraud_rate DESC,
  transaction_count DESC
```

### Velocity Check Analysis

```sql
WITH daily_transactions AS (
  SELECT
    transaction_date,
    customer_id,
    COUNT(*) as transaction_count,
    SUM(transaction_amount) as total_amount,
    COUNT(DISTINCT shop_id) as unique_shops,
    COUNT(DISTINCT payment_method) as unique_payment_methods,
    COUNTIF(is_fraud_confirmed = TRUE) as fraud_count
  FROM
    `sdp-prd-cti-data.fraud.transaction_fraud_flags`
  WHERE
    transaction_date >= DATE_SUB(CURRENT_DATE(), INTERVAL 30 DAY)
  GROUP BY
    transaction_date, customer_id
)

SELECT
  customer_id,
  MAX(transaction_count) as max_daily_transactions,
  MAX(total_amount) as max_daily_amount,
  MAX(unique_shops) as max_daily_shops,
  MAX(unique_payment_methods) as max_daily_payment_methods,
  SUM(fraud_count) as total_fraud_transactions,
  COUNT(*) as days_with_transactions
FROM
  daily_transactions
GROUP BY
  customer_id
HAVING
  max_daily_transactions > 10
  OR max_daily_amount > 5000
  OR max_daily_shops > 3
  OR total_fraud_transactions > 0
ORDER BY
  total_fraud_transactions DESC,
  max_daily_transactions DESC
```

## Fraud Indicators Reference 📋

Common fraud indicators tracked in our system:

1. **Customer Velocity** - Unusual transaction frequency or amount in short timeframe
2. **Device/IP Reputation** - History of fraudulent activity from device or IP
3. **Billing/Shipping Mismatch** - Discrepancy between billing and shipping information
4. **Payment Method Anomalies** - Unusual patterns in payment method usage
5. **Transaction Amount Outliers** - Transaction amounts that deviate from typical patterns

## Notes and Best Practices 📝

- Combine multiple fraud indicators for more accurate detection
- Regularly tune thresholds based on performance metrics
- Consider both rule-based and machine learning approaches
- Monitor false positive rates to balance security with customer experience
- Update detection mechanisms as new fraud patterns emerge

Feel free to contribute new queries or improve the existing ones. Together, we can build a comprehensive resource for fraud detection and prevention.

Happy fraud hunting! 🕵️ 