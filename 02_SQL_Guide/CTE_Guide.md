# Common Table Expressions (CTEs) in SQL

## Overview

The example files demonstrate extensive use of Common Table Expressions (CTEs) in SQL queries for credit risk analysis. CTEs are temporary named result sets that exist only within the execution scope of a specific SQL statement and appear to be a key pattern in the team's query structure.

## Table of Contents

- [Available Information](#available-information)
- [Examples from Source Materials](#examples-from-source-materials)
- [Information Required](#information-required)

## Available Information

Based on the example files, we can observe that:

- CTEs are used to break down complex credit risk queries into logical components
- Multiple CTEs are chained together to build up complex analyses
- CTEs are named to describe their function (e.g., `shop_gpv`, `unfulfilled_120d_exposure`, `chargeback_dominate`)
- Window functions like `RANK()` and `QUALIFY` are used within CTEs
- Date-based calculations appear frequently in CTE filtering

## Examples from Source Materials

The following CTE examples are extracted directly from the source materials:

### Example 1: Shop GPV Calculation

```sql
-- Calculate total GPV (Gross Processing Volume) for a shop over the last 120 days
WITH shop_gpv AS (
    SELECT
        shop_id,
        ROUND(SUM(f.daily_sp_paid_amount_usd), 4) AS total_gpv 
    FROM
        `sdp-prd-cti-data.intermediate.shop_daily_order_fraud_facts` AS f
    WHERE date >= DATE_SUB(CURRENT_DATE(), INTERVAL 120 DAY)
    AND shop_id = @input_shop_id
    GROUP BY
        shop_id 
)
```

### Example 2: Handling Null Values

```sql
-- Calculate average shop review rating and handle null values
WITH shop_app AS ( 
    SELECT 
        shop_id,
        ROUND(AVG(rating), 1) AS shop_reviews
    FROM 
        `shopify-dw.intermediate.shop_product_reviews_history_v1_0`
    WHERE 
        shop_id = @input_shop_id
        AND valid_to IS NULL  -- Required filter on partition column
    GROUP BY 
        shop_id
)
SELECT
    shop_id,
    COALESCE(CAST(shop_reviews AS STRING), 'No reviews') AS shop_reviews
FROM
    shop_app
```

### Example 3: Dominant Chargeback Reason

```sql
-- Identify the most common chargeback reason for a shop
WITH chargeback_dominate AS (
    SELECT 
        shop_id,
        reason as reasons,
        COUNT(*) AS occurrence_count
    FROM 
        `shopify-dw.money_products.chargebacks_summary`
    WHERE shop_id = @input_shop_id
    AND provider_chargeback_created_at >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 120 DAY)  -- Using correct TIMESTAMP function for timestamp field
    GROUP BY 
        shop_id, reasons 
    QUALIFY 
        RANK() OVER (
            PARTITION BY shop_id 
            ORDER BY 
                occurrence_count DESC, 
                CASE -- if there's a tie, use this ranking
                    WHEN reasons = 'product_not_received' THEN 1
                    WHEN reasons = 'product_unacceptable' THEN 2
                    WHEN reasons = 'credit_not_processed' THEN 3
                    WHEN reasons = 'fraudulent' THEN 4
                    WHEN reasons = 'general' THEN 5
                    WHEN reasons = 'noncompliant' THEN 6
                    WHEN reasons = 'customer_initiated' THEN 7
                    WHEN reasons = 'incorrect_account_details' THEN 8
                    WHEN reasons = 'unrecognized' THEN 9
                    WHEN reasons = 'subscription_canceled' THEN 10
                    ELSE 11
                END
        ) = 1
)
```

### Example 4: Multiple Sequential CTEs (Reserve Analysis)

From the reserve recommendation query, we can see how multiple CTEs are combined:

```sql
-- Example of chaining multiple CTEs for complex analysis
WITH shop_gpv AS (
    -- GPV calculation
    SELECT
        shop_id,
        ROUND(SUM(f.daily_sp_paid_amount_usd), 4) AS total_gpv 
    FROM
        `sdp-prd-cti-data.intermediate.shop_daily_order_fraud_facts` AS f
    WHERE date >= DATE_SUB(CURRENT_DATE(), INTERVAL 120 DAY)
    AND shop_id = @input_shop_id
    GROUP BY shop_id
), 
unfulfilled_120d_exposure AS (
    -- Unfulfilled order exposure calculation
    SELECT
        shop_id,
        SUM(daily_unfulfilled_paid_sp_order_amount_usd) AS unfulfilled_exposure
    FROM
        `sdp-prd-cti-data.intermediate.shop_daily_order_fraud_facts`
    WHERE date >= DATE_SUB(CURRENT_DATE(), INTERVAL 120 DAY)
    AND shop_id = @input_shop_id
    GROUP BY shop_id
),
chargeback_dominate AS (
    -- Dominant chargeback reason calculation
    SELECT 
        shop_id,
        reason as reasons,
        COUNT(*) AS occurrence_count
    FROM 
        `shopify-dw.money_products.chargebacks_summary`
    WHERE shop_id = @input_shop_id
    AND provider_chargeback_created_at >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 120 DAY)
    GROUP BY shop_id, reasons
    QUALIFY RANK() OVER (
        PARTITION BY shop_id 
        ORDER BY occurrence_count DESC
    ) = 1
), 
-- Additional CTEs for other data points
-- ...

-- Main query combining all CTEs
SELECT 
    -- Combined fields from all CTEs
    sg.shop_id,
    sg.total_gpv,
    ue.unfulfilled_exposure,
    cd.reasons AS dominant_chargeback_reason
FROM 
    shop_gpv sg
LEFT JOIN 
    unfulfilled_120d_exposure ue ON sg.shop_id = ue.shop_id
LEFT JOIN
    chargeback_dominate cd ON sg.shop_id = cd.shop_id
-- Additional joins
```

## Temporal Fields Reference

| Field Name | Data Type | Comparison Function | Example |
|------------|-----------|---------------------|---------|
| date | DATE | DATE_SUB() | `WHERE date >= DATE_SUB(CURRENT_DATE(), INTERVAL 7 DAY)` |
| provider_chargeback_created_at | TIMESTAMP | TIMESTAMP_SUB() | `WHERE provider_chargeback_created_at >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 30 DAY)` |
| valid_to | TIMESTAMP | IS NULL / TIMESTAMP_SUB() | `WHERE valid_to IS NULL` |

## Information Required

To complete a comprehensive CTE Guide, the following information is needed:

### General CTE Information

- Question 1: What are the team's specific conventions for naming CTEs?
- Question 2: Are there performance considerations specific to Shopify's BigQuery implementation?
- Question 3: What is the maximum recommended number of CTEs in a single query?

### Best Practices

- Question 1: What are the Credit Risk team's best practices for CTE structure?
- Question 2: Are there standard CTE patterns that should be used for specific types of analyses?
- Question 3: How should CTEs be commented for maximum clarity?

### Troubleshooting

- Question 1: What are common CTE-related errors encountered by the team?
- Question 2: What debugging techniques are recommended for complex CTE chains?
- Question 3: Are there known limitations or pitfalls with CTEs in the team's environment?

---

*Note: This document provides factual information extracted directly from the source materials and validated through MCP. It requires additional input from the Credit Risk team to complete. Once finalized, it will provide comprehensive guidance on using CTEs for credit risk analysis.*