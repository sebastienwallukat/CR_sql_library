# Credit Risk Data Tables Overview

This document provides a comprehensive overview of all data tables used by the Credit Risk team. Tables are organized by domain to help you quickly find the relevant data for your analysis.

## Table Navigation

- [Shop Data Tables](#shop-data-tables)
- [Payment Data Tables](#payment-data-tables)
- [Chargeback and Dispute Tables](#chargeback-and-dispute-tables)
- [Trust Platform Tables](#trust-platform-tables)
- [Financial Data Tables](#financial-data-tables)
- [Risk Assessment Tables](#risk-assessment-tables)
- [Miscellaneous Tables](#miscellaneous-tables)

## Table of Contents

- [Chargeback and Dispute Tables](#chargeback-and-dispute-tables)
- [Risk Assessment Tables](#risk-assessment-tables)
- [Payment Processing Tables](#payment-processing-tables)
- [Shop and Merchant Tables](#shop-and-merchant-tables)
- [Order and Transaction Tables](#order-and-transaction-tables)
- [Financial Data Tables](#financial-data-tables)

## Shop Data Tables

| Table Name | Description | Primary Keys | Data Types | Refresh Frequency |
|------------|-------------|--------------|------------|-------------------|
| `sdp-prd-cti-data.intermediate.shop_insights_shops` | Core shop information with trust battery data | shop_id | Various including PII fields | Daily |
| `shopify-dw.mart_core_deliver.shop_back_office_summary_current` | Shop industry and business metadata | shop_id | `shop_id` (INTEGER), `date` (DATE), `shop_created_at` (TIMESTAMP) | Daily |
| `shopify-dw.accounts_and_administration.shop_profile_current` | Shop profile information including country | shop_id | Various, includes TIMESTAMP fields | Daily |
| `shopify-dw.accounts_and_administration.shop_billing_info_current` | Shop billing and subscription plans | shop_id | Various, includes TIMESTAMP fields | Daily |
| `shopify-dw.intermediate.shop_product_reviews_history_v1_0` | Product reviews for shop products | review_id, shop_id | Various, includes TIMESTAMP fields | Daily |
| `sdp-prd-cti-data.intermediate.linksphere_shop_nodes_for_fraud` | Shop fraud risk indicators and metadata | shop_id | Various | Daily |
| `sdp-prd-cti-data.intermediate.linksphere_shop_to_shop` | Links between related shops | origin_shop_id, result_shop_id | Various | Daily |
| `sdp-prd-cti-data.base.base__subscriptions` | Shop subscription plan information | shop_id | Various, includes TIMESTAMP fields | Daily |

## Payment Data Tables

| Table Name | Description | Primary Keys | Data Types | Refresh Frequency |
|------------|-------------|--------------|------------|-------------------|
| `sdp-prd-cti-data.intermediate.shop_daily_order_fraud_facts` | Daily shop payment activity metrics | shop_id, date | `shop_id` (INTEGER), `date` (DATE) | Daily |
| `sdp-prd-cti-data.intermediate.shop_current_shopify_payments_status` | Current shop Shopify Payments status | shop_id | Various | Daily |
| `sdp-prd-cti-data.base.base__payments_balance_transactions` | Payment balance transactions | balance_transaction_id, shop_id | Various, includes TIMESTAMP fields | Hourly |
| `shopify-dw.money_products.shopify_payments_balance_transactions` | Shopify Payments balance transactions | balance_transaction_id, shop_id | Various, includes TIMESTAMP fields | Hourly |
| `sdp-prd-cti-data.base.base__orders` | Order information including fulfillment status | order_id, shop_id | Various, includes TIMESTAMP fields | Hourly |
| `shopify-dw.finance.currency_rate_daily_snapshot` | Currency exchange rates | currency_code, date | `currency_code` (STRING), `date` (DATE) | Daily |
| `sdp-prd-cti-data.base.base__payments_refunds` | Payment refund information | id, shop_id | Various, includes TIMESTAMP fields | Hourly |

## Chargeback and Dispute Tables

Tables for analyzing chargeback data, dispute patterns, and merchant chargeback rates.

| Table Name | Description | Primary Keys | Data Types | Refresh Frequency | Verification Status |
|------------|-------------|--------------|------------|-------------------|---------------------|
| `shopify-dw.money_products.chargebacks_summary` | Comprehensive view of all chargebacks across Shopify Payments | chargeback_id | `chargeback_id` (INTEGER), `provider_chargeback_created_at` (TIMESTAMP), `shopify_chargeback_created_at` (TIMESTAMP) | Daily, partitioned by `provider_chargeback_created_at` | ✓ Verified |
| `sdp-prd-cti-data.base.base__payments_disputes` | Detailed dispute information including chargebacks and inquiries | dispute_id, shop_id, order_id | Various, includes TIMESTAMP fields | Daily | ✓ Verified |
| `sdp-prd-cti-data.intermediate.shop_chargeback_rates_current` | Current chargeback rates for shops with various time windows | shop_id | Various | Daily | ✓ Verified |
| `sdp-prd-cti-data.intermediate.shop_chargeback_rates_daily_snapshot` | Historical daily snapshots of shop chargeback rates | shop_id, snapshot_date | `shop_id` (INTEGER), `snapshot_date` (DATE) | Daily | ✓ Verified |

[View detailed documentation for Chargeback Tables](./Chargeback_Tables.md)

## Trust Platform Tables

| Table Name | Description | Primary Keys | Data Types | Refresh Frequency |
|------------|-------------|--------------|------------|-------------------|
| `shopify-dw.risk.trust_platform_tickets_summary_v1` | Trust Platform ticket information | trust_platform_ticket_id | `trust_platform_ticket_id` (INTEGER), `created_at` (TIMESTAMP) | Hourly |
| `shopify-dw.risk.trust_platform_actions_summary_v1` | Trust Platform actions data | id, actionable_id | Various, includes TIMESTAMP fields | Hourly |
| `shopify-dw.risk.trust_platform_ticket_escalations_v1` | Trust Platform ticket escalations | id, from_trust_platform_ticket_id | Various, includes TIMESTAMP fields | Hourly |
| `sdp-prd-cti-data.intermediate.trust_platform_ticket_deduplication` | Deduplicated ticket information | shop_id, detection_identifier | Various | Daily |
| `sdp-prd-cti-data.intermediate.trust_platform_shop_risk_attributes_daily` | Daily shop risk metrics | shop_id, date | `shop_id` (INTEGER), `date` (DATE) | Daily |
| `sdp-prd-cti-data.base.base__sensitive_reports` | Sensitive reports generated | report_id | Various, includes TIMESTAMP fields | Daily |
| `sdp-prd-cti-data.base.base__tickets` | Base ticket information | signal_id | Various, includes TIMESTAMP fields | Daily |
| `sdp-prd-cti-data.base.base__decisions` | Decision information for tickets | signal_id, subject_id | Various, includes TIMESTAMP fields | Daily |

## Financial Data Tables

Tables with financial data including exchange rates and transaction values.

| Table Name | Description | Primary Keys | Data Types | Refresh Frequency | Verification Status |
|------------|-------------|--------------|------------|-------------------|---------------------|
| `shopify-dw.finance.shop_gmv_daily_summary` | Daily GMV/GPV metrics | shop_id, date | `shop_id` (INTEGER), `date` (DATE) | Daily | ✓ Verified |
| `shopify-dw.money_products.shopify_payments_balance_account_daily_cumulative_summary` | Cumulative balance account summary | shop_id, remote_account_id, date | `shop_id` (INTEGER), `remote_account_id` (STRING), `date` (DATE) | Daily | ✓ Verified |
| `shopify-dw.mart_cti_data.shop_loss_metrics__wide` | Shop loss metrics table | shop_id, date | `shop_id` (INTEGER), `date` (DATE) | Daily | ✓ Verified |
| `sdp-prd-cti-data.mart.detection_performance_debugger` | Detection performance metrics | ticket_id | Various, includes TIMESTAMP fields | Daily | ✓ Verified |
| `shopify-dw.money_products.shopify_capital_bankrupt_shops_current` | Shops tagged as bankrupt in Capital | shop_id | `shop_id` (INTEGER) | Daily | ✓ Verified |
| `shopify-dw.money_products.shopify_payments_bankrupt_shops_current` | Shops tagged as bankrupt in Payments | shop_id | `shop_id` (INTEGER) | Daily | ✓ Verified |

## Risk Assessment Tables

Tables containing merchant risk scores, predictive model data, and risk monitoring metrics.

| Table Name | Description | Primary Keys | Data Types | Refresh Frequency | Verification Status |
|------------|-------------|--------------|------------|-------------------|---------------------|
| `sdp-prd-cti-data.intermediate.hvcr_predictions_v2` | High Value Credit Risk model predictions | remote_account_id, provider_name, shop_id, model_id, uuid | `shop_id` (INTEGER), `processed_at` (TIMESTAMP), `observed_at` (TIMESTAMP) | Daily, partitioned by `observed_at`, clustered by `shop_id` | ✓ Verified |
| `sdp-prd-cti-data.intermediate.trust_platform_shop_risk_attributes_daily` | Daily aggregated risk attributes for merchant monitoring | shop_id, date | `shop_id` (INTEGER), `date` (DATE) | Daily | ✓ Verified |
| `sdp-prd-cti-data.intermediate.shop_insights_shops` | Comprehensive shop information with Trust Battery scores | shop_id | Various, includes PII fields | Daily | ✓ Verified |
| `shopify-dw.money_products.shopify_payments_reserve_configurations_v1` | Information about merchant reserves and configurations | reserve_config_id | Various, includes TIMESTAMP fields | Daily | ✓ Verified |

[View detailed documentation for Risk Assessment Tables](./Risk_Assessment_Tables.md)

## Payment Processing Tables

Tables related to payment processing, transfers, and transactions.

| Table Name | Description | Primary Keys | Data Types | Refresh Frequency | Verification Status |
|------------|-------------|--------------|------------|-------------------|---------------------|
| `shopify-dw.money_products.shopify_payments_balance_transactions` | Payment transaction details and balance adjustments | balance_transaction_id | Various, includes TIMESTAMP fields | Daily | ✓ Verified |
| `sdp-prd-cti-data.intermediate.shop_daily_order_fraud_facts` | Daily shop GPV, order counts, and fraud metrics | shop_id, date | `shop_id` (INTEGER), `date` (DATE) | Daily | ✓ Verified |
| `shopify-dw.money_products.shopify_payments_payouts` | Information about payouts to merchants | payout_id | Various, includes TIMESTAMP fields | Daily | ✓ Verified |

## Shop and Merchant Tables

Tables containing shop information, merchant profiles, and account details.

| Table Name | Description | Primary Keys | Data Types | Refresh Frequency | Verification Status |
|------------|-------------|--------------|------------|-------------------|---------------------|
| `shopify-dw.mart_core_deliver.shop_back_office_summary_current` | Shop summary information including industry | shop_id | `shop_id` (INTEGER), `date` (DATE), `shop_created_at` (TIMESTAMP) | Daily | ✓ Verified |
| `shopify-dw.accounts_and_administration.shop_profile_current` | Shop profile information including geography | shop_id | Various, includes TIMESTAMP fields | Daily | ✓ Verified |
| `shopify-dw.accounts_and_administration.shop_billing_info_current` | Shop billing and plan information | shop_id | Various, includes TIMESTAMP fields | Daily | ✓ Verified |

## Order and Transaction Tables

Tables with order data, fulfillment details, and transaction information.

| Table Name | Description | Primary Keys | Data Types | Refresh Frequency | Verification Status |
|------------|-------------|--------------|------------|-------------------|---------------------|
| `sdp-prd-cti-data.base.base__orders` | Base order information | order_id | Various, includes TIMESTAMP fields | Daily | ✓ Verified |
| `sdp-prd-cti-data.intermediate.order_fulfillment_facts` | Order fulfillment metrics and status | order_id, shop_id | Various, includes TIMESTAMP fields | Daily | ✓ Verified |

## Miscellaneous Tables

| Table Name | Description | Primary Keys | Data Types | Refresh Frequency |
|------------|-------------|--------------|------------|-------------------|
| `shopify-dw.base.base__payments_balance_transactions` | Legacy payment balance transactions | shop_id, balance_transaction_id | Various, includes TIMESTAMP fields | Hourly |

## Table Relationships

For a visual representation of how these tables relate to each other, see the [Table Relationships](./Table_Relationships.md) document.

## Temporal Field Types

When working with date and timestamp fields in these tables, be careful to use the appropriate comparison functions:

| Field Type | Comparison Function | Example |
|------------|---------------------|---------|
| DATE | DATE_SUB() | `WHERE date >= DATE_SUB(CURRENT_DATE(), INTERVAL 30 DAY)` |
| TIMESTAMP | TIMESTAMP_SUB() | `WHERE created_at >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 30 DAY)` |

Mismatching these types is a common query error. When necessary, convert between types:

```sql
-- Converting TIMESTAMP to DATE for comparison
WHERE DATE(timestamp_column) >= DATE_SUB(CURRENT_DATE(), INTERVAL 30 DAY)
```

## Usage Guidelines

When selecting tables for your analysis:

1. Start with intermediate tables when available as they contain pre-aggregated data
2. Use base tables when you need the most granular data
3. Consider time windows carefully in your queries (30d, 60d, 90d)
4. Always include appropriate filters to optimize query performance
5. Reference the detailed table documentation for specific usage examples

## Need a Table Not Listed?

If you need a table that isn't listed here, please:

1. Check if an equivalent table exists in the verified tables
2. Consult with the data platform team to identify the appropriate source
3. Submit a request to add the table to this documentation via [process TBD]

## Data Domains

Each table belongs to one or more data domains, which are logical groupings of related data concepts:

- **Shop Data**: Basic shop information like country, plan, status
- **Payment Processing**: Order and payment processing data
- **Risk Management**: Risk scores, fraud indicators, HVCR model
- **Financial**: Balance, GPV, loss data
- **Chargebacks**: Dispute and chargeback information
- **Trust Platform**: Tickets, actions, analyst activities

For more detailed documentation on each table's schema, sample data, and common query patterns, refer to the domain-specific documentation files.

## Access and Permissions

Most tables require specific access permissions:

- `shopify-dw.*` tables require access to the shopify-dw project
- `sdp-prd-cti-data.*` tables require specific CTI data permissions
- PII data requires the `sdp-pii` Clouddo permit

See the [Access and Permissions](../01_Getting_Started/Access_and_Permissions.md) guide for detailed instructions.

## Data Limitations and Caveats

- Most tables have a data retention period of 2 years
- Some tables are updated with a delay (check refresh frequency)
- Historical data quality may vary before 2022
- Currency values typically require conversion to USD for cross-shop analysis

## Common Query Patterns

For examples of how to use these tables together effectively, see the [Common Query Patterns](../05_Example_Queries/Common_Query_Patterns.md) documentation.

--- 