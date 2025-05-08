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

**Primary Keys:** chargeback_id

**Refresh Frequency:** Daily

**Schema:**

| Column Name | Data Type | Description | Example |
|-------------|-----------|-------------|---------|
| chargeback_id | INTEGER | Primary key of payments_disputes | 12345678 |
| shop_id | INTEGER | The shop ID associated with the payment charge transaction | 12345678 |
| order_id | INTEGER | Primary key of orders. The order the dispute is filed on. | 12345678 |
| reason | STRING | Reason for the dispute, e.g. product_not_received, fraudulent, product_unacceptable, etc. | product_not_received |
| status | STRING | Current status of the dispute. e.g. lost, won, etc. | won |
| chargeback_amount | NUMERIC | Amount filed for chargeback in the currency of the charge in chargeback_currency_code | 99.99 |
| chargeback_currency_code | STRING | The three-letter abbreviation currency (e.g. CAD) of the chargeback amount | USD |
| chargeback_amount_usd | NUMERIC | Amount filed for chargeback in USD | 99.99 |
| provider_chargeback_created_at | TIMESTAMP | The timestamp at which the chargeback was created by the provider | 2023-02-15 10:30:15 UTC |
| shopify_chargeback_created_at | TIMESTAMP | The timestamp at which the chargeback record was created in Shopify Core | 2023-02-15 10:30:15 UTC |
| evidence_due_by | TIMESTAMP | The timestamp at which the evidence is due | 2023-03-15 23:59:59 UTC |
| evidence_sent_at | TIMESTAMP | The timestamp at which the evidence was sent | 2023-03-10 16:45:20 UTC |
| order_transaction_id | INTEGER | Unique Identifier for each order transaction | 12345678 |
| payments_charge_id | INTEGER | Primary key of payments_charges. The charge transaction the dispute is attributed to. | 12345678 |
| network_reason_code | STRING | The network reason code for the dispute | 10.4 |
| is_first_chargeback_per_charge | BOOLEAN | Whether the chargeback is the first chargeback for the payments_charge_id | TRUE |
| is_first_chargeback_per_shop | BOOLEAN | Whether the chargeback is the first chargeback for the shop_id | TRUE |

**Notes:**
- This table only includes chargebacks from 2018-01-01 onwards
- All amounts are converted to USD based on the exchange rate at `provider_chargeback_created_at`
- Common `status` values include: pending, under_review, lost, won, needs_response
- Common `reason` values include: product_not_received, product_unacceptable, fraudulent, general
- This table is a view on top of `shopify-dw.intermediate.chargebacks_summary_v1`
- The table is partitioned by `provider_chargeback_created_at`

### base__payments_disputes

**Full Path:** `sdp-prd-cti-data.base.base__payments_disputes`

**Description:** Detailed dispute information from the payments system, including both chargebacks and inquiries.

**Primary Keys:** payments_dispute_id, shop_id, order_id

**Refresh Frequency:** Daily

**Schema:**

| Column Name | Data Type | Description | Example |
|-------------|-----------|-------------|---------|
| payments_dispute_id | INTEGER | Unique identifier for the dispute | 12345678 |
| shop_id | INTEGER | Unique identifier for the shop | 12345678 |
| order_id | INTEGER | Related order ID | 12345678 |
| payment_charge_id | INTEGER | ID of the original payment charge | 12345678 |
| amount | NUMERIC | Amount of the dispute | 99.99 |
| currency | STRING | Currency code | USD |
| created | TIMESTAMP | When the dispute record was created | 2023-02-15 10:30:15 UTC |
| reason | STRING | Reason for the dispute | fraudulent |
| status | STRING | Current status of the dispute | needs_response |
| evidence_due_by | TIMESTAMP | Deadline for evidence submission | 2023-03-15 23:59:59 UTC |
| created_at | TIMESTAMP | When the dispute was created | 2023-02-15 10:30:15 UTC |
| updated_at | TIMESTAMP | When the dispute was last updated | 2023-03-01 14:22:30 UTC |
| evidence_sent_on | TIMESTAMP | When evidence was submitted | 2023-03-10 16:45:20 UTC |
| initiated_as_inquiry | BOOLEAN | Whether the dispute was initiated as an inquiry | TRUE |
| customized_by_merchant | BOOLEAN | Whether the merchant customized the dispute | TRUE |
| network_reason_code | STRING | Network reason code for the dispute | 10.4 |

