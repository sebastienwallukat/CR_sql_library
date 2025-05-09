# Guide: Working with DATE vs TIMESTAMP Fields in Credit Risk Data

This guide explains how to properly work with temporal fields in the Credit Risk data tables. Handling these fields correctly is critical for accurate analysis and query performance.

## Common Date/Timestamp Error

The #1 error encountered during MCP validation is the mismatch between DATE and TIMESTAMP types:

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
-- CORRECT: Using DATE_SUB with DATE columns
SELECT 
  shop_id, 
  chargeback_rate_90d
FROM `sdp-prd-cti-data.intermediate.shop_chargeback_rates_daily_snapshot`
WHERE date >= DATE_SUB(CURRENT_DATE(), INTERVAL 30 DAY)
```

### For TIMESTAMP Fields

```sql
-- CORRECT: Using TIMESTAMP_SUB with TIMESTAMP columns
SELECT 
  trust_platform_ticket_id, 
  subjectable_id AS shop_id
FROM `shopify-dw.risk.trust_platform_tickets_summary_v1`
WHERE created_at >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 30 DAY)
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
```

MCP error message:
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
-- Converting TIMESTAMP to DATE for comparison
SELECT 
  chargeback_id, 
  shop_id
FROM `shopify-dw.money_products.chargebacks_summary`
WHERE DATE(provider_chargeback_created_at) = CURRENT_DATE() -- Extract date portion
```

### Converting DATE to TIMESTAMP

```sql
-- Converting DATE to TIMESTAMP for comparison
-- This creates a timestamp at midnight UTC for the given date
SELECT 
  shop_id, 
  chargeback_rate_90d
FROM `sdp-prd-cti-data.intermediate.shop_chargeback_rates_daily_snapshot`
WHERE TIMESTAMP(date) <= CURRENT_TIMESTAMP()
```

## Combining DATE and TIMESTAMP Tables

When joining tables with different temporal types, convert them to the same type:

```sql
-- Joining tables with different date types
SELECT
  cb.shop_id,
  DATE(cb.provider_chargeback_created_at) AS chargeback_date,
  rates.chargeback_rate_90d
FROM `shopify-dw.money_products.chargebacks_summary` cb
JOIN `sdp-prd-cti-data.intermediate.shop_chargeback_rates_daily_snapshot` rates
  ON cb.shop_id = rates.shop_id
  AND DATE(cb.provider_chargeback_created_at) = rates.date
WHERE cb.provider_chargeback_created_at >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 90 DAY)
  AND rates.date >= DATE_SUB(CURRENT_DATE(), INTERVAL 90 DAY)
```

## Real-World Example from MCP Validation

During MCP validation, we fixed the following query that had date/timestamp mismatches:

### Original Query with Error

```sql
SELECT 
  trust_platform_ticket_id,
  subjectable_id AS shop_id,
  created_at,
  latest_report_type AS ticket_type,
  rule_group AS risk_level,
  status AS ticket_status,
  latest_action_type AS action_taken
FROM `shopify-dw.risk.trust_platform_tickets_summary_v1`
WHERE created_at >= DATE_SUB(CURRENT_DATE(), INTERVAL 30 DAY)  -- ERROR: Timestamp vs Date mismatch
  AND rule_group = 'HIGH'
  AND latest_action_type IS NOT NULL
ORDER BY created_at DESC
```

MCP error message:
```
No matching signature for operator >= for argument types: TIMESTAMP, DATE
Signature: T1 >= T1
Unable to find common supertype for templated argument <T1>
Input types for <T1>: {DATE, TIMESTAMP}
```

### Corrected Query

```sql
SELECT 
  trust_platform_ticket_id,
  subjectable_id AS shop_id,
  created_at,
  latest_report_type AS ticket_type,
  rule_group AS risk_level,
  status AS ticket_status,
  latest_action_type AS action_taken
FROM `shopify-dw.risk.trust_platform_tickets_summary_v1`
WHERE created_at >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 30 DAY)  -- FIXED: Using TIMESTAMP functions
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
   SELECT column_name, data_type
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

---

*Last Validated with MCP: 2023-05-07*