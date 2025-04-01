# Chargeback Analysis

## Introduction
Chargebacks occur when customers dispute transactions with their financial institutions. This page provides SQL queries to analyze chargeback trends, identify high-risk merchants, and calculate key metrics for risk management.

## Datasets
The following datasets contain chargeback-related information:

- `shopify-dw.money_products.disputes` - Primary source for chargeback/dispute data
- `shopify-dw.money_products.dispute_evidence` - Evidence submitted for disputes
- `shopify-dw.money_products.payment_transactions` - Original transaction data
- `shopify-dw.merchant_trust.risk_signals` - Risk signals associated with chargebacks
- `shopify-dw.risk.fraud_indicators` - Fraud indicators related to disputes

## Common Queries

### Basic Chargeback Rate Calculation
```sql
WITH transaction_totals AS (
  SELECT
    merchant_id,
    COUNT(*) AS total_transactions,
    SUM(amount_usd) AS total_amount_usd
  FROM
    `shopify-dw.money_products.payment_transactions`
  WHERE
    status = 'SUCCESS'
    AND transaction_date >= DATE_SUB(CURRENT_DATE(), INTERVAL 90 DAY)
  GROUP BY
    merchant_id
),

chargeback_totals AS (
  SELECT
    merchant_id,
    COUNT(*) AS total_chargebacks,
    SUM(amount_usd) AS chargeback_amount_usd
  FROM
    `shopify-dw.money_products.disputes`
  WHERE
    dispute_type = 'CHARGEBACK'
    AND created_at >= DATE_SUB(CURRENT_DATE(), INTERVAL 90 DAY)
  GROUP BY
    merchant_id
)

SELECT
  t.merchant_id,
  t.total_transactions,
  t.total_amount_usd,
  COALESCE(c.total_chargebacks, 0) AS total_chargebacks,
  COALESCE(c.chargeback_amount_usd, 0) AS chargeback_amount_usd,
  ROUND(COALESCE(c.total_chargebacks, 0) / NULLIF(t.total_transactions, 0) * 100, 2) AS chargeback_rate_pct,
  ROUND(COALESCE(c.chargeback_amount_usd, 0) / NULLIF(t.total_amount_usd, 0) * 100, 2) AS chargeback_amount_rate_pct
FROM
  transaction_totals t
LEFT JOIN
  chargeback_totals c
  ON t.merchant_id = c.merchant_id
WHERE
  t.total_transactions >= 100  -- Only consider merchants with meaningful volume
ORDER BY
  chargeback_rate_pct DESC
LIMIT 100
```

### Chargeback Reason Analysis
```sql
SELECT
  merchant_id,
  reason_code,
  COUNT(*) AS dispute_count,
  SUM(amount_usd) AS disputed_amount_usd,
  ROUND(AVG(CASE WHEN outcome = 'WON' THEN 1.0 ELSE 0.0 END) * 100, 2) AS win_rate_pct
FROM
  `shopify-dw.money_products.disputes`
WHERE
  dispute_type = 'CHARGEBACK'
  AND created_at >= DATE_SUB(CURRENT_DATE(), INTERVAL 180 DAY)
GROUP BY
  merchant_id, reason_code
ORDER BY
  merchant_id, dispute_count DESC
LIMIT 100
```

### Monthly Chargeback Trend
```sql
SELECT
  DATE_TRUNC(created_at, MONTH) AS month,
  COUNT(*) AS dispute_count,
  SUM(amount_usd) AS disputed_amount_usd,
  COUNT(CASE WHEN outcome = 'WON' THEN 1 END) AS won_count,
  COUNT(CASE WHEN outcome = 'LOST' THEN 1 END) AS lost_count,
  COUNT(CASE WHEN outcome = 'PENDING' THEN 1 END) AS pending_count
FROM
  `shopify-dw.money_products.disputes`
WHERE
  dispute_type = 'CHARGEBACK'
  AND created_at >= DATE_SUB(CURRENT_DATE(), INTERVAL 12 MONTH)
GROUP BY
  month
ORDER BY
  month
```

### Time-to-Dispute Analysis
```sql
SELECT
  merchant_id,
  ROUND(AVG(DATE_DIFF(dispute_date, transaction_date, DAY)), 1) AS avg_days_to_dispute,
  MIN(DATE_DIFF(dispute_date, transaction_date, DAY)) AS min_days_to_dispute,
  MAX(DATE_DIFF(dispute_date, transaction_date, DAY)) AS max_days_to_dispute,
  COUNT(*) AS dispute_count
FROM (
  SELECT
    d.merchant_id,
    d.transaction_id,
    d.created_at AS dispute_date,
    t.transaction_date
  FROM
    `shopify-dw.money_products.disputes` d
  JOIN
    `shopify-dw.money_products.payment_transactions` t
    ON d.transaction_id = t.transaction_id
  WHERE
    d.dispute_type = 'CHARGEBACK'
    AND d.created_at >= DATE_SUB(CURRENT_DATE(), INTERVAL 180 DAY)
)
GROUP BY
  merchant_id
HAVING
  COUNT(*) >= 5  -- Only include merchants with at least 5 disputes
ORDER BY
  avg_days_to_dispute
LIMIT 100
```

### Chargeback Risk Signals Correlation
```sql
SELECT
  d.merchant_id,
  COUNT(DISTINCT d.dispute_id) AS total_disputes,
  COUNT(DISTINCT rs.signal_id) AS total_risk_signals,
  COUNT(DISTINCT CASE WHEN rs.signal_type = 'AVS_MISMATCH' THEN rs.signal_id END) AS avs_mismatch_count,
  COUNT(DISTINCT CASE WHEN rs.signal_type = 'CVV_FAILURE' THEN rs.signal_id END) AS cvv_failure_count,
  COUNT(DISTINCT CASE WHEN rs.signal_type = 'IP_LOCATION_MISMATCH' THEN rs.signal_id END) AS ip_mismatch_count,
  COUNT(DISTINCT CASE WHEN rs.signal_type = 'VELOCITY_LIMIT_EXCEEDED' THEN rs.signal_id END) AS velocity_exceed_count
FROM
  `shopify-dw.money_products.disputes` d
LEFT JOIN
  `shopify-dw.merchant_trust.risk_signals` rs
  ON d.transaction_id = rs.transaction_id
WHERE
  d.dispute_type = 'CHARGEBACK'
  AND d.created_at >= DATE_SUB(CURRENT_DATE(), INTERVAL 90 DAY)
GROUP BY
  d.merchant_id
ORDER BY
  total_disputes DESC
LIMIT 100
```

## Best Practices

- Monitor chargeback rates by both count and amount; they can tell different stories
- Segment analysis by product category, transaction size, and geography to identify risk patterns
- Track win rates by reason code to prioritize evidence collection strategies
- Establish baseline chargeback rates for different merchant verticals
- Focus on merchants with a high number of disputes within first 30 days of transaction
- Chargebacks with reason codes "fraud" or "not recognized" deserve special scrutiny
- Always correlate chargebacks with other risk signals for a comprehensive view

## References
- [Dispute Resolution Guide](https://shopify.dev/docs)
- [Chargeback Reason Codes](https://shopify.dev/api)
- [Risk Management Documentation](https://shopify.dev/docs)