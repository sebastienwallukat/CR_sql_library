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
WITH shop_gpv AS (
    SELECT
        shop_id,
        ROUND(SUM(f.daily_sp_paid_amount_usd), 4) AS total_gpv 
    FROM
        `sdp-prd-cti-data.intermediate.shop_daily_order_fraud_facts` AS f
    WHERE DATE(date) >= DATE_SUB(CURRENT_DATE(), INTERVAL 120 DAY)
    AND CAST(shop_id AS STRING) = @input_shop_id
    GROUP BY
        shop_id 
)
```

### Example 2: Handling Null Values

```sql
WITH shop_app AS ( 
    SELECT 
        shop_id,
        ROUND(AVG(rating), 1) AS shop_reviews
    FROM 
        `shopify-dw.intermediate.shop_product_reviews_history_v1_0`
    WHERE 
        CAST(shop_id AS STRING) = @input_shop_id
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
WITH chargeback_dominate AS (
    SELECT 
        shop_id,
        reason as reasons,
        COUNT(*) AS occurrence_count
    FROM 
        `shopify-dw.money_products.chargebacks_summary`
    WHERE CAST(shop_id AS STRING) = @input_shop_id
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
WITH shop_gpv AS (
    -- GPV calculation
), 
unfulfilled_120d_exposure AS (
    -- Unfulfilled order exposure calculation
),
chargeback_dominate AS (
    -- Dominant chargeback reason calculation
),
failed_debits AS (
    -- Failed transfers check
), 
shop_industry AS ( 
    -- Shop industry retrieval
), 
-- Additional CTEs for other data points
-- ...

-- Main query combining all CTEs
SELECT 
    -- Combined fields from all CTEs
FROM 
    shop_gpv sg
LEFT JOIN 
    shop_industry si ON sg.shop_id = si.shop_id
-- Additional joins
```

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

*Note: This document provides factual information extracted directly from the source materials. It requires additional input from the Credit Risk team to complete. Once finalized, it will provide comprehensive guidance on using CTEs for credit risk analysis.*

*Last Updated: May 2024* 