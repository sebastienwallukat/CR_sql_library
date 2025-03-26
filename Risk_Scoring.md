# Transaction Risk Scoring Queries 🛡️

Welcome to the Transaction Risk Scoring section of our SQL Query Resource Center. This page contains models and queries for evaluating transaction risk.

## Datasets 📁

### `sdp-prd-cti-data.risk.transaction_risk_scores`

This dataset contains risk scores assigned to individual transactions, along with the contributing factors.

🔗 [Access on Dataplex](https://console.cloud.google.com/dataplex/projects/sdp-prd-cti-data/locations/us/entryGroups/@bigquery/entries/path_to_dataset)

### `sdp-prd-cti-data.risk.shop_risk_profile`

This dataset provides aggregated risk profiles for shops based on their transaction history.

🔗 [Access on Dataplex](https://console.cloud.google.com/dataplex/projects/sdp-prd-cti-data/locations/us/entryGroups/@bigquery/entries/path_to_dataset)

## Common Queries 💻

### High-Risk Transactions by Shop

```sql
SELECT
  t.shop_id,
  COUNT(*) as total_transactions,
  COUNTIF(risk_score > 0.8) as high_risk_transactions,
  COUNTIF(risk_score > 0.8) / COUNT(*) as high_risk_ratio
FROM
  `sdp-prd-cti-data.risk.transaction_risk_scores` t
WHERE
  transaction_date >= DATE_SUB(CURRENT_DATE(), INTERVAL 30 DAY)
GROUP BY
  shop_id
HAVING
  total_transactions > 50
  AND high_risk_ratio > 0.1
ORDER BY
  high_risk_ratio DESC
```

### Risk Factor Analysis

```sql
SELECT
  risk_factor,
  COUNT(*) as occurrences,
  AVG(factor_weight) as avg_weight,
  AVG(risk_score) as avg_risk_score
FROM
  `sdp-prd-cti-data.risk.transaction_risk_scores` t,
  UNNEST(risk_factors) as risk_factor
WHERE
  transaction_date >= DATE_SUB(CURRENT_DATE(), INTERVAL 90 DAY)
GROUP BY
  risk_factor
ORDER BY
  occurrences DESC
```

### Shop Risk Profile Trends

```sql
WITH monthly_risk AS (
  SELECT
    shop_id,
    DATE_TRUNC(profile_date, MONTH) as month,
    AVG(risk_score) as avg_risk_score
  FROM
    `sdp-prd-cti-data.risk.shop_risk_profile`
  WHERE
    profile_date >= DATE_SUB(CURRENT_DATE(), INTERVAL 6 MONTH)
  GROUP BY
    shop_id, month
)

SELECT
  shop_id,
  month,
  avg_risk_score,
  LAG(avg_risk_score) OVER (PARTITION BY shop_id ORDER BY month) as prev_month_score,
  avg_risk_score - LAG(avg_risk_score) OVER (PARTITION BY shop_id ORDER BY month) as score_change
FROM
  monthly_risk
ORDER BY
  score_change DESC
```

## Risk Model Information 📊

Our transaction risk scoring model incorporates multiple factors including:

1. **Transaction Characteristics**
   - Amount relative to shop average
   - Time of day
   - Payment method
   - Device information

2. **Customer Behavior**
   - Velocity checks
   - Location vs billing address
   - Customer history

3. **Merchant Profile**
   - Industry risk category
   - Processing history
   - Chargeback rate

The model outputs a risk score between 0 and 1, with higher values indicating higher risk.

## Notes and Best Practices 📝

- Risk scores should be used as indicators, not absolute determinants
- Consider multiple risk factors when evaluating transactions
- Regularly review and update risk model thresholds based on performance

Feel free to contribute new queries or improve the existing ones. Together, we can build a comprehensive resource for transaction risk scoring.

Happy risk analyzing! 🔍 