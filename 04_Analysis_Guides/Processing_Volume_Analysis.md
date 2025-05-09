# Processing Volume Analysis (GPV)

## Introduction
Gross Payment Volume (GPV) is a critical metric for credit risk management, representing the total transaction volume processed through Shopify Payments. Analyzing GPV patterns helps identify changes in merchant behavior, growth trends, and potential risk indicators.

## Table of Contents
- [Introduction](#introduction)
- [Datasets](#datasets)
- [Common Queries](#common-queries)
  - [Daily GPV by Merchant](#daily-gpv-by-merchant)
  - [Monthly GPV Growth Rate](#monthly-gpv-growth-rate)
  - [GPV vs Risk Score Correlation](#gpv-vs-risk-score-correlation)
  - [GPV Anomaly Detection](#gpv-anomaly-detection)
- [Best Practices](#best-practices)
- [Temporal Field Handling](#temporal-field-handling)
- [Key Considerations](#key-considerations-for-gpv-analysis)
- [Related Resources](#related-resources)

## Datasets
The following datasets contain GPV-related information:

- `shopify-dw.money_products.order_transactions` - Primary source for payment processing metrics
- `shopify-dw.money_products.order_transactions_payments_summary` - Detailed transaction data with payment information
- `shopify-dw.base.base__banking_card_authorizations` - Payment authorization records
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

**Description:** This query calculates the daily GPV for each merchant over the past 30 days. It filters for successful capture transactions that are included in GPV calculations. The query uses both a time filter on transaction creation date and the extracted_at field to ensure data completeness.

**Example:** The results show each merchant's daily transaction volume, allowing analysts to identify sudden spikes or drops in processing volume that may indicate changes in risk profile.

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

**Description:** This query calculates the month-over-month GPV growth rate for each merchant over the past year. It first creates a CTE that aggregates monthly GPV values, then joins this CTE to itself to compare each month with the previous month. The growth rate is calculated as a percentage change.

**Example:** The results show merchants with the highest growth rates, which can help identify rapidly growing merchants that may require additional monitoring or merchants with unusual spikes that could indicate potential issues.

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

**Description:** This query correlates weekly GPV with average risk scores for each merchant over the past 90 days. It joins transaction data with risk assessment data to understand how risk scores evolve with GPV fluctuations.

**Example:** The results help identify correlations between transaction volume and risk scores. For instance, merchants with increasing risk scores alongside increasing GPV may warrant closer monitoring.

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

**Description:** This query identifies anomalous daily GPV values for merchants over the past 90 days. It calculates a z-score for each day's GPV relative to the merchant's own historical average and standard deviation. Days with z-scores exceeding 3 (meaning they are 3+ standard deviations from the mean) are flagged as anomalies.

**Example:** Results show merchants with unusual daily GPV patterns, including the exact date of the anomaly, the actual GPV for that day, and baseline metrics for comparison. This helps risk analysts quickly identify potential fraud or other issues requiring investigation.

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

- [Financial Data](../03_Table_References/Financial_Data.md)
- [Chargeback Analysis](./Chargeback_Analysis.md)
- [Writing Better Queries](../02_Analysis_Tools/Writing_Better_Queries.md)
- [Complete Shop Analysis](../05_Example_Analysis/Complete_Shop_Analysis.md)

---