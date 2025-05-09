# BigQuery Guide for Credit Risk Analysis

This guide provides practical instructions for using Google BigQuery for credit risk analysis at Shopify, focusing on best practices, optimization, and common use cases.

## Accessing BigQuery

### Web Console Access

1. Go to [console.cloud.google.com/bigquery](https://console.cloud.google.com/bigquery)
2. Ensure you're signed in with your Shopify account
3. Select the `shopify-dw` project from the project selector


# Example query
query = """
-- High Chargeback Rate Merchants in Last 30 Days
SELECT shop_id, COUNT(*) as chargeback_count
FROM `shopify-dw.money_products.chargebacks_summary`
WHERE provider_chargeback_created_at >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 30 DAY)
GROUP BY shop_id
ORDER BY chargeback_count DESC
LIMIT 100
"""


## Basic Query Structure

### Standard Query Format

```sql
-- Always include a comment describing the query purpose
-- Query: High Chargeback Rate Merchants in Last 30 Days

-- Use CTEs for improved readability
WITH merchant_transactions AS (
  SELECT 
    shop_id,
    COUNT(*) AS transaction_count
  FROM `shopify-dw.money_products.order_transactions_payments_summary`
  WHERE order_transaction_created_at >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 30 DAY)
    AND _extracted_at >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 365 DAY)
    AND order_transaction_status = 'success'
  GROUP BY shop_id
),

merchant_chargebacks AS (
  SELECT 
    shop_id,
    COUNT(*) AS chargeback_count
  FROM `shopify-dw.money_products.chargebacks_summary`
  WHERE provider_chargeback_created_at >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 30 DAY)
  GROUP BY shop_id
)

-- Final query joining CTEs
SELECT 
  c.shop_id,
  t.transaction_count,
  c.chargeback_count,
  SAFE_DIVIDE(c.chargeback_count, t.transaction_count) AS chargeback_rate
FROM merchant_chargebacks c
JOIN merchant_transactions t ON c.shop_id = t.shop_id
WHERE t.transaction_count >= 100  -- Only consider merchants with sufficient volume
ORDER BY chargeback_rate DESC
LIMIT 100
```

### Query Organization Best Practices

1. Start with a comment describing the query's purpose and validation date
2. Use Common Table Expressions (CTEs) for modular structure
3. Follow consistent indentation
4. Use meaningful alias names
5. Include appropriate filters and limits
6. Include partition column filters (e.g., _extracted_at)
7. Use correct date/time functions based on column types:
   - TIMESTAMP_SUB() with DAY intervals for timestamp columns
   - DATE_SUB() for date columns or when using DATE() conversion
   - Avoid using MONTH/YEAR intervals with TIMESTAMP_SUB()

## Essential BigQuery Functions for Credit Risk

### Date and Time Functions

```sql
-- Last 30 days of data
-- For TIMESTAMP columns (most common in Shopify DW)
WHERE order_transaction_created_at >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 30 DAY)

-- For DATE columns (less common)
WHERE date_field >= DATE_SUB(CURRENT_DATE(), INTERVAL 30 DAY)

-- Converting TIMESTAMP to DATE for comparison
WHERE DATE(timestamp_column) >= DATE_SUB(CURRENT_DATE(), INTERVAL 30 DAY)

-- Month-to-date metrics for TIMESTAMP columns 
-- NOTE: TIMESTAMP_SUB does not support MONTH date part with TIMESTAMP type
-- Use one of these approaches instead:

-- Option 1: DATE_TRUNC with current month
WHERE DATE_TRUNC(order_transaction_created_at, MONTH) = DATE_TRUNC(CURRENT_TIMESTAMP(), MONTH)

-- Option 2: Using DAY intervals as a workaround for months
WHERE order_transaction_created_at >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 30*3 DAY) -- Approximately 3 months

-- Year-over-year comparison for TIMESTAMP columns
-- NOTE: Be careful with TIMESTAMP_SUB and MONTH intervals
WHERE (
  (order_transaction_created_at >= TIMESTAMP_TRUNC(TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 365 DAY), MONTH) 
   AND order_transaction_created_at < TIMESTAMP_ADD(TIMESTAMP_TRUNC(TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 365 DAY), MONTH), INTERVAL 30 DAY))
  OR
  (order_transaction_created_at >= TIMESTAMP_TRUNC(CURRENT_TIMESTAMP(), MONTH) 
   AND order_transaction_created_at < CURRENT_TIMESTAMP())
)
```

### Aggregation Functions

```sql
-- Calculating risk metrics
SELECT
  shop_id,
  COUNT(*) AS transaction_count,
  SUM(amount_local) AS total_amount,
  AVG(amount_local) AS average_amount,
  STDDEV(amount_local) AS amount_stddev,
  MIN(amount_local) AS min_amount,
  MAX(amount_local) AS max_amount,
  APPROX_QUANTILES(amount_local, 100)[OFFSET(50)] AS median_amount,
  COUNTIF(is_test) / COUNT(*) AS risky_transaction_rate
FROM `shopify-dw.money_products.order_transactions_payments_summary`
WHERE _extracted_at >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 365 DAY)
GROUP BY shop_id
```

### Window Functions

```sql
-- Identifying trend changes
SELECT
  shop_id,
  date,
  chargeback_count,
  AVG(chargeback_count) OVER(
    PARTITION BY shop_id 
    ORDER BY date 
    ROWS BETWEEN 6 PRECEDING AND CURRENT ROW
  ) AS seven_day_avg,
  chargeback_count - LAG(chargeback_count) OVER(
    PARTITION BY shop_id 
    ORDER BY date
  ) AS daily_change
FROM daily_chargeback_data
```

### Safe Operations

Always use safe operations to prevent errors:

```sql
-- Division that handles null/zero denominators
SAFE_DIVIDE(numerator, denominator) AS ratio

-- Array access that handles out-of-bounds indices
SAFE_OFFSET(array, index)

-- Casting that doesn't fail on invalid values
SAFE_CAST(value AS TYPE)
```

## Query Optimization

### Filter Optimization

1. **Use partitioned columns in filters**
   - Most tables are partitioned by date or _extracted_at, so always include these filters
   ```sql
   -- For tables partitioned by _extracted_at
   WHERE _extracted_at >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 365 DAY)
   AND order_transaction_created_at >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 30 DAY)
   ```

2. **Apply filters early**
   - Filter in CTEs rather than the final query
   - Push filters into subqueries

3. **Use appropriate operators**
   - Prefer `=` over `IN` when possible
   - Use `BETWEEN` for date ranges

### Join Optimization

1. **Filter before joining**
   ```sql
   WITH filtered_shops AS (
     SELECT shop_id 
     FROM `shopify-dw.money_products.chargebacks_summary`
     WHERE provider_chargeback_created_at >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 30 DAY)
     GROUP BY shop_id
     HAVING COUNT(*) > 5
   )
   
   SELECT * 
   FROM filtered_shops f
   JOIN large_transaction_table t ON f.shop_id = t.shop_id
   ```

2. **Join on indexed columns**
   - Typically primary keys or fields ending with `_id`

3. **Use appropriate join types**
   - Use `INNER JOIN` when both sides are required
   - Use `LEFT JOIN` when preserving the left table is needed
   - Avoid `CROSS JOIN` for large tables

### Aggregation Optimization

1. **Pre-aggregate in CTEs**
   ```sql
   WITH daily_agg AS (
     SELECT 
       shop_id,
       DATE(order_transaction_created_at) AS date,
       COUNT(*) AS transaction_count
     FROM `shopify-dw.money_products.order_transactions_payments_summary`
     WHERE order_transaction_created_at >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 90 DAY)
       AND _extracted_at >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 365 DAY)
     GROUP BY shop_id, DATE(order_transaction_created_at)
   )
   
   SELECT 
     shop_id,
     AVG(transaction_count) AS avg_daily_transactions
   FROM daily_agg
   GROUP BY shop_id
   ```

2. **Use approximate functions for large datasets**
   - `APPROX_COUNT_DISTINCT()` instead of `COUNT(DISTINCT)`
   - `APPROX_QUANTILES()` for percentiles

## Common Credit Risk Query Patterns

### Chargeback Rate Calculation

```sql
-- Chargeback rate calculation
WITH transactions AS (
  SELECT 
    shop_id,
    COUNT(*) AS transaction_count
  FROM `shopify-dw.money_products.order_transactions_payments_summary`
  WHERE order_transaction_created_at >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 90 DAY)
    AND order_transaction_status = 'success'
    AND _extracted_at >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 365 DAY)
  GROUP BY shop_id
),

chargebacks AS (
  SELECT 
    shop_id,
    COUNT(*) AS chargeback_count
  FROM `shopify-dw.money_products.chargebacks_summary`
  WHERE provider_chargeback_created_at >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 90 DAY)
  GROUP BY shop_id
)

SELECT 
  t.shop_id,
  t.transaction_count,
  COALESCE(c.chargeback_count, 0) AS chargeback_count,
  SAFE_DIVIDE(COALESCE(c.chargeback_count, 0), t.transaction_count) AS chargeback_rate
FROM transactions t
LEFT JOIN chargebacks c ON t.shop_id = c.shop_id
WHERE t.transaction_count >= 100  -- Only include merchants with sufficient volume
```

### Time Series Analysis

```sql
-- Monthly trend analysis
SELECT 
  DATE_TRUNC(order_transaction_created_at, MONTH) AS month,
  COUNT(*) AS transaction_count,
  SUM(amount_local) AS total_amount,
  COUNT(DISTINCT shop_id) AS active_merchants
FROM `shopify-dw.money_products.order_transactions_payments_summary`
WHERE order_transaction_created_at >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 12*30 DAY)  -- Use DAY intervals with multiplier instead of MONTH with TIMESTAMP_SUB
  AND _extracted_at >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 365 DAY)
GROUP BY month
ORDER BY month
```

### Risk Segment Analysis

```sql
-- Segmenting merchants by risk level
WITH merchant_metrics AS (
  SELECT 
    shop_id,
    COUNT(*) AS transaction_count,
    COUNTIF(order_transaction_status = 'declined') / COUNT(*) AS decline_rate,
    AVG(amount_local) AS avg_amount,
    STDDEV(amount_local) AS amount_stddev
  FROM `shopify-dw.money_products.order_transactions_payments_summary`
  WHERE order_transaction_created_at >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 30 DAY)
    AND _extracted_at >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 365 DAY)
  GROUP BY shop_id
  HAVING transaction_count >= 50
)

SELECT 
  shop_id,
  CASE
    WHEN decline_rate > 0.2 THEN 'High Risk'
    WHEN decline_rate > 0.1 THEN 'Medium Risk'
    ELSE 'Low Risk'
  END AS risk_segment,
  transaction_count,
  decline_rate,
  avg_amount,
  amount_stddev
FROM merchant_metrics
ORDER BY decline_rate DESC
```

## Saving and Sharing Queries

### Saving Queries

1. In the BigQuery web interface, click "Save query"
2. Choose a descriptive name and add relevant labels
3. For team queries, save to a shared folder
4. Include a description of the query's purpose

### Creating Views

For frequently used queries, create views:

```sql
CREATE OR REPLACE VIEW `project.dataset.view_name` AS
SELECT 
  shop_id,
  chargeback_rate,
  risk_score
FROM your_query_logic
```


## Troubleshooting Common Errors

### "Resources exceeded" Error

**Cause**: Query processing requires more resources than allocated
**Solutions**:
- Filter data more aggressively
- Use more efficient joins
- Pre-aggregate data
- Split into multiple queries


### "Syntax error" Issues

**Cause**: SQL syntax problems
**Solutions**:
- Check for missing commas, parentheses, or quotation marks
- Verify table and column names
- Use the "Format query" button to identify syntax issues

### "No matching signature for operator" Error

**Cause**: Type mismatches between DATE and TIMESTAMP columns
**Solutions**:
- For TIMESTAMP columns (most common):
  ```sql
  WHERE created_at >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 30 DAY)
  ```
- For DATE columns:
  ```sql
  WHERE date_field >= DATE_SUB(CURRENT_DATE(), INTERVAL 30 DAY)
  ```
- For conversion when needed:
  ```sql
  WHERE DATE(timestamp_column) >= DATE_SUB(CURRENT_DATE(), INTERVAL 30 DAY)
  ```

### "TIMESTAMP_SUB does not support the MONTH date part" Error

**Cause**: Attempting to use MONTH, YEAR, or other non-supported intervals with TIMESTAMP_SUB on TIMESTAMP columns
**Solutions**:
1. Use DAY intervals with multipliers instead:
   ```sql
   -- INCORRECT: Will generate error
   WHERE created_at >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 3 MONTH)
   
   -- CORRECT: Use days instead (30*3 = ~3 months)
   WHERE created_at >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 30*3 DAY)
   ```

2. Use TIMESTAMP_TRUNC with appropriate INTERVAL:
   ```sql
   -- For beginning of month 3 months ago
   WHERE created_at >= TIMESTAMP_TRUNC(TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 90 DAY), MONTH)
   ```

3. Convert to DATE type if appropriate:
   ```sql
   -- If you only need date-level granularity
   WHERE DATE(created_at) >= DATE_SUB(CURRENT_DATE(), INTERVAL 3 MONTH)
   ```

### "Partitioned table requires filter" Error

**Cause**: Missing filter on partitioned column (_extracted_at)
**Solution**: Add a filter on the partition column
```sql
WHERE _extracted_at >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 365 DAY)
```

## Credit Risk Team Query Guidelines

1. **Always use `shopify-dw` billing project**
2. **Include partition column filters on all queries** (usually _extracted_at)
3. **Use appropriate date/time functions for column types**
   - TIMESTAMP_SUB() for timestamp columns with supported intervals (DAY, HOUR, MINUTE, SECOND)
   - DATE_SUB() for date columns (supports all intervals including MONTH, YEAR)
   - Never use MONTH, YEAR intervals with TIMESTAMP_SUB()
   - Use DATE() conversion if you need to use MONTH, YEAR intervals with timestamp columns
4. **Document all queries with comments and last validation date**
5. **Verify results against standard tables**
6. **Follow PII data handling guidelines**
7. **Use consistent naming conventions**
8. **Keep queries modular and readable**
9. **Test on limited data before running on full datasets**

### Temporal Fields Reference

| Field Name | Data Type | Comparison Function | Example | Notes |
|------------|-----------|---------------------|---------|-------|
| order_transaction_created_at | TIMESTAMP | TIMESTAMP_SUB() | `WHERE order_transaction_created_at >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 7 DAY)` | Supports DAY, HOUR, MINUTE, SECOND intervals only. For MONTH intervals, use alternative approaches. |
| provider_chargeback_created_at | TIMESTAMP | TIMESTAMP_SUB() | `WHERE provider_chargeback_created_at >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 7 DAY)` | Supports DAY, HOUR, MINUTE, SECOND intervals only. For MONTH intervals, use alternative approaches. |
| _extracted_at | TIMESTAMP | TIMESTAMP_SUB() | `WHERE _extracted_at >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 365 DAY)` | Partition column - always include in queries. |
| Date fields (when using DATE()) | DATE | DATE_SUB() | `WHERE DATE(timestamp_field) >= DATE_SUB(CURRENT_DATE(), INTERVAL 7 DAY)` | Supports all intervals including MONTH, YEAR. |
| Date-only fields | DATE | DATE_SUB() | `WHERE date_field >= DATE_SUB(CURRENT_DATE(), INTERVAL 1 MONTH)` | Native DATE fields support all interval types. |

### Alternative Approaches for Month/Year-based Filtering on TIMESTAMP Fields

| Need | Approach | Example |
|------|----------|---------|
| Filter for last N months | Use DAY intervals | `WHERE field >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 30*N DAY)` |
| Filter for specific month | Use TIMESTAMP_TRUNC | `WHERE TIMESTAMP_TRUNC(field, MONTH) = TIMESTAMP_TRUNC(TIMESTAMP '2023-06-01', MONTH)` |
| Month-to-date comparison | Use DATE_TRUNC | `WHERE DATE_TRUNC(field, MONTH) = DATE_TRUNC(CURRENT_TIMESTAMP(), MONTH)` |
| Year-over-year | Convert to DATE if appropriate | `WHERE DATE(field) BETWEEN DATE_SUB(CURRENT_DATE(), INTERVAL 1 YEAR) AND DATE_SUB(CURRENT_DATE(), INTERVAL 364 DAY)` |

## Additional Resources

For more information on using SQL effectively for credit risk analysis:

- [SQL Basics Guide](../02_SQL_Guide/SQL_Basics.md)
- [Common SQL Patterns](../02_SQL_Guide/Common_SQL_Patterns.md)
- [Table Relationships](../03_Data_Dictionary/Table_Relationships.md)

## Further Resources

- [BigQuery Documentation](https://cloud.google.com/bigquery/docs)
- [SQL Basics Guide](../02_SQL_Guide/SQL_Basics.md)
- [Common SQL Patterns](../02_SQL_Guide/Common_SQL_Patterns.md)
- [Table Relationships](../03_Data_Dictionary/Table_Relationships.md)


---