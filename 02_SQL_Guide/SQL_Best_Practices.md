# SQL Best Practices for Credit Risk Analysis

This guide outlines best practices for writing SQL queries for credit risk analysis at Shopify, focusing on query optimization, naming conventions, and data handling guidelines.

## Query Structure and Organization

### 1. Use Consistent Query Structure

```sql
-- 1. Start with a comment describing the query purpose
-- Query: High-risk merchants with elevated chargeback rates

-- Author: Credit Risk Team

-- 2. Use CTEs for better readability and modularity
WITH merchant_transactions AS (
  -- 3. Include comments for complex logic
  -- Get transaction counts for the last 90 days
  SELECT 
    shop_id,
    COUNT(*) AS transaction_count
  FROM `shopify-dw.money_products.order_transactions_payments_summary`
  WHERE order_transaction_created_at >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 90 DAY)
    AND order_transaction_status = 'success'
    AND _extracted_at >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 90 DAY) -- Include partition filter
  GROUP BY shop_id
),

merchant_chargebacks AS (
  -- Calculate chargeback counts for the same period
  SELECT 
    shop_id,
    COUNT(*) AS chargeback_count
  FROM `shopify-dw.money_products.chargebacks_summary`
  WHERE provider_chargeback_created_at >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 90 DAY)
  GROUP BY shop_id
)

-- 4. Final query joining CTEs with clear alias names
SELECT 
  m.shop_id,
  s.name AS shop_name,
  m.transaction_count,
  c.chargeback_count,
  SAFE_DIVIDE(c.chargeback_count, m.transaction_count) AS chargeback_rate
FROM merchant_transactions m
-- 5. Join with shop information for context
JOIN `shopify-dw.accounts_and_administration.shop_profile_current` s ON m.shop_id = s.shop_id
LEFT JOIN merchant_chargebacks c ON m.shop_id = c.shop_id
-- 6. Include appropriate filters
WHERE m.transaction_count >= 100  -- Only include merchants with sufficient volume
-- 7. Clear sorting
ORDER BY chargeback_rate DESC
-- 8. Use limits for performance
LIMIT 100;
```

### 2. Use Descriptive Naming

- **CTE names**: Use clear, descriptive names for CTEs (e.g., `merchant_transactions` instead of `t1`)
- **Column aliases**: Use meaningful column names (e.g., `chargeback_rate` instead of `rate`)
- **Table aliases**: Use short but clear table aliases (e.g., `t` for transactions, `c` for chargebacks)

### 3. Format SQL Consistently

- Use uppercase for SQL keywords (SELECT, FROM, WHERE, etc.)
- Use consistent indentation (2 or 4 spaces)
- Align related items vertically for readability
- Place each field on a separate line in complex queries
- Use parentheses to clarify complex conditions

## Query Optimization for BigQuery

### 1. Filter Optimization

- **Always use date filters on partitioned columns**
  ```sql
  -- Good: Filtering on a partitioned column (order_transaction_created_at) with correct timestamp function and partition filter
  WHERE order_transaction_created_at >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 90 DAY)
    AND _extracted_at >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 90 DAY)
  
  -- Bad: Not using date filters or using wrong data type
  WHERE shop_id = 12345  -- This will scan the entire table
  -- or
  WHERE order_transaction_created_at >= DATE_SUB(CURRENT_DATE(), INTERVAL 90 DAY) -- Type mismatch
  ```

- **Push filters down to the earliest stage possible**
  ```sql
  -- Good: Filtering in the CTE with correct data types
  WITH filtered_transactions AS (
    SELECT * FROM `shopify-dw.money_products.order_transactions_payments_summary`
    WHERE order_transaction_created_at >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 90 DAY)
      AND _extracted_at >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 90 DAY)
  )
  
  -- Bad: Filtering in the final query
  WITH all_transactions AS (
    SELECT * FROM `shopify-dw.money_products.order_transactions_payments_summary`  -- Pulls all data, then filters later
  )
  SELECT * FROM all_transactions
  WHERE order_transaction_created_at >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 90 DAY)
  ```

### 2. SELECT Only Needed Columns

