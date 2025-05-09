# Guide: Working with DATE vs TIMESTAMP Fields in Credit Risk Data

This guide explains how to properly work with temporal fields in the Credit Risk data tables. Handling these fields correctly is critical for accurate analysis and query performance.

## Table of Contents
- [Common Date/Timestamp Error](#common-datetimestamp-error)
- [Field Types in Credit Risk Tables](#field-types-in-credit-risk-tables)
- [Correct Time Functions](#correct-time-functions)
- [Correct Query Examples](#correct-query-examples)
- [Incorrect Query Examples](#incorrect-query-examples)
- [Type Conversion Solutions](#type-conversion-solutions)
- [Combining DATE and TIMESTAMP Tables](#combining-date-and-timestamp-tables)
- [Real-World Example from MCP Validation](#real-world-example-from-mcp-validation)
- [Important: Partitioned Fields Require Filtering](#important-partitioned-fields-require-filtering)
- [Best Practices](#best-practices)
- [Troubleshooting Query Errors](#troubleshooting-query-errors)
- [Credit Risk Table Temporal Field Types Quick Reference](#credit-risk-table-temporal-field-types-quick-reference)
- [Sample Query Outputs](#sample-query-outputs)

## Common Date/Timestamp Error

The #1 error encountered during validation is the mismatch between DATE and TIMESTAMP types:

```
No matching signature for operator >= for argument types: TIMESTAMP, DATE
```

This error occurs when you compare a TIMESTAMP column with a DATE value or vice versa.

## Field Types in Credit Risk Tables

### DATE Fields

Date fields are calendar dates without time information. Examples include:

- `date` in `sdp-prd-cti-data.intermediate.shop_chargeback_rates_daily_snapshot`
- `date` in `shopify-dw.intermediate.shop_gmv_daily_summary_v1_1`
- `date` in `shopify-dw.money_products.shopify_payments_balance_account_daily_cumulative_summary`

### TIMESTAMP Fields

Timestamp fields include both date and time information in UTC. Examples include:

- `created_at` in `shopify-dw.risk.trust_platform_tickets_summary_v1`
- `provider_chargeback_created_at` in `shopify-dw.money_products.chargebacks_summary`
- `order_transaction_created_at` in `shopify-dw.money_products.order_transactions_payments_summary`
- `reserve_created_at` in `sdp-prd-cti-data.base.base__shopify_payments_reserve_configurations`

## Correct Time Functions

Always use the matching function for the field type:

| Field Type | Correct Function | Incorrect Function |
|------------|------------------|--------------------|
| DATE | `DATE_SUB()`, `CURRENT_DATE()` | `TIMESTAMP_SUB()`, `CURRENT_TIMESTAMP()` |
| TIMESTAMP | `TIMESTAMP_SUB()`, `CURRENT_TIMESTAMP()` | `DATE_SUB()`, `CURRENT_DATE()` |

## Correct Query Examples

### For DATE Fields

```sql
-- Purpose: Retrieve recent shop chargeback rates for risk analysis
-- This query pulls the 90-day chargeback rates for shops from the past 30 days
SELECT 
  shop_id,                  -- Unique identifier for the shop
  chargeback_rate_90d       -- Calculated chargeback rate over the past 90 days
FROM `sdp-prd-cti-data.intermediate.shop_chargeback_rates_daily_snapshot`
WHERE date >= DATE_SUB(CURRENT_DATE(), INTERVAL 30 DAY)  -- Only consider recent data
```

### For TIMESTAMP Fields

```sql
-- Purpose: Find recently created high-risk trust platform tickets
-- This query identifies tickets created in the last 30 days
SELECT 
  trust_platform_ticket_id,     -- Unique identifier for the ticket
  subjectable_id AS shop_id,    -- Shop associated with the ticket
  created_at,                   -- When the ticket was created
  latest_report_type            -- Type of report that triggered the ticket
FROM `shopify-dw.risk.trust_platform_tickets_summary_v1`
WHERE created_at >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 30 DAY)  -- Only recent tickets
  AND rule_group = 'HIGH'       -- Focusing on high-risk tickets only
```

## Incorrect Query Examples

### Mixing DATE and TIMESTAMP

```sql
-- INCORRECT: This will fail with type mismatch error
SELECT 
  trust_platform_ticket_id,
  subjectable_id AS shop_id
FROM `shopify-dw.risk.trust_platform_tickets_summary_v1`
WHERE created_at >= DATE_SUB(CURRENT_DATE(), INTERVAL 30 DAY)  -- ERROR: TIMESTAMP vs DATE
  AND rule_group = 'HIGH'
```

Error message:
```
No matching signature for operator >= for argument types: TIMESTAMP, DATE
Signature: T1 >= T1
Unable to find common supertype for templated argument <T1>
Input types for <T1>: {DATE, TIMESTAMP}
```

```sql
-- INCORRECT: This will fail with type mismatch error
SELECT 
  shop_id, 
  chargeback_rate_90d
FROM `sdp-prd-cti-data.intermediate.shop_chargeback_rates_daily_snapshot`
WHERE date >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 30 DAY)  -- ERROR: DATE vs TIMESTAMP
```

## Type Conversion Solutions

When you need to compare different types, convert them explicitly:

### Converting TIMESTAMP to DATE

```sql
-- Purpose: Find chargebacks that occurred on the current date
-- Used when you need only the date portion of a timestamp
SELECT 
  chargeback_id,            -- Unique identifier for the chargeback
  shop_id,                  -- Shop affected by the chargeback 
  provider_chargeback_created_at  -- When the chargeback was created at the payment provider
FROM `shopify-dw.money_products.chargebacks_summary`
WHERE DATE(provider_chargeback_created_at) = CURRENT_DATE()  -- Extract date portion for comparison
```

### Converting DATE to TIMESTAMP

```sql
-- Purpose: Compare daily snapshot dates with current time
-- This converts a DATE to a TIMESTAMP at midnight UTC
SELECT 
  shop_id,                  -- Shop identifier
  chargeback_rate_90d       -- 90-day chargeback rate
FROM `sdp-prd-cti-data.intermediate.shop_chargeback_rates_daily_snapshot`
WHERE TIMESTAMP(date) <= CURRENT_TIMESTAMP()  -- Convert date to timestamp for comparison
```

## Combining DATE and TIMESTAMP Tables

When joining tables with different temporal types, convert them to the same type:

```sql
-- Purpose: Combine chargeback data with calculated rates for comprehensive analysis
-- This query demonstrates proper joining of tables with different date types
SELECT
  cb.shop_id,                                           -- Shop identifier
  DATE(cb.provider_chargeback_created_at) AS chargeback_date,  -- Convert timestamp to date for display
  rates.chargeback_rate_90d                             -- 90-day chargeback rate
FROM `shopify-dw.money_products.chargebacks_summary` cb
JOIN `sdp-prd-cti-data.intermediate.shop_chargeback_rates_daily_snapshot` rates
  ON cb.shop_id = rates.shop_id
  AND DATE(cb.provider_chargeback_created_at) = rates.date  -- Convert TIMESTAMP to DATE for join
WHERE cb.provider_chargeback_created_at >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 90 DAY)  -- Recent chargebacks
  AND rates.date >= DATE_SUB(CURRENT_DATE(), INTERVAL 90 DAY)  -- Recent rate calculations
```

## Real-World Example from MCP Validation

During validation, we fixed the following query that had date/timestamp mismatches:

### Original Query with Error

```sql
-- Purpose: Analyze high risk tickets with actions taken during the past month
SELECT 
  trust_platform_ticket_id,     -- Ticket identifier
  subjectable_id AS shop_id,    -- Associated shop 
  created_at,                   -- When the ticket was created
  latest_report_type AS ticket_type,  -- Type of report
  rule_group AS risk_level,     -- Risk assessment level
  status AS ticket_status,      -- Current status of the ticket
  latest_action_type AS action_taken  -- Last action applied to the ticket
FROM `shopify-dw.risk.trust_platform_tickets_summary_v1`
WHERE created_at >= DATE_SUB(CURRENT_DATE(), INTERVAL 30 DAY)  -- ERROR: Timestamp vs Date mismatch
  AND rule_group = 'HIGH'
  AND latest_action_type IS NOT NULL
ORDER BY created_at DESC
```

Error message:
```
No matching signature for operator >= for argument types: TIMESTAMP, DATE
Signature: T1 >= T1
Unable to find common supertype for templated argument <T1>
Input types for <T1>: {DATE, TIMESTAMP}
```

### Corrected Query

```sql
-- Purpose: Analyze high risk tickets with actions taken during the past month
SELECT 
  trust_platform_ticket_id,     -- Ticket identifier
  subjectable_id AS shop_id,    -- Associated shop
  created_at,                   -- When the ticket was created
  latest_report_type AS ticket_type,  -- Type of report
  rule_group AS risk_level,     -- Risk assessment level
  status AS ticket_status,      -- Current status of the ticket
  latest_action_type AS action_taken  -- Last action applied to the ticket
FROM `shopify-dw.risk.trust_platform_tickets_summary_v1`
WHERE created_at >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 30 DAY)  -- FIXED: Using proper TIMESTAMP function
  AND rule_group = 'HIGH'
  AND latest_action_type IS NOT NULL
ORDER BY created_at DESC
```

## Important: Partitioned Fields Require Filtering

Many BigQuery tables in the Credit Risk data set are partitioned on date or timestamp fields for performance. When querying these tables, you **must** include a filter on the partition field:

- `shopify-dw.money_products.chargebacks_summary` is partitioned on `provider_chargeback_created_at`
- `shopify-dw.risk.trust_platform_tickets_summary_v1` is partitioned on `created_at`
- `sdp-prd-cti-data.intermediate.shop_chargeback_rates_daily_snapshot` is partitioned on `date`

Failing to filter on the partition field will result in validation errors like:

```
Query validation failed: Table is partitioned by column 'provider_chargeback_created_at', 
but no filter on this column was found in the WHERE clause.
```

## Best Practices

1. **Know Your Fields**: Understand which fields are DATE vs TIMESTAMP before writing queries
2. **Use Correct Functions**: Always use `DATE_SUB()` for DATE fields and `TIMESTAMP_SUB()` for TIMESTAMP fields
3. **Explicit Conversion**: When comparing different types, use explicit conversion functions (`DATE()` or `TIMESTAMP()`)
4. **Use EXTRACT**: For complex time logic, use `EXTRACT()` to get specific parts (year, month, day, hour)
5. **Partitioned Tables**: Always include filters on partition fields to improve query performance and avoid validation errors
6. **Reference Data Type Column**: Always refer to the Data Type column in the [README.md](../README.md) table for quick reference on field types

## Troubleshooting Query Errors

If you encounter temporal comparison errors:

1. Check the field types using:
   ```sql
   -- Purpose: Inspect column data types in a table schema
   -- Use this query to verify data types before writing complex queries
   SELECT 
     column_name,    -- Name of the column
     data_type       -- Data type (e.g., DATE, TIMESTAMP, etc.)
   FROM `project_id`.dataset_id.INFORMATION_SCHEMA.COLUMNS
   WHERE table_name = 'table_name'
   ```

2. Fix by using the appropriate function or converting types explicitly
3. Ensure partitioned fields are included in your query filters

## Credit Risk Table Temporal Field Types Quick Reference

| Table | Temporal Field | Data Type | Proper Function |
|-------|----------------|-----------|-----------------|
| `shopify-dw.money_products.chargebacks_summary` | `provider_chargeback_created_at` | TIMESTAMP | `TIMESTAMP_SUB()` |
| `sdp-prd-cti-data.intermediate.shop_chargeback_rates_daily_snapshot` | `date` | DATE | `DATE_SUB()` |
| `shopify-dw.risk.trust_platform_tickets_summary_v1` | `created_at` | TIMESTAMP | `TIMESTAMP_SUB()` |
| `shopify-dw.finance.shop_gmv_current` | `first_order_with_gmv_date` | DATE | `DATE_SUB()` |
| `shopify-dw.intermediate.shop_gmv_daily_summary_v1_1` | `date` | DATE | `DATE_SUB()` |
| `shopify-dw.money_products.shopify_payments_balance_account_daily_cumulative_summary` | `date` | DATE | `DATE_SUB()` |
| `shopify-dw.money_products.order_transactions_payments_summary` | `order_transaction_created_at` | TIMESTAMP | `TIMESTAMP_SUB()` |

## Sample Query Outputs

### Example 1: Shop Chargeback Rates

The following query retrieves recent shop chargeback rates:

```sql
-- Purpose: Get a sample of shop chargeback rates for risk analysis
SELECT 
  shop_id,                    -- Shop identifier
  chargeback_rate_90d         -- Calculated 90-day chargeback rate 
FROM `sdp-prd-cti-data.intermediate.shop_chargeback_rates_daily_snapshot`
WHERE date >= DATE_SUB(CURRENT_DATE(), INTERVAL 30 DAY)
LIMIT 5
```

Sample output:

| shop_id      | chargeback_rate_90d |
|--------------|---------------------|
| 66511503669  | 0.012195            |
| 80821354820  | 0.000000            |
| 63596560541  | 0.001019            |
| 63899730144  | 0.000000            |
| 62284300480  | 0.000000            |

Basic statistics:
- Average chargeback rate: 0.002643
- Maximum rate: 0.012195
- Minimum rate: 0.000000

This shows that most shops have very low or zero chargeback rates, but there are occasional shops with higher rates that may need attention from the risk team.

### Example 2: Distribution Analysis

When analyzing temporal data, visualization often helps identify patterns:

![Chargeback Rate Distribution](sample_chargeback_distribution.png)

A histogram of chargeback rates typically shows that:
- Most shops have very low (near zero) chargeback rates
- A long tail distribution with few shops having significantly higher rates
- Clear thresholds can be established for risk categorization

---

*Last Validated with MCP: 2023-05-07*