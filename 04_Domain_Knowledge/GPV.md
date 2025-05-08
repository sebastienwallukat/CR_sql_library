# Gross Payment Volume (GPV) Analysis

## Introduction
Gross Payment Volume (GPV) is a critical metric for credit risk management, representing the total transaction volume processed through Shopify Payments. Analyzing GPV patterns helps identify changes in merchant behavior, growth trends, and potential risk indicators.

## Datasets
The following datasets contain GPV-related information:

- `shopify-dw.money_products.order_transactions` - Primary source for payment processing metrics
- `shopify-dw.money_products.order_transactions_payments_summary` - Detailed transaction data with payment information
- `shopify-dw.money_products.banking_card_authorizations` - Payment authorization records
- `shopify-dw.money_products.banking_balance_transactions` - Payment capture and transaction data
- `sdp-prd-payments.intermediate.order_risk_assessments` - Risk scores associated with transactions

## Common Queries

### Daily GPV by Merchant
```sql
SELECT 
  shop_id AS merchant_id,
  DATE(order_transaction_created_at) AS transaction_day,
  SUM(amount_local) AS daily_gpv_usd
FROM 
  `shopify-dw.money_products.order_transactions_payments_summary`
WHERE 
  order_transaction_status = 'SUCCESS'
  AND order_transaction_created_at >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 30 DAY)
  AND order_transaction_kind = 'capture'
  AND is_included_in_gpv = TRUE
  AND _extracted_at >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 60 DAY)
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
    shop_id AS merchant_id,
    DATE_TRUNC(DATE(order_transaction_created_at), MONTH) AS month,
    SUM(amount_local) AS monthly_gpv_usd
  FROM 
    `shopify-dw.money_products.order_transactions_payments_summary`
  WHERE 
    order_transaction_status = 'SUCCESS'
    AND order_transaction_created_at >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 365 DAY)
    AND order_transaction_kind = 'capture'
    AND is_included_in_gpv = TRUE
    AND _extracted_at >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 400 DAY)
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
  t.shop_id AS merchant_id,
  DATE_TRUNC(DATE(t.order_transaction_created_at), WEEK) AS week,
  SUM(t.amount_local) AS weekly_gpv_usd,
  AVG(r.risk_score) AS avg_risk_score
FROM 
  `shopify-dw.money_products.order_transactions_payments_summary` t
JOIN 
  `sdp-prd-payments.intermediate.order_risk_assessments` r
  ON t.order_id = r.order_id
WHERE 
  t.order_transaction_status = 'SUCCESS'
  AND t.order_transaction_created_at >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 90 DAY)
  AND t.order_transaction_kind = 'capture'
  AND t.is_included_in_gpv = TRUE
  AND t._extracted_at >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 100 DAY)
GROUP BY 
  merchant_id, week
ORDER BY 
  merchant_id, week
LIMIT 100
```

### GPV Anomaly Detection
```sql
WITH merchant_daily_stats AS (
  SELECT 
    shop_id AS merchant_id,
    DATE(order_transaction_created_at) AS day,
    SUM(amount_local) AS daily_gpv_usd
  FROM 
    `shopify-dw.money_products.order_transactions_payments_summary`
  WHERE 
    order_transaction_status = 'SUCCESS'
    AND order_transaction_created_at >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 90 DAY)
    AND order_transaction_kind = 'capture'
    AND is_included_in_gpv = TRUE
    AND _extracted_at >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 100 DAY)
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

- Always filter by `order_transaction_status = 'SUCCESS'` to ensure accurate GPV calculations
- Use `order_transaction_kind = 'capture'` to focus on completed transactions
- Use `is_included_in_gpv = TRUE` to identify transactions processed through Shopify Payments
- Always include a partition filter on `_extracted_at` when querying `order_transactions_payments_summary`
- Consider currency exchange fluctuations when analyzing multi-currency merchants
- Use Z-scores to identify abnormal GPV patterns that may indicate fraud or risk
- Compare GPV trends against industry benchmarks to identify outliers
- Analyze authorization-to-capture ratios as part of risk assessment
- Consider seasonal patterns when evaluating GPV changes
- For TIMESTAMP type columns, always use TIMESTAMP functions (TIMESTAMP_SUB) not DATE functions
- When working with dates derived from timestamps, convert using DATE() function first

## Temporal Field Handling

### Proper Date/Timestamp Handling
| Field Name | Data Type | Comparison Function | Example |
|------------|-----------|---------------------|---------|
| order_transaction_created_at | TIMESTAMP | TIMESTAMP_SUB() | `WHERE order_transaction_created_at >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 7 DAY)` |
| day (derived) | DATE | DATE_SUB() | `WHERE DATE(order_transaction_created_at) >= DATE_SUB(CURRENT_DATE(), INTERVAL 7 DAY)` |

## Key Considerations for GPV Analysis

- **GPV Time Lag:** There can be 1-2 days of lag in GPV data appearing in reporting tables
- **Currency Conversion:** All monetary values should be in the same currency (USD is standard) for comparison
- **Refunds Impact:** Remember that refunds can reduce both GMV and GPV
- **Time Dimension:** Always analyze GPV over appropriate time intervals (7d, 28d, 90d, etc.)
- **Payment Methods:** Consider filtering by payment type when analyzing specific segments
- **Region Specifics:** Different regions may have different payment method mixes affecting GPV

## Related Resources

- [Financial Tables Documentation](../03_Data_Dictionary/Financial_Tables.md)
- [Chargeback Analysis](./Chargeback.md)
- [SQL Best Practices](../02_SQL_Guide/SQL_Best_Practices.md)
- [Shop Analysis](../05_Example_Queries/Full_Query_Shop.md)

---