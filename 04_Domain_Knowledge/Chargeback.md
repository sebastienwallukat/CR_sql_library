# Chargeback Analysis

## Introduction
Chargebacks occur when customers dispute transactions with their financial institutions. This page provides SQL queries to analyze chargeback trends, identify high-risk merchants, and calculate key metrics for risk management.

## Datasets
The following datasets contain chargeback-related information:

- `shopify-dw.money_products.chargebacks_summary` - Comprehensive, pre-aggregated chargeback data (recommended)
- `shopify-dw.base.base__payments_disputes` - Primary source for chargeback/dispute data
- `shopify-dw.base.base__payments_dispute_evidences` - Evidence submitted for disputes
- `shopify-dw.money_products.payment_transactions` - Original transaction data
- `shopify-dw.risk.risk_signals` - Risk signals associated with chargebacks
- `shopify-dw.risk.fraud_indicators` - Fraud indicators related to disputes

## Common Queries

### Merchant Chargeback Analysis Using Recommended Table
```sql
SELECT
  shop_id AS merchant_id,
  DATE_TRUNC(provider_chargeback_created_at, MONTH) AS month,
  COUNT(chargeback_id) AS chargeback_count,
  SUM(chargeback_amount_usd) AS chargeback_amount_usd,
  AVG(chargeback_amount_usd) AS avg_chargeback_amount_usd,
  COUNTIF(status = 'won') AS won_count,
  COUNTIF(status = 'lost') AS lost_count,
  COUNTIF(status = 'pending') AS pending_count,
  ROUND(COUNTIF(status = 'won') / NULLIF(COUNT(chargeback_id), 0) * 100, 2) AS win_rate_pct
FROM
  `shopify-dw.money_products.chargebacks_summary`
WHERE 
  provider_chargeback_created_at >= TIMESTAMP(DATE_SUB(CURRENT_DATE(), INTERVAL 6 MONTH))
  AND shop_id = @merchant_id
GROUP BY
  merchant_id, month
ORDER BY
  month DESC
```

### Top Merchants by Chargeback Rate Using Recommended Table
```sql
WITH merchant_metrics AS (
  SELECT
    c.shop_id AS merchant_id,
    COUNT(DISTINCT c.chargeback_id) AS chargeback_count,
    SUM(c.chargeback_amount_usd) AS chargeback_amount_usd,
    MAX(t.transaction_count) AS transaction_count,
    MAX(t.transaction_amount_usd) AS transaction_amount_usd
  FROM
    `shopify-dw.money_products.chargebacks_summary` c
  JOIN (
    SELECT
      shop_id AS merchant_id,
      COUNT(*) AS transaction_count,
      SUM(amount_usd) AS transaction_amount_usd
    FROM
      `shopify-dw.money_products.payment_transactions`
    WHERE
      status = 'SUCCESS'
      AND transaction_date >= DATE_SUB(CURRENT_DATE(), INTERVAL 90 DAY)
    GROUP BY
      merchant_id
  ) t ON c.shop_id = t.merchant_id
  WHERE
    c.provider_chargeback_created_at >= TIMESTAMP(DATE_SUB(CURRENT_DATE(), INTERVAL 90 DAY))
  GROUP BY
    c.shop_id
)

SELECT
  merchant_id,
  chargeback_count,
  chargeback_amount_usd,
  transaction_count,
  transaction_amount_usd,
  ROUND(chargeback_count / NULLIF(transaction_count, 0) * 100, 3) AS chargeback_rate_pct,
  ROUND(chargeback_amount_usd / NULLIF(transaction_amount_usd, 0) * 100, 3) AS chargeback_amount_rate_pct
FROM
  merchant_metrics
WHERE
  transaction_count >= 100  -- Only merchants with meaningful volume
ORDER BY
  chargeback_rate_pct DESC
LIMIT 100
```

### Basic Chargeback Rate Calculation
```sql
WITH transaction_totals AS (
  SELECT
    shop_id AS merchant_id,
    COUNT(*) AS total_transactions,
    SUM(amount_usd) AS total_amount_usd
  FROM
    `shopify-dw.money_products.payment_transactions`
  WHERE
    status = 'SUCCESS'
    AND transaction_date >= DATE_SUB(CURRENT_DATE(), INTERVAL 90 DAY)
  GROUP BY
    shop_id
),

chargeback_totals AS (
  SELECT
    shop_id AS merchant_id,
    COUNT(*) AS total_chargebacks,
    SUM(amount_usd) AS chargeback_amount_usd
  FROM
    `shopify-dw.base.base__payments_disputes`
  WHERE
    dispute_type = 'CHARGEBACK'
    AND dispute_created_at >= TIMESTAMP(DATE_SUB(CURRENT_DATE(), INTERVAL 90 DAY))
  GROUP BY
    shop_id
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
  shop_id AS merchant_id,
  reason_code,
  COUNT(*) AS dispute_count,
  SUM(amount_usd) AS disputed_amount_usd,
  ROUND(AVG(CASE WHEN status = 'won' THEN 1.0 ELSE 0.0 END) * 100, 2) AS win_rate_pct
FROM
  `shopify-dw.base.base__payments_disputes`
WHERE
  dispute_type = 'CHARGEBACK'
  AND dispute_created_at >= TIMESTAMP(DATE_SUB(CURRENT_DATE(), INTERVAL 180 DAY))
GROUP BY
  merchant_id, reason_code
ORDER BY
  merchant_id, dispute_count DESC
LIMIT 100
```

