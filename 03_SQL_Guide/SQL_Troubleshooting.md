# SQL Troubleshooting Guide

This guide covers common SQL errors encountered during credit risk analysis, their causes, and solutions. Use this resource to diagnose and fix issues with your queries.

## Common Error Types

### 1. Syntax Errors

#### Error: `Syntax error: Expected ... but got ...`

**Cause**: SQL syntax is incorrectly written.

**Common fixes**:
- Check for missing or extra commas
- Ensure parentheses are balanced and properly closed
- Verify keywords are spelled correctly
- Check for incorrect alias definitions

**Example Fix**:
```sql
-- Error: Syntax error: Expected ")" but got "," at [6:19]
SELECT 
  shop_id, 
  COUNT(*) as count,  -- Extra comma here
FROM transactions

-- Fixed:
SELECT 
  shop_id, 
  COUNT(*) as count   -- Removed the extra comma
FROM transactions
```

#### Error: `Syntax error: Unexpected keyword ...`

**Cause**: Using a keyword in the wrong context or misspelling it.

**Common fixes**:
- Check for correct SQL keyword usage
- Ensure reserved words used as identifiers are properly quoted
- Verify the SQL dialect (BigQuery has specific syntax requirements)

**Example Fix**:
```sql
-- Error: Syntax error: Unexpected keyword GROUP
SELECT 
  GROUP,  -- GROUP is a reserved keyword
  COUNT(*) 
FROM transactions

-- Fixed:
SELECT 
  `GROUP`,  -- Enclosed in backticks
  COUNT(*) 
FROM transactions
```

### 2. Name Resolution Errors

#### Error: `Table not found: project:dataset.table`

**Cause**: Referenced table doesn't exist or you don't have access.

**Common fixes**:
- Check table name spelling
- Verify the project and dataset names
- Ensure you have appropriate permissions
- Check if you're using the right billing project

**Example Fix**:
```sql
-- Error: Table not found: shopify-dw.money_product.chargebacks_summary
SELECT * FROM `shopify-dw.money_product.chargebacks_summary`

-- Fixed:
SELECT * FROM `shopify-dw.money_products.chargebacks_summary`  -- Corrected "money_products"
```

#### Error: `Column ... not found in any table`

**Cause**: Referenced column doesn't exist in the tables in scope.

**Common fixes**:
- Check column name spelling
- Verify column exists in the specified table
- Check table aliases if using them
- Ensure column is available in the current scope

**Example Fix**:
```sql
-- Error: Column 'chargeback_id' not found in any table
SELECT 
  shop_id,
  chargeback_id  -- This column doesn't exist in transactions table
FROM `shopify-dw.money_products.order_transactions_payments_summary`

-- Fixed:
SELECT 
  t.shop_id,
  c.chargeback_id  -- Properly referenced from chargebacks table
FROM `shopify-dw.money_products.order_transactions_payments_summary` t
JOIN `shopify-dw.money_products.chargebacks_summary` c
  ON t.transaction_id = c.transaction_id
```

### 3. Type Mismatch Errors

#### Error: `Cannot coerce type ... to ...`

**Cause**: Trying to compare or operate on incompatible data types.

**Common fixes**:
- Cast variables to appropriate types
- Use `SAFE_CAST` to prevent errors
- Ensure date/timestamp formats are correct

**Example Fix**:
```sql
-- Error: Cannot coerce string 'high' to double
SELECT 
  *
FROM transactions
WHERE amount > 'high'  -- Comparing number to string

-- Fixed:
SELECT 
  *
FROM transactions
WHERE amount > 1000  -- Using a numeric value
```

#### Error: `No matching signature for function ...`

**Cause**: Function called with incorrect parameter types or count.

**Common fixes**:
- Check function documentation for correct parameters
- Ensure parameters are of the expected type
- Convert parameters to the required type

**Example Fix**:
```sql
-- Error: No matching signature for function DATE_DIFF for argument types: STRING, STRING, STRING
SELECT 
  DATE_DIFF('2023-01-01', '2023-02-01', 'day')  -- All parameters are strings

-- Fixed:
SELECT 
  DATE_DIFF(DATE '2023-01-01', DATE '2023-02-01', DAY)  -- Properly typed parameters
```

