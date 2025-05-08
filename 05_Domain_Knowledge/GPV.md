# Gross Payment Volume (GPV) Analysis

## Introduction
Gross Payment Volume (GPV) is a critical metric for credit risk management, representing the total transaction volume processed through Shopify Payments. Analyzing GPV patterns helps identify changes in merchant behavior, growth trends, and potential risk indicators.

## Datasets
The following datasets contain GPV-related information:

- `shopify-dw.finance.gpv` - Primary source for payment processing metrics
- `shopify-dw.money_products.payment_transactions` - Individual transaction data
- `shopify-dw.money_products.authorizations` - Payment authorization records
- `shopify-dw.money_products.captures` - Payment capture data
- `shopify-dw.risk.risk_assessments` - Risk scores associated with transactions

## Common Queries

### Daily GPV by Merchant
```sql
SELECT 
  merchant_id,
  DATE(transaction_date) AS transaction_day,
  SUM(amount_usd) AS daily_gpv_usd
FROM 
  `shopify-dw.money_products.payment_transactions`
WHERE 
  status = 'SUCCESS'
  AND transaction_date >= DATE_SUB(CURRENT_DATE(), INTERVAL 30 DAY)
GROUP BY 
  merchant_id, transaction_day
ORDER BY 
  merchant_id, transaction_day
LIMIT 100
```

### Monthly GPV Growth Rate
```sql
WITH monthly_gpv AS (
  SELECT 
    merchant_id,
    DATE_TRUNC(transaction_date, MONTH) AS month,
    SUM(amount_usd) AS monthly_gpv_usd
  FROM 
    `shopify-dw.money_products.payment_transactions`
  WHERE 
    status = 'SUCCESS'
    AND transaction_date >= DATE_SUB(CURRENT_DATE(), INTERVAL 12 MONTH)
  GROUP BY 
    merchant_id, month
)

SELECT 
  current_month.merchant_id,
  current_month.month,
  current_month.monthly_gpv_usd AS current_month_gpv,
  previous_month.monthly_gpv_usd AS previous_month_gpv,
  ROUND((current_month.monthly_gpv_usd - previous_month.monthly_gpv_usd) / NULLIF(previous_month.monthly_gpv_usd, 0) * 100, 2) AS growth_rate_percent
FROM 
  monthly_gpv current_month
LEFT JOIN 
  monthly_gpv previous_month
  ON current_month.merchant_id = previous_month.merchant_id
  AND previous_month.month = DATE_SUB(current_month.month, INTERVAL 1 MONTH)
WHERE 
  previous_month.monthly_gpv_usd IS NOT NULL
ORDER BY 
  growth_rate_percent DESC
LIMIT 100
```

### GPV vs Risk Score Correlation
```sql
SELECT 
  r.merchant_id,
  DATE_TRUNC(t.transaction_date, WEEK) AS week,
  SUM(t.amount_usd) AS weekly_gpv_usd,
  AVG(r.risk_score) AS avg_risk_score
FROM 
  `shopify-dw.money_products.payment_transactions` t
JOIN 
  `shopify-dw.risk.risk_assessments` r
  ON t.merchant_id = r.merchant_id
  AND t.transaction_date = r.assessment_date
WHERE 
  t.status = 'SUCCESS'
  AND t.transaction_date >= DATE_SUB(CURRENT_DATE(), INTERVAL 90 DAY)
GROUP BY 
  r.merchant_id, week
ORDER BY 
  r.merchant_id, week
LIMIT 100
```

### GPV Anomaly Detection
```sql
WITH merchant_daily_stats AS (
  SELECT 
    merchant_id,
    DATE(transaction_date) AS day,
    SUM(amount_usd) AS daily_gpv_usd
  FROM 
    `shopify-dw.money_products.payment_transactions`
  WHERE 
    status = 'SUCCESS'
    AND transaction_date >= DATE_SUB(CURRENT_DATE(), INTERVAL 90 DAY)
  GROUP BY 
    merchant_id, day
),

merchant_baselines AS (
  SELECT 
    merchant_id,
    AVG(daily_gpv_usd) AS avg_daily_gpv,
    STDDEV(daily_gpv_usd) AS stddev_daily_gpv
  FROM 
    merchant_daily_stats
  GROUP BY 
    merchant_id
)

SELECT 
  stats.merchant_id,
  stats.day,
  stats.daily_gpv_usd,
  base.avg_daily_gpv,
  base.stddev_daily_gpv,
  (stats.daily_gpv_usd - base.avg_daily_gpv) / NULLIF(base.stddev_daily_gpv, 0) AS z_score
FROM 
  merchant_daily_stats stats
JOIN 
  merchant_baselines base
  ON stats.merchant_id = base.merchant_id
WHERE 
  ABS((stats.daily_gpv_usd - base.avg_daily_gpv) / NULLIF(base.stddev_daily_gpv, 0)) > 3
  AND base.stddev_daily_gpv > 0
ORDER BY 
  ABS((stats.daily_gpv_usd - base.avg_daily_gpv) / NULLIF(base.stddev_daily_gpv, 0)) DESC
LIMIT 100
```

## Best Practices

- Always filter by `status = 'SUCCESS'` to ensure accurate GPV calculations
- Consider currency exchange fluctuations when analyzing multi-currency merchants
- Use Z-scores to identify abnormal GPV patterns that may indicate fraud or risk
- Compare GPV trends against industry benchmarks to identify outliers
- Analyze authorization-to-capture ratios as part of risk assessment
- Consider seasonal patterns when evaluating GPV changes

## References
- [Finance Data Domain Documentation](https://shopify.dev/api)
- [Money Products Domain Documentation](https://shopify.dev/api)
- [Risk Management Best Practices](https://shopify.dev/docs) 