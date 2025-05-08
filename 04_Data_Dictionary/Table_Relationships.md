# Table Relationships

This document outlines how key tables in the credit risk data ecosystem relate to each other, providing relationship diagrams, join fields, and example join patterns.

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
| Transactions | `transaction_id` | Links payment transactions to other events |
| Chargebacks | `chargeback_id`, `order_id`, `transaction_id` | Connected to transactions and orders |
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
│ - shop_id                     │          │ - snapshot_date                     │
│ - order_id                    │          │ - chargeback_count                  │
│ - transaction_id              │          │ - chargeback_rate                   │
│ - created_at                  │          │ - transaction_count                 │
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
│ - transaction_id              │          │ - chargeback_count                  │
│ - order_id                    │          │ - chargeback_rate                   │
│ - created_at                  │          │ - transaction_count                 │
│ - status                      │          │ - last_updated_at                   │
│ - amount                      │          └─────────────────────────────────────┘
│ - currency                    │
└───────────────────────────────┘
```

**Key Join Pattern:**
```sql
-- Joining transaction data with chargebacks
SELECT 
  t.shop_id,
  t.order_id,
  t.transaction_id,
  t.amount AS transaction_amount,
  c.amount AS chargeback_amount,
  c.reason AS chargeback_reason
FROM `shopify-dw.money_products.order_transactions_payments_summary` t
LEFT JOIN `shopify-dw.money_products.chargebacks_summary` c 
  ON t.transaction_id = c.transaction_id
WHERE t.created_at >= DATE_SUB(CURRENT_DATE(), INTERVAL 90 DAY)
```

### 2. Financial Data Relationships

```
┌───────────────────────────────┐          ┌─────────────────────────────────────┐
│ finance.                      │          │ finance.                            │
│ shop_gmv_current              │          │ shop_gmv_daily_summary_v1_1         │
│                               │          │                                     │
│ - shop_id                     │◄─────────┤ - shop_id                           │
│ - currency                    │          │ - date                              │
│ - gmv_1d                      │          │ - currency                          │
│ - gmv_7d                      │          │ - gmv                               │
│ - gmv_28d                     │          │ - gpv                               │
│ - gpv_1d                      │          │ - order_count                       │
│ - gpv_7d                      │          └─────────────────────────────────────┘
│ - gpv_28d                     │                          ▲
└───────────────────────────────┘                          │
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
SELECT 
  g.shop_id,
  g.date,
  g.gpv,
  b.balance
FROM `shopify-dw.finance.shop_gmv_daily_summary_v1_1` g
LEFT JOIN `shopify-dw.money_products.shopify_payments_balance_account_daily_cumulative_summary` b
  ON g.shop_id = b.shop_id
  AND g.date = b.date
  AND g.currency = b.currency
WHERE g.date >= DATE_SUB(CURRENT_DATE(), INTERVAL 90 DAY)
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
SELECT 
  s.shop_id,
  s.shopify_payments_status,
  s.account_active,
  c.chargeback_rate,
  COUNT(t.ticket_id) AS ticket_count
FROM `sdp-prd-cti-data.intermediate.shop_current_shopify_payments_status` s
LEFT JOIN `sdp-prd-cti-data.intermediate.shop_chargeback_rates_current` c
  ON s.shop_id = c.shop_id
LEFT JOIN `shopify-dw.risk.trust_platform_tickets_summary_v1` t
  ON s.shop_id = t.shop_id
  AND t.created_at >= DATE_SUB(CURRENT_DATE(), INTERVAL 90 DAY)
WHERE s.account_active = TRUE
GROUP BY s.shop_id, s.shopify_payments_status, s.account_active, c.chargeback_rate
```

## Domain-Specific Table Groups

### Chargeback Domain

| Table | Purpose | Join Fields |
|-------|---------|------------|
| `shopify-dw.money_products.chargebacks_summary` | Primary chargeback data | `shop_id`, `order_id`, `transaction_id` |
| `sdp-prd-cti-data.intermediate.shop_chargeback_rates_daily_snapshot` | Daily snapshots of chargeback rates | `shop_id`, `snapshot_date` |
| `sdp-prd-cti-data.intermediate.shop_chargeback_rates_current` | Current chargeback rates | `shop_id` |

### Financial Domain

| Table | Purpose | Join Fields |
|-------|---------|------------|
| `shopify-dw.finance.shop_gmv_current` | Current GMV/GPV metrics | `shop_id`, `currency` |
| `shopify-dw.finance.shop_gmv_daily_summary_v1_1` | Daily GMV/GPV data | `shop_id`, `date`, `currency` |
| `shopify-dw.money_products.shopify_payments_balance_account_daily_cumulative_summary` | Balance information | `shop_id`, `date`, `currency` |
| `shopify-dw.raw_shopify.payments_refunds` | Refund data | `shop_id`, `order_id`, `transaction_id` |
| `shopify-dw.money_products.order_transactions_payments_summary` | Transaction data | `shop_id`, `order_id`, `transaction_id` |

### Shop Domain

| Table | Purpose | Join Fields |
|-------|---------|------------|
| `sdp-prd-cti-data.intermediate.shop_current_shopify_payments_status` | Current shop status | `shop_id` |

### Risk Domain

| Table | Purpose | Join Fields |
|-------|---------|------------|
| `shopify-dw.risk.trust_platform_tickets_summary_v1` | Risk investigation tickets | `shop_id`, `ticket_id` |

### Reserve Domain

| Table | Purpose | Join Fields |
|-------|---------|------------|
| `sdp-prd-cti-data.base.base__shopify_payments_reserve_configurations` | Reserve settings | `shop_id` |

## Common Join Patterns

### 1. Comprehensive Merchant Risk Profile

```sql
-- Comprehensive merchant risk profile query
WITH merchant_status AS (
  SELECT 
    shop_id,
    shopify_payments_status,
    account_active,
    disabled_reason
  FROM `sdp-prd-cti-data.intermediate.shop_current_shopify_payments_status`
  WHERE account_active = TRUE
),