### 4. Resource Exceeded Errors

#### Error: `Resources exceeded during query execution: Not enough resources ...`

**Cause**: Query requires more resources than allocated (memory, CPU, etc.).

**Common fixes**:
- Filter data earlier in the query
- Reduce the date range being queried
- Break the query into smaller parts
- Add more specific filters
- Use partitioned fields for filtering
- Avoid SELECT * when only specific columns are needed

**Example Fix**:
```sql
-- Error: Resources exceeded during query execution
SELECT 
  *  -- Selecting all columns
FROM `shopify-dw.money_products.order_transactions_payments_summary`
JOIN `shopify-dw.money_products.chargebacks_summary` 
  ON transaction_id = transaction_id  -- Missing table aliases causing cross join

-- Fixed:
SELECT 
  t.shop_id,
  t.transaction_id,
  t.amount,
  c.chargeback_id  -- Only selecting needed columns
FROM `shopify-dw.money_products.order_transactions_payments_summary` t
JOIN `shopify-dw.money_products.chargebacks_summary` c
  ON t.transaction_id = c.transaction_id  -- Proper join condition
WHERE t.created_at >= DATE_SUB(CURRENT_DATE(), INTERVAL 90 DAY)  -- Date filter
```

#### Error: `Query exceeded resource limits...`

**Cause**: Query processing exceeded time or slot limits.

**Common fixes**:
- Simplify the query
- Use partitioning to reduce data scanned
- Pre-aggregate data in CTEs
- Avoid complex joins between large tables
- Create materialized views for frequently used data

**Example Fix**:
```sql
-- Error: Query exceeded resource limits
WITH all_transactions AS (
  SELECT * FROM `shopify-dw.money_products.order_transactions_payments_summary`
)
SELECT 
  shop_id,
  COUNT(*) as transaction_count
FROM all_transactions
GROUP BY shop_id

-- Fixed:
SELECT 
  shop_id,
  COUNT(*) as transaction_count
FROM `shopify-dw.money_products.order_transactions_payments_summary`
WHERE created_at >= DATE_SUB(CURRENT_DATE(), INTERVAL 90 DAY)  -- Added date filter
GROUP BY shop_id
```

### 5. Join and Aggregation Errors

#### Error: `GROUP BY expression references column ... which is neither grouped nor aggregated`

**Cause**: Column in SELECT not included in GROUP BY or aggregation function.

**Common fixes**:
- Add the column to the GROUP BY clause
- Apply an aggregation function to the column
- Remove the column from the SELECT list

**Example Fix**:
```sql
-- Error: GROUP BY expression references column shop_name which is neither grouped nor aggregated
SELECT 
  shop_id,
  shop_name,  -- This is not in the GROUP BY
  COUNT(*) as transaction_count
FROM transactions
GROUP BY shop_id

-- Fixed:
SELECT 
  shop_id,
  shop_name,
  COUNT(*) as transaction_count
FROM transactions
GROUP BY shop_id, shop_name  -- Added shop_name to GROUP BY
```

#### Error: `Ambiguous column name ...`

**Cause**: Column name exists in multiple tables and is not fully qualified.

**Common fixes**:
- Use table aliases to qualify the column
- Provide explicit aliases for ambiguous columns
- Use fully qualified column names

**Example Fix**:
```sql
-- Error: Ambiguous column name 'shop_id'
SELECT 
  shop_id,  -- Ambiguous - exists in both tables
  COUNT(*) as count
FROM `shopify-dw.money_products.order_transactions_payments_summary`
JOIN `shopify-dw.money_products.chargebacks_summary`
  ON transaction_id = transaction_id

-- Fixed:
SELECT 
  t.shop_id,  -- Clearly from transactions table
  COUNT(*) as count
FROM `shopify-dw.money_products.order_transactions_payments_summary` t
JOIN `shopify-dw.money_products.chargebacks_summary` c
  ON t.transaction_id = c.transaction_id
```

### 6. Permissions Errors

#### Error: `Access Denied: Project [project_id]: User does not have permission to access project`

**Cause**: User lacks necessary permissions for the project or dataset.

**Common fixes**:
- Request access through Clouddo
- Check if you're using the correct billing project
- Verify your IAM roles and permissions
- Ask an admin to grant required access

