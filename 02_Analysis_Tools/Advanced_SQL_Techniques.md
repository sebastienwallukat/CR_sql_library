# Common Table Expressions (CTEs) in SQL

## Introduction

Common Table Expressions (CTEs) are temporary named result sets that exist only within the execution scope of a specific SQL statement. CTEs make complex queries more readable by breaking them into logical, named components that can be referenced multiple times. This guide explains how to effectively use CTEs for credit risk analysis at Shopify.

## Table of Contents

- [Introduction](#introduction)
- [Basic CTE Syntax](#basic-cte-syntax)
- [Chaining Multiple CTEs](#chaining-multiple-ctEs)
- [CTEs with Window Functions](#ctes-with-window-functions)
- [Common CTE Patterns](#common-cte-patterns)
- [Performance Considerations](#performance-considerations)
- [Troubleshooting](#troubleshooting)

## Basic CTE Syntax

A CTE is defined using the `WITH` clause followed by the name of the CTE and its query definition.

```sql
-- Basic CTE syntax example
WITH shop_gpv AS (
    SELECT
        shop_id,
        ROUND(SUM(daily_sp_paid_amount_usd), 4) AS total_gpv 
    FROM
        `sdp-prd-cti-data.intermediate.shop_daily_order_fraud_facts` AS f
    WHERE 
        date >= DATE_SUB(CURRENT_DATE(), INTERVAL 120 DAY)
        AND shop_id = @input_shop_id
    GROUP BY
        shop_id 
)

-- Using the CTE in the main query
SELECT 
    shop_id,
    total_gpv,
    CASE
        WHEN total_gpv > 100000 THEN 'High Volume'
        WHEN total_gpv > 10000 THEN 'Medium Volume'
        ELSE 'Low Volume'
    END AS volume_category
FROM 
    shop_gpv
```

## Description

The example above demonstrates:

1. **CTE Definition**: The `WITH` clause creates a named result set (`shop_gpv`) that calculates the total Gross Processing Volume (GPV) for a specific shop over the last 120 days.
2. **Parameter Usage**: The query uses the parameter `@input_shop_id` to filter for a specific shop.
3. **Date Filtering**: Uses `DATE_SUB()` to create a relative date range of the last 120 days.
4. **Result Formatting**: `ROUND()` is used to format the monetary values to 4 decimal places.
5. **Main Query**: After the CTE is defined, the main query references it like a table.

## Chaining Multiple CTEs

For complex analyses, multiple CTEs can be chained together, each building on previous data transformations.

```sql
-- Example of multiple chained CTEs for comprehensive risk analysis
WITH 
-- Calculate total GPV for a shop over the last 120 days
shop_gpv AS (
    SELECT
        shop_id,
        ROUND(SUM(f.daily_sp_paid_amount_usd), 4) AS total_gpv 
    FROM
        `sdp-prd-cti-data.intermediate.shop_daily_order_fraud_facts` AS f
    WHERE 
        date >= DATE_SUB(CURRENT_DATE(), INTERVAL 120 DAY)
        AND shop_id = @input_shop_id
    GROUP BY 
        shop_id
), 
-- Calculate unfulfilled order exposure
unfulfilled_120d_exposure AS (
    SELECT
        shop_id,
        SUM(daily_unfulfilled_paid_sp_order_amount_usd) AS unfulfilled_exposure
    FROM
        `sdp-prd-cti-data.intermediate.shop_daily_order_fraud_facts`
    WHERE 
        date >= DATE_SUB(CURRENT_DATE(), INTERVAL 120 DAY)
        AND shop_id = @input_shop_id
    GROUP BY 
        shop_id
),
-- Identify the most common chargeback reason
chargeback_dominate AS (
    SELECT 
        shop_id,
        reason AS reasons,
        COUNT(*) AS occurrence_count
    FROM 
        `shopify-dw.money_products.chargebacks_summary`
    WHERE 
        shop_id = @input_shop_id
        AND provider_chargeback_created_at >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 120 DAY)
    GROUP BY 
        shop_id, reasons
    QUALIFY 
        RANK() OVER (
            PARTITION BY shop_id 
            ORDER BY 
                occurrence_count DESC,
                -- If there's a tie, use this ranking for consistent results
                CASE 
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

-- Main query combining all CTEs
SELECT 
    sg.shop_id,
    sg.total_gpv,
    ue.unfulfilled_exposure,
    cd.reasons AS dominant_chargeback_reason,
    -- Calculate exposure as a percentage of GPV
    ROUND((ue.unfulfilled_exposure / NULLIF(sg.total_gpv, 0)) * 100, 2) AS exposure_percentage
FROM 
    shop_gpv sg
LEFT JOIN 
    unfulfilled_120d_exposure ue ON sg.shop_id = ue.shop_id
LEFT JOIN
    chargeback_dominate cd ON sg.shop_id = cd.shop_id
```

## Description

This complex query demonstrates:

1. **Multiple CTEs**: Three separate CTEs each perform a specific calculation:
   - `shop_gpv`: Calculates total GPV for a shop
   - `unfulfilled_120d_exposure`: Calculates unfulfilled order exposure
   - `chargeback_dominate`: Identifies the most common chargeback reason

2. **Window Functions**: The `QUALIFY` clause with `RANK()` window function selects only the top-ranked chargeback reason.

3. **Tie-Breaking Logic**: If multiple chargeback reasons have the same count, the `CASE` statement provides a consistent tie-breaking mechanism.

4. **CTE Joining**: The main query joins all CTEs together to create a comprehensive view.

5. **Calculated Fields**: The main query calculates additional metrics based on the CTE results.

## CTEs with Window Functions

Window functions are powerful when combined with CTEs, especially for ranking and analytical calculations.

```sql
-- Using CTEs with window functions to analyze shop review trends
WITH shop_reviews AS ( 
    SELECT 
        shop_id,
        rating,
        created_at,
        -- Calculate running average of ratings
        AVG(rating) OVER (
            PARTITION BY shop_id 
            ORDER BY created_at
            ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
        ) AS running_avg_rating,
        -- Calculate month-over-month change
        LAG(rating, 1) OVER (
            PARTITION BY shop_id 
            ORDER BY created_at
        ) AS previous_rating
    FROM 
        `shopify-dw.intermediate.shop_product_reviews_history_v1_0`
    WHERE 
        shop_id = @input_shop_id
        AND valid_to IS NULL  -- Required filter on partition column
        AND created_at >= DATE_SUB(CURRENT_DATE(), INTERVAL 180 DAY)
)

SELECT
    shop_id,
    created_at,
    rating,
    running_avg_rating,
    -- Calculate change from previous rating
    rating - previous_rating AS rating_change,
    -- Categorize the trend
    CASE 
        WHEN rating > previous_rating THEN 'Improving'
        WHEN rating = previous_rating THEN 'Stable'
        WHEN rating < previous_rating THEN 'Declining'
        ELSE 'First Review'
    END AS trend
FROM
    shop_reviews
ORDER BY 
    created_at DESC
```

## Description

This example demonstrates:

1. **Window Functions in CTEs**: Using `AVG() OVER()` and `LAG()` functions to calculate running averages and previous values.

2. **Partitioning**: The `PARTITION BY` clause segments calculations by shop_id.

3. **Window Frames**: Specifying which rows to include in calculations with `ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW`.

4. **Trend Analysis**: Using the window function results to calculate changes and categorize trends.

## Common CTE Patterns

### Handling Null Values

```sql
-- Pattern for handling null values in CTEs
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
    -- Handle NULL values with COALESCE
    COALESCE(CAST(shop_reviews AS STRING), 'No reviews') AS shop_reviews,
    -- Alternative null handling using IFNULL
    IFNULL(shop_reviews, 0) AS shop_reviews_numeric
FROM
    shop_app
```

### Recursive CTEs

BigQuery supports recursive CTEs for hierarchical or iterative calculations.

```sql
-- Example of a recursive CTE to find order dependencies
WITH RECURSIVE order_hierarchy AS (
    -- Base case: starting with parent orders
    SELECT 
        order_id,
        parent_order_id,
        1 AS level
    FROM 
        `shopify-dw.transaction_data.orders`
    WHERE 
        parent_order_id IS NULL
        AND shop_id = @input_shop_id
    
    UNION ALL
    
    -- Recursive case: add child orders
    SELECT 
        o.order_id,
        o.parent_order_id,
        oh.level + 1 AS level
    FROM 
        `shopify-dw.transaction_data.orders` o
    INNER JOIN 
        order_hierarchy oh ON o.parent_order_id = oh.order_id
    WHERE 
        o.shop_id = @input_shop_id
)

SELECT 
    order_id,
    parent_order_id,
    level
FROM 
    order_hierarchy
ORDER BY 
    level, order_id
```

## Performance Considerations

When using CTEs, consider these performance tips:

1. **Materialization**: CTEs are not materialized by default. Each reference to a CTE causes it to be re-executed.

2. **Filtering Early**: Apply filters in the CTE definition rather than in the main query to reduce processed data.

3. **Partition Pruning**: Always include partition field filters in your CTEs to leverage BigQuery's partition pruning.

4. **Avoid Overuse**: Breaking queries into too many CTEs can hinder optimization. Use CTEs for logical separation, not for every small operation.

5. **Temporal Field Types**: Use appropriate comparison functions based on field types:

| Field Name | Data Type | Comparison Function | Example |
|------------|-----------|---------------------|---------|
| date | DATE | DATE_SUB() | `WHERE date >= DATE_SUB(CURRENT_DATE(), INTERVAL 7 DAY)` |
| provider_chargeback_created_at | TIMESTAMP | TIMESTAMP_SUB() | `WHERE provider_chargeback_created_at >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 30 DAY)` |
| valid_to | TIMESTAMP | IS NULL / TIMESTAMP_SUB() | `WHERE valid_to IS NULL` |

## Troubleshooting

### Common CTE Errors

1. **Name Conflicts**: Ensure unique CTE names within a query.

   ```sql
   -- Incorrect: Name conflict
   WITH shop_data AS (...),
        shop_data AS (...)  -- Error: duplicate name
   ```

2. **Invalid References**: CTEs can only reference preceding CTEs.

   ```sql
   -- Incorrect: Forward reference
   WITH cte1 AS (SELECT * FROM cte2),  -- Error: cte2 not yet defined
        cte2 AS (SELECT * FROM table)
   ```

3. **Missing Column References**: Verify column names in CTE definitions.

   ```sql
   -- Correct way to reference columns
   WITH cte AS (
       SELECT id, name FROM table
   )
   SELECT id, name FROM cte  -- Use exact column names
   ```

### Debugging Complex CTEs

To debug complex CTE chains:

1. Test each CTE individually before combining them.
2. Use `LIMIT` clauses in each CTE during development.
3. Add debugging columns to inspect intermediate values.

Example debugging approach:

```sql
WITH cte1 AS (
    SELECT 
        *,
        'From CTE1' AS debug_source  -- Add debugging information
    FROM table
    LIMIT 100  -- Limit rows during development
)

SELECT * FROM cte1
-- During development, run this query instead of the full chain
```

## Example

A complete example analyzing shop risk factors:

```sql
-- Purpose: Comprehensive shop risk analysis using CTEs
WITH 
-- Base shop information
shop_base AS (
    SELECT 
        shop_id,
        shop_created_at,
        DATE_DIFF(CURRENT_DATE(), DATE(shop_created_at), DAY) AS shop_age_days,
        shopify_payments_status
    FROM 
        `shopify-dw.risk.shop_current_shopify_payments_status`
    WHERE 
        shop_id = @input_shop_id
),
-- Chargeback metrics
chargeback_metrics AS (
    SELECT 
        shop_id,
        COUNT(*) AS total_chargebacks,
        SUM(chargeback_amount_usd) AS total_chargeback_amount_usd
    FROM 
        `shopify-dw.money_products.chargebacks_summary`
    WHERE 
        shop_id = @input_shop_id
        AND provider_chargeback_created_at >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 180 DAY)
    GROUP BY 
        shop_id
),
-- Order velocity
order_velocity AS (
    SELECT 
        shop_id,
        COUNT(*) AS total_orders,
        COUNT(*) / 180 AS daily_order_rate
    FROM 
        `sdp-prd-cti-data.intermediate.shop_daily_order_fraud_facts`
    WHERE 
        shop_id = @input_shop_id
        AND date >= DATE_SUB(CURRENT_DATE(), INTERVAL 180 DAY)
    GROUP BY 
        shop_id
)

-- Main query combining all risk factors
SELECT 
    sb.shop_id,
    sb.shopify_payments_status,
    sb.shop_age_days,
    
    -- Order metrics
    ov.total_orders,
    ov.daily_order_rate,
    
    -- Chargeback metrics
    cm.total_chargebacks,
    cm.total_chargeback_amount_usd,
    
    -- Calculated risk metrics
    SAFE_DIVIDE(cm.total_chargebacks, ov.total_orders) * 100 AS chargeback_rate_pct,
    
    -- Risk categorization
    CASE 
        WHEN sb.shop_age_days < 30 AND SAFE_DIVIDE(cm.total_chargebacks, ov.total_orders) > 0.01 THEN 'High Risk'
        WHEN sb.shop_age_days < 90 AND SAFE_DIVIDE(cm.total_chargebacks, ov.total_orders) > 0.005 THEN 'Medium Risk'
        WHEN sb.shop_age_days < 180 THEN 'New Shop - Monitor'
        ELSE 'Standard Risk'
    END AS risk_category
FROM 
    shop_base sb
LEFT JOIN 
    chargeback_metrics cm ON sb.shop_id = cm.shop_id
LEFT JOIN 
    order_velocity ov ON sb.shop_id = ov.shop_id
```

The above query would generate a report like:

| shop_id | shopify_payments_status | shop_age_days | total_orders | daily_order_rate | total_chargebacks | total_chargeback_amount_usd | chargeback_rate_pct | risk_category |
|---------|------------------------|---------------|--------------|------------------|-------------------|----------------------------|---------------------|---------------|
| 12345   | active                 | 45            | 250          | 1.38             | 3                 | 149.95                     | 1.2                 | Medium Risk   |
| 67890   | pending                | 12            | 32           | 0.17             | 1                 | 74.99                      | 3.13                | High Risk     |
| 24680   | active                 | 365           | 4380         | 24.33            | 12                | 599.88                     | 0.27                | Standard Risk |