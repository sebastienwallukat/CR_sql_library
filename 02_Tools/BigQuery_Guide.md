# BigQuery Guide for Credit Risk Analysis

This guide provides practical instructions for using Google BigQuery for credit risk analysis at Shopify, focusing on best practices, optimization, and common use cases.

## Accessing BigQuery

### Web Console Access

1. Go to [console.cloud.google.com/bigquery](https://console.cloud.google.com/bigquery)
2. Ensure you're signed in with your Shopify account
3. Select the `shopify-dw` project from the project selector
4. Confirm the billing project is set to `shopify-dw` in the query editor

### API Access

For programmatic access (Python):
```python
from google.cloud import bigquery

# Always specify shopify-dw as the billing project
client = bigquery.Client(project='shopify-dw')

# Example query
query = """
SELECT shop_id, COUNT(*) as chargeback_count
FROM `shopify-dw.money_products.chargebacks_summary`
WHERE created_at >= DATE_SUB(CURRENT_DATE(), INTERVAL 30 DAY)
GROUP BY shop_id
ORDER BY chargeback_count DESC
LIMIT 100
"""

# Run the query
query_job = client.query(query)
results = query_job.result()

# Process results
for row in results:
    print(f"Shop ID: {row.shop_id}, Chargebacks: {row.chargeback_count}")
```

## Basic Query Structure

### Standard Query Format

```sql
-- Always include a comment describing the query purpose
-- Query: High Chargeback Rate Merchants in Last 30 Days
-- Last validated: May 2024

-- Use CTEs for improved readability
WITH merchant_transactions AS (
  SELECT 
    shop_id,
    COUNT(*) AS transaction_count
  FROM `shopify-dw.money_products.order_transactions_payments_summary`
  WHERE created_at >= DATE_SUB(CURRENT_DATE(), INTERVAL 30 DAY)
  GROUP BY shop_id
),

merchant_chargebacks AS (
  SELECT 
    shop_id,
    COUNT(*) AS chargeback_count
  FROM `shopify-dw.money_products.chargebacks_summary`
  WHERE created_at >= DATE_SUB(CURRENT_DATE(), INTERVAL 30 DAY)
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

1. Start with a comment describing the query's purpose
2. Include a "Last validated" date
3. Use Common Table Expressions (CTEs) for modular structure
4. Follow consistent indentation
5. Use meaningful alias names
6. Include appropriate filters and limits

## Essential BigQuery Functions for Credit Risk

### Date and Time Functions

```sql
-- Last 30 days of data
WHERE created_at >= DATE_SUB(CURRENT_DATE(), INTERVAL 30 DAY)

-- Month-to-date metrics
WHERE DATE_TRUNC(created_at, MONTH) = DATE_TRUNC(CURRENT_DATE(), MONTH)

-- Year-over-year comparison
WHERE (
  (created_at >= DATE_TRUNC(DATE_SUB(CURRENT_DATE(), INTERVAL 1 YEAR), MONTH) 
   AND created_at < DATE_ADD(DATE_TRUNC(DATE_SUB(CURRENT_DATE(), INTERVAL 1 YEAR), MONTH), INTERVAL 1 MONTH))
  OR
  (created_at >= DATE_TRUNC(CURRENT_DATE(), MONTH) 
   AND created_at < CURRENT_DATE())
)
```

### Aggregation Functions

```sql
-- Calculating risk metrics
SELECT
  shop_id,
  COUNT(*) AS transaction_count,
  SUM(amount) AS total_amount,
  AVG(amount) AS average_amount,
  STDDEV(amount) AS amount_stddev,
  MIN(amount) AS min_amount,
  MAX(amount) AS max_amount,
  APPROX_QUANTILES(amount, 100)[OFFSET(50)] AS median_amount,
  COUNTIF(is_risky) / COUNT(*) AS risky_transaction_rate
FROM `shopify-dw.money_products.order_transactions_payments_summary`
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
   - Most tables are partitioned by date, so always include date filters
   ```sql
   WHERE created_at >= DATE_SUB(CURRENT_DATE(), INTERVAL 30 DAY)
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
     WHERE created_at >= DATE_SUB(CURRENT_DATE(), INTERVAL 30 DAY)
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
       DATE(created_at) AS date,
       COUNT(*) AS transaction_count
     FROM `shopify-dw.money_products.order_transactions_payments_summary`
     WHERE created_at >= DATE_SUB(CURRENT_DATE(), INTERVAL 90 DAY)
     GROUP BY shop_id, DATE(created_at)
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
  WHERE created_at >= DATE_SUB(CURRENT_DATE(), INTERVAL 90 DAY)
    AND status = 'success'
  GROUP BY shop_id
),

chargebacks AS (
  SELECT 
    shop_id,
    COUNT(*) AS chargeback_count
  FROM `shopify-dw.money_products.chargebacks_summary`
  WHERE created_at >= DATE_SUB(CURRENT_DATE(), INTERVAL 90 DAY)
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
  DATE_TRUNC(created_at, MONTH) AS month,
  COUNT(*) AS transaction_count,
  SUM(amount) AS total_amount,
  COUNT(DISTINCT shop_id) AS active_merchants
FROM `shopify-dw.money_products.order_transactions_payments_summary`
WHERE created_at >= DATE_SUB(CURRENT_DATE(), INTERVAL 12 MONTH)
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
    COUNTIF(status = 'declined') / COUNT(*) AS decline_rate,
    AVG(amount) AS avg_amount,
    STDDEV(amount) AS amount_stddev
  FROM `shopify-dw.money_products.order_transactions_payments_summary`
  WHERE created_at >= DATE_SUB(CURRENT_DATE(), INTERVAL 30 DAY)
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

### Scheduling Queries

For recurring analyses:
1. Save your query
2. Click "Schedule" 
3. Set the run frequency
4. Configure destination table
5. Set email notifications

## Troubleshooting Common Errors

### "Resources exceeded" Error

**Cause**: Query processing requires more resources than allocated
**Solutions**:
- Filter data more aggressively
- Use more efficient joins
- Pre-aggregate data
- Split into multiple queries

### "Access denied" Error

**Cause**: Insufficient permissions
**Solution**: Follow the [Access and Permissions](../01_Getting_Started/Access_and_Permissions.md) guide

### "Syntax error" Issues

**Cause**: SQL syntax problems
**Solutions**:
- Check for missing commas, parentheses, or quotation marks
- Verify table and column names
- Use the "Format query" button to identify syntax issues

## Credit Risk Team Query Guidelines

1. **Always use `shopify-dw` billing project**
2. **Include date filters on all queries**
3. **Document all queries with comments and last validation date**
4. **Verify results against standard tables**
5. **Follow PII data handling guidelines**
6. **Use consistent naming conventions**
7. **Keep queries modular and readable**
8. **Test on limited data before running on full datasets**

## Further Resources

- [BigQuery Documentation](https://cloud.google.com/bigquery/docs)
- [SQL Basics Guide](../03_SQL_Guide/SQL_Basics.md)
- [Common SQL Patterns](../03_SQL_Guide/Common_SQL_Patterns.md)
- [Table Relationships](../04_Data_Dictionary/Table_Relationships.md)
- Internal Shopify BigQuery documentation on [internal-docs.shopify.io](https://internal-docs.shopify.io)

---

*Last Updated: May 2024* 