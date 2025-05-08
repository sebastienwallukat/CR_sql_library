# Booked Losses Analysis

## Introduction
Booked losses represent financial impacts from various risk events, including chargebacks, fraud, and merchant bankruptcies. This page provides SQL queries to analyze losses, identify trends, and categorize losses for better risk management.

## Datasets
The following datasets contain information related to booked losses:

- `shopify-dw.finance.booked_losses` - Primary source for financial loss data
- `shopify-dw.risk.loss_events` - Details about events that resulted in losses
- `shopify-dw.money_products.disputes` - Dispute information related to losses
- `shopify-dw.finance.merchant_write_offs` - Merchant-related write-offs
- `shopify-dw.merchant_trust.merchant_terminations` - Merchant termination records

## Common Queries

### Monthly Loss Trend by Category
```sql
SELECT
  DATE_TRUNC(loss_date, MONTH) AS month,
  loss_category,
  COUNT(*) AS loss_count,
  SUM(amount_usd) AS total_loss_usd,
  AVG(amount_usd) AS avg_loss_usd
FROM
  `shopify-dw.finance.booked_losses`
WHERE
  loss_date >= DATE_SUB(CURRENT_DATE(), INTERVAL 12 MONTH)
GROUP BY
  month, loss_category
ORDER BY
  month DESC, total_loss_usd DESC
```

### Loss Recovery Rate Analysis
```sql
SELECT
  loss_category,
  COUNT(*) AS total_cases,
  SUM(amount_usd) AS total_loss_usd,
  SUM(recovered_amount_usd) AS total_recovered_usd,
  ROUND(SUM(recovered_amount_usd) / NULLIF(SUM(amount_usd), 0) * 100, 2) AS recovery_rate_pct,
  AVG(TIMESTAMP_DIFF(recovery_date, loss_date, DAY)) AS avg_days_to_recovery
FROM
  `shopify-dw.finance.booked_losses`
WHERE
  loss_date >= DATE_SUB(CURRENT_DATE(), INTERVAL 6 MONTH)
  AND recovery_status IS NOT NULL
GROUP BY
  loss_category
ORDER BY
  total_loss_usd DESC
```

### High-Value Loss Merchants
```sql
SELECT
  merchant_id,
  COUNT(*) AS loss_count,
  SUM(amount_usd) AS total_loss_usd,
  MAX(amount_usd) AS max_single_loss_usd,
  MIN(loss_date) AS first_loss_date,
  MAX(loss_date) AS last_loss_date
FROM
  `shopify-dw.finance.booked_losses`
WHERE
  loss_date >= DATE_SUB(CURRENT_DATE(), INTERVAL 12 MONTH)
GROUP BY
  merchant_id
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
    m.merchant_id,
    m.merchant_category,
    SUM(p.amount_usd) AS total_gpv_usd
  FROM
    `shopify-dw.money_products.payment_transactions` p
  JOIN
    `shopify-dw.accounts_and_administration.merchants` m
    ON p.merchant_id = m.merchant_id
  WHERE
    p.transaction_date >= DATE_SUB(CURRENT_DATE(), INTERVAL 6 MONTH)
    AND p.status = 'SUCCESS'
  GROUP BY
    m.merchant_id, m.merchant_category
),

merchant_losses AS (
  SELECT
    l.merchant_id,
    SUM(l.amount_usd) AS total_loss_usd
  FROM
    `shopify-dw.finance.booked_losses` l
  WHERE
    l.loss_date >= DATE_SUB(CURRENT_DATE(), INTERVAL 6 MONTH)
  GROUP BY
    l.merchant_id
)

SELECT
  g.merchant_category,
  COUNT(DISTINCT g.merchant_id) AS merchant_count,
  SUM(g.total_gpv_usd) AS category_gpv_usd,
  SUM(COALESCE(l.total_loss_usd, 0)) AS category_loss_usd,
  ROUND(SUM(COALESCE(l.total_loss_usd, 0)) / NULLIF(SUM(g.total_gpv_usd), 0) * 100, 4) AS loss_to_gpv_ratio_pct
FROM
  merchant_gpv g
LEFT JOIN
  merchant_losses l
  ON g.merchant_id = l.merchant_id
GROUP BY
  g.merchant_category
HAVING
  category_gpv_usd > 1000000  -- Only include categories with meaningful GPV
ORDER BY
  loss_to_gpv_ratio_pct DESC
```

### Time-to-Detection Analysis
```sql
SELECT
  loss_category,
  ROUND(AVG(TIMESTAMP_DIFF(detection_date, event_date, DAY)), 1) AS avg_days_to_detection,
  MIN(TIMESTAMP_DIFF(detection_date, event_date, DAY)) AS min_days_to_detection,
  MAX(TIMESTAMP_DIFF(detection_date, event_date, DAY)) AS max_days_to_detection,
  APPROX_QUANTILES(TIMESTAMP_DIFF(detection_date, event_date, DAY), 100)[OFFSET(50)] AS median_days_to_detection,
  COUNT(*) AS event_count
FROM
  `shopify-dw.risk.loss_events`
WHERE
  event_date >= DATE_SUB(CURRENT_DATE(), INTERVAL 12 MONTH)
  AND detection_date IS NOT NULL
GROUP BY
  loss_category
HAVING
  event_count >= 10  -- Only include categories with sufficient data
ORDER BY
  avg_days_to_detection DESC
```

## Best Practices

- Segment loss analysis by merchant category, size, and tenure to identify risk patterns
- Review recovery rates by loss type to optimize collection strategies
- Monitor the time gap between loss events and detection to improve early warning systems
- Correlate losses with specific risk indicators to enhance predictive models
- Establish loss reserves based on historical patterns by merchant category
- Conduct quarterly reviews of high-value loss merchants for risk mitigation
- Track loss-to-GPV ratios to establish healthy benchmarks by merchant category
- Document root causes for significant losses to prevent similar events

## References
- [Financial Loss Classification Guide](https://shopify.dev/docs)
- [Risk Mitigation Strategies](https://shopify.dev/api)
- [Loss Recovery Best Practices](https://shopify.dev/docs) 