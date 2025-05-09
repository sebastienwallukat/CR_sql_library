# Fraud Detection Analysis

## Introduction
Fraud detection is a critical component of credit risk management. This page provides SQL queries to identify fraud patterns, analyze fraud indicators, and measure the effectiveness of detection mechanisms.

## Table of Contents
- [Fraud Ticket Analysis by Type](#fraud-ticket-analysis-by-type)
- [Fraud Detection Time Analysis](#fraud-detection-time-analysis)
- [Fraud-Related Chargebacks Analysis](#fraud-related-chargebacks-analysis)
- [Shop Termination Analysis](#shop-termination-analysis)
- [Shop Loss Metrics Analysis](#shop-loss-metrics-analysis)
- [Best Practices](#best-practices)
- [References](#references)

## Datasets
The following datasets contain fraud-related information:

- `shopify-dw.risk.trust_platform_tickets_summary_v1` - Tickets for fraud cases and investigations
- `shopify-dw.money_products.chargebacks_summary` - Disputes and chargebacks with fraud-related reason codes
- `shopify-dw.risk.shop_terminations_history` - History of shop terminations with reasons
- `shopify-dw.risk.shop_loss_metrics_daily_summary` - Shop loss metrics for trust reporting
- `shopify-dw.risk.trust_platform_shopify_payments_account_activity` - Shopify Payments account activity related to fraud

## Fraud Ticket Analysis by Type

### SQL Query
```sql
-- Purpose: Analyze fraud tickets by type and source to identify most common fraud patterns
-- This query categorizes fraud tickets by their report type and source, providing counts and response metrics
SELECT
  latest_report_type AS fraud_type,                       -- Type of fraud report
  source,                                                 -- Origin of the fraud report
  COUNT(*) AS fraud_count,                                -- Total number of tickets
  COUNT(DISTINCT subjectable_id) AS unique_entities,      -- Number of unique affected entities
  AVG(time_to_first_action) AS avg_time_to_action_minutes -- Average response time in minutes
FROM
  `shopify-dw.risk.trust_platform_tickets_summary_v1`
WHERE
  created_at >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 90 DAY)  -- Only consider recent tickets (last 90 days)
  AND latest_report_group = 'fraud'                                   -- Filter for fraud reports only
GROUP BY
  fraud_type, source
ORDER BY
  fraud_count DESC                                        -- Show most common fraud types first
```

### Description
This query analyzes fraud tickets created in the last 90 days by categorizing them according to the type of fraud and the source of the report. It calculates the total count of fraud tickets for each category, the number of unique entities affected, and the average time taken to respond to these tickets. This analysis helps identify the most prevalent types of fraud and assess the efficiency of the response mechanisms.

### Example
The query results might look like:

| fraud_type | source | fraud_count | unique_entities | avg_time_to_action_minutes |
|------------|--------|-------------|-----------------|----------------------------|
| card_testing | automated | 1250 | 842 | 12.5 |
| unauthorized_transactions | manual | 520 | 498 | 25.8 |
| stolen_financial_info | customer | 315 | 293 | 18.2 |
| identity_theft | partner | 105 | 102 | 30.1 |

## Fraud Detection Time Analysis

### SQL Query
```sql
-- Purpose: Analyze the time taken to detect and respond to fraud over time
-- This query tracks monthly averages and trends in fraud detection efficiency
SELECT
  DATE_TRUNC(DATE(created_at), MONTH) AS month,                   -- Group by month
  COUNT(*) AS fraud_count,                                         -- Total fraud tickets in month
  AVG(time_to_first_action) AS avg_minutes_to_detection,           -- Average detection time
  MIN(time_to_first_action) AS min_minutes_to_detection,           -- Fastest detection time
  MAX(time_to_first_action) AS max_minutes_to_detection,           -- Slowest detection time
  -- Median detection time (useful for eliminating outlier effects)
  APPROX_QUANTILES(time_to_first_action, 100)[OFFSET(50)] AS median_minutes_to_detection
FROM
  `shopify-dw.risk.trust_platform_tickets_summary_v1`
WHERE
  created_at >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 365 DAY)  -- Look back one year
  AND latest_report_group = 'fraud'                                    -- Only fraud reports
  AND first_action_at IS NOT NULL                                      -- Only include tickets with actions taken
GROUP BY
  month
ORDER BY
  month                                                               -- Sort chronologically
```

### Description
This query provides a monthly breakdown of fraud detection efficiency metrics over the past year. It calculates the average, minimum, maximum, and median time to detect and respond to fraud incidents. By tracking these metrics over time, analysts can identify trends and improvements in fraud detection capabilities, as well as potential areas where response times may be degrading. The median is particularly useful to eliminate the effect of outliers on the average.

### Example
The query results might look like:

| month | fraud_count | avg_minutes_to_detection | min_minutes_to_detection | max_minutes_to_detection | median_minutes_to_detection |
|-------|-------------|--------------------------|--------------------------|--------------------------|----------------------------|
| 2023-10-01 | 2850 | 18.2 | 0.5 | 2544.3 | 12.7 |
| 2023-11-01 | 3120 | 16.8 | 0.4 | 1987.2 | 11.5 |
| 2023-12-01 | 4250 | 15.3 | 0.3 | 2103.7 | 10.8 |
| 2024-01-01 | 3580 | 14.1 | 0.3 | 1856.4 | 9.7 |

## Fraud-Related Chargebacks Analysis

### SQL Query
```sql
-- Purpose: Analyze chargebacks specifically related to fraud reasons
-- This CTE-based query isolates fraud chargebacks and calculates key performance metrics
WITH fraud_chargebacks AS (
  SELECT
    chargeback_id,                      -- Unique identifier for each chargeback
    reason,                             -- Reason code for the chargeback
    status,                             -- Current status of the chargeback
    shop_id,                            -- Shop affected by the chargeback
    chargeback_amount_usd,              -- Monetary impact in USD
    provider_chargeback_created_at      -- When the chargeback was filed
  FROM
    `shopify-dw.money_products.chargebacks_summary`
  WHERE
    provider_chargeback_created_at >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 180 DAY)  -- Last 6 months
    AND reason IN ('fraudulent', 'fraud_monitoring')                                         -- Only fraud reasons
)

SELECT
  reason,                                                 -- Fraud reason code
  COUNT(DISTINCT chargeback_id) AS chargeback_count,      -- Total number of chargebacks
  COUNT(DISTINCT shop_id) AS affected_shops,              -- Number of unique shops affected
  SUM(chargeback_amount_usd) AS total_chargeback_amount_usd,  -- Total financial impact
  AVG(chargeback_amount_usd) AS avg_chargeback_amount_usd,    -- Average chargeback amount
  COUNTIF(status = 'won') / COUNT(*) AS win_rate          -- Rate of successfully disputed chargebacks
FROM
  fraud_chargebacks
GROUP BY
  reason
ORDER BY
  chargeback_count DESC                                   -- Show most common reasons first
```

### Description
This query analyzes chargebacks that are specifically related to fraud over the past six months. It first filters chargebacks to only those with fraud-related reason codes using a Common Table Expression (CTE). Then it calculates several key metrics for each fraud reason, including the total count of chargebacks, the number of unique shops affected, the total financial impact, the average chargeback amount, and the win rate for disputed chargebacks. This analysis helps quantify the financial impact of fraud and assess the effectiveness of chargeback dispute processes.

### Example
The query results might look like:

| reason | chargeback_count | affected_shops | total_chargeback_amount_usd | avg_chargeback_amount_usd | win_rate |
|--------|------------------|----------------|------------------------------|----------------------------|----------|
| fraudulent | 3856 | 2145 | 958745.23 | 248.64 | 0.32 |
| fraud_monitoring | 1257 | 836 | 412568.75 | 328.22 | 0.21 |

## Shop Termination Analysis

### SQL Query
```sql
-- Purpose: Analyze shop terminations related to fraud to understand enforcement patterns
-- This query examines termination reasons, counts, and durations
SELECT
  termination_reason,                                   -- Reason for shop termination
  COUNT(DISTINCT shop_id) AS terminated_shops_count,    -- Number of shops terminated
  -- Average time (in days) shops remained terminated before restoration or current time
  AVG(TIMESTAMP_DIFF(COALESCE(valid_to, CURRENT_TIMESTAMP()), valid_from, DAY)) AS avg_termination_duration_days
FROM
  `shopify-dw.risk.shop_terminations_history`
WHERE
  valid_from >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 90 DAY)  -- Only recent terminations (90 days)
  AND termination_reason LIKE '%fraud%'                               -- Filter for fraud-related terminations
GROUP BY
  termination_reason
ORDER BY
  terminated_shops_count DESC                                         -- Show most common reasons first
```

### Description
This query analyzes shop terminations related to fraud over the past 90 days. It groups terminations by the specific reason code and calculates the count of shops terminated for each reason, as well as the average duration of these terminations in days. For shops that have not been restored, the current timestamp is used to calculate the ongoing duration. This analysis helps understand the severity and frequency of different types of fraud that lead to shop terminations, and how long these enforcement actions typically last.

### Example
The query results might look like:

| termination_reason | terminated_shops_count | avg_termination_duration_days |
|--------------------|-----------------------|-------------------------------|
| credit_card_fraud | 587 | 365.2 |
| identity_fraud | 245 | 420.8 |
| transaction_fraud | 128 | 180.5 |
| merchant_fraud | 93 | 540.3 |

## Shop Loss Metrics Analysis

### SQL Query
```sql
-- Purpose: Analyze financial losses due to fraud over time
-- This query tracks monthly loss trends and metrics
SELECT
  DATE_TRUNC(date, MONTH) AS month,                       -- Group by month
  COUNT(DISTINCT shop_id) AS shops_with_losses,           -- Number of shops with losses
  SUM(net_sp_booked_loss_usd) AS total_loss_amount_usd,   -- Total financial impact
  AVG(net_sp_booked_loss_usd) AS avg_loss_per_shop_usd    -- Average loss per affected shop
FROM
  `shopify-dw.risk.shop_loss_metrics_daily_summary`
WHERE
  date >= DATE_SUB(CURRENT_DATE(), INTERVAL 180 DAY)      -- Last 6 months of data
GROUP BY
  month
ORDER BY
  month DESC                                              -- Most recent months first
```

### Description
This query analyzes the financial losses associated with shop activities over the past six months, aggregated by month. It calculates the number of shops experiencing losses each month, the total financial impact of these losses, and the average loss amount per shop. This analysis helps track trends in financial losses over time, which can provide insights into the effectiveness of fraud prevention measures and identify potential seasonal patterns in fraud-related losses.

### Example
The query results might look like:

| month | shops_with_losses | total_loss_amount_usd | avg_loss_per_shop_usd |
|-------|-------------------|------------------------|------------------------|
| 2024-08-01 | 542 | 1254823.58 | 2315.17 |
| 2024-07-01 | 628 | 1587412.36 | 2527.73 |
| 2024-06-01 | 712 | 1897632.45 | 2665.21 |
| 2024-05-01 | 657 | 1743217.89 | 2653.30 |

## Best Practices

- Monitor fraud patterns regularly to adapt detection rules
- Balance false positive and false negative rates for optimal protection
- Implement layered fraud detection with multiple signal types
- Analyze fraud by merchant category to develop targeted prevention strategies
- Track time-to-detection to measure effectiveness of real-time systems
- Review rule precision and recall regularly to improve fraud models
- Combine machine learning predictions with rule-based systems
- Maintain reference data for known fraud patterns and evolving techniques

## References
- [Fraud Detection Methodology](https://shopify.dev/docs)
- [Risk Signal Definitions](https://shopify.dev/api)
- [Transaction Monitoring Best Practices](https://shopify.dev/docs) 
