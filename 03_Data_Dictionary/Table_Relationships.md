# Table Relationships

## Introduction
This document outlines how key tables in the credit risk data ecosystem relate to each other, providing relationship diagrams, join fields, and example join patterns to facilitate data analysis.

## Table of Contents
- [Table Relationship Overview](#table-relationship-overview)
- [Key Join Fields](#key-join-fields)
- [Core Table Relationships](#core-table-relationships)
  - [Chargeback-Related Tables](#1-chargeback-related-tables)
  - [Financial Data Relationships](#2-financial-data-relationships)
  - [Shop Status and Risk Relationships](#3-shop-status-and-risk-relationships)
- [Domain-Specific Table Groups](#domain-specific-table-groups)
  - [Chargeback Domain](#chargeback-domain)
  - [Financial Domain](#financial-domain)
  - [Shop Domain](#shop-domain)
  - [Risk Domain](#risk-domain)
  - [Reserve Domain](#reserve-domain)
- [Common Join Patterns](#common-join-patterns)
  - [Comprehensive Merchant Risk Profile](#1-comprehensive-merchant-risk-profile)
  - [Chargeback to Transaction Ratio Over Time](#2-chargeback-to-transaction-ratio-over-time)
  - [Reserve to Balance Relationship](#3-reserve-to-balance-relationship)
  - [Union Examples](#union-examples)
  - [Many-to-Many Shop Relationships](#many-to-many-shop-relationships)
  - [Temporal Table Relationships](#temporal-table-relationships)
  - [Summary Statistics by Join](#summary-statistics-by-join)
  - [Multi-Hop Relationships](#multi-hop-relationships)
- [Table Update Frequency](#table-update-frequency)
- [Best Practices for Table Joins](#best-practices-for-table-joins)
- [Temporal Fields Reference](#temporal-fields-reference)

## Table Relationship Overview

The credit risk data ecosystem is organized around several core entities:
- **Shops/Merchants** - The businesses using Shopify Payments
- **Transactions** - Payment processing events
- **Chargebacks** - Disputed transactions
- **Reserves** - Funds held to mitigate risk
- **Trust Platform Tickets** - Risk-related investigations
- **Financial Data** - GPV, GMV, and balance information

## Key Join Fields

| Entity Type | Primary Join Field | Related Tables |
|-------------|-------------------|----------------|
| Shops | `shop_id` | Most tables use this as the primary merchant identifier |
| Orders | `order_id` | Used to connect order data across tables |
| Transactions | `order_transaction_id` | Links payment transactions to other events |
| Chargebacks | `chargeback_id`, `order_id`, `order_transaction_id` | Connected to transactions and orders |
| Trust Platform | `ticket_id` | Links risk investigations to shops |
| Payments | `payment_id` | Used for payment-specific data linkage |

## Core Table Relationships

### 1. Chargeback-Related Tables

```
┌───────────────────────────────┐          ┌─────────────────────────────────────┐
│ money_products.               │          │ sdp-prd-cti-data.intermediate.      │
│ chargebacks_summary           │          │ shop_chargeback_rates_daily_snapshot│
│                               │          │                                     │
│ - chargeback_id               │◄─────────┤ - shop_id                           │
│ - shop_id                     │          │ - date                              │
│ - order_id                    │          │ - chargeback_count                  │
│ - order_transaction_id        │          │ - chargeback_rate                   │
│ - provider_chargeback_created_at  │      │ - transaction_count                 │
│ - amount                      │          └─────────────────────────────────────┘
│ - currency                    │                          ▲
│ - reason                      │                          │
└───────────────────────────────┘                          │
               ▲                                           │
               │                                           │
               │                                           │
┌───────────────────────────────┐                          │
│ money_products.               │          ┌─────────────────────────────────────┐
│ order_transactions_payments_  │          │ sdp-prd-cti-data.intermediate.      │
│ summary                       │          │ shop_chargeback_rates_current       │
│                               │          │                                     │
│ - shop_id                     │◄─────────┤ - shop_id                           │
│ - order_transaction_id        │          │ - chargeback_count                  │
│ - order_id                    │          │ - chargeback_rate                   │
│ - order_transaction_created_at  │        │ - transaction_count                 │
│ - _extracted_at               │          │ - last_updated_at                   │
│ - status                      │          └─────────────────────────────────────┘
│ - amount                      │
│ - currency                    │
└───────────────────────────────┘
```

**Key Join Pattern:**
```sql
-- Joining transaction data with chargebacks
-- This query connects transactions with any associated chargebacks
-- to analyze disputed transactions
SELECT 
  t.shop_id,                                         -- Shop identifier
  t.order_id,                                        -- Order identifier
  t.order_transaction_id,                            -- Transaction identifier
  t.amount_local AS transaction_amount,              -- Original transaction amount
  c.chargeback_amount,                               -- Amount disputed in chargeback
  c.reason AS chargeback_reason                      -- Reason code for the chargeback
FROM `shopify-dw.money_products.order_transactions_payments_summary` t
LEFT JOIN `shopify-dw.money_products.chargebacks_summary` c 
  ON t.order_transaction_id = c.order_transaction_id -- Join on transaction ID
WHERE t._extracted_at >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 7 DAY)  -- Recent data only
  AND t.order_transaction_created_at >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 90 DAY)  -- Last 90 days of transactions
  AND (c.provider_chargeback_created_at >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 90 DAY) 
       OR c.provider_chargeback_created_at IS NULL)  -- Include recent chargebacks or transactions without chargebacks
```

### 2. Financial Data Relationships

```
┌───────────────────────────────┐          ┌─────────────────────────────────────┐
│ finance.                      │          │ intermediate.                        │
│ shop_gmv_current              │          │ shop_gmv_daily_summary_v1_1         │
│                               │          │                                     │
│ - shop_id                     │◄─────────┤ - shop_id                           │
│ - currency                    │          │ - date                              │
│ - gmv_usd_l7d                 │          │ - gmv_usd                           │
│ - gmv_usd_l28d                │          │ - gpv_usd                           │
│ - gpv_usd_l7d                 │          │ - order_count                       │
│ - gpv_usd_l28d                │          │                                     │
│                               │          └─────────────────────────────────────┘
└───────────────────────────────┘                          ▲
               ▲                                           │
               │                                           │
               │                                           │
┌───────────────────────────────┐          ┌─────────────────────────────────────┐
│ money_products.               │          │ base.                               │
│ shopify_payments_balance_     │          │ base__shopify_payments_reserve_     │
│ account_daily_cumulative_     │          │ configurations                      │
│ summary                       │          │                                     │
│                               │          │ - shop_id                           │
│ - shop_id                     │◄─────────┤ - reserve_type                      │
│ - date                        │          │ - percentage                        │
│ - balance                     │          │ - period_in_days                    │
│ - currency                    │          │ - created_at                        │
└───────────────────────────────┘          │ - updated_at                        │
                                           └─────────────────────────────────────┘
```

**Key Join Pattern:**
```sql
-- Joining daily GPV with balance data
-- This query connects shop GMV/GPV data with their Shopify Payments balance
-- to analyze financial activity and available funds
SELECT 
  g.shop_id,                           -- Shop identifier
  g.date,                              -- Date of activity
  g.gmv_usd AS gpv,                    -- Gross payment volume in USD
  SUM(b.cumulative_net_usd_sum) AS balance  -- Consolidated balance across all payment accounts
FROM `shopify-dw.intermediate.shop_gmv_daily_summary_v1_1` g
LEFT JOIN (
  -- Subquery to get shop balances by joining provider accounts to balance data
  SELECT
    sppa.shop_id,
    sp.date,
    sp.cumulative_net_usd_sum
  FROM
    `shopify-dw.money_products.shopify_payments_balance_account_daily_cumulative_summary` AS sp
    LEFT JOIN `shopify-dw.money_products.shopify_payments_provider_accounts` AS sppa
      ON sp.remote_account_id = sppa.remote_account_id
) b
  ON g.shop_id = b.shop_id
  AND g.date = b.date
WHERE g.date >= DATE_SUB(CURRENT_DATE(), INTERVAL 90 DAY)  -- Limiting to past 90 days
GROUP BY g.shop_id, g.date, g.gmv_usd  -- Group to account for multiple payment accounts per shop
```

### 3. Shop Status and Risk Relationships

```
┌───────────────────────────────┐          ┌─────────────────────────────────────┐
│ sdp-prd-cti-data.intermediate.│          │ shopify-dw.risk.                    │
│ shop_current_shopify_payments_│          │ trust_platform_tickets_summary_v1   │
│ status                        │          │                                     │
│                               │          │ - ticket_id                         │
│ - shop_id                     │◄─────────┤ - shop_id                           │
│ - shopify_payments_status     │          │ - created_at                        │
│ - account_active              │          │ - ticket_status                     │
│ - last_updated_at             │          │ - ticket_type                       │
│ - disabled_reason             │          │ - risk_level                        │
└───────────────────────────────┘          │ - trust_subject                     │
               ▲                           └─────────────────────────────────────┘
               │                                           ▲
               │                                           │
               │                                           │
┌───────────────────────────────┐          ┌─────────────────────────────────────┐
│ sdp-prd-cti-data.intermediate.│          │ base.                               │
│ shop_chargeback_rates_current │          │ base__shopify_payments_reserve_     │
│                               │          │ configurations                      │
│ - shop_id                     │◄─────────┤ - shop_id                           │
│ - chargeback_count            │          │ - reserve_type                      │
│ - chargeback_rate             │          │ - percentage                        │
│ - transaction_count           │          │ - period_in_days                    │
│ - last_updated_at             │          └─────────────────────────────────────┘
└───────────────────────────────┘
```

**Key Join Pattern:**
```sql
-- Joining shop status with chargeback data and Trust tickets
-- This query creates a comprehensive risk profile by combining
-- shop payment status, chargeback metrics, and risk investigations
SELECT 
  s.shop_id,                                    -- Shop identifier
  s.shopify_payments_status,                    -- Current Shopify Payments status
  s.account_active,                             -- Whether the account is active
  c.chargeback_rate AS chargeback_rate_90d,     -- Chargeback rate for the past 90 days
  COUNT(t.ticket_id) AS ticket_count            -- Count of Trust Platform tickets
FROM `sdp-prd-cti-data.intermediate.shop_current_shopify_payments_status` s
LEFT JOIN `sdp-prd-cti-data.intermediate.shop_chargeback_rates_current` c
  ON s.shop_id = c.shop_id                      -- Join on shop_id for chargeback data
LEFT JOIN `shopify-dw.risk.trust_platform_tickets_summary_v1` t
  ON s.shop_id = t.shop_id                      -- Join on shop_id for ticket data
  AND t.subjectable_type = 'Shop'               -- Ensure tickets are for shops
  AND t.created_at >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 90 DAY)  -- Only recent tickets
WHERE s.account_active = TRUE                   -- Only active accounts
GROUP BY s.shop_id, s.shopify_payments_status, s.account_active, c.chargeback_rate
```

## Domain-Specific Table Groups

### Chargeback Domain

| Table | Purpose | Join Fields | Partition Key |
|-------|---------|------------|--------------|
| `shopify-dw.money_products.chargebacks_summary` | Primary chargeback data | `shop_id`, `order_id`, `order_transaction_id` | `provider_chargeback_created_at` |
| `sdp-prd-cti-data.intermediate.shop_chargeback_rates_daily_snapshot` | Daily snapshots of chargeback rates | `shop_id`, `date` | `date` |
| `sdp-prd-cti-data.intermediate.shop_chargeback_rates_current` | Current chargeback rates | `shop_id` | N/A (view) |

### Financial Domain

| Table | Purpose | Join Fields | Partition Key |
|-------|---------|------------|--------------|
| `shopify-dw.finance.shop_gmv_current` | Current GMV/GPV metrics | `shop_id`, `currency` | N/A (table) |
| `shopify-dw.intermediate.shop_gmv_daily_summary_v1_1` | Daily GMV/GPV data | `shop_id`, `date` | `date` |
| `shopify-dw.money_products.shopify_payments_balance_account_daily_cumulative_summary` | Balance information | `remote_account_id`, `date`, `currency_code` | `date` |
| `shopify-dw.base.base__payments_refunds` | Refund data | `shop_id`, `order_transaction_id` | Not verified |
| `shopify-dw.money_products.order_transactions_payments_summary` | Transaction data | `shop_id`, `order_id`, `order_transaction_id` | `_extracted_at` |

### Shop Domain

| Table | Purpose | Join Fields | Partition Key |
|-------|---------|------------|--------------|
| `sdp-prd-cti-data.intermediate.shop_current_shopify_payments_status` | Current shop status | `shop_id` | N/A (view) |

### Risk Domain

| Table | Purpose | Join Fields | Partition Key |
|-------|---------|------------|--------------|
| `shopify-dw.risk.trust_platform_tickets_summary_v1` | Risk investigation tickets | `shop_id`, `ticket_id` | Not verified |

### Reserve Domain

| Table | Purpose | Join Fields | Partition Key |
|-------|---------|------------|--------------|
| `sdp-prd-cti-data.base.base__shopify_payments_reserve_configurations` | Reserve settings | `shop_id` | Not verified |

## Common Join Patterns

### 1. Comprehensive Merchant Risk Profile

```sql
-- Comprehensive merchant risk profile query
-- This query combines multiple data sources to create a complete
-- risk profile for merchants with sufficient transaction volume
WITH merchant_status AS (
  -- Get current merchant status information
  SELECT 
    shop_id,
    shopify_payments_status,
    account_active,
    disabled_reason
  FROM `sdp-prd-cti-data.intermediate.shop_current_shopify_payments_status`
  WHERE account_active = TRUE  -- Only include active merchants
),

merchant_risk AS (
  -- Get chargeback metrics for risk evaluation
  SELECT 
    shop_id,
    chargeback_rate AS chargeback_rate_90d,
    transaction_count AS sp_transaction_count_90d
  FROM `sdp-prd-cti-data.intermediate.shop_chargeback_rates_current`
),

merchant_financial AS (
  -- Get financial performance metrics
  SELECT 
    shop_id,
    gmv_usd_l28d
  FROM `shopify-dw.finance.shop_gmv_current`
),

merchant_reserves AS (
  -- Get reserve configuration details
  SELECT
    shop_id,
    reserve_type,
    percentage,
    period_in_days
  FROM `sdp-prd-cti-data.base.base__shopify_payments_reserve_configurations`
  WHERE is_active = TRUE  -- Only include active reserves
),

merchant_tickets AS (
  -- Get open risk investigation tickets
  SELECT
    shop_id,
    COUNT(*) AS open_ticket_count
  FROM `shopify-dw.risk.trust_platform_tickets_summary_v1`
  WHERE ticket_status = 'OPEN'
  AND created_at >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 90 DAY)
  GROUP BY shop_id
)

-- Join all data together for a comprehensive view
SELECT
  s.shop_id,
  s.shopify_payments_status,
  r.chargeback_rate_90d,
  r.sp_transaction_count_90d,
  f.gmv_usd_l28d,
  COALESCE(t.open_ticket_count, 0) AS open_ticket_count,  -- Handle shops with no tickets
  rsv.reserve_type,
  rsv.percentage AS reserve_percentage,
  rsv.period_in_days AS reserve_period
FROM merchant_status s
LEFT JOIN merchant_risk r ON s.shop_id = r.shop_id
LEFT JOIN merchant_financial f ON s.shop_id = f.shop_id
LEFT JOIN merchant_tickets t ON s.shop_id = t.shop_id
LEFT JOIN merchant_reserves rsv ON s.shop_id = rsv.shop_id
WHERE r.sp_transaction_count_90d >= 100  -- Only include merchants with sufficient volume
ORDER BY r.chargeback_rate_90d DESC  -- Sort highest risk first
LIMIT 100
```

### 2. Chargeback to Transaction Ratio Over Time

```sql
-- Time series of chargeback rates
-- This query calculates daily chargeback rates over time
-- to identify trends in disputed transactions
WITH daily_transactions AS (
  -- Get count of successful transactions by day
  SELECT
    shop_id,
    DATE(order_transaction_created_at) AS date,
    COUNT(*) AS transaction_count
  FROM `shopify-dw.money_products.order_transactions_payments_summary`
  WHERE _extracted_at >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 7 DAY)  -- Recent data only
    AND order_transaction_created_at >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 90 DAY)  -- Last 90 days
    AND order_transaction_status = 'success'  -- Only successful transactions
  GROUP BY shop_id, DATE(order_transaction_created_at)
),

daily_chargebacks AS (
  -- Get count of chargebacks by day
  SELECT
    shop_id,
    DATE(provider_chargeback_created_at) AS date,
    COUNT(*) AS chargeback_count
  FROM `shopify-dw.money_products.chargebacks_summary`
  WHERE provider_chargeback_created_at >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 90 DAY)  -- Last 90 days
  GROUP BY shop_id, DATE(provider_chargeback_created_at)
)

-- Join and calculate daily rates
SELECT
  t.shop_id,
  t.date,
  t.transaction_count,
  COALESCE(c.chargeback_count, 0) AS chargeback_count,  -- Handle days with no chargebacks
  SAFE_DIVIDE(COALESCE(c.chargeback_count, 0), t.transaction_count) AS daily_chargeback_rate  -- Safe division
FROM daily_transactions t
LEFT JOIN daily_chargebacks c 
  ON t.shop_id = c.shop_id 
  AND t.date = c.date
ORDER BY t.shop_id, t.date
```

### 3. Reserve to Balance Relationship

```sql
-- Reserve impact on merchant balance
-- This query analyzes how reserves affect merchant available balances
-- by comparing GPV to reserve percentage and actual balance
WITH merchant_reserves AS (
  -- Get current reserve configurations
  SELECT
    shop_id,
    reserve_type,
    percentage
  FROM `sdp-prd-cti-data.base.base__shopify_payments_reserve_configurations`
  WHERE is_active = TRUE  -- Only active reserves
),

merchant_balances AS (
  -- Get daily balance information
  SELECT
    sppa.shop_id,
    sp.date,
    SUM(sp.cumulative_net_usd_sum) AS balance
  FROM `shopify-dw.money_products.shopify_payments_balance_account_daily_cumulative_summary` sp
  LEFT JOIN `shopify-dw.money_products.shopify_payments_provider_accounts` sppa
    ON sp.remote_account_id = sppa.remote_account_id
  WHERE sp.date >= DATE_SUB(CURRENT_DATE(), INTERVAL 30 DAY)
    AND sp.currency_code = 'USD'  -- Standardize on USD
  GROUP BY sppa.shop_id, sp.date
),

merchant_gpv AS (
  -- Get daily GPV metrics
  SELECT
    shop_id,
    date,
    gmv_usd AS gpv  -- Using GMV as approximate GPV for this example
  FROM `shopify-dw.intermediate.shop_gmv_daily_summary_v1_1`
  WHERE date >= DATE_SUB(CURRENT_DATE(), INTERVAL 30 DAY)
)

-- Join data and calculate estimated reserve amount
SELECT
  b.shop_id,
  b.date,
  b.balance,
  g.gpv,
  r.reserve_type,
  r.percentage,
  CASE
    WHEN r.reserve_type = 'percentage' THEN g.gpv * r.percentage / 100
    ELSE NULL
  END AS estimated_reserve_amount  -- Calculate estimated reserve based on configuration
FROM merchant_balances b
LEFT JOIN merchant_gpv g ON b.shop_id = g.shop_id AND b.date = g.date
LEFT JOIN merchant_reserves r ON b.shop_id = r.shop_id
ORDER BY b.shop_id, b.date
```

### Union Examples

Sometimes you need to combine records from different tables that have similar structures:

```sql
-- This query unifies data from multiple sources to create a consolidated view
-- of different risk indicators across the merchant base
WITH shop_data AS (
  -- Get shops with chargebacks
  SELECT 
    shop_id,
    'Has Chargebacks' as category,
    COUNT(*) as count_value
  FROM `shopify-dw.money_products.chargebacks_summary`
  WHERE provider_chargeback_created_at >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 90 DAY)
  GROUP BY shop_id
  
  UNION ALL
  
  -- Get shops with refunds
  SELECT 
    shop_id,
    'Has Refunds' as category,
    COUNT(*) as count_value
  FROM `shopify-dw.base.base__payments_refunds`
  WHERE created_at >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 90 DAY)
  GROUP BY shop_id
  
  UNION ALL
  
  -- Get shops with trust platform cases
  SELECT 
    shop_id,
    'Has TP Cases' as category,
    COUNT(*) as count_value
  FROM `shopify-dw.risk.trust_platform_tickets_summary_v1`
  WHERE created_at >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 90 DAY)
  GROUP BY shop_id
)

-- Aggregate results by category to compare prevalence
SELECT
  category,
  COUNT(DISTINCT shop_id) as unique_shop_count,
  SUM(count_value) as total_count,
  ROUND(SUM(count_value) / COUNT(DISTINCT shop_id), 1) as avg_per_shop
FROM shop_data
GROUP BY category
ORDER BY unique_shop_count DESC
```

### Many-to-Many Shop Relationships

This example shows how to navigate complex many-to-many relationships:

```sql
-- This query analyzes shops connected through partners
-- to identify multi-shop relationships and partner networks
WITH shop_partners AS (
  -- Get all partners for each shop
  SELECT 
    shop_id,
    ARRAY_AGG(DISTINCT partner_id) as partner_ids,
    COUNT(DISTINCT partner_id) as partner_count
  FROM `shopify-dw.shops.shop_partner_relationships`
  WHERE is_active = TRUE
  GROUP BY shop_id
),

partner_shops AS (
  -- Get shop count per partner
  SELECT
    partner_id,
    COUNT(DISTINCT shop_id) as shop_count
  FROM `shopify-dw.shops.shop_partner_relationships`
  WHERE is_active = TRUE
  GROUP BY partner_id
)

-- Join shop and partner data to create relationship network
SELECT
  s.shop_id,
  s.shop_name,
  sp.partner_ids,
  sp.partner_count,
  -- Create array of partner details with their shop counts
  ARRAY_AGG(STRUCT(p.partner_id, p.partner_name, ps.shop_count as partner_shop_count)) as partner_details
FROM `sdp-prd-cti-data.intermediate.shop_insights_shops` s
JOIN shop_partners sp ON s.shop_id = sp.shop_id
JOIN `shopify-dw.shops.shop_partner_relationships` r ON s.shop_id = r.shop_id
JOIN `shopify-dw.shops.partners` p ON r.partner_id = p.partner_id
JOIN partner_shops ps ON p.partner_id = ps.partner_id
WHERE r.is_active = TRUE
GROUP BY s.shop_id, s.shop_name, sp.partner_ids, sp.partner_count
HAVING sp.partner_count > 1  -- Only shops with multiple partners
ORDER BY sp.partner_count DESC
LIMIT 100
```

### Temporal Table Relationships

This example shows how to join tables with different temporal dimensions:

```sql
-- This query combines current snapshot metrics with daily historical data
-- to provide trending analysis for shop performance
WITH shop_daily_data AS (
  -- Get daily metrics for the past month
  SELECT
    shop_id,
    date,
    SUM(gmv_usd) as daily_gmv_usd,
    SUM(gross_orders_with_gmv) as daily_order_count
  FROM `shopify-dw.intermediate.shop_gmv_daily_summary_v1_1`
  WHERE date >= DATE_SUB(CURRENT_DATE(), INTERVAL 30 DAY)
  GROUP BY shop_id, date
),

shop_current_data AS (
  -- Get current rolling period metrics
  SELECT
    shop_id,
    gmv_usd_l28d,
    gross_orders_with_gmv_l28d as order_count_l28d,
    gmv_usd_l7d,
    gross_orders_with_gmv_l7d as order_count_l7d
  FROM `shopify-dw.finance.shop_gmv_current`
)

-- Join current and historical data
SELECT
  s.shop_id,
  s.shop_name,
  sc.gmv_usd_l28d,
  sc.order_count_l28d,
  -- Create array of daily stats for time series analysis
  ARRAY_AGG(STRUCT(d.date, d.daily_order_count, d.daily_gmv_usd) ORDER BY d.date DESC) as daily_stats
FROM `sdp-prd-cti-data.intermediate.shop_insights_shops` s
JOIN shop_current_data sc ON s.shop_id = sc.shop_id
LEFT JOIN shop_daily_data d ON s.shop_id = d.shop_id
WHERE sc.order_count_l28d > 0  -- Only shops with recent orders
GROUP BY s.shop_id, s.shop_name, sc.gmv_usd_l28d, sc.order_count_l28d
ORDER BY sc.order_count_l28d DESC
LIMIT 50
```

### Summary Statistics by Join

Joining tables to create aggregated statistics:

```sql
-- This query analyzes payment method distribution and chargeback rates
-- to identify correlations between payment types and dispute frequencies
WITH shop_summary AS (
  -- Get average chargeback metrics for shops with sufficient data
  SELECT
    shop_id,
    AVG(chargeback_count_180d) as avg_cb_count,
    AVG(chargeback_rate_180d) as avg_cb_rate,
    COUNT(*) as data_points
  FROM `sdp-prd-cti-data.intermediate.shop_chargeback_rates_daily_snapshot`
  WHERE date >= DATE_SUB(CURRENT_DATE(), INTERVAL 90 DAY)
  GROUP BY shop_id
  HAVING COUNT(*) >= 30  -- Only include shops with at least 30 days of data
),

shop_payments AS (
  -- Get payment method distribution for each shop
  SELECT
    shop_id,
    SUM(CASE WHEN payment_method = 'credit_card' THEN count ELSE 0 END) as cc_count,
    SUM(CASE WHEN payment_method = 'paypal' THEN count ELSE 0 END) as paypal_count,
    SUM(CASE WHEN payment_method = 'bank' THEN count ELSE 0 END) as bank_count,
    SUM(count) as total_count
  FROM (
    -- Subquery to get payment method counts
    SELECT
      shop_id,
      payment_method_name as payment_method,
      COUNT(*) as count
    FROM `shopify-dw.money_products.order_transactions_payments_summary`
    WHERE order_transaction_created_at >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 90 DAY)
    GROUP BY shop_id, payment_method_name
  )
  GROUP BY shop_id
)

-- Join summary data with payment distribution metrics
SELECT
  s.shop_id,
  s.shop_name,
  ss.avg_cb_count,
  ss.avg_cb_rate,
  sp.cc_count,
  sp.paypal_count,
  sp.bank_count,
  sp.total_count,
  ROUND(sp.cc_count / sp.total_count, 3) as cc_percentage,
  ROUND(sp.paypal_count / sp.total_count, 3) as paypal_percentage,
  ROUND(sp.bank_count / sp.total_count, 3) as bank_percentage
FROM `sdp-prd-cti-data.intermediate.shop_insights_shops` s
JOIN shop_summary ss ON s.shop_id = ss.shop_id
JOIN shop_payments sp ON s.shop_id = sp.shop_id
WHERE ss.avg_cb_rate > 0.005  -- Focus on shops with significant chargeback rates
ORDER BY ss.avg_cb_rate DESC
LIMIT 100
```

### Multi-Hop Relationships

This example demonstrates navigating through multiple relationship hops:

```sql
-- This query finds connections between Trust Platform tickets and chargebacks
-- to identify potential relationships between risk events
WITH ticket_data AS (
  -- Get relevant Trust Platform tickets
  SELECT
    ticket_id,
    shop_id,
    created_at,
    ticket_type,
    ticket_status,
    ticket_severity,
    resolution_type
  FROM `shopify-dw.risk.trust_platform_tickets_summary_v1`
  WHERE created_at >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 180 DAY)
    AND (
      ticket_type LIKE '%chargeback%' OR
      ticket_type LIKE '%dispute%' OR
      ticket_type LIKE '%fraud%'
    )
),

chargeback_data AS (
  -- Get chargeback details
  SELECT
    shop_id,
    order_transaction_id as transaction_id,
    order_id,
    provider_chargeback_created_at as created_at,
    evidence_due_by as disputed_at,
    reason as dispute_reason,
    status as dispute_status,
    is_rapid_dispute_resolution as chargeback_protected
  FROM `shopify-dw.money_products.chargebacks_summary`
  WHERE provider_chargeback_created_at >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 180 DAY)
)

-- Join ticket and chargeback data with transaction details
-- to find temporal relationships within 72 hours
SELECT
  t.shop_id,
  s.shop_name,
  t.ticket_id,
  t.ticket_type,
  t.ticket_status,
  t.ticket_severity,
  c.transaction_id,
  c.order_id,
  c.dispute_reason,
  c.dispute_status,
  c.chargeback_protected,
  o.payment_method_name as payment_method,
  TIMESTAMP_DIFF(t.created_at, c.created_at, HOUR) as hours_between_chargeback_and_ticket
FROM ticket_data t
JOIN chargeback_data c 
  ON t.shop_id = c.shop_id
  AND ABS(TIMESTAMP_DIFF(t.created_at, c.created_at, HOUR)) <= 72  -- Events within 72 hours
JOIN `shopify-dw.money_products.order_transactions_payments_summary` o
  ON c.transaction_id = o.order_transaction_id
JOIN `sdp-prd-cti-data.intermediate.shop_insights_shops` s
  ON t.shop_id = s.shop_id
ORDER BY ABS(TIMESTAMP_DIFF(t.created_at, c.created_at, HOUR))  -- Sort by closest temporal relationship
LIMIT 100
```

## Table Update Frequency

| Table | Update Frequency | Lag Time |
|-------|------------------|----------|
| `shopify-dw.money_products.chargebacks_summary` | Daily | 1 day |
| `sdp-prd-cti-data.intermediate.shop_chargeback_rates_daily_snapshot` | Daily | 1 day |
| `sdp-prd-cti-data.intermediate.shop_chargeback_rates_current` | Daily | 1 day |
| `shopify-dw.finance.shop_gmv_current` | Daily | 1 day |
| `shopify-dw.intermediate.shop_gmv_daily_summary_v1_1` | Daily | 1 day |
| `shopify-dw.money_products.shopify_payments_balance_account_daily_cumulative_summary` | Daily | 1 day |
| `sdp-prd-cti-data.intermediate.shop_current_shopify_payments_status` | Daily | 1 day |
| `shopify-dw.risk.trust_platform_tickets_summary_v1` | Hourly | 1 hour |
| `sdp-prd-cti-data.base.base__shopify_payments_reserve_configurations` | Real-time | Near real-time |

## Best Practices for Table Joins

1. **Always join on complete keys** - Include all parts of composite keys in joins
2. **Filter before joining** - Apply filters to individual tables before joining
3. **Be aware of data freshness** - Consider update frequencies when joining tables
4. **Handle missing data** - Use appropriate outer joins and COALESCE functions
5. **Check join cardinality** - Be aware of one-to-many relationships
6. **Use date alignment** - When joining time-series data, ensure dates are aligned
7. **Consider currency** - Many financial tables require joining on currency as well as shop_id
8. **Include partition field filters** - Always filter on partition fields in WHERE clauses

## Temporal Fields Reference

| Table | Temporal Field | Data Type | Filter Function Example |
|-------|---------------|-----------|------------------------|
| `money_products.chargebacks_summary` | `provider_chargeback_created_at` | TIMESTAMP | `WHERE provider_chargeback_created_at >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 90 DAY)` |
| `money_products.order_transactions_payments_summary` | `order_transaction_created_at` | TIMESTAMP | `WHERE order_transaction_created_at >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 90 DAY)` |
| `money_products.order_transactions_payments_summary` | `_extracted_at` | TIMESTAMP | `WHERE _extracted_at >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 7 DAY)` |
| `intermediate.shop_gmv_daily_summary_v1_1` | `date` | DATE | `WHERE date >= DATE_SUB(CURRENT_DATE(), INTERVAL 90 DAY)` |
| `risk.trust_platform_tickets_summary_v1` | `created_at` | TIMESTAMP | `WHERE created_at >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 90 DAY)` |