**Notes:**
- Includes both chargebacks and inquiries (pre-chargeback disputes)
- `initiated_as_inquiry` field distinguishes between chargeback and inquiry
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
| shop_id | INTEGER | Unique identifier of Shopify shop | 12345678 |
| chargeback_count_30d | INTEGER | Number of chargebacks in last 30 days | 5 |
| chargeback_count_60d | INTEGER | Number of chargebacks in last 60 days | 8 |
| chargeback_count_90d | INTEGER | Number of chargebacks in last 90 days | 12 |
| chargeback_amount_usd_30d | NUMERIC | Total chargeback amount (USD) in 30 days | 495.75 |
| chargeback_amount_usd_60d | NUMERIC | Total chargeback amount (USD) in 60 days | 789.50 |
| chargeback_amount_usd_90d | NUMERIC | Total chargeback amount (USD) in 90 days | 1250.25 |
| sp_transaction_amount_usd_30d | NUMERIC | Gross payment volume (USD) in 30 days | 15000.00 |
| sp_transaction_amount_usd_60d | NUMERIC | Gross payment volume (USD) in 60 days | 32000.00 |
| sp_transaction_amount_usd_90d | NUMERIC | Gross payment volume (USD) in 90 days | 45000.00 |
| chargeback_rate_30d | FLOAT | Chargeback rate for 30-day window | 0.033 |
| chargeback_rate_60d | FLOAT | Chargeback rate for 60-day window | 0.025 |
| chargeback_rate_90d | FLOAT | Chargeback rate for 90-day window | 0.027 |
| sp_transaction_count_30d | INTEGER | Number of Shopify Payments transactions in 30 days | 450 |
| sp_transaction_count_60d | INTEGER | Number of Shopify Payments transactions in 60 days | 950 |
| sp_transaction_count_90d | INTEGER | Number of Shopify Payments transactions in 90 days | 1500 |

**Notes:**
- Chargeback rates calculated as chargeback_count / sp_transaction_count for each time window
- All amounts are in USD for consistent comparison
- The table provides multiple time windows to identify trends
- This table is updated daily with the most current rates
- Used extensively in risk monitoring and reserve calculations

### shop_chargeback_rates_daily_snapshot

**Full Path:** `sdp-prd-cti-data.intermediate.shop_chargeback_rates_daily_snapshot`

**Description:** Historical daily snapshots of shop chargeback rates, providing a time series of rate changes.

**Primary Keys:** shop_id, date

**Refresh Frequency:** Daily

**Schema:**

| Column Name | Data Type | Description | Example |
|-------------|-----------|-------------|---------|
| shop_id | INTEGER | Unique identifier of Shopify shop | 12345678 |
| date | DATE | Date of this snapshot | 2023-03-15 |
| chargeback_count_30d | INTEGER | Number of chargebacks in last 30 days | 5 |
| chargeback_count_60d | INTEGER | Number of chargebacks in last 60 days | 8 |
| chargeback_count_90d | INTEGER | Number of chargebacks in last 90 days | 12 |
| chargeback_amount_usd_30d | NUMERIC | Total chargeback amount (USD) in 30 days | 495.75 |
| chargeback_amount_usd_60d | NUMERIC | Total chargeback amount (USD) in 60 days | 789.50 |
| chargeback_amount_usd_90d | NUMERIC | Total chargeback amount (USD) in 90 days | 1250.25 |
| sp_transaction_amount_usd_30d | NUMERIC | Gross payment volume (USD) in 30 days | 15000.00 |
| sp_transaction_amount_usd_60d | NUMERIC | Gross payment volume (USD) in 60 days | 32000.00 |
| sp_transaction_amount_usd_90d | NUMERIC | Gross payment volume (USD) in 90 days | 45000.00 |
| chargeback_rate_30d | FLOAT | Chargeback rate for 30-day window | 0.033 |
| chargeback_rate_60d | FLOAT | Chargeback rate for 60-day window | 0.025 |
| chargeback_rate_90d | FLOAT | Chargeback rate for 90-day window | 0.027 |
| sp_transaction_count_30d | INTEGER | Count of transactions in 30-day window | 450 |
| sp_transaction_count_60d | INTEGER | Count of transactions in 60-day window | 950 |
| sp_transaction_count_90d | INTEGER | Count of transactions in 90-day window | 1500 |