merchant_risk AS (
  SELECT 
    shop_id,
    chargeback_rate,
    transaction_count
  FROM `sdp-prd-cti-data.intermediate.shop_chargeback_rates_current`
),

merchant_financial AS (
  SELECT 
    shop_id,
    gpv_28d
  FROM `shopify-dw.finance.shop_gmv_current`
  WHERE currency = 'USD'
),

merchant_reserves AS (
  SELECT
    shop_id,
    reserve_type,
    percentage,
    period_in_days
  FROM `sdp-prd-cti-data.base.base__shopify_payments_reserve_configurations`
  WHERE is_active = TRUE
),

merchant_tickets AS (
  SELECT
    shop_id,
    COUNT(*) AS open_ticket_count
  FROM `shopify-dw.risk.trust_platform_tickets_summary_v1`
  WHERE ticket_status = 'OPEN'
  GROUP BY shop_id
)

SELECT
  s.shop_id,
  s.shopify_payments_status,
  r.chargeback_rate,
  r.transaction_count,
  f.gpv_28d,
  COALESCE(t.open_ticket_count, 0) AS open_ticket_count,
  rsv.reserve_type,
  rsv.percentage AS reserve_percentage,
  rsv.period_in_days AS reserve_period
FROM merchant_status s
LEFT JOIN merchant_risk r ON s.shop_id = r.shop_id
LEFT JOIN merchant_financial f ON s.shop_id = f.shop_id
LEFT JOIN merchant_tickets t ON s.shop_id = t.shop_id
LEFT JOIN merchant_reserves rsv ON s.shop_id = rsv.shop_id
WHERE r.transaction_count >= 100  -- Only include merchants with sufficient volume
ORDER BY r.chargeback_rate DESC
LIMIT 100
```

### 2. Chargeback to Transaction Ratio Over Time

```sql
-- Time series of chargeback rates
WITH daily_transactions AS (
  SELECT
    shop_id,
    DATE(created_at) AS date,
    COUNT(*) AS transaction_count
  FROM `shopify-dw.money_products.order_transactions_payments_summary`
  WHERE created_at >= DATE_SUB(CURRENT_DATE(), INTERVAL 90 DAY)
    AND status = 'success'
  GROUP BY shop_id, DATE(created_at)
),

daily_chargebacks AS (
  SELECT
    shop_id,
    DATE(created_at) AS date,
    COUNT(*) AS chargeback_count
  FROM `shopify-dw.money_products.chargebacks_summary`
  WHERE created_at >= DATE_SUB(CURRENT_DATE(), INTERVAL 90 DAY)
  GROUP BY shop_id, DATE(created_at)
)

SELECT
  t.shop_id,
  t.date,
  t.transaction_count,
  COALESCE(c.chargeback_count, 0) AS chargeback_count,
  SAFE_DIVIDE(COALESCE(c.chargeback_count, 0), t.transaction_count) AS daily_chargeback_rate
FROM daily_transactions t
LEFT JOIN daily_chargebacks c 
  ON t.shop_id = c.shop_id 
  AND t.date = c.date
ORDER BY t.shop_id, t.date
```

### 3. Reserve to Balance Relationship

```sql
-- Reserve impact on merchant balance
WITH merchant_reserves AS (
  SELECT
    shop_id,
    reserve_type,
    percentage
  FROM `sdp-prd-cti-data.base.base__shopify_payments_reserve_configurations`
  WHERE is_active = TRUE
),

merchant_balances AS (
  SELECT
    shop_id,
    date,
    balance
  FROM `shopify-dw.money_products.shopify_payments_balance_account_daily_cumulative_summary`
  WHERE date >= DATE_SUB(CURRENT_DATE(), INTERVAL 30 DAY)
    AND currency = 'USD'
),

merchant_gpv AS (
  SELECT
    shop_id,
    date,
    gpv
  FROM `shopify-dw.finance.shop_gmv_daily_summary_v1_1`
  WHERE date >= DATE_SUB(CURRENT_DATE(), INTERVAL 30 DAY)
    AND currency = 'USD'
)

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
  END AS estimated_reserve_amount
FROM merchant_balances b
LEFT JOIN merchant_gpv g ON b.shop_id = g.shop_id AND b.date = g.date
LEFT JOIN merchant_reserves r ON b.shop_id = r.shop_id
ORDER BY b.shop_id, b.date
```

## Table Update Frequency

| Table | Update Frequency | Lag Time |
|-------|------------------|----------|
| `shopify-dw.money_products.chargebacks_summary` | Daily | 1 day |
| `sdp-prd-cti-data.intermediate.shop_chargeback_rates_daily_snapshot` | Daily | 1 day |
| `sdp-prd-cti-data.intermediate.shop_chargeback_rates_current` | Daily | 1 day |
| `shopify-dw.finance.shop_gmv_current` | Daily | 1 day |
| `shopify-dw.finance.shop_gmv_daily_summary_v1_1` | Daily | 1 day |
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

---

*Last Updated: May 2024* 