- **Avoid `SELECT *` except for development**
  ```sql
  -- Good: Selecting only needed columns
  SELECT shop_id, order_transaction_id, amount_local, order_transaction_created_at
  FROM `shopify-dw.money_products.order_transactions_payments_summary`
  
  -- Bad: Selecting all columns
  SELECT *
  FROM `shopify-dw.money_products.order_transactions_payments_summary`
  ```

### 3. Join Optimization

- **Filter tables before joining**
  ```sql
  -- Good: Filtering before joining
  WITH small_shop_set AS (
    SELECT shop_id
    FROM `shopify-dw.accounts_and_administration.shop_profile_current`
    WHERE country_code = 'US' 
      AND is_test = FALSE
  )
  SELECT t.*
  FROM small_shop_set s
  JOIN `shopify-dw.money_products.order_transactions_payments_summary` t 
    ON s.shop_id = t.shop_id
    AND t._extracted_at >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 90 DAY)
  ```

- **Join on indexed/key columns**
  - Use `shop_id`, `order_id`, `transaction_id`, etc.
  - Composite keys should include all relevant fields

- **Choose the appropriate join type**
  - Use `INNER JOIN` when data must exist in both tables
  - Use `LEFT JOIN` when preserving the left table is needed
  - Avoid `CROSS JOIN` for large tables

### 4. Efficient Aggregations

- **Pre-aggregate in CTEs for complex aggregations**
  ```sql
  -- Good: Pre-aggregating data
  WITH daily_agg AS (
    SELECT 
      shop_id,
      DATE(order_transaction_created_at) AS date,
      COUNT(*) AS tx_count
    FROM `shopify-dw.money_products.order_transactions_payments_summary`
    WHERE order_transaction_created_at >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 90 DAY)
      AND _extracted_at >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 90 DAY)
    GROUP BY shop_id, DATE(order_transaction_created_at)
  )
  SELECT 
    shop_id,
    AVG(tx_count) AS avg_daily_tx
  FROM daily_agg
  GROUP BY shop_id
  ```

- **Use approximate functions for large datasets**
  ```sql
  -- Good: Using approximate functions
  SELECT 
    APPROX_COUNT_DISTINCT(shop_id) AS approximate_shop_count
  FROM `shopify-dw.money_products.order_transactions_payments_summary`
  WHERE order_transaction_created_at >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 90 DAY)
    AND _extracted_at >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 90 DAY)
  
  -- Also good: Using APPROX_QUANTILES for percentiles
  SELECT
    APPROX_QUANTILES(amount_local, 100)[OFFSET(50)] AS median_amount
  FROM `shopify-dw.money_products.order_transactions_payments_summary`
  WHERE order_transaction_created_at >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 90 DAY)
    AND _extracted_at >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 90 DAY)
  ```

### 5. Window Functions

- **Use window functions instead of self-joins when possible**
  ```sql
  -- Good: Using window functions
  SELECT
    shop_id,
    date,
    amount,
    SUM(amount) OVER(PARTITION BY shop_id ORDER BY date 
                     ROWS BETWEEN 6 PRECEDING AND CURRENT ROW) AS rolling_7day_amount
  FROM daily_shop_transactions
  
  -- Avoid: Self-joining for the same result
  ```

### 6. Use LIMIT with ORDER BY

- **Always include LIMIT when using ORDER BY**
  ```sql
  -- Good: Using LIMIT with ORDER BY
  SELECT shop_id, chargeback_rate
  FROM shop_metrics
  ORDER BY chargeback_rate DESC
  LIMIT 100
  ```

## Data Handling Best Practices

### 1. PII Data Handling

- **Minimize PII usage in queries**
  - Only include PII fields when absolutely necessary
  - Prefer shop_id over merchant email/name when possible
  - Aggregate data to remove individual identifiers

- **Apply proper masking for PII in results**
  ```sql
  -- Good: Masking PII in results
  SELECT
    shop_id,
    CONCAT(SUBSTR(domain, 1, 2), '****', SUBSTR(domain, -4)) AS masked_domain
  FROM `shopify-dw.accounts_and_administration.shop_profile_current`
  ```