### Monthly Chargeback Trend
```sql
SELECT
  DATE_TRUNC(dispute_created_at, MONTH) AS month,
  COUNT(*) AS dispute_count,
  SUM(amount_usd) AS disputed_amount_usd,
  COUNT(CASE WHEN status = 'won' THEN 1 END) AS won_count,
  COUNT(CASE WHEN status = 'lost' THEN 1 END) AS lost_count,
  COUNT(CASE WHEN status = 'pending' THEN 1 END) AS pending_count
FROM
  `shopify-dw.base.base__payments_disputes`
WHERE
  dispute_type = 'CHARGEBACK'
  AND dispute_created_at >= TIMESTAMP(DATE_SUB(CURRENT_DATE(), INTERVAL 12 MONTH))
GROUP BY
  month
ORDER BY
  month
```

### Time-to-Dispute Analysis
```sql
SELECT
  shop_id AS merchant_id,
  ROUND(AVG(DATE_DIFF(dispute_date, transaction_date, DAY)), 1) AS avg_days_to_dispute,
  MIN(DATE_DIFF(dispute_date, transaction_date, DAY)) AS min_days_to_dispute,
  MAX(DATE_DIFF(dispute_date, transaction_date, DAY)) AS max_days_to_dispute,
  COUNT(*) AS dispute_count
FROM (
  SELECT
    d.shop_id,
    d.transaction_id,
    d.dispute_created_at AS dispute_date,
    t.transaction_date
  FROM
    `shopify-dw.base.base__payments_disputes` d
  JOIN
    `shopify-dw.money_products.payment_transactions` t
    ON d.transaction_id = t.transaction_id
  WHERE
    d.dispute_type = 'CHARGEBACK'
    AND d.dispute_created_at >= TIMESTAMP(DATE_SUB(CURRENT_DATE(), INTERVAL 180 DAY))
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
  d.shop_id AS merchant_id,
  COUNT(DISTINCT d.payments_dispute_id) AS total_disputes,
  COUNT(DISTINCT rs.signal_id) AS total_risk_signals,
  COUNT(DISTINCT CASE WHEN rs.signal_type = 'AVS_MISMATCH' THEN rs.signal_id END) AS avs_mismatch_count,
  COUNT(DISTINCT CASE WHEN rs.signal_type = 'CVV_FAILURE' THEN rs.signal_id END) AS cvv_failure_count,
  COUNT(DISTINCT CASE WHEN rs.signal_type = 'IP_LOCATION_MISMATCH' THEN rs.signal_id END) AS ip_mismatch_count,
  COUNT(DISTINCT CASE WHEN rs.signal_type = 'VELOCITY_LIMIT_EXCEEDED' THEN rs.signal_id END) AS velocity_exceed_count
FROM
  `shopify-dw.base.base__payments_disputes` d
LEFT JOIN
  `shopify-dw.risk.risk_signals` rs
  ON d.transaction_id = rs.transaction_id
WHERE
  d.dispute_type = 'CHARGEBACK'
  AND d.dispute_created_at >= TIMESTAMP(DATE_SUB(CURRENT_DATE(), INTERVAL 90 DAY))
GROUP BY
  d.shop_id
ORDER BY
  total_disputes DESC
LIMIT 100
```

## Best Practices

- Use `shopify-dw.money_products.chargebacks_summary` for most chargeback analyses as it's optimized and comprehensive
- Monitor chargeback rates by both count and amount; they can tell different stories
- Segment analysis by product category, transaction size, and geography to identify risk patterns
- Track win rates by reason code to prioritize evidence collection strategies
- Establish baseline chargeback rates for different merchant verticals
- Focus on merchants with a high number of disputes within first 30 days of transaction
- Chargebacks with reason codes "fraud" or "not recognized" deserve special scrutiny
- Always correlate chargebacks with other risk signals for a comprehensive view
- Remember to use `shopify-dw` as the billing project for all queries

## Temporal Fields Documentation

| Field Name | Data Type | Comparison Function | Example |
|------------|-----------|---------------------|---------|
| provider_chargeback_created_at | TIMESTAMP | TIMESTAMP() | `WHERE provider_chargeback_created_at >= TIMESTAMP(DATE_SUB(CURRENT_DATE(), INTERVAL 7 DAY))` |
| dispute_created_at | TIMESTAMP | TIMESTAMP() | `WHERE dispute_created_at >= TIMESTAMP(DATE_SUB(CURRENT_DATE(), INTERVAL 7 DAY))` |
| transaction_date | DATE | DATE_SUB() | `WHERE transaction_date >= DATE_SUB(CURRENT_DATE(), INTERVAL 7 DAY)` |

## References
- [Dispute Resolution Guide](https://shopify.dev/docs)
- [Chargeback Reason Codes](https://shopify.dev/api)
- [Risk Management Documentation](https://shopify.dev/docs)