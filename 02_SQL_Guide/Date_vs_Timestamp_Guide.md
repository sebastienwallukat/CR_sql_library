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

- `date` in `shopify-dw.mart_core_deliver.shop_back_office_summary_current`
- `snapshot_date` in `sdp-prd-cti-data.intermediate.shop_chargeback_rates_daily_snapshot`
- `date` in `shopify-dw.mart_cti_data.shop_loss_metrics__wide`

### TIMESTAMP Fields

Timestamp fields include both date and time information in UTC. Examples include:

- `created_at` in `shopify-dw.risk.trust_platform_tickets_summary_v1`
- `provider_chargeback_created_at` in `shopify-dw.money_products.chargebacks_summary`
- `observed_at` and `processed_at` in `sdp-prd-cti-data.intermediate.hvcr_predictions_v2`

## Correct Time Functions

Always use the matching function for the field type:

| Field Type | Correct Function | Incorrect Function |
|------------|------------------|--------------------|
| DATE | `DATE_SUB()` | `TIMESTAMP_SUB()` |
| TIMESTAMP | `TIMESTAMP_SUB()` | `DATE_SUB()` |

## Correct Query Examples

### For DATE Fields

```sql
-- CORRECT: Using DATE_SUB with DATE columns
SELECT shop_id, gmv_usd
FROM `shopify-dw.mart_core_deliver.shop_back_office_summary_current`
WHERE date >= DATE_SUB(CURRENT_DATE(), INTERVAL 30 DAY)
```

### For TIMESTAMP Fields

```sql
-- CORRECT: Using TIMESTAMP_SUB with TIMESTAMP columns
SELECT trust_platform_ticket_id, subjectable_id AS shop_id
FROM `shopify-dw.risk.trust_platform_tickets_summary_v1`
WHERE created_at >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 30 DAY)
```

## Incorrect Query Examples

### Mixing DATE and TIMESTAMP

```sql
-- INCORRECT: This will fail with type mismatch error
SELECT trust_platform_ticket_id, subjectable_id AS shop_id
FROM `shopify-dw.risk.trust_platform_tickets_summary_v1`
WHERE created_at >= DATE_SUB(CURRENT_DATE(), INTERVAL 30 DAY)
```

```sql
-- INCORRECT: This will fail with type mismatch error
SELECT shop_id, gmv_usd
FROM `shopify-dw.mart_core_deliver.shop_back_office_summary_current`
WHERE date >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 30 DAY)
```

## Type Conversion Solutions

When you need to compare different types, convert them explicitly:

### Converting TIMESTAMP to DATE

```sql
-- Converting TIMESTAMP to DATE for comparison
SELECT chargeback_id, shop_id
FROM `shopify-dw.money_products.chargebacks_summary`
WHERE DATE(provider_chargeback_created_at) = CURRENT_DATE() -- Extract date portion
```

### Converting DATE to TIMESTAMP

```sql
-- Converting DATE to TIMESTAMP for comparison
-- This creates a timestamp at midnight UTC for the given date
SELECT shop_id, gmv_usd
FROM `shopify-dw.mart_core_deliver.shop_back_office_summary_current`
WHERE TIMESTAMP(date) <= CURRENT_TIMESTAMP()
```

## Combining DATE and TIMESTAMP Tables

When joining tables with different temporal types, convert them to the same type:

```sql
-- Joining tables with different date types
SELECT
  shop.shop_id,
  shop.shop_name,
  COUNT(cb.chargeback_id) AS chargeback_count
FROM `shopify-dw.mart_core_deliver.shop_back_office_summary_current` shop
JOIN `shopify-dw.money_products.chargebacks_summary` cb
  ON shop.shop_id = cb.shop_id
  AND shop.date = DATE(cb.provider_chargeback_created_at)
WHERE shop.date >= DATE_SUB(CURRENT_DATE(), INTERVAL 30 DAY)
  AND cb.provider_chargeback_created_at >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 90 DAY) -- Filter on partition field
GROUP BY shop.shop_id, shop.shop_name
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

---