- **Document PII usage in query comments**
  ```sql
  -- Query includes PII: domain
  -- Purpose: Merchant outreach for high-risk merchants
  -- Approved by: [Manager Name], [Date]
  ```

### 2. Currency Handling

- **Always join financial tables on currency**
  ```sql
  -- Good: Joining on currency
  SELECT
    t.shop_id,
    t.amount_local,
    b.balance
  FROM `shopify-dw.money_products.order_transactions_payments_summary` t
  JOIN balances b ON t.shop_id = b.shop_id AND t.currency_code_local = b.currency
  WHERE t._extracted_at >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 90 DAY)
  ```

- **Use currency conversion functions for multi-currency analysis**
  ```sql
  -- Converting currencies to a common base (USD)
  SELECT
    shop_id,
    SUM(CASE 
      WHEN currency_code_local = 'USD' THEN amount_local
      WHEN currency_code_local = 'CAD' THEN amount_local * 0.75 -- Example conversion rate
      WHEN currency_code_local = 'EUR' THEN amount_local * 1.1  -- Example conversion rate
      ELSE 0
    END) AS total_usd_equivalent
  FROM `shopify-dw.money_products.order_transactions_payments_summary`
  WHERE _extracted_at >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 90 DAY)
  GROUP BY shop_id
  ```

### 3. Error Handling

- **Use SAFE functions to prevent errors**
  ```sql
  -- Good: Using SAFE_DIVIDE for division
  SELECT
    shop_id,
    SAFE_DIVIDE(chargeback_count, transaction_count) AS chargeback_rate
  FROM shop_metrics
  
  -- Good: Using SAFE_CAST for casting
  SELECT
    SAFE_CAST(value AS FLOAT64) AS numeric_value
  FROM string_values
  ```

- **Handle NULL values appropriately**
  ```sql
  -- Good: Using COALESCE to handle NULLs
  SELECT
    shop_id,
    COALESCE(chargeback_count, 0) AS chargeback_count,
    COALESCE(transaction_count, 0) AS transaction_count
  FROM shop_metrics
  ```

## Documentation and Maintainability

### 1. Query Documentation

- **Include a header comment with:**
  - Query purpose/description
  - Last validation date
  - Author/team
  - Any assumptions or limitations
  
- **Document complex logic inline**
  ```sql
  -- Calculate the weighted risk score:
  -- - 60% based on chargeback rate
  -- - 30% based on decline rate
  -- - 10% based on refund rate
  SELECT
    shop_id,
    (chargeback_rate * 0.6) + (decline_rate * 0.3) + (refund_rate * 0.1) AS weighted_risk_score
  FROM shop_metrics
  ```

### 2. Version Control

- **Include version information in saved queries**
  ```sql
  -- Version: 1.2
  -- Change history:
  -- v1.2 (2024-05-15): Added refund rate to risk calculation
  -- v1.1 (2024-04-30): Updated chargeback_rate calculation
  -- v1.0 (2024-04-01): Initial version
  ```

### 3. Query Reusability

- **Parameterize queries when possible**
  ```sql
  -- Declare parameters for easier updates
  DECLARE lookback_days INT64 DEFAULT 90;
  
  SELECT
    shop_id,
    chargeback_count,
    transaction_count
  FROM shop_metrics
  WHERE date >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL lookback_days DAY)
  ```

- **Create Views for frequently used queries**
  ```sql
  CREATE OR REPLACE VIEW `project.dataset.high_risk_merchants` AS
  SELECT
    shop_id,
    chargeback_rate,
    risk_score
  FROM shop_metrics
  WHERE risk_score > 0.7
  ```

## Shopify-Specific Guidelines

### 1. Always Use shopify-dw Billing Project

- Set `shopify-dw` as the billing project for all queries
- In the BigQuery UI: Ensure the project is selected in the query editor
- In API calls: Specify the project
  ```python
  client = bigquery.Client(project='shopify-dw')
  ```

### 2. Use Recommended Tables

- Prefer aggregated/derived tables over raw tables when available
- Use the tables listed in the [Data Dictionary](../03_Data_Dictionary/Table_Overview.md)
- Verify table freshness before critical analyses