**Example Fix**:
```sql
-- Error: Access Denied: Project [my-personal-project]...
-- Using personal project for billing instead of shopify-dw

-- Fixed:
-- Set shopify-dw as the billing project in the BigQuery UI
-- Or use the appropriate API setting in code
```

#### Error: `Access Denied: Table [project:dataset.table]: User does not have permission to query table`

**Cause**: User lacks permission to access a specific table.

**Common fixes**:
- Request specific table access
- For PII data, obtain the `sdp-pii` permit
- Check if you need domain-specific permissions
- Use alternative tables you have access to

**Example**:
```
# Request PII access through Clouddo:
1. Navigate to clouddo.shopify.io/permits
2. Search for "sdp-pii"
3. Click "Claim Permit"
4. Provide justification
```

### 7. Data Quality Errors

#### Error: `Division by zero: ...`

**Cause**: Attempting to divide by zero or NULL.

**Common fixes**:
- Use SAFE_DIVIDE() function
- Add conditional checks for zero/NULL values
- Use NULLIF() to prevent division by zero

**Example Fix**:
```sql
-- Error: Division by zero
SELECT 
  shop_id,
  chargeback_count / transaction_count as chargeback_rate  -- Division by zero for some rows
FROM shop_metrics

-- Fixed:
SELECT 
  shop_id,
  SAFE_DIVIDE(chargeback_count, transaction_count) as chargeback_rate  -- Safe division
FROM shop_metrics
```

#### Error: `Cannot read field 'X' from NULL value`

**Cause**: Trying to access a field of a NULL struct or array.

**Common fixes**:
- Use COALESCE() to provide default values
- Add IS NOT NULL checks before accessing fields
- Use LEFT JOIN instead of INNER JOIN if appropriate

**Example Fix**:
```sql
-- Error: Cannot read field 'amount' from NULL value
SELECT 
  t.transaction_id,
  c.amount  -- c might be NULL for some rows
FROM transactions t
LEFT JOIN chargebacks c ON t.transaction_id = c.transaction_id

-- Fixed:
SELECT 
  t.transaction_id,
  COALESCE(c.amount, 0) as chargeback_amount  -- Default to 0 if NULL
FROM transactions t
LEFT JOIN chargebacks c ON t.transaction_id = c.transaction_id
```

## Performance Troubleshooting

### 1. Slow Query Execution

**Symptoms**:
- Query takes longer than expected to run
- Query times out or fails
- Results appear but take minutes instead of seconds

**Diagnosis techniques**:
- Check the query execution plan in BigQuery
- Look for table scans without filters
- Identify expensive joins or aggregations
- Check for inefficient data access patterns

**Common solutions**:
- Add filters on partitioned columns
- Minimize the amount of data scanned
- Pre-aggregate data in CTEs
- Simplify complex joins
- Use date range restrictions
- Avoid SELECT * when possible
- Create materialized views for frequent queries

**Example Optimization**:
```sql
-- Slow query
SELECT 
  s.shop_id,
  s.shop_name, 
  COUNT(t.transaction_id) as transaction_count,
  SUM(t.amount) as total_amount
FROM `shopify-dw.shopify.shops` s  -- Large table scan
JOIN `shopify-dw.money_products.order_transactions_payments_summary` t
  ON s.id = t.shop_id
GROUP BY s.shop_id, s.shop_name

-- Optimized query
SELECT 
  s.shop_id,
  s.shop_name, 
  COUNT(t.transaction_id) as transaction_count,
  SUM(t.amount) as total_amount
FROM `shopify-dw.money_products.order_transactions_payments_summary` t
JOIN `shopify-dw.shopify.shops` s   -- Join after filtering
  ON s.id = t.shop_id
WHERE t.created_at >= DATE_SUB(CURRENT_DATE(), INTERVAL 90 DAY)  -- Date filter added
GROUP BY s.shop_id, s.shop_name
```

### 2. Unexpected Results

**Symptoms**:
- Results don't match expectations
- Missing or extra rows
- Incorrect calculations

**Diagnosis techniques**:
- Test with simplified queries
- Check join conditions (inner vs outer joins)
- Verify aggregation logic
- Check for unexpected NULL handling
- Validate date/time handling
- Examine data types and conversions

**Common solutions**:
- Use LEFT JOIN instead of INNER JOIN to preserve rows
- Handle NULL values with COALESCE or IFNULL
- Use proper date/time functions for time comparisons
- Check for duplicate rows in source data
- Verify currency handling in financial calculations

**Example Fix**:
```sql
-- Query with unexpected results (missing shops with no transactions)
SELECT 
  s.shop_id,
  COUNT(t.transaction_id) as transaction_count  -- Counts NULL as zero
FROM `shopify-dw.shopify.shops` s
INNER JOIN `shopify-dw.money_products.order_transactions_payments_summary` t  -- Inner join excludes shops with no transactions
  ON s.id = t.shop_id
GROUP BY s.shop_id

-- Fixed query
SELECT 
  s.shop_id,
  COUNT(t.transaction_id) as transaction_count  -- Counts NULL as zero
FROM `shopify-dw.shopify.shops` s
LEFT JOIN `shopify-dw.money_products.order_transactions_payments_summary` t  -- Left join preserves all shops
  ON s.id = t.shop_id
GROUP BY s.shop_id
```

## Query Validation Techniques

### 1. Incremental Testing

1. Start with a simplified version of your query
2. Add complexity one step at a time
3. Verify results at each step
4. Use LIMIT to test with small result sets first

**Example**:
```sql
-- Step 1: Test base table access
SELECT COUNT(*) FROM `shopify-dw.money_products.chargebacks_summary`
WHERE created_at >= DATE_SUB(CURRENT_DATE(), INTERVAL 30 DAY)

-- Step 2: Add basic grouping
SELECT shop_id, COUNT(*) as count
FROM `shopify-dw.money_products.chargebacks_summary`
WHERE created_at >= DATE_SUB(CURRENT_DATE(), INTERVAL 30 DAY)
GROUP BY shop_id
LIMIT 10  -- Test with limited results

-- Step 3: Add join to another table
SELECT 
  c.shop_id, 
  COUNT(c.chargeback_id) as chargeback_count,
  COUNT(t.transaction_id) as transaction_count
FROM `shopify-dw.money_products.chargebacks_summary` c
JOIN `shopify-dw.money_products.order_transactions_payments_summary` t
  ON c.shop_id = t.shop_id
WHERE c.created_at >= DATE_SUB(CURRENT_DATE(), INTERVAL 30 DAY)
  AND t.created_at >= DATE_SUB(CURRENT_DATE(), INTERVAL 30 DAY)
GROUP BY c.shop_id
LIMIT 10
```

### 2. Cross-Validation with Known Results

1. Compare results with existing reports or tables
2. Validate against standard tables like `shop_chargeback_rates_current`
3. Use sample calculations to verify logic
4. Check row counts against expected values

**Example**:
```sql
-- Custom chargeback rate calculation
WITH my_chargeback_rates AS (
  SELECT 
    shop_id,
    COUNT(DISTINCT c.chargeback_id) AS calculated_chargeback_count,
    COUNT(DISTINCT t.transaction_id) AS calculated_transaction_count,
    SAFE_DIVIDE(COUNT(DISTINCT c.chargeback_id), COUNT(DISTINCT t.transaction_id)) 
      AS calculated_chargeback_rate
  FROM `shopify-dw.money_products.order_transactions_payments_summary` t
  LEFT JOIN `shopify-dw.money_products.chargebacks_summary` c
    ON t.transaction_id = c.transaction_id
  WHERE t.created_at >= DATE_SUB(CURRENT_DATE(), INTERVAL 90 DAY)
  GROUP BY shop_id
)

-- Cross-validate with standard table
SELECT 
  m.shop_id,
  m.calculated_chargeback_count,
  s.chargeback_count AS standard_chargeback_count,
  m.calculated_transaction_count,
  s.transaction_count AS standard_transaction_count,
  m.calculated_chargeback_rate,
  s.chargeback_rate AS standard_chargeback_rate,
  -- Calculate differences
  m.calculated_chargeback_rate - s.chargeback_rate AS rate_difference
FROM my_chargeback_rates m
JOIN `sdp-prd-cti-data.intermediate.shop_chargeback_rates_current` s
  ON m.shop_id = s.shop_id
WHERE ABS(m.calculated_chargeback_rate - s.chargeback_rate) > 0.0001  -- Show only significant differences
ORDER BY rate_difference DESC
```

### 3. Sanity Checks

1. Look for impossible values (negative amounts, future dates)
2. Check for unreasonable rates (>100%)
3. Verify totals match expected magnitude
4. Look for suspiciously high or low values

**Example**:
```sql
-- Sanity check for transaction data
SELECT 
  -- Date range check
  MIN(created_at) AS oldest_transaction,
  MAX(created_at) AS newest_transaction,
  
  -- Amount checks
  MIN(amount) AS min_amount,
  MAX(amount) AS max_amount,
  AVG(amount) AS avg_amount,
  
  -- Volume checks
  COUNT(*) AS total_transactions,
  COUNT(DISTINCT shop_id) AS unique_shops,
  
  -- Suspicious values
  COUNTIF(amount < 0) AS negative_amounts,
  COUNTIF(amount > 10000) AS large_amounts,
  COUNTIF(created_at > CURRENT_TIMESTAMP()) AS future_dates
FROM `shopify-dw.money_products.order_transactions_payments_summary`
WHERE created_at >= DATE_SUB(CURRENT_DATE(), INTERVAL 90 DAY)
```

## Resource Limits and Workarounds

### BigQuery Limits

| Limit Type | Default Value | Workaround |
|------------|--------------|------------|
| Query timeout | 6 hours | Break into smaller queries |
| Maximum query length | 1 MB | Use views or stored procedures |
| Maximum result size | 10 GB | Stream results or use pagination |
| Concurrency limit | Project-based | Schedule queries at different times |
| Daily quota | Project-based | Monitor usage and optimize queries |

### Workarounds for Resource Limits

#### 1. Breaking Up Large Queries

```sql
-- Instead of one large date range:
SELECT * FROM huge_table WHERE date BETWEEN '2020-01-01' AND '2023-12-31'

-- Break into yearly chunks:
-- Year 2020
SELECT * FROM huge_table WHERE date BETWEEN '2020-01-01' AND '2020-12-31'
-- Year 2021
SELECT * FROM huge_table WHERE date BETWEEN '2021-01-01' AND '2021-12-31'
-- etc.
```

#### 2. Creating Materialized Views or Tables

```sql
-- Create a materialized view for frequently accessed aggregations
CREATE OR REPLACE TABLE `your_project.dataset.materialized_chargeback_metrics` AS
SELECT 
  shop_id,
  DATE_TRUNC(created_at, MONTH) AS month,
  COUNT(*) AS chargeback_count,
  SUM(amount) AS chargeback_amount
FROM `shopify-dw.money_products.chargebacks_summary`
WHERE created_at >= DATE_SUB(CURRENT_DATE(), INTERVAL 12 MONTH)
GROUP BY shop_id, DATE_TRUNC(created_at, MONTH)
```

#### 3. Using Query Parameters

```sql
-- Using parameters to make queries more efficient
DECLARE start_date DATE DEFAULT DATE_SUB(CURRENT_DATE(), INTERVAL 90 DAY);
DECLARE end_date DATE DEFAULT CURRENT_DATE();

SELECT 
  shop_id,
  COUNT(*) AS transaction_count
FROM `shopify-dw.money_products.order_transactions_payments_summary`
WHERE created_at >= start_date AND created_at < end_date
GROUP BY shop_id
```

#### 4. Limiting Columns Selected

```sql
-- Instead of:
SELECT * FROM `shopify-dw.money_products.order_transactions_payments_summary`

-- Select only needed columns:
SELECT 
  shop_id,
  transaction_id,
  order_id,
  amount,
  currency,
  created_at,
  status
FROM `shopify-dw.money_products.order_transactions_payments_summary`
```

## Getting Help

If you're still encountering issues after trying the solutions in this guide:

1. Check the [BigQuery Documentation](https://cloud.google.com/bigquery/docs)
2. Ask in the `#credit-risk-data` Slack channel
3. Consult with the Data team
4. Check the [BigQuery official error messages documentation](https://cloud.google.com/bigquery/docs/error-messages)
5. For access issues, follow the [Access and Permissions](../01_Getting_Started/Access_and_Permissions.md) guide

---

*Last Updated: May 2024* 