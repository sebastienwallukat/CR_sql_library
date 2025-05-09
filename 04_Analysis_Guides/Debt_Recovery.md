# Capital Recovery Analysis Queries 💸

## Introduction
This document contains datasets and queries for analyzing capital recovery activities and effectiveness at Shopify. Capital recovery refers to the process of recouping funds through various collection methods, payment plans, and settlement arrangements. These queries help Credit Risk analysts track recovery performance metrics and identify optimization opportunities.

## Table of Contents
- [Primary Datasets](#primary-datasets)
- [Recovery Rate Analysis](#recovery-rate-analysis)
- [Aging Analysis](#aging-analysis)
- [Merchant Recovery Effectiveness](#merchant-recovery-effectiveness)
- [Recovery Trend Analysis](#recovery-trend-analysis)
- [Recovery Method Effectiveness](#recovery-method-effectiveness)
- [Recovery Methods Reference](#recovery-methods-reference)
- [Temporal Fields Documentation](#temporal-fields-documentation)
- [Notes and Best Practices](#notes-and-best-practices)

## Primary Datasets

### `shopify-dw.risk.recovery_collections`

This dataset tracks capital recovery remittances and outcomes, including recovery amounts, methods, and success rates. Each row represents a successful collection (remittance) for a recovery payment plan.

**Key fields:**
- `remittance_id`: Unique identifier for the remittance
- `shop_id`: Identifier of the merchant shop
- `payment_plan_id`: Reference to the associated payment plan
- `reason`: Method used for recovery
- `amount_usd`: Recovery amount in USD

### `shopify-dw.risk.recovery_payment_plans`

This dataset contains payment plan information related to capital recovery efforts, including repayment plans and settlement plans. Each row represents a payment plan arrangement with a merchant.

**Key fields:**
- `payment_plan_id`: Unique identifier for the payment plan
- `shop_id`: Identifier of the merchant shop
- `type`: Type of payment plan (repayment or settlement)
- `status`: Current status of the payment plan (open, completed, etc.)
- `amount_usd`: Total amount of the payment plan in USD
- `is_latest_payment_plan`: Flag indicating the most recent plan per shop

## Recovery Rate Analysis

This query analyzes the effectiveness of different recovery methods by calculating success rates and amounts recovered.

```sql
-- Purpose: Analyze effectiveness of different recovery methods
-- Shows attempt counts, success rates, and recovery amounts by method

SELECT
  reason as recovery_method,
  COUNT(*) as attempt_count,
  COUNT(*) as successful_attempts,
  ROUND(COUNT(*) / NULLIF(COUNT(*), 0) * 100, 2) as success_rate_pct,  -- Calculate percentage of successful attempts
  SUM(amount_usd) as total_attempted_usd,
  SUM(amount_usd) as total_recovered_usd,
  ROUND(SUM(amount_usd) / NULLIF(SUM(amount_usd), 0) * 100, 2) as recovery_rate_pct  -- Calculate percentage of recovered amounts
FROM
  `shopify-dw.risk.recovery_collections`
WHERE
  created_at >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 90 DAY)  -- Only consider recent collections (last 90 days)
GROUP BY
  recovery_method
ORDER BY
  total_recovered_usd DESC  -- Sort by most successful methods first
```

### Description
This query helps identify which recovery methods are most effective by comparing success rates and recovered amounts. It groups collections by their reason/method and calculates both the count of successful attempts and the total monetary value recovered. The results are ordered by the highest recovery amount to highlight the most impactful methods.

### Example Output
The query returns a table with recovery methods ranked by effectiveness. For instance, "payment_plan_scheduled" might show as the top method with high success rates, while "merchant_initiated" collections might yield higher per-attempt values.

## Aging Analysis

This query segments open payment plans by age to track recovery timelines and identify aging cases.

```sql
-- Purpose: Track recovery payment plans by age bucket
-- Identifies outstanding amounts and aging patterns in open recovery cases

WITH aging_buckets AS (
  SELECT
    payment_plan_id,
    shop_id,
    created_at as attempt_date,
    status as recovery_status,
    amount_usd as recovery_amount_usd,
    DATE_DIFF(CURRENT_DATE(), DATE(created_at), DAY) as days_since_event,  -- Calculate days since plan creation
    CASE  -- Categorize into aging buckets
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
    AND created_at >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 365 DAY)  -- Required partition filter
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
  CASE  -- Custom sorting for aging buckets
    WHEN aging_bucket = '0-30 days' THEN 1
    WHEN aging_bucket = '31-60 days' THEN 2
    WHEN aging_bucket = '61-90 days' THEN 3
    WHEN aging_bucket = '91-180 days' THEN 4
    WHEN aging_bucket = '180+ days' THEN 5
  END
```

### Description
This query creates aging buckets for open payment plans to analyze the age distribution of outstanding recovery cases. It calculates the number of open cases, total outstanding amount, and average case value for each aging bucket. This helps identify where recovery efforts might be stalling and where the largest outstanding amounts exist.

### Example Output
The results show statistics for each aging bucket, with counts and amounts increasing visibility into how much capital is outstanding in each timeframe. For example, you might see that the "91-180 days" bucket has the highest total outstanding amount, indicating a potential area for focused recovery efforts.

## Merchant Recovery Effectiveness

This query analyzes recovery performance at the merchant level, combining recovery data with merchant information.

```sql
-- Purpose: Analyze recovery effectiveness by merchant
-- Combines recovery metrics with merchant attributes for deeper analysis

WITH merchant_recovery AS (
  SELECT
    shop_id as merchant_id,
    COUNT(*) as total_attempts,  -- Count of recovery attempts
    SUM(amount_usd) as total_attempted_usd,
    SUM(amount_usd) as total_recovered_usd,
    SAFE_DIVIDE(SUM(amount_usd), NULLIF(SUM(amount_usd), 0)) as recovery_rate,  -- Safe division to avoid errors
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
  r.merchant_id,
  i.merchant_tier,
  i.industry_category,
  i.country_code,
  0 as merchant_age_days,
  r.total_attempts,
  r.total_attempted_usd,
  r.total_recovered_usd,
  ROUND(r.recovery_rate * 100, 2) as recovery_rate_pct,  -- Convert to percentage
  r.avg_days_to_recovery,
  -- Calculate recovery amount as percentage of merchant's total processing volume
  ROUND(r.total_attempted_usd / NULLIF(i.total_processing_volume_usd, 0) * 100, 4) as attempted_vs_gpv_pct  
FROM
  merchant_recovery r
JOIN
  merchant_info i ON r.merchant_id = i.merchant_id
WHERE
  r.total_attempted_usd > 1000  -- Focus on significant recovery amounts
ORDER BY
  r.recovery_rate DESC
```

### Description
This query combines recovery metrics with merchant information to provide a holistic view of recovery effectiveness. It calculates total recovery amounts, success rates, and contextualizes these against the merchant's overall processing volume. This helps identify patterns in recovery success based on merchant attributes such as country and tier.

### Example Output
The results provide merchant-level recovery metrics with contextual information such as country code and processing volume. This allows for identification of trends in recovery effectiveness across different merchant segments, such as potentially higher recovery rates in certain countries or merchant tiers.

## Recovery Trend Analysis

This query analyzes recovery trends over time, broken down by recovery method.

```sql
-- Purpose: Track recovery trends over time by method
-- Shows month-over-month performance for each recovery method

SELECT
  DATE_TRUNC(created_at, MONTH) as month,  -- Group by month
  reason as recovery_method,
  COUNT(*) as attempt_count,
  SUM(amount_usd) as attempted_amount_usd,
  SUM(amount_usd) as recovered_amount_usd,
  ROUND(SUM(amount_usd) / NULLIF(SUM(amount_usd), 0) * 100, 2) as recovery_rate_pct,  -- Calculate success percentage
  NULL as avg_days_to_recovery
FROM
  `shopify-dw.risk.recovery_collections`
WHERE
  created_at >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 365 DAY)  -- Get data for past year
GROUP BY
  month, recovery_method
ORDER BY
  month DESC, recovered_amount_usd DESC  -- Sort by most recent month and highest recovery amount
```

### Description
This query tracks recovery performance trends over time by grouping collections by month and recovery method. It calculates monthly metrics such as attempt counts and recovery amounts for each method, allowing for tracking of method effectiveness over time and identifying seasonal patterns or improvements in recovery strategies.

### Example Output
The results provide a month-by-month breakdown of recovery metrics by method. This allows for tracking trends such as increasing effectiveness of certain methods or seasonal variations in recovery success. For example, you might observe that "payment_plan_scheduled" collections have been steadily increasing in effectiveness over recent months.

## Recovery Method Effectiveness

This query analyzes the effectiveness of different recovery methods across various amount ranges.

```sql
-- Purpose: Compare recovery method effectiveness by amount range
-- Identifies which methods work best for different recovery amount tiers

WITH amount_buckets AS (
  SELECT
    payment_plan_id as recovery_id,
    type as recovery_method,
    amount_usd as attempted_amount_usd,
    amount_usd as recovered_amount_usd,
    status as recovery_status,
    CASE  -- Categorize into amount buckets
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
    created_at >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 180 DAY)  -- Recent payment plans only
)

SELECT
  amount_bucket,
  recovery_method,
  COUNT(*) as attempt_count,
  SUM(attempted_amount_usd) as attempted_amount_usd,
  SUM(recovered_amount_usd) as recovered_amount_usd,
  -- Calculate recovery rate percentage with null handling
  ROUND(SUM(recovered_amount_usd) / NULLIF(SUM(attempted_amount_usd), 0) * 100, 2) as recovery_rate_pct,  
  COUNTIF(recovery_status = 'completed') as successful_count,
  -- Calculate success rate percentage with null handling
  ROUND(COUNTIF(recovery_status = 'completed') / NULLIF(COUNT(*), 0) * 100, 2) as success_rate_pct  
FROM
  amount_buckets
GROUP BY
  amount_bucket, recovery_method
ORDER BY
  CASE  -- Custom sorting for amount buckets
    WHEN amount_bucket = '0-$100' THEN 1
    WHEN amount_bucket = '$101-$500' THEN 2
    WHEN amount_bucket = '$501-$1,000' THEN 3
    WHEN amount_bucket = '$1,001-$5,000' THEN 4
    WHEN amount_bucket = '$5,001-$10,000' THEN 5
    WHEN amount_bucket = '$10,000+' THEN 6
  END,
  recovery_rate_pct DESC  -- Sort by highest recovery rate within each bucket
```

### Description
This query segments recovery payment plans into amount buckets and analyzes the effectiveness of different recovery methods within each bucket. It calculates metrics such as attempt counts, success rates, and recovery amounts to identify which methods work best for different recovery amount ranges. This helps optimize recovery strategy by matching the most effective method to each case based on amount.

### Example Output
The results show which recovery methods are most effective for different amount ranges. For example, you might find that "settlement" plans have higher success rates for larger amounts ($5,001-$10,000), while "repayment" plans are more effective for moderate amounts ($501-$1,000).

## Recovery Methods Reference

Common recovery methods include:

1. **Payment Plan Scheduled** - Regular scheduled payments according to a payment plan
2. **Merchant Initiated** - Payments initiated directly by the merchant
3. **Automatically Initiated** - System-triggered collection attempts
4. **Manually Initiated** - Collections initiated by Shopify support or risk team
5. **Settlement** - One-time lump sum payments to settle outstanding amounts
6. **Repayment** - Structured recurring payment plans (weekly or monthly)
7. **Chargeback** - Issuing a chargeback through the customer's card issuer
8. **Reserve Capture** - Using reserved funds to cover losses

## Temporal Fields Documentation

| Field Name | Data Type | Comparison Function | Example |
|------------|-----------|---------------------|---------|
| created_at | TIMESTAMP | TIMESTAMP_SUB() | `WHERE created_at >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 7 DAY)` |
| updated_at | TIMESTAMP | TIMESTAMP_SUB() | `WHERE updated_at >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 7 DAY)` |
| starts_at | TIMESTAMP | TIMESTAMP_SUB() | `WHERE starts_at >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 7 DAY)` |

## Notes and Best Practices

- Always include partition filters on `created_at` fields to optimize query performance
- Use `is_latest_payment_plan = TRUE` when querying payment plans to handle cases where plans are reopened
- For merchant-level analysis, consider adding context like GMV or merchant tier
- Use `NULLIF()` or `SAFE_DIVIDE()` functions to avoid division-by-zero errors
- When analyzing trends, use appropriate time windows (30/90/180/365 days) based on your analysis goals
- For small amounts (under $100), evaluate whether the recovery effort cost justifies the potential return
- Track recovery rate trends over time to identify process improvements or deterioration
- Document all recovery attempts for compliance and audit purposes
- Review aging recovery cases regularly to ensure timely action

Happy recovering! 💰 