### 3. Test Queries Before Adding to Repository

- Run on a limited dataset first (use LIMIT)
- Validate results against known benchmarks
- Check for reasonable execution time and resource usage

### 4. Apply Table-Specific Best Practices

- For `chargebacks_summary`:
  - Always filter by provider_chargeback_created_at (partition column)
  - Join with transaction data for complete context

- For `order_transactions_payments_summary`:
  - Always filter by _extracted_at (partition column)
  - Filter on specific currencies when analyzing amounts
  - Use appropriate date/timestamp functions (TIMESTAMP_SUB, CURRENT_TIMESTAMP()) for timestamp fields

## Common Pitfalls to Avoid

### 1. Performance Issues

- Joining large tables without appropriate filters
- Using ORDER BY without LIMIT
- Selecting unnecessary columns
- Not using partitioning columns in filters
- Using complex UDFs when standard functions would work

### 2. Accuracy Issues

- Dividing by zero (use SAFE_DIVIDE)
- Not handling NULL values properly
- Double-counting in joins (check join cardinality)
- Mixing currencies without conversion
- Incorrect date/time handling across timezones
- Using DATE functions with TIMESTAMP columns or vice versa

### 3. Maintainability Issues

- Overly complex queries without documentation
- Hardcoded values instead of parameters
- No version control or change history
- Inconsistent naming or formatting
- Lack of error handling

## Query Performance Optimization

BigQuery performance optimization is critical for both query speed and cost efficiency when working with large datasets. These tips will help you write more efficient queries.

### 1. Optimize Table Scanning

- **Leverage partitioned tables**: Always filter on partitioned columns to reduce data scanned
  ```sql
  -- Good: Using partition column
  WHERE order_transaction_created_at >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 30 DAY)
  ```

- **Use clustering columns**: For tables with clustering, include clustering columns in filters and joins
  ```sql
  -- Good: Using a clustering column like shop_id in filters
  WHERE shop_id IN (SELECT shop_id FROM high_risk_merchants)
  ```

- **Avoid full table scans**: Use selective filters early in query logic
  ```sql
  -- Bad: Scanning entire table before filtering
  SELECT * FROM large_table ORDER BY date_column LIMIT 100
  
  -- Good: Filter first, then order and limit
  SELECT * FROM large_table 
  WHERE date_column >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 7 DAY)
  ORDER BY date_column LIMIT 100
  ```

### 2. Optimize Joins

- **Join order matters**: Put the smallest table first in the FROM clause
  ```sql
  -- Good: Small table first when joining
  SELECT *
  FROM small_table s
  JOIN large_table l ON s.id = l.id
  ```

- **Denormalize when appropriate**: For repeated analyses, consider pre-joining frequently used tables
  ```sql
  -- Create materialized view for commonly joined data
  CREATE MATERIALIZED VIEW project.dataset.denormalized_shop_data AS
  SELECT 
    s.shop_id, 
    s.name,
    s.created_at,
    c.chargeback_rate_90d
  FROM `shopify-dw.accounts_and_administration.shop_profile_current` s
  JOIN `sdp-prd-cti-data.intermediate.shop_chargeback_rates_current` c
    ON s.shop_id = c.shop_id
  ```

- **Use broadcast joins for small tables**: If one table is very small (< 10 MB), explicitly hint a broadcast join
  ```sql
  -- Good: Using broadcast join hint for small lookup table
  SELECT /*+ BROADCAST(small_table) */
    large_table.*, 
    small_table.property
  FROM large_table
  JOIN small_table ON large_table.id = small_table.id
  ```

### 3. Data Reduction Techniques

- **Project early**: Select only needed columns as early as possible
  ```sql
  -- Good: Select only needed fields in subquery
  WITH relevant_data AS (
    SELECT shop_id, amount_local, currency_code_local
    FROM `shopify-dw.money_products.order_transactions_payments_summary`
    WHERE order_transaction_created_at >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 90 DAY)
  )
  ```

