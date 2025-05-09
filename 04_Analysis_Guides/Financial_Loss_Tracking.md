# Booked Losses Analysis

## Introduction
Booked losses represent financial impacts from various risk events, including chargebacks, fraud, and merchant bankruptcies. This analysis provides SQL queries to analyze losses, identify trends, and categorize losses for better risk management at Shopify.

## Table of Contents
- [Introduction](#introduction)
- [Datasets](#datasets)
- [Common Queries](#common-queries)
  - [Monthly Loss Trend by Provider](#monthly-loss-trend-by-provider)
  - [Loss by Merchant Category](#loss-by-merchant-category)
  - [High-Value Loss Merchants](#high-value-loss-merchants)
  - [Loss-to-GPV Ratio by Merchant Category](#loss-to-gpv-ratio-by-merchant-category)
  - [Balance-Exacerbated Exposure Analysis](#balance-exacerbated-exposure-analysis)
  - [Termination Analysis for Fraud Shops with Losses](#termination-analysis-for-fraud-shops-with-losses)
- [Basic Booked Loss Query](#basic-booked-loss-query)
- [Aggregate Loss Analysis](#aggregate-loss-analysis)
- [Merchants with High Losses](#merchants-with-high-losses)
- [Timeframe Comparison](#timeframe-comparison)
- [Loss and Chargeback Relationship](#loss-and-chargeback-relationship)
- [Best Practices](#best-practices)
- [Temporal Fields Documentation](#temporal-fields-documentation)
- [References](#references)

## Datasets
The following datasets contain information related to booked losses:

- `shopify-dw.money_products.shopify_payments_estimated_booked_losses` - Primary source for Shopify Payments booked loss data
- `shopify-dw.mart_cti_data.shop_loss_metrics__wide` - Merchant-level loss metrics and associated attributes
- `shopify-dw.money_products.banking_balance_exacerbated_shopify_payments_exposure` - Balance-related payment exposure data
- `shopify-dw.risk.shop_terminations_summary` - Merchant termination records
- `shopify-dw.risk.shop_terminations_history` - Detailed history of merchant terminations

## Common Queries

### Monthly Loss Trend by Provider
```sql
SELECT
  DATE_TRUNC(booked_date, MONTH) AS month,
  provider_name,
  COUNT(*) AS loss_count,
  SUM(booked_loss_usd) AS total_loss_usd,
  AVG(booked_loss_usd) AS avg_loss_usd
FROM
  `shopify-dw.money_products.shopify_payments_estimated_booked_losses`
WHERE
  booked_date >= DATE_SUB(CURRENT_DATE(), INTERVAL 12 MONTH)
GROUP BY
  month, provider_name
ORDER BY
  month DESC, total_loss_usd DESC
```

#### Description
This query analyzes monthly booked losses by payment provider over the past 12 months. It helps identify trends and anomalies in the loss patterns across different payment providers. The query:
- Groups data by month and provider
- Counts the number of loss events
- Calculates total and average loss amounts in USD
- Orders results by most recent month and highest total loss

#### Example
| month      | provider_name | loss_count | total_loss_usd  | avg_loss_usd |
|------------|---------------|------------|-----------------|--------------|
| 2025-05-01 | stripe        | 12,022     | 679,398.77      | 56.51        |
| 2025-05-01 | paypal        | 640        | 227,805.58      | 355.95       |
| 2025-04-01 | stripe        | 49,924     | 12,715,232.86   | 254.69       |
| 2025-04-01 | paypal        | 2,840      | 1,261,874.43    | 444.32       |
| 2025-03-01 | stripe        | 63,751     | 13,880,926.04   | 217.74       |

### Loss by Merchant Category
```sql
SELECT
  s.shop_primary_predicted_product_category AS merchant_category,
  COUNT(DISTINCT s.shop_id) AS merchant_count,
  SUM(s.net_sp_booked_loss_usd) AS total_loss_usd,
  AVG(s.net_sp_booked_loss_usd) AS avg_loss_usd
FROM
  `shopify-dw.mart_cti_data.shop_loss_metrics__wide` s
WHERE
  s.date >= DATE_SUB(CURRENT_DATE(), INTERVAL 6 MONTH)
GROUP BY
  merchant_category
HAVING
  merchant_count > 10
ORDER BY
  total_loss_usd DESC
```

#### Description
This query analyzes booked losses by merchant product category over the past 6 months. It helps identify which product categories are associated with higher loss rates. The query:
- Groups merchants by their primary predicted product category
- Calculates total losses and average loss per merchant
- Filters out categories with fewer than 10 merchants for statistical significance
- Orders results by highest total loss

#### Example
| merchant_category            | merchant_count | total_loss_usd  | avg_loss_usd |
|------------------------------|----------------|-----------------|--------------|
| Clothing & Fashion           | 1,523          | 5,245,678.91    | 3,443.65     |
| Electronics & Accessories    | 742            | 4,578,921.34    | 6,170.65     |
| Beauty & Personal Care       | 893            | 2,784,562.18    | 3,118.21     |
| Health & Wellness            | 621            | 2,156,789.45    | 3,473.09     |
| Home & Garden               | 458            | 1,845,672.89    | 4,030.07     |

### High-Value Loss Merchants
```sql
SELECT
  shop_id,
  COUNT(*) AS loss_count,
  SUM(net_sp_booked_loss_usd) AS total_loss_usd,
  MAX(net_sp_booked_loss_usd) AS max_single_loss_usd,
  MIN(date) AS first_loss_date,
  MAX(date) AS last_loss_date
FROM
  `shopify-dw.mart_cti_data.shop_loss_metrics__wide`
WHERE
  date >= DATE_SUB(CURRENT_DATE(), INTERVAL 12 MONTH)
GROUP BY
  shop_id
HAVING
  total_loss_usd > 10000  -- Focus on high-value losses
ORDER BY
  total_loss_usd DESC
LIMIT 100
```

#### Description
This query identifies merchants with high-value losses over the past 12 months. It helps focus risk mitigation efforts on the most impactful cases. The query:
- Groups loss data by merchant
- Calculates total loss amount and maximum single loss
- Identifies first and last loss dates to understand loss timeline
- Filters for merchants with total losses exceeding $10,000
- Limits results to top 100 merchants by total loss amount

#### Example
| shop_id    | loss_count | total_loss_usd  | max_single_loss_usd | first_loss_date | last_loss_date |
|------------|------------|-----------------|---------------------|-----------------|----------------|
| 12345678   | 8          | 87,452.32       | 32,456.78           | 2024-06-15      | 2025-03-22     |
| 23456789   | 12         | 64,789.45       | 18,234.56           | 2024-08-03      | 2025-05-11     |
| 34567890   | 5          | 52,345.67       | 28,765.43           | 2024-09-18      | 2025-02-27     |
| 45678901   | 7          | 48,912.34       | 15,678.90           | 2024-07-22      | 2025-04-30     |
| 56789012   | 9          | 42,567.89       | 14,321.56           | 2024-11-10      | 2025-05-03     |

### Loss-to-GPV Ratio by Merchant Category
```sql
WITH merchant_gpv AS (
  SELECT
    m.shop_id,
    m.shop_primary_predicted_product_category AS merchant_category,
    SUM(m.gpv_usd) AS total_gpv_usd
  FROM
    `sdp-prd-cti-data.intermediate.shop_trust_daily_snapshot` m
  WHERE
    m.date >= DATE_SUB(CURRENT_DATE(), INTERVAL 6 MONTH)
  GROUP BY
    m.shop_id, m.shop_primary_predicted_product_category
),

merchant_losses AS (
  SELECT
    l.shop_id,
    SUM(l.net_sp_booked_loss_usd) AS total_loss_usd
  FROM
    `shopify-dw.mart_cti_data.shop_loss_metrics__wide` l
  WHERE
    l.date >= DATE_SUB(CURRENT_DATE(), INTERVAL 6 MONTH)
  GROUP BY
    l.shop_id
)

SELECT
  g.merchant_category,
  COUNT(DISTINCT g.shop_id) AS merchant_count,
  SUM(g.total_gpv_usd) AS category_gpv_usd,
  SUM(COALESCE(l.total_loss_usd, 0)) AS category_loss_usd,
  ROUND(SUM(COALESCE(l.total_loss_usd, 0)) / NULLIF(SUM(g.total_gpv_usd), 0) * 100, 4) AS loss_to_gpv_ratio_pct
FROM
  merchant_gpv g
LEFT JOIN
  merchant_losses l
  ON g.shop_id = l.shop_id
GROUP BY
  g.merchant_category
HAVING
  category_gpv_usd > 1000000  -- Only include categories with meaningful GPV
ORDER BY
  loss_to_gpv_ratio_pct DESC
```

#### Description
This query calculates the loss-to-GPV (Gross Payment Volume) ratio by merchant category over the past 6 months. It helps understand the relative risk of different product categories. The query:
- First calculates total GPV by merchant and category
- Then calculates total losses by merchant
- Joins the two datasets and calculates the loss-to-GPV ratio as a percentage
- Filters out categories with less than $1 million in GPV for statistical significance
- Orders results by highest loss-to-GPV ratio

#### Example
| merchant_category            | merchant_count | category_gpv_usd   | category_loss_usd | loss_to_gpv_ratio_pct |
|------------------------------|----------------|--------------------|--------------------|------------------------|
| Digital Products             | 378            | 45,678,921.45      | 1,245,678.23       | 2.7272                 |
| Dropshipping                 | 512            | 78,923,456.78      | 1,789,234.56       | 2.2671                 |
| Collectibles                 | 267            | 34,567,892.12      | 756,432.12         | 2.1883                 |
| Jewelry & Accessories        | 423            | 56,789,234.56      | 987,654.32         | 1.7392                 |
| Clothing & Fashion           | 1,523          | 123,456,789.12     | 1,654,321.98       | 1.3400                 |

### Balance-Exacerbated Exposure Analysis
```sql
SELECT
  DATE_TRUNC(negative_balance_created_on, MONTH) AS month,
  COUNT(*) AS exposure_events,
  SUM(balance_exacerbated_sp_exposure_usd) AS balance_exacerbated_exposure_usd,
  AVG(balance_exacerbated_sp_exposure_usd) AS avg_exposure_per_event_usd,
  SUM(balance_exacerbated_sp_exposure_usd) / COUNT(*) AS mean_exposure_per_event_usd
FROM
  `shopify-dw.money_products.banking_balance_exacerbated_shopify_payments_exposure`
WHERE
  negative_balance_created_on >= DATE_SUB(CURRENT_DATE(), INTERVAL 12 MONTH)
GROUP BY
  month
ORDER BY
  month DESC
```

#### Description
This query analyzes balance-exacerbated exposure events by month over the past 12 months. It helps understand the impact of Shopify Balance's faster payout speeds on potential losses. The query:
- Groups exposure events by month
- Counts the number of exposure events
- Calculates total and average exposure amounts in USD
- Orders results by most recent month

#### Example
| month      | exposure_events | balance_exacerbated_exposure_usd | avg_exposure_per_event_usd | mean_exposure_per_event_usd |
|------------|-----------------|----------------------------------|----------------------------|----------------------------|
| 2025-05-01 | 1,245           | 2,345,678.90                     | 1,883.28                   | 1,883.28                   |
| 2025-04-01 | 1,342           | 2,678,921.34                     | 1,996.22                   | 1,996.22                   |
| 2025-03-01 | 1,567           | 3,123,456.78                     | 1,992.63                   | 1,992.63                   |
| 2025-02-01 | 1,432           | 2,876,543.21                     | 2,009.46                   | 2,009.46                   |
| 2025-01-01 | 1,378           | 2,765,432.10                     | 2,006.85                   | 2,006.85                   |

### Termination Analysis for Fraud Shops with Losses
```sql
SELECT
  t.latest_termination_reason_category,
  COUNT(DISTINCT l.shop_id) AS merchant_count,
  SUM(l.net_sp_booked_loss_usd) AS total_loss_usd,
  AVG(l.net_sp_booked_loss_usd) AS avg_loss_per_merchant_usd,
  MIN(TIMESTAMP_DIFF(l.date, CAST(t.first_termination_at AS DATE), DAY)) AS min_days_to_first_loss,
  AVG(TIMESTAMP_DIFF(l.date, CAST(t.first_termination_at AS DATE), DAY)) AS avg_days_to_first_loss,
  MAX(TIMESTAMP_DIFF(l.date, CAST(t.first_termination_at AS DATE), DAY)) AS max_days_to_first_loss
FROM
  `shopify-dw.mart_cti_data.shop_loss_metrics__wide` l
JOIN
  `shopify-dw.risk.shop_terminations_summary` t
  ON l.shop_id = t.shop_id
WHERE
  l.is_shop_still_terminated_for_fraud = TRUE
  AND l.date >= DATE_SUB(CURRENT_DATE(), INTERVAL 12 MONTH)
GROUP BY
  t.latest_termination_reason_category
ORDER BY
  total_loss_usd DESC
```

#### Description
This query analyzes the relationship between shop terminations and booked losses over the past 12 months. It helps understand how different termination reasons correlate with loss patterns. The query:
- Joins loss data with termination data for currently terminated shops
- Groups by termination reason category
- Calculates total losses and average loss per merchant
- Analyzes the time gap between termination and first loss
- Orders results by highest total loss

#### Example
| latest_termination_reason_category | merchant_count | total_loss_usd | avg_loss_per_merchant_usd | min_days_to_first_loss | avg_days_to_first_loss | max_days_to_first_loss |
|------------------------------------|----------------|----------------|---------------------------|------------------------|------------------------|------------------------|
| Unauthorized Reseller              | 256            | 4,567,892.34   | 17,842.55                 | -45                    | 12.3                   | 143                    |
| Payment Fraud                      | 423            | 3,789,234.56   | 8,957.77                  | -30                    | 5.7                    | 95                     |
| Counterfeit                        | 187            | 2,345,678.90   | 12,543.74                 | -22                    | 8.9                    | 112                    |
| High Risk                          | 345            | 1,987,654.32   | 5,760.74                  | -18                    | 15.2                   | 132                    |
| Prohibited Business                | 178            | 1,456,789.23   | 8,184.21                  | -27                    | 7.5                    | 87                     |

## Basic Booked Loss Query

```sql
SELECT
  bl.shop_id,
  bl.transaction_id,
  bl.shop_currency_code,
  bl.booked_loss_amount_usd,
  bl.booked_loss_amount_shop_currency,
  bl.created_at as booked_loss_created_at,
  bl.updated_at as booked_loss_updated_at, 
  bl.booked_loss_category,
  bl.booked_loss_reason,
  bl.booked_loss_description,
  o.order_name,
  o.payment_method,
  o.processor_type
FROM `shopify-dw.money_products.booked_losses` bl
JOIN `shopify-dw.money_products.order_transactions_payments_summary` o
  ON bl.transaction_id = o.transaction_id
WHERE bl.shop_id = 12345678
ORDER BY bl.created_at DESC
LIMIT 100
```

#### Description
This query retrieves basic information about booked losses for a specific merchant, joining with transaction data to provide context. The query:
- Selects key booked loss attributes including amounts, timestamps, and categories
- Joins with transaction data to get order and payment information
- Filters for a specific merchant (shop_id)
- Orders results by most recent losses
- Limits results to 100 records for performance

#### Example
| shop_id  | transaction_id | shop_currency_code | booked_loss_amount_usd | booked_loss_amount_shop_currency | booked_loss_created_at | updated_at           | booked_loss_category | booked_loss_reason | booked_loss_description | order_name | payment_method | processor_type |
|----------|---------------|---------------------|------------------------|----------------------------------|------------------------|----------------------|----------------------|--------------------|-----------------------|-----------|---------------|---------------|
| 12345678 | tx_abc123def  | USD                 | 345.67                 | 345.67                           | 2025-05-01 14:23:45    | 2025-05-01 14:23:45  | Chargeback           | Fraud               | Customer disputed charge | #1001      | credit_card    | stripe         |
| 12345678 | tx_def456ghi  | USD                 | 189.99                 | 189.99                           | 2025-04-28 09:12:34    | 2025-04-28 09:12:34  | Chargeback           | Product not received | Item never delivered    | #1002      | credit_card    | stripe         |
| 12345678 | tx_ghi789jkl  | USD                 | 567.89                 | 567.89                           | 2025-04-15 16:45:23    | 2025-04-15 16:45:23  | Merchant bankruptcy  | Business closed     | Merchant terminated     | #1003      | credit_card    | stripe         |

## Aggregate Loss Analysis

```sql
WITH loss_data AS (
  SELECT
    DATE_TRUNC(created_at, MONTH) as month,
    booked_loss_category,
    booked_loss_reason,
    SUM(booked_loss_amount_usd) as total_loss_usd,
    COUNT(*) as loss_count
  FROM `shopify-dw.money_products.booked_losses`
  WHERE created_at >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 12 MONTH)
  GROUP BY month, booked_loss_category, booked_loss_reason
)

SELECT
  month,
  booked_loss_category,
  booked_loss_reason,
  total_loss_usd,
  loss_count,
  ROUND(total_loss_usd / loss_count, 2) as avg_loss_per_incident
FROM loss_data
ORDER BY month DESC, total_loss_usd DESC
```

#### Description
This query provides an aggregate analysis of booked losses by category and reason over the past 12 months. It helps identify patterns in loss types and their financial impact. The query:
- Groups losses by month, category, and reason
- Calculates total loss amount and count by group
- Calculates average loss per incident
- Orders results by most recent month and highest total loss

#### Example
| month      | booked_loss_category | booked_loss_reason      | total_loss_usd | loss_count | avg_loss_per_incident |
|------------|----------------------|-------------------------|----------------|------------|------------------------|
| 2025-05-01 | Chargeback           | Fraud                   | 1,234,567.89   | 876        | 1,409.32               |
| 2025-05-01 | Chargeback           | Product not received    | 876,543.21     | 654        | 1,340.28               |
| 2025-05-01 | Merchant bankruptcy  | Business closed         | 567,890.12     | 234        | 2,426.88               |
| 2025-04-01 | Chargeback           | Fraud                   | 1,345,678.90   | 923        | 1,458.00               |
| 2025-04-01 | Chargeback           | Product not as described| 987,654.32     | 712        | 1,387.15               |

## Merchants with High Losses

```sql
SELECT
  bl.shop_id,
  s.shop_name,
  s.shop_domain,
  s.trust_battery,
  COUNT(*) as loss_count,
  SUM(bl.booked_loss_amount_usd) as total_loss_usd,
  MIN(bl.created_at) as first_loss_date,
  MAX(bl.created_at) as latest_loss_date,
  ARRAY_AGG(DISTINCT bl.booked_loss_category ORDER BY bl.booked_loss_category) as loss_categories
FROM `shopify-dw.money_products.booked_losses` bl
JOIN sdp-prd-cti-data.intermediate.shop_insights_shops s ON bl.shop_id = s.shop_id
WHERE bl.created_at >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 180 DAY)
GROUP BY bl.shop_id, s.shop_name, s.shop_domain, s.trust_battery
HAVING total_loss_usd > 1000
ORDER BY total_loss_usd DESC
LIMIT 100
```

#### Description
This query identifies merchants with high booked losses over the past 180 days, including shop profile information and trust signals. The query:
- Joins booked loss data with shop information
- Groups by merchant and calculates total loss amount and count
- Records first and latest loss dates to understand loss timeline
- Aggregates unique loss categories to understand loss diversity
- Filters for merchants with losses exceeding $1,000
- Orders results by highest total loss
- Limits results to top 100 merchants

#### Example
| shop_id  | shop_name        | shop_domain         | trust_battery | loss_count | total_loss_usd | first_loss_date      | latest_loss_date     | loss_categories                            |
|----------|------------------|---------------------|---------------|------------|----------------|----------------------|----------------------|--------------------------------------------|
| 12345678 | Tech Gadgets     | techgadgets.myshopify.com | 68      | 12         | 45,678.90      | 2024-12-15 08:23:45  | 2025-05-02 14:56:12  | ["Chargeback", "Merchant bankruptcy"]       |
| 23456789 | Fashion Hub      | fashionhub.myshopify.com  | 42      | 18         | 32,456.78      | 2025-01-03 12:34:56  | 2025-04-28 09:45:23  | ["Chargeback"]                             |
| 34567890 | Home Essentials  | homeessentials.myshopify.com | 51  | 8          | 28,765.43      | 2025-02-17 16:45:23  | 2025-05-04 11:23:45  | ["Chargeback", "Fraud complaint"]          |

## Timeframe Comparison

```sql
WITH monthly_losses AS (
  SELECT
    DATE_TRUNC(created_at, MONTH) as month,
    SUM(booked_loss_amount_usd) as monthly_loss_usd,
    COUNT(*) as loss_count,
    COUNT(DISTINCT shop_id) as affected_merchant_count
  FROM `shopify-dw.money_products.booked_losses`
  WHERE created_at >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 24 MONTH)
  GROUP BY month
)

SELECT
  month,
  monthly_loss_usd,
  loss_count,
  affected_merchant_count,
  ROUND(monthly_loss_usd / loss_count, 2) as avg_loss_per_incident,
  ROUND(monthly_loss_usd / affected_merchant_count, 2) as avg_loss_per_merchant,
  ROUND(loss_count / affected_merchant_count, 2) as avg_incidents_per_merchant
FROM monthly_losses
ORDER BY month DESC
```

#### Description
This query compares booked losses across different monthly time periods over the past 24 months. It helps identify seasonal patterns and long-term trends in loss metrics. The query:
- Groups losses by month
- Calculates total loss amount, count, and number of affected merchants
- Derives average metrics including loss per incident, loss per merchant, and incidents per merchant
- Orders results by most recent month

#### Example
| month      | monthly_loss_usd | loss_count | affected_merchant_count | avg_loss_per_incident | avg_loss_per_merchant | avg_incidents_per_merchant |
|------------|------------------|------------|-------------------------|------------------------|------------------------|----------------------------|
| 2025-05-01 | 2,789,456.78     | 1,876      | 1,234                   | 1,486.92               | 2,260.50               | 1.52                       |
| 2025-04-01 | 3,456,789.23     | 2,345      | 1,567                   | 1,474.54               | 2,205.99               | 1.50                       |
| 2025-03-01 | 3,987,654.32     | 2,678      | 1,789                   | 1,489.04               | 2,228.99               | 1.50                       |
| 2025-02-01 | 3,234,567.89     | 2,234      | 1,456                   | 1,447.88               | 2,221.54               | 1.53                       |
| 2025-01-01 | 2,876,543.21     | 1,987      | 1,345                   | 1,447.68               | 2,138.69               | 1.48                       |

## Loss and Chargeback Relationship

```sql
WITH loss_data AS (
  SELECT
    shop_id,
    SUM(booked_loss_amount_usd) as total_loss_usd,
    COUNT(*) as loss_count
  FROM `shopify-dw.money_products.booked_losses`
  WHERE created_at >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 90 DAY)
  GROUP BY shop_id
),

chargeback_data AS (
  SELECT
    shop_id,
    COUNT(*) as chargeback_count,
    SUM(CASE WHEN dispute_reason LIKE '%fraud%' THEN 1 ELSE 0 END) as fraud_chargeback_count,
    SUM(CASE WHEN dispute_reason NOT LIKE '%fraud%' THEN 1 ELSE 0 END) as non_fraud_chargeback_count
  FROM `shopify-dw.money_products.chargebacks_summary`
  WHERE created_at >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 180 DAY)
  GROUP BY shop_id
)

SELECT
  l.shop_id,
  s.shop_name,
  s.shop_domain,
  l.total_loss_usd,
  l.loss_count,
  c.chargeback_count,
  c.fraud_chargeback_count,
  c.non_fraud_chargeback_count,
  SAFE_DIVIDE(c.chargeback_count, l.loss_count) as chargebacks_per_loss
FROM loss_data l
JOIN chargeback_data c ON l.shop_id = c.shop_id
JOIN sdp-prd-cti-data.intermediate.shop_insights_shops s ON l.shop_id = s.shop_id
WHERE l.total_loss_usd > 500
  AND c.chargeback_count > 10
ORDER BY l.total_loss_usd DESC
LIMIT 100
```

#### Description
This query analyzes the relationship between booked losses and chargebacks, focusing on merchants with significant activity in both areas. It helps understand how chargebacks contribute to financial losses. The query:
- Creates separate aggregations for loss data and chargeback data
- Joins these datasets along with shop information
- Categorizes chargebacks as fraud vs. non-fraud
- Calculates the ratio of chargebacks to losses
- Filters for merchants with meaningful loss amounts and chargeback volumes
- Orders results by highest total loss

#### Example
| shop_id  | shop_name        | shop_domain         | total_loss_usd | loss_count | chargeback_count | fraud_chargeback_count | non_fraud_chargeback_count | chargebacks_per_loss |
|----------|------------------|---------------------|----------------|------------|------------------|------------------------|----------------------------|----------------------|
| 12345678 | Tech Gadgets     | techgadgets.myshopify.com | 32,456.78 | 24        | 42               | 28                     | 14                         | 1.75                 |
| 23456789 | Fashion Hub      | fashionhub.myshopify.com  | 28,765.43 | 18        | 32               | 15                     | 17                         | 1.78                 |
| 34567890 | Jewelry Store    | jewelrystore.myshopify.com| 25,432.12 | 15        | 29               | 22                     | 7                          | 1.93                 |
| 45678901 | Sports Gear      | sportsgear.myshopify.com  | 21,876.54 | 12        | 24               | 9                      | 15                         | 2.00                 |
| 56789012 | Home Decor       | homedecor.myshopify.com   | 18,543.21 | 14        | 21               | 11                     | 10                         | 1.50                 |

## Best Practices

- Segment loss analysis by merchant category, size, and tenure to identify risk patterns
- Review recovery rates by loss type to optimize collection strategies
- Monitor the time gap between termination events and losses to improve early warning systems
- Correlate losses with specific risk indicators to enhance predictive models
- Establish loss reserves based on historical patterns by merchant category
- Conduct quarterly reviews of high-value loss merchants for risk mitigation
- Track loss-to-GPV ratios to establish healthy benchmarks by merchant category
- Document root causes for significant losses to prevent similar events

## Temporal Fields Documentation

### Key Temporal Fields

| Field Name | Data Type | Comparison Function | Example |
|------------|-----------|---------------------|---------|
| booked_date | DATE | DATE_SUB() | `WHERE booked_date >= DATE_SUB(CURRENT_DATE(), INTERVAL 7 DAY)` |
| date | DATE | DATE_SUB() | `WHERE date >= DATE_SUB(CURRENT_DATE(), INTERVAL 7 DAY)` |
| negative_balance_created_on | DATE | DATE_SUB() | `WHERE negative_balance_created_on >= DATE_SUB(CURRENT_DATE(), INTERVAL 7 DAY)` |
| shop_created_at | TIMESTAMP | TIMESTAMP_SUB() | `WHERE shop_created_at >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 7 DAY)` |
| first_termination_at | TIMESTAMP | TIMESTAMP_SUB() | `WHERE first_termination_at >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 7 DAY)` |

## References
- [Financial Loss Classification Guide](https://shopify.dev/docs)
- [Risk Mitigation Strategies](https://shopify.dev/api)
- [Loss Recovery Best Practices](https://shopify.dev/docs) 