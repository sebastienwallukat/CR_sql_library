# Capital Recovery Analysis Queries 💸

Welcome to the Capital Recovery section of our SQL Query Resource Center. This page contains datasets and queries for analyzing capital recovery activities and effectiveness.

## Primary Datasets 📁

### `shopify-dw.risk.recovery_collections`

This dataset tracks capital recovery remittances and outcomes, including recovery amounts, methods, and success rates.

### `shopify-dw.risk.recovery_payment_plans`

This dataset contains payment plan information related to capital recovery efforts, including repayment plans and settlement plans.

## Common Queries 💻

### Recovery Rate by Recovery Method

```sql
-- Version: 1.1
-- Author: Credit Risk Team

SELECT
  reason as recovery_method,
  COUNT(*) as attempt_count,
  COUNT(*) as successful_attempts,
  ROUND(COUNT(*) / COUNT(*) * 100, 2) as success_rate_pct,
  SUM(amount_usd) as total_attempted_usd,
  SUM(amount_usd) as total_recovered_usd,
  ROUND(SUM(amount_usd) / SUM(amount_usd) * 100, 2) as recovery_rate_pct
FROM
  `shopify-dw.risk.recovery_collections`
WHERE
  created_at >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 90 DAY)
GROUP BY
  recovery_method
ORDER BY
  total_recovered_usd DESC
```

### Recovery Aging Analysis

```sql
-- Version: 1.1
-- Author: Credit Risk Team

WITH aging_buckets AS (
  SELECT
    payment_plan_id,
    shop_id,
    created_at as attempt_date,
    status as recovery_status,
    amount_usd as recovery_amount_usd,
    DATE_DIFF(CURRENT_DATE(), DATE(created_at), DAY) as days_since_event,
    CASE
      WHEN DATE_DIFF(CURRENT_DATE(), DATE(created_at), DAY) <= 30 THEN '0-30 days'
      WHEN DATE_DIFF(CURRENT_DATE(), DATE(created_at), DAY) <= 60 THEN '31-60 days'
      WHEN DATE_DIFF(CURRENT_DATE(), DATE(created_at), DAY) <= 90 THEN '61-90 days'
      WHEN DATE_DIFF(CURRENT_DATE(), DATE(created_at), DAY) <= 180 THEN '91-180 days'
      ELSE '180+ days'
    END as aging_bucket
  FROM
    `shopify-dw.risk.recovery_payment_plans`
  WHERE
    status = 'open'
)

SELECT
  aging_bucket,
  COUNT(*) as open_cases,
  SUM(recovery_amount_usd) as outstanding_amount_usd,
  AVG(recovery_amount_usd) as avg_case_amount_usd,
  MAX(days_since_event) as max_days_outstanding,
  MIN(days_since_event) as min_days_outstanding
FROM
  aging_buckets
GROUP BY
  aging_bucket
ORDER BY
  CASE
    WHEN aging_bucket = '0-30 days' THEN 1
    WHEN aging_bucket = '31-60 days' THEN 2
    WHEN aging_bucket = '61-90 days' THEN 3
    WHEN aging_bucket = '91-180 days' THEN 4
    WHEN aging_bucket = '180+ days' THEN 5
  END
```

### Merchant Recovery Effectiveness

```sql
-- Version: 1.1
-- Author: Credit Risk Team

WITH merchant_recovery AS (
  SELECT
    shop_id as merchant_id,
    COUNT(*) as total_attempts,
    SUM(amount_usd) as total_attempted_usd,
    SUM(amount_usd) as total_recovered_usd,
    SAFE_DIVIDE(SUM(amount_usd), SUM(amount_usd)) as recovery_rate,
    NULL as avg_days_to_recovery
  FROM
    `shopify-dw.risk.recovery_collections`
  WHERE
    created_at >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 180 DAY)
  GROUP BY
    merchant_id
),

merchant_info AS (
  SELECT 
    shop_details.shop_id as merchant_id,
    'Standard' as merchant_tier,
    NULL as industry_category,
    shop_details.country_code,
    NULL as first_transaction_date,
    shop_details.gmv_usd as total_processing_volume_usd
  FROM 
    `shopify-dw.mart_core_operate.identity_account_shop_summary_current`,
    UNNEST(shop_details) as shop_details
)

SELECT
  i.merchant_id,
  i.merchant_tier,
  i.industry_category,
  i.country_code,
  0 as merchant_age_days,
  r.total_attempts,
  r.total_attempted_usd,
  r.total_recovered_usd,
  ROUND(r.recovery_rate * 100, 2) as recovery_rate_pct,
  r.avg_days_to_recovery,
  ROUND(r.total_attempted_usd / NULLIF(i.total_processing_volume_usd, 0) * 100, 4) as attempted_vs_gpv_pct
FROM
  merchant_recovery r
JOIN
  merchant_info i ON r.merchant_id = i.merchant_id
WHERE
  r.total_attempted_usd > 1000
ORDER BY
  r.recovery_rate DESC
LIMIT 100
```