- **Aggregate early**: Apply aggregations as early as possible to reduce data volume
  ```sql
  -- Good: Aggregate in CTE before joining
  WITH daily_txns AS (
    SELECT 
      shop_id, 
      DATE(order_transaction_created_at) AS date,
      COUNT(*) AS transaction_count,
      SUM(amount_local) AS total_amount
    FROM `shopify-dw.money_products.order_transactions_payments_summary`
    WHERE order_transaction_created_at >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 90 DAY)
    GROUP BY shop_id, DATE(order_transaction_created_at)
  )
  ```

- **Use subqueries for filtering**: Reduce the size of large tables with subqueries before joining
  ```sql
  -- Good: Filter with subquery before joining
  SELECT t.* 
  FROM transactions t
  JOIN (SELECT shop_id FROM shops WHERE country_code = 'US') s
    ON t.shop_id = s.shop_id
  ```

### 4. Caching and Materialization

- **Use query results cache**: Identical queries will use cached results for ~24 hours
  ```sql
  -- Query caching happens automatically for identical queries
  -- Add a comment to distinguish queries if you want to bypass the cache
  -- /*+ NO_CACHE */
  ```

- **Create materialized views for frequent queries**: Pre-compute and save results
  ```sql
  CREATE MATERIALIZED VIEW `project.dataset.high_risk_shops` AS
  SELECT 
    shop_id, 
    chargeback_rate_90d
  FROM `sdp-prd-cti-data.intermediate.shop_chargeback_rates_current`
  WHERE chargeback_rate_90d > 0.01
    AND sp_transaction_count_90d > 100
  ```

- **Consider table snapshots**: For frequently needed historical data points
  ```sql
  -- Create a snapshot of current metrics (pseudo-code)
  CREATE OR REPLACE TABLE `project.dataset.metrics_snapshot_20240601` AS
  SELECT * FROM `project.dataset.current_metrics`
  ```

### 5. Query Structure Optimization

- **Use CTEs instead of multiple subqueries**: Easier to read, debug and optimize
  ```sql
  -- Good: Using CTEs for clarity and potential optimization
  WITH filtered_data AS (
    SELECT * FROM large_table WHERE date > '2024-01-01'
  ),
  aggregated_data AS (
    SELECT id, SUM(value) as total FROM filtered_data GROUP BY id
  )
  SELECT * FROM aggregated_data WHERE total > 1000
  ```

- **Use appropriate aggregation functions**: Choose exact or approximate based on needs
  ```sql
  -- Exact count (slower but precise)
  SELECT COUNT(DISTINCT shop_id) 
  
  -- Approximate count (faster, slight accuracy trade-off)
  SELECT APPROX_COUNT_DISTINCT(shop_id)
  ```

- **Optimize window functions**: Limit window frame size when possible
  ```sql
  -- Limited window frame (more efficient)
  SUM(amount) OVER (PARTITION BY shop_id ORDER BY date ROWS BETWEEN 6 PRECEDING AND CURRENT ROW)
  
  -- vs unbounded frame (less efficient)
  SUM(amount) OVER (PARTITION BY shop_id ORDER BY date ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW)
  ```

### 6. Query Monitoring and Tuning

- **Use EXPLAIN to analyze query plans**
  ```sql
  EXPLAIN SELECT * FROM `shopify-dw.money_products.order_transactions_payments_summary`
  WHERE shop_id = 12345
  ```

- **Monitor slot usage**: Check if queries are using excessive resources

- **Use query profiling**: Identify bottlenecks in query execution
  ```sql
  -- Add profiling hint
  SELECT /*+ PROFILE */
    shop_id,
    COUNT(*) as tx_count
  FROM `shopify-dw.money_products.order_transactions_payments_summary`
  GROUP BY shop_id
  ```

- **Split complex queries**: Break very complex queries into smaller, more manageable pieces

## Further Resources

- [SQL Basics Guide](./SQL_Basics.md)
- [CTE Guide](./CTE_Guide.md)
- [Common SQL Patterns](./Common_SQL_Patterns.md)
- [Table Relationships](../03_Data_Dictionary/Table_Relationships.md)
- [BigQuery Documentation](https://cloud.google.com/bigquery/docs)

---