**Notes:**
- Contains the same metrics as shop_chargeback_rates_current but with historical daily values
- Useful for trend analysis and identifying patterns over time
- Data is partitioned by date for efficient querying
- Snapshots are created daily with a 1-day lag
- Used for historical reporting and time-series analysis
- Only includes the most recent 3 years of data
- Does not include inquiries or Visa Rapid Dispute Resolution "chargebacks"

## Common Metrics

When working with chargeback data, the following metrics are commonly used:

1. **Chargeback Rate**: The ratio of chargeback count to total transaction count
   ```sql
   SELECT 
       shop_id,
       chargeback_rate_30d,
       chargeback_rate_60d,
       chargeback_rate_90d
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
   AND provider_chargeback_created_at >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 7 DAY)
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
    c.chargeback_count_30d,
    c.sp_transaction_amount_usd_30d,
    c.chargeback_rate_30d
FROM `sdp-prd-cti-data.intermediate.shop_chargeback_rates_current` c
JOIN `sdp-prd-cti-data.intermediate.shop_insights_shops` s 
  ON c.shop_id = s.shop_id
WHERE c.chargeback_rate_30d > 0.01  -- 1% threshold
  AND c.sp_transaction_amount_usd_30d > 5000  -- Minimum payment volume
  AND c.chargeback_count_30d >= 3   -- Minimum chargeback count
ORDER BY c.chargeback_rate_30d DESC
LIMIT 100
```

### Query 2: Chargeback Reason Analysis

```sql
WITH reason_counts AS (
    SELECT
        shop_id,
        reason,
        COUNT(*) as count,
        SUM(chargeback_amount_usd) as total_amount_usd
    FROM `shopify-dw.money_products.chargebacks_summary`
    WHERE provider_chargeback_created_at >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 90 DAY)
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
    date,
    chargeback_rate_30d,
    chargeback_count_30d,
    sp_transaction_amount_usd_30d
FROM `sdp-prd-cti-data.intermediate.shop_chargeback_rates_daily_snapshot`
WHERE shop_id = 12345678  -- Replace with specific shop ID
  AND date >= DATE_SUB(CURRENT_DATE(), INTERVAL 90 DAY)
ORDER BY date
```

## Table Relationships

Chargeback tables relate to other tables in the following ways:

1. `chargebacks_summary` links to:
   - Order data via `order_id`
   - Shop data via `shop_id`
   - Payment data via `order_transaction_id` and `payments_charge_id`

2. `base__payments_disputes` links to:
   - Order data via `order_id`
   - Shop data via `shop_id`
   - Payment data via `payment_charge_id`
   - Processor-specific data via network information

3. `shop_chargeback_rates_current` and `shop_chargeback_rates_daily_snapshot` aggregate data from:
   - `chargebacks_summary` for chargeback amounts and counts
   - Payment transaction tables for transaction counts and volumes

## Best Practices

When working with chargeback data, follow these best practices:

1. **For Rate Calculations**:
   - Always use multiple time windows (30d, 60d, 90d)
   - Compare rates across windows to identify trends
   - Consider both count and amount-based rates
   - Set minimum thresholds for transaction volume to avoid skewed ratios

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

### Temporal Fields Documentation

| Field Name | Data Type | Comparison Function | Example |
|------------|-----------|---------------------|---------|
| provider_chargeback_created_at | TIMESTAMP | TIMESTAMP_SUB() | `WHERE provider_chargeback_created_at >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 7 DAY)` |
| date | DATE | DATE_SUB() | `WHERE date >= DATE_SUB(CURRENT_DATE(), INTERVAL 90 DAY)` |

## Related Resources

- [Risk Assessment Tables](./Risk_Assessment_Tables.md)
- [Payment Transaction Tables](./Payment_Transaction_Tables.md)
- [Reserve Management](../04_Domain_Knowledge/Reserves.md)

--- 