### Recovery Trend Analysis

```sql
-- Version: 1.1
-- Author: Credit Risk Team

SELECT
  DATE_TRUNC(created_at, MONTH) as month,
  reason as recovery_method,
  COUNT(*) as attempt_count,
  SUM(amount_usd) as attempted_amount_usd,
  SUM(amount_usd) as recovered_amount_usd,
  ROUND(SUM(amount_usd) / NULLIF(SUM(amount_usd), 0) * 100, 2) as recovery_rate_pct,
  NULL as avg_days_to_recovery
FROM
  `shopify-dw.risk.recovery_collections`
WHERE
  created_at >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 12 MONTH)
GROUP BY
  month, recovery_method
ORDER BY
  month DESC, recovered_amount_usd DESC
```

## Recovery Method Effectiveness

```sql
-- Version: 1.1
-- Author: Credit Risk Team

WITH amount_buckets AS (
  SELECT
    payment_plan_id as recovery_id,
    type as recovery_method,
    amount_usd as attempted_amount_usd,
    amount_usd as recovered_amount_usd,
    status as recovery_status,
    CASE
      WHEN amount_usd <= 100 THEN '0-$100'
      WHEN amount_usd <= 500 THEN '$101-$500'
      WHEN amount_usd <= 1000 THEN '$501-$1,000'
      WHEN amount_usd <= 5000 THEN '$1,001-$5,000'
      WHEN amount_usd <= 10000 THEN '$5,001-$10,000'
      ELSE '$10,000+'
    END as amount_bucket
  FROM
    `shopify-dw.risk.recovery_payment_plans`
  WHERE
    created_at >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 180 DAY)
)

SELECT
  amount_bucket,
  recovery_method,
  COUNT(*) as attempt_count,
  SUM(attempted_amount_usd) as attempted_amount_usd,
  SUM(recovered_amount_usd) as recovered_amount_usd,
  ROUND(SUM(recovered_amount_usd) / NULLIF(SUM(attempted_amount_usd), 0) * 100, 2) as recovery_rate_pct,
  COUNTIF(recovery_status = 'completed') as successful_count,
  ROUND(COUNTIF(recovery_status = 'completed') / NULLIF(COUNT(*), 0) * 100, 2) as success_rate_pct
FROM
  amount_buckets
GROUP BY
  amount_bucket, recovery_method
ORDER BY
  CASE
    WHEN amount_bucket = '0-$100' THEN 1
    WHEN amount_bucket = '$101-$500' THEN 2
    WHEN amount_bucket = '$501-$1,000' THEN 3
    WHEN amount_bucket = '$1,001-$5,000' THEN 4
    WHEN amount_bucket = '$5,001-$10,000' THEN 5
    WHEN amount_bucket = '$10,000+' THEN 6
  END,
  recovery_rate_pct DESC
```

## Recovery Methods Reference 📋

Common recovery methods include:

1. **Direct Refund Request** - Processing a refund through the merchant's payment processor
2. **Chargeback** - Issuing a chargeback through the customer's card issuer
3. **Merchant Direct Engagement** - Direct outreach to merchants for repayment
4. **Legal Action** - Formal legal proceedings for recovery
5. **Reserve Capture** - Using reserved funds to cover losses
6. **Debt Collection** - Engaging third-party collection agencies
7. **Payment Plan** - Establishing a structured repayment schedule with merchants

## Temporal Fields Documentation

| Field Name | Data Type | Comparison Function | Example |
|------------|-----------|---------------------|---------|
| created_at | TIMESTAMP | TIMESTAMP_SUB() | `WHERE created_at >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 7 DAY)` |
| updated_at | TIMESTAMP | TIMESTAMP_SUB() | `WHERE updated_at >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 7 DAY)` |
| starts_at | TIMESTAMP | TIMESTAMP_SUB() | `WHERE starts_at >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 7 DAY)` |

## Notes and Best Practices 📝

- Always prioritize recovery methods based on recovery rate and average time to recovery
- For small amounts (under $100), evaluate whether the recovery effort cost justifies the potential return
- Track recovery rate trends over time to identify process improvements or deterioration
- Consider industry norms when evaluating recovery performance
- Document all recovery attempts for compliance and audit purposes
- Review aging recovery cases regularly to ensure timely action
- Compare recovery rates against industry benchmarks

Feel free to contribute new queries or improve the existing ones. Together, we can build a comprehensive resource for capital recovery analysis.

Happy recovering! 💰 