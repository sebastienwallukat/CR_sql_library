# Chargeback Tables Documentation

This document provides detailed information about the chargeback-related tables used by the Credit Risk team. These tables are central to analyzing dispute patterns, calculating chargeback rates, and identifying high-risk merchants.

## Table of Contents

- [Overview](#overview)
- [Key Tables](#key-tables)
  - [chargebacks_summary](#chargebacks_summary)
  - [base__payments_disputes](#base__payments_disputes)
  - [shop_chargeback_rates_current](#shop_chargeback_rates_current)
  - [shop_chargeback_rates_daily_snapshot](#shop_chargeback_rates_daily_snapshot)
- [Common Metrics](#common-metrics)
- [Sample Queries](#sample-queries)
- [Table Relationships](#table-relationships)
- [Best Practices](#best-practices)

## Overview

Chargeback data is critical for credit risk analysis, allowing the team to:

1. Identify merchants with high chargeback rates
2. Analyze patterns and trends in dispute behavior
3. Detect potential fraud or policy violations
4. Calculate accurate risk metrics for reserve recommendations
5. Monitor the effectiveness of risk mitigation measures

## Key Tables

### chargebacks_summary

**Full Path:** `shopify-dw.money_products.chargebacks_summary`

**Description:** A comprehensive view of all chargebacks across Shopify Payments, including details on status, reason, amount, and related transaction information.

**Primary Keys:** chargeback_id, shop_id, created_at

**Refresh Frequency:** Daily

**Schema:**

| Column Name | Data Type | Description | Example |
|-------------|-----------|-------------|---------|
| chargeback_id | INTEGER | Unique identifier for the chargeback | 12345678 |
| shop_id | INTEGER | Unique identifier for the shop | 12345678 |
| order_id | INTEGER | ID of the order related to the chargeback | 12345678 |
| reason | STRING | Reason for the chargeback | product_not_received |
| status | STRING | Current status of the chargeback | won |
| amount | NUMERIC | Amount of the chargeback | 99.99 |
| currency | STRING | Currency code | USD |
| amount_usd | NUMERIC | Amount in USD (converted) | 99.99 |
| created_at | TIMESTAMP | When the chargeback was created | 2023-02-15 10:30:15 UTC |
| updated_at | TIMESTAMP | When the chargeback was last updated | 2023-03-01 14:22:30 UTC |
| payment_id | INTEGER | ID of the original payment | 12345678 |
| payment_method | STRING | Method used for payment | credit_card |
| provider | STRING | Payment provider | shopify_payments |
| remote_reference | STRING | External reference ID | sp_chxxx12345 |
| evidence_due_by | TIMESTAMP | When evidence must be submitted by | 2023-03-15 23:59:59 UTC |
| evidence_submitted_at | TIMESTAMP | When evidence was submitted | 2023-03-10 16:45:20 UTC |

**Notes:**
- This table only includes chargebacks from 2018-01-01 onwards
- All amounts are converted to USD based on the exchange rate at `created_at`
- Common `status` values include: pending, under_review, lost, won, needs_response
- Common `reason` values include: product_not_received, product_unacceptable, fraudulent, general
- This table is a view on top of `shopify-dw.intermediate.chargebacks_summary_v1`

### base__payments_disputes

**Full Path:** `sdp-prd-cti-data.base.base__payments_disputes`

**Description:** Detailed dispute information from the payments system, including both chargebacks and inquiries.

**Primary Keys:** dispute_id, shop_id, order_id

**Refresh Frequency:** Daily

**Schema:**

| Column Name | Data Type | Description | Example |
|-------------|-----------|-------------|---------|
| dispute_id | STRING | Unique identifier for the dispute | disp_123xyz |
| shop_id | INTEGER | Unique identifier for the shop | 12345678 |
| order_id | INTEGER | Related order ID | 12345678 |
| type | STRING | Type of dispute | chargeback |
| reason | STRING | Reason for the dispute | fraudulent |
| status | STRING | Current status of the dispute | needs_response |
| amount | NUMERIC | Amount of the dispute | 99.99 |
| currency | STRING | Currency code | USD |
| processor | STRING | Payment processor | shopify_payments |
| processor_dispute_id | STRING | ID from processor | ch_12345 |
| payment_id | STRING | ID of the original payment | pay_12345 |
| created_at | TIMESTAMP | When the dispute was created | 2023-02-15 10:30:15 UTC |
| updated_at | TIMESTAMP | When the dispute was last updated | 2023-03-01 14:22:30 UTC |
| evidence_due_date | TIMESTAMP | Deadline for evidence submission | 2023-03-15 23:59:59 UTC |
| submitted_evidence | BOOLEAN | Whether evidence was submitted | TRUE |

**Notes:**
- Includes both chargebacks and inquiries (pre-chargeback disputes)
- `type` field distinguishes between chargeback and inquiry
- Links to order and payment data via respective IDs
- Status transitions can be tracked via the created_at and updated_at fields
- This is a base layer table in the medallion architecture

### shop_chargeback_rates_current

**Full Path:** `sdp-prd-cti-data.intermediate.shop_chargeback_rates_current`

**Description:** Current chargeback rates for shops, including rolling windows of various durations.

**Primary Keys:** shop_id

**Refresh Frequency:** Daily

**Schema:**

| Column Name | Data Type | Description | Example |
|-------------|-----------|-------------|---------|
| shop_id | INTEGER | Unique identifier for the shop | 12345678 |
| cb_count_30d | INTEGER | Number of chargebacks in last 30 days | 5 |
| cb_count_60d | INTEGER | Number of chargebacks in last 60 days | 8 |
| cb_count_90d | INTEGER | Number of chargebacks in last 90 days | 12 |
| cb_amount_usd_30d | NUMERIC | Total chargeback amount (USD) in 30 days | 495.75 |
| cb_amount_usd_60d | NUMERIC | Total chargeback amount (USD) in 60 days | 789.50 |
| cb_amount_usd_90d | NUMERIC | Total chargeback amount (USD) in 90 days | 1250.25 |
| gpv_usd_30d | NUMERIC | Gross payment volume (USD) in 30 days | 15000.00 |
| gpv_usd_60d | NUMERIC | Gross payment volume (USD) in 60 days | 32000.00 |
| gpv_usd_90d | NUMERIC | Gross payment volume (USD) in 90 days | 45000.00 |
| cb_rate_30d | NUMERIC | Chargeback rate for 30-day window | 0.033 |
| cb_rate_60d | NUMERIC | Chargeback rate for 60-day window | 0.025 |
| cb_rate_90d | NUMERIC | Chargeback rate for 90-day window | 0.027 |
| last_cb_date | DATE | Date of most recent chargeback | 2023-03-10 |
| processed_at | TIMESTAMP | When this record was calculated | 2023-03-15 12:00:00 UTC |

**Notes:**
- Chargeback rates calculated as cb_amount_usd / gpv_usd for each time window
- All amounts are in USD for consistent comparison
- The table provides multiple time windows to identify trends
- This table is updated daily with the most current rates
- Used extensively in risk monitoring and reserve calculations

### shop_chargeback_rates_daily_snapshot

**Full Path:** `sdp-prd-cti-data.intermediate.shop_chargeback_rates_daily_snapshot`

**Description:** Historical daily snapshots of shop chargeback rates, providing a time series of rate changes.

**Primary Keys:** shop_id, snapshot_date

**Refresh Frequency:** Daily

**Schema:**

| Column Name | Data Type | Description | Example |
|-------------|-----------|-------------|---------|
| shop_id | INTEGER | Unique identifier for the shop | 12345678 |
| snapshot_date | DATE | Date of this snapshot | 2023-03-15 |
| cb_count_30d | INTEGER | Number of chargebacks in last 30 days | 5 |
| cb_count_60d | INTEGER | Number of chargebacks in last 60 days | 8 |
| cb_count_90d | INTEGER | Number of chargebacks in last 90 days | 12 |
| cb_amount_usd_30d | NUMERIC | Total chargeback amount (USD) in 30 days | 495.75 |
| cb_amount_usd_60d | NUMERIC | Total chargeback amount (USD) in 60 days | 789.50 |
| cb_amount_usd_90d | NUMERIC | Total chargeback amount (USD) in 90 days | 1250.25 |
| gpv_usd_30d | NUMERIC | Gross payment volume (USD) in 30 days | 15000.00 |
| gpv_usd_60d | NUMERIC | Gross payment volume (USD) in 60 days | 32000.00 |
| gpv_usd_90d | NUMERIC | Gross payment volume (USD) in 90 days | 45000.00 |
| cb_rate_30d | NUMERIC | Chargeback rate for 30-day window | 0.033 |
| cb_rate_60d | NUMERIC | Chargeback rate for 60-day window | 0.025 |
| cb_rate_90d | NUMERIC | Chargeback rate for 90-day window | 0.027 |

**Notes:**
- Contains the same metrics as shop_chargeback_rates_current but with historical daily values
- Useful for trend analysis and identifying patterns over time
- Data is partitioned by snapshot_date for efficient querying
- Snapshots are created daily with a 1-day lag
- Used for historical reporting and time-series analysis

## Common Metrics

When working with chargeback data, the following metrics are commonly used:

1. **Chargeback Rate**: The ratio of chargeback amount to total payment volume
   ```sql
   SELECT 
       shop_id,
       cb_rate_30d,
       cb_rate_60d,
       cb_rate_90d
   FROM `sdp-prd-cti-data.intermediate.shop_chargeback_rates_current`
   WHERE shop_id = 12345678
   ```

2. **Chargeback Velocity**: The rate at which chargebacks are occurring
   ```sql
   SELECT 
       shop_id,
       COUNT(*) as chargebacks_last_7_days
   FROM `shopify-dw.money_products.chargebacks_summary`
   WHERE shop_id = 12345678
   AND created_at >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 7 DAY)
   GROUP BY shop_id
   ```

3. **Dominant Chargeback Reason**: The most common reason for chargebacks
   ```sql
   SELECT 
       shop_id,
       reason,
       COUNT(*) as occurrence_count
   FROM `shopify-dw.money_products.chargebacks_summary`
   WHERE shop_id = 12345678
   GROUP BY shop_id, reason
   ORDER BY occurrence_count DESC
   LIMIT 1
   ```

4. **Chargeback Win Rate**: The percentage of chargebacks that were successfully disputed
   ```sql
   SELECT 
       shop_id,
       COUNTIF(status = 'won') / COUNT(*) as win_rate
   FROM `shopify-dw.money_products.chargebacks_summary`
   WHERE shop_id = 12345678
   GROUP BY shop_id
   ```

## Sample Queries

### Query 1: Identify Shops with High Chargeback Rates

```sql
SELECT 
    s.shop_id,
    s.shop_name,
    c.cb_count_30d,
    c.gpv_usd_30d,
    c.cb_rate_30d,
    c.last_cb_date,
    DATE_DIFF(CURRENT_DATE(), c.last_cb_date, DAY) as days_since_last_cb
FROM `sdp-prd-cti-data.intermediate.shop_chargeback_rates_current` c
JOIN `sdp-prd-cti-data.intermediate.shop_insights_shops` s ON c.shop_id = s.shop_id
WHERE c.cb_rate_30d > 0.01  -- 1% threshold
  AND c.gpv_usd_30d > 5000  -- Minimum payment volume
  AND c.cb_count_30d >= 3   -- Minimum chargeback count
ORDER BY c.cb_rate_30d DESC
LIMIT 100
```

### Query 2: Chargeback Reason Analysis

```sql
WITH reason_counts AS (
    SELECT
        shop_id,
        reason,
        COUNT(*) as count,
        SUM(amount_usd) as total_amount_usd
    FROM `shopify-dw.money_products.chargebacks_summary`
    WHERE created_at >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 90 DAY)
    GROUP BY shop_id, reason
)

SELECT
    shop_id,
    reason,
    count,
    total_amount_usd,
    ROUND(count / SUM(count) OVER (PARTITION BY shop_id) * 100, 2) as percent_of_total_count,
    ROUND(total_amount_usd / SUM(total_amount_usd) OVER (PARTITION BY shop_id) * 100, 2) as percent_of_total_amount
FROM reason_counts
WHERE shop_id = 12345678  -- Replace with specific shop ID
ORDER BY count DESC
```

### Query 3: Chargeback Rate Trend Analysis

```sql
SELECT
    snapshot_date,
    cb_rate_30d,
    cb_count_30d,
    gpv_usd_30d
FROM `sdp-prd-cti-data.intermediate.shop_chargeback_rates_daily_snapshot`
WHERE shop_id = 12345678  -- Replace with specific shop ID
  AND snapshot_date >= DATE_SUB(CURRENT_DATE(), INTERVAL 90 DAY)
ORDER BY snapshot_date
```

## Table Relationships

Chargeback tables relate to other tables in the following ways:

1. `chargebacks_summary` links to:
   - Order data via `order_id`
   - Shop data via `shop_id`
   - Payment data via `payment_id`

2. `base__payments_disputes` links to:
   - Order data via `order_id`
   - Shop data via `shop_id`
   - Payment data via `payment_id`
   - Processor-specific data via `processor_dispute_id`

3. `shop_chargeback_rates_current` and `shop_chargeback_rates_daily_snapshot` aggregate data from:
   - `chargebacks_summary` for chargeback amounts
   - Payment transaction tables for GPV calculation

## Best Practices

When working with chargeback data, follow these best practices:

1. **For Rate Calculations**:
   - Always use multiple time windows (30d, 60d, 90d)
   - Compare rates across windows to identify trends
   - Consider both count and amount-based rates
   - Set minimum thresholds for GPV to avoid skewed ratios

2. **For Chargeback Reason Analysis**:
   - Group reasons into meaningful categories
   - Weight by both frequency and amount
   - Consider shop-specific context (industry, business model)
   - Track changes in dominant reasons over time

3. **For Snapshot Analysis**:
   - Use time-series analysis to identify patterns
   - Look for sudden spikes or gradual increases
   - Correlate with other shop events or changes
   - Consider seasonality in certain industries

4. **For Risk Assessment**:
   - Combine chargeback data with other risk signals
   - Weight recent chargebacks more heavily
   - Consider disposition status (won vs. lost)
   - Adjust for shop size and processing volume

## Related Documentation

- [Risk Assessment Tables](./Risk_Assessment_Tables.md)
- [Payment Transaction Tables](./Payment_Transaction_Tables.md)
- [Reserve Management](../05_Domain_Knowledge/Reserves.md)

---

*Last Updated: May 2024* 