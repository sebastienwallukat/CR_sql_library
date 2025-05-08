# Risk Scoring Analysis

## Introduction
Risk scoring is essential for identifying potentially fraudulent transactions and high-risk merchants. This page provides SQL queries to analyze risk scores, identify patterns, and evaluate the effectiveness of risk models.

## Datasets
The following datasets contain risk scoring information:

- `shopify-dw.risk.risk_assessments` - Primary source for risk scores and assessments
- `shopify-dw.risk.risk_factors` - Factors contributing to risk scores
- `shopify-dw.merchant_trust.risk_signals` - Signals indicating potential risk
- `shopify-dw.money_products.payment_transactions` - Transaction data for correlation
- `shopify-dw.merchant_trust.shop_trust_metrics` - Shop-level trust indicators

## Common Queries

### Transaction Risk Score Distribution
```sql
SELECT
  ROUND(risk_score * 10) / 10 AS risk_score_bucket,
  COUNT(*) AS assessment_count,
  COUNT(*) / SUM(COUNT(*)) OVER () AS percentage
FROM
  `shopify-dw.risk.risk_assessments`
WHERE
  assessment_date >= DATE_SUB(CURRENT_DATE(), INTERVAL 30 DAY)
  AND assessment_type = 'TRANSACTION'
GROUP BY
  risk_score_bucket
ORDER BY
  risk_score_bucket
```

### High-Risk Transactions by Merchant
```sql
SELECT
  merchant_id,
  COUNT(*) AS total_transactions,
  COUNT(CASE WHEN risk_score >= 0.8 THEN 1 END) AS high_risk_count,
  ROUND(COUNT(CASE WHEN risk_score >= 0.8 THEN 1 END) / COUNT(*) * 100, 2) AS high_risk_percentage,
  AVG(risk_score) AS avg_risk_score,
  MAX(risk_score) AS max_risk_score
FROM
  `shopify-dw.risk.risk_assessments`
WHERE
  assessment_date >= DATE_SUB(CURRENT_DATE(), INTERVAL 90 DAY)
  AND assessment_type = 'TRANSACTION'
GROUP BY
  merchant_id
HAVING
  COUNT(*) >= 50  -- Only include merchants with at least 50 assessments
  AND COUNT(CASE WHEN risk_score >= 0.8 THEN 1 END) >= 5  -- With at least 5 high-risk transactions
ORDER BY
  high_risk_percentage DESC
LIMIT 100
```

### Risk Factor Analysis
```sql
SELECT
  rf.factor_name,
  COUNT(*) AS occurrence_count,
  AVG(ra.risk_score) AS avg_risk_score,
  MIN(ra.risk_score) AS min_risk_score,
  MAX(ra.risk_score) AS max_risk_score
FROM
  `shopify-dw.risk.risk_factors` rf
JOIN
  `shopify-dw.risk.risk_assessments` ra
  ON rf.assessment_id = ra.assessment_id
WHERE
  ra.assessment_date >= DATE_SUB(CURRENT_DATE(), INTERVAL 60 DAY)
  AND ra.assessment_type = 'TRANSACTION'
GROUP BY
  rf.factor_name
ORDER BY
  occurrence_count DESC
LIMIT 20
```

### Risk Score Correlation with Fraud
```sql
WITH transaction_data AS (
  SELECT
    t.transaction_id,
    t.merchant_id,
    t.amount_usd,
    ra.risk_score,
    CASE WHEN d.dispute_id IS NOT NULL AND d.reason_code = 'FRAUD' THEN 1 ELSE 0 END AS is_fraud
  FROM
    `shopify-dw.money_products.payment_transactions` t
  LEFT JOIN
    `shopify-dw.risk.risk_assessments` ra
    ON t.transaction_id = ra.transaction_id
    AND ra.assessment_type = 'TRANSACTION'
  LEFT JOIN
    `shopify-dw.money_products.disputes` d
    ON t.transaction_id = d.transaction_id
    AND d.dispute_type = 'CHARGEBACK'
    AND d.reason_code = 'FRAUD'
  WHERE
    t.transaction_date >= DATE_SUB(CURRENT_DATE(), INTERVAL 180 DAY)
    AND t.status = 'SUCCESS'
)

SELECT
  ROUND(risk_score * 10) / 10 AS risk_score_bucket,
  COUNT(*) AS transaction_count,
  SUM(is_fraud) AS fraud_count,
  ROUND(SUM(is_fraud) / COUNT(*) * 100, 2) AS fraud_rate_pct
FROM
  transaction_data
WHERE
  risk_score IS NOT NULL
GROUP BY
  risk_score_bucket
ORDER BY
  risk_score_bucket
```

### Risk Signals by Transaction Amount
```sql
SELECT
  CASE
    WHEN amount_usd < 50 THEN 'Under $50'
    WHEN amount_usd BETWEEN 50 AND 100 THEN '$50-$100'
    WHEN amount_usd BETWEEN 100 AND 250 THEN '$100-$250'
    WHEN amount_usd BETWEEN 250 AND 500 THEN '$250-$500'
    WHEN amount_usd BETWEEN 500 AND 1000 THEN '$500-$1000'
    ELSE 'Over $1000'
  END AS amount_range,
  COUNT(*) AS transaction_count,
  COUNT(DISTINCT signal_id) AS signal_count,
  COUNT(DISTINCT signal_id) / COUNT(*) AS signals_per_transaction,
  COUNT(DISTINCT CASE WHEN signal_type = 'AVS_MISMATCH' THEN signal_id END) AS avs_mismatch_count,
  COUNT(DISTINCT CASE WHEN signal_type = 'CVV_FAILURE' THEN signal_id END) AS cvv_failure_count,
  COUNT(DISTINCT CASE WHEN signal_type = 'IP_LOCATION_MISMATCH' THEN signal_id END) AS ip_mismatch_count
FROM
  `shopify-dw.money_products.payment_transactions` t
LEFT JOIN
  `shopify-dw.merchant_trust.risk_signals` rs
  ON t.transaction_id = rs.transaction_id
WHERE
  t.transaction_date >= DATE_SUB(CURRENT_DATE(), INTERVAL 90 DAY)
  AND t.status = 'SUCCESS'
GROUP BY
  amount_range
ORDER BY
  MIN(amount_usd)
```

## Best Practices

- Regularly monitor the effectiveness of risk scores by comparing to actual fraud outcomes
- Consider multiple risk thresholds based on merchant category and transaction size
- Examine outliers: both high-risk transactions that weren't fraudulent and low-risk transactions that were
- Create merchant risk segments based on historical patterns and industry categories
- Review risk factors for their predictive power and adjust weights accordingly
- Combine transaction risk scores with merchant trust metrics for a comprehensive view
- Document false positives to improve future risk model iterations

## References
- [Risk Assessment Documentation](https://shopify.dev/docs)
- [Risk Factor Definitions](https://shopify.dev/api)
- [Fraud Prevention Best Practices](https://shopify.dev/docs) 