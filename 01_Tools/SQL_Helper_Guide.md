# SQL Helper Guide

This guide explains how to use SQL Helper, a tool that assists with SQL query development for credit risk analysis at Shopify.

## What is SQL Helper?

SQL Helper is an AI-powered assistant that helps you write, improve, and understand SQL queries. It can:

- Generate SQL queries based on natural language descriptions
- Explain complex SQL queries in plain language
- Optimize existing queries for better performance
- Identify and fix errors in problematic queries
- Suggest improvements to query structure

## When to Use SQL Helper

Use SQL Helper when you need to:

- Create a new query but aren't sure about the exact SQL syntax
- Understand what an existing query is doing
- Optimize a slow-running query
- Fix errors in a query that isn't working
- Learn about SQL best practices
- Convert a business question into a SQL query

## Accessing SQL Helper

SQL Helper is available at [https://sql-helper.shopify.io/](https://sql-helper.shopify.io/)

## Key Features for Credit Risk Analysis

### 1. Query Generation

SQL Helper can generate queries from natural language descriptions:

```
Example prompt: "Show me the chargeback rate for each shop over the last 90 days, 
only for shops with at least 100 transactions, using the chargebacks_summary and 
order_transactions_payments_summary tables."
```

The tool will generate a properly structured query like:

```sql
WITH transactions AS (
  SELECT 
    shop_id,
    COUNT(*) AS transaction_count
  FROM `shopify-dw.money_products.order_transactions_payments_summary`
  WHERE order_transaction_created_at >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 90 DAY)
    AND order_transaction_status = 'success'
    AND _extracted_at >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 90 DAY)
  GROUP BY shop_id
  HAVING COUNT(*) >= 100
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
ORDER BY chargeback_rate DESC
```

### 2. Query Explanation

SQL Helper can explain what a query is doing in plain language:

```
Example prompt: "Explain what this query is doing: [paste your query here]"
```

The tool will provide a step-by-step explanation of the query logic.

### 3. Query Optimization

SQL Helper can suggest performance improvements:

```
Example prompt: "Optimize this query for better performance: [paste your query here]"
```

The tool will suggest changes like:
- Adding filters on partitioned columns
- Simplifying complex joins
- Pre-aggregating data
- Reducing data scanned

### 4. Error Diagnosis

SQL Helper can identify and fix errors:

```
Example prompt: "Fix the errors in this query: [paste your query here]"
```

The tool will identify issues like:
- Syntax errors
- Missing table references
- Incorrect join conditions
- Type mismatches

## Best Practices for Using SQL Helper

### 1. Be Specific with Your Requests

Provide detailed context in your prompts:
- Mention specific table names
- Specify time periods
- Include business context
- Mention any specific calculations

**Good example**: "Generate a query to find shops with chargeback rates above 1% in the last 90 days using the chargebacks_summary and order_transactions_payments_summary tables."

**Less effective**: "Find high risk shops."

### 2. Review and Validate Generated Queries

Always review SQL Helper outputs before running them:
- Check table names and field references
- Verify join conditions are correct
- Ensure filters are appropriate
- Test on small data samples first

### 3. Iterate for Better Results

Use a conversational approach to refine queries:
1. Start with a basic request
2. Review the generated query
3. Ask for specific modifications
4. Repeat until you have the right query

### 4. Learn from the Tool

Use SQL Helper as a learning opportunity:
- Study the patterns in generated queries
- Ask for explanations of unfamiliar techniques
- Request alternative approaches to the same problem

## Example Workflows for Credit Risk Analysis

### Scenario 1: Investigating High-Risk Merchants

```
Step 1: "Generate a query to identify merchants with chargeback rates over 0.5% 
in the last 30 days, with at least 100 transactions."

Step 2: "Modify the query to also include the merchant's GPV from the 
shop_gmv_current table and whether they have an active reserve from the 
base__shopify_payments_reserve_configurations table."

Step 3: "Add a risk categorization column that classifies merchants as 'High Risk' 
if chargeback rate > 1%, 'Medium Risk' if between 0.5% and 1%, and 'Low Risk' if below 0.5%."
```

### Scenario 2: Time Series Analysis of Chargebacks

```
Step 1: "Create a query that shows daily chargeback counts and rates for the last 90 days."

Step 2: "Modify the query to include a 7-day rolling average of the chargeback rate."

Step 3: "Add a comparison to the same period last year."
```

### Scenario 3: Reserve Impact Analysis

```
Step 1: "Generate a query to show current reserve configurations for merchants."

Step 2: "Extend the query to calculate the estimated reserve amount based on the 
merchant's GPV from shop_gmv_daily_summary_v1_1."

Step 3: "Compare the reserve amount to the merchant's balance from 
shopify_payments_balance_account_daily_cumulative_summary."
```

## Limitations and Considerations

### 1. Data Access Limitations

SQL Helper may generate queries referencing tables you don't have access to:
- Always verify you have appropriate permissions
- Be aware that PII data requires specific access
- Check for alternative tables if you encounter access issues

### 2. Query Complexity Limitations

Some complex analyses may exceed the tool's capabilities:
- Break down complex questions into smaller components
- Build up the query piece by piece
- Manually review and adjust complex logic

### 3. Performance Considerations

Generated queries might not always be optimized:
- Review for inefficient patterns
- Check for appropriate filters
- Verify partitioning columns are used
- Test on small data samples before running on full datasets

### 4. Business Logic Verification

The tool doesn't understand all Shopify-specific business logic:
- Verify calculations match business definitions
- Check assumptions about data relationships
- Confirm time periods align with business reporting cycles

### 5. Date and Timestamp Handling

Shopify's BigQuery tables strictly enforce type matching between dates and timestamps:

#### Common Date/Timestamp Issues

- **TIMESTAMP columns** (most common in Shopify data):
  ```sql
  -- CORRECT
  WHERE order_transaction_created_at >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 30 DAY)
  
  -- INCORRECT - will cause type mismatch error
  WHERE order_transaction_created_at >= DATE_SUB(CURRENT_DATE(), INTERVAL 30 DAY)
  ```

- **DATE columns**:
  ```sql
  -- CORRECT
  WHERE date_field >= DATE_SUB(CURRENT_DATE(), INTERVAL 30 DAY)
  
  -- INCORRECT - will cause type mismatch error
  WHERE date_field >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 30 DAY)
  ```

- **Type conversion when needed**:
  ```sql
  -- Converting TIMESTAMP to DATE for comparison
  WHERE DATE(timestamp_column) >= DATE_SUB(CURRENT_DATE(), INTERVAL 30 DAY)
  ```

#### Partitioned Tables

Many Shopify BigQuery tables are partitioned by timestamp columns (often `_extracted_at`):
- Always include a filter on the partitioning column in your WHERE clause
- This improves performance and is required for most tables:

```sql
-- Including partition filter (required for most tables)
WHERE order_transaction_created_at >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 90 DAY)
  AND _extracted_at >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 90 DAY)
```

## Getting Help

If you need assistance with SQL Helper:

1. Check the internal documentation
2. Ask in the `#credit-risk-data` Slack channel
3. Contact the Data Platform team
4. Refer to the [SQL Best Practices](../02_SQL_Guide/SQL_Best_Practices.md) guide

## Additional Resources

For more robust SQL support, consider:

1. Using pre-built queries from the [Example Queries](../05_Example_Queries/) section
2. Reviewing the [Common SQL Patterns](../02_SQL_Guide/Common_SQL_Patterns.md) guide
3. Referring to the [BigQuery Guide](./BigQuery_Guide.md) 
4. Refer to the [SQL Best Practices](../02_SQL_Guide/SQL_Best_Practices.md) guide

---