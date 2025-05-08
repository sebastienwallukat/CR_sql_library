# Full Query - Shop 🏪

Welcome to the Full Query - Shop section of our SQL Query Resource Center. This page contains comprehensive end-to-end queries for shop analysis, combining multiple data sources.

## Comprehensive Shop Risk Profile

This query provides a complete risk profile for a shop by joining multiple datasets:

```sql
WITH shop_base AS (
  SELECT
    s.shop_id,
    shop_detail.shop_name,
    shop_detail.created_at as shop_created_at,
    shop_detail.industry AS industry_category,
    shop_detail.country_code,
    shop_detail.primary_category AS risk_tier,
    DATE_DIFF(CURRENT_DATE(), DATE(shop_detail.created_at), DAY) as days_since_creation
  FROM
    `shopify-dw.mart_core_operate.identity_account_shop_summary_current` s,
    UNNEST(s.shop_details) AS shop_detail
  WHERE
    shop_detail.shop_id = 12345
),

shop_processing AS (
  SELECT
    p.shop_id,
    SUM(p.processing_volume_usd) as total_gpv_lifetime,
    AVG(p.processing_volume_usd) as avg_daily_gpv,
    COUNT(DISTINCT p.transaction_date) as active_processing_days
  FROM
    `sdp-prd-cti-data.finance.shop_processing_daily` p
  WHERE
    p.shop_id = 12345
  GROUP BY
    p.shop_id
),

shop_chargeback AS (
  SELECT
    c.shop_id,
    c.chargeback_count_30d,
    c.chargeback_amount_usd_30d,
    c.gpv_amount_usd_30d,
    c.chargeback_rate_30d,
    c.chargeback_count_90d,
    c.chargeback_amount_usd_90d,
    c.gpv_amount_usd_90d,
    c.chargeback_rate_90d
  FROM
    `sdp-prd-cti-data.intermediate.shop_chargeback_rates_current` c
  WHERE
    c.shop_id = 12345
),

shop_fraud AS (
  SELECT
    f.shop_id,
    COUNT(*) as total_fraud_flags,
    COUNTIF(f.is_fraud_confirmed = TRUE) as confirmed_fraud_count,
    SUM(CASE WHEN f.is_fraud_confirmed = TRUE THEN f.transaction_amount ELSE 0 END) as confirmed_fraud_amount_usd
  FROM
    `sdp-prd-cti-data.fraud.transaction_fraud_flags` f
  WHERE
    f.shop_id = 12345
    AND f.transaction_date >= DATE_SUB(CURRENT_DATE(), INTERVAL 90 DAY)
  GROUP BY
    f.shop_id
),

shop_tickets AS (
  SELECT
    t.subjectable_id as shop_id,
    COUNT(*) as total_tickets,
    COUNTIF(t.status = 'open' OR t.status = 'in_progress') as open_tickets,
    COUNTIF(t.latest_report_type = 'risk_review') as risk_review_tickets,
    COUNTIF(t.latest_report_type = 'chargeback_dispute') as chargeback_dispute_tickets,
    COUNTIF(t.latest_report_type = 'fraud_investigation') as fraud_investigation_tickets,
    MAX(t.created_at) as latest_ticket_date
  FROM
    `shopify-dw.risk.trust_platform_tickets_summary_v1` t
  WHERE
    t.subjectable_id = 12345
    AND t.created_at >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 180 DAY)
  GROUP BY
    t.subjectable_id
),

shop_losses AS (
  SELECT
    l.shop_id,
    COUNT(*) as total_losses,
    SUM(l.loss_amount_usd) as total_loss_amount_usd,
    MAX(l.booking_date) as latest_loss_date,
    ARRAY_AGG(DISTINCT l.loss_category ORDER BY l.loss_category) as loss_categories
  FROM
    `sdp-prd-cti-data.finance.booked_losses` l
  WHERE
    l.shop_id = 12345
  GROUP BY
    l.shop_id
),

shop_risk_indicators AS (
  SELECT
    r.shop_id,
    r.risk_score,
    r.velocity_score,
    r.customer_complaint_count_30d,
    r.avg_transaction_amount,
    r.transaction_count_daily
  FROM
    `sdp-prd-cti-data.merchant.shop_risk_indicators` r
  WHERE
    r.shop_id = 12345
    AND r.snapshot_date = (
      SELECT MAX(snapshot_date)
      FROM `sdp-prd-cti-data.merchant.shop_risk_indicators`
      WHERE shop_id = 12345
    )
),

shop_reserves AS (
  SELECT
    r.merchant_id as shop_id,
    r.reserve_rate,
    r.reserve_type,
    r.rolling_window_days,
    r.minimum_reserve_amount_usd,
    r.effective_from_date,
    r.effective_to_date
  FROM
    `sdp-prd-cti-data.base.base__shopify_payments_reserve_configurations` r
  WHERE
    r.merchant_id = 12345
    AND CURRENT_DATE() BETWEEN r.effective_from_date AND COALESCE(r.effective_to_date, '9999-12-31')
)

SELECT
  b.*,
  p.total_gpv_lifetime,
  p.avg_daily_gpv,
  p.active_processing_days,
  c.chargeback_count_30d,
  c.chargeback_amount_usd_30d,
  c.gpv_amount_usd_30d,
  c.chargeback_rate_30d,
  c.chargeback_count_90d,
  c.chargeback_amount_usd_90d,
  c.gpv_amount_usd_90d,
  c.chargeback_rate_90d,
  f.total_fraud_flags,
  f.confirmed_fraud_count,
  f.confirmed_fraud_amount_usd,
  t.total_tickets,
  t.open_tickets,
  t.risk_review_tickets,
  t.chargeback_dispute_tickets,
  t.fraud_investigation_tickets,
  t.latest_ticket_date,
  l.total_losses,
  l.total_loss_amount_usd,
  l.latest_loss_date,
  l.loss_categories,
  r.risk_score,
  r.velocity_score,
  r.customer_complaint_count_30d,
  r.avg_transaction_amount,
  r.transaction_count_daily,
  res.reserve_rate,
  res.reserve_type,
  res.minimum_reserve_amount_usd
FROM
  shop_base b
  LEFT JOIN shop_processing p ON b.shop_id = p.shop_id
  LEFT JOIN shop_chargeback c ON b.shop_id = c.shop_id
  LEFT JOIN shop_fraud f ON b.shop_id = f.shop_id
  LEFT JOIN shop_tickets t ON b.shop_id = t.shop_id
  LEFT JOIN shop_losses l ON b.shop_id = l.shop_id
  LEFT JOIN shop_risk_indicators r ON b.shop_id = r.shop_id
  LEFT JOIN shop_reserves res ON b.shop_id = res.shop_id
```

## Shop Risk Comparison with Industry Peers

This query compares a shop's risk metrics with peers in the same industry:

```sql
WITH shop_metrics AS (
  SELECT
    shop_detail.shop_id,
    shop_detail.shop_name,
    shop_detail.industry AS industry_category,
    shop_detail.shop_score AS risk_score
  FROM
    `shopify-dw.mart_core_operate.identity_account_shop_summary_current` s,
    UNNEST(s.shop_details) AS shop_detail
  WHERE
    shop_detail.shop_id = 12345
),

industry_metrics AS (
  SELECT
    shop_detail.industry AS industry_category,
    COUNT(DISTINCT shop_detail.shop_id) as shop_count,
    AVG(shop_detail.shop_score) as avg_risk_score,
    APPROX_QUANTILES(shop_detail.shop_score, 100)[OFFSET(75)] as p75_risk_score
  FROM
    `shopify-dw.mart_core_operate.identity_account_shop_summary_current` s,
    UNNEST(s.shop_details) AS shop_detail
  WHERE
    shop_detail.industry IS NOT NULL
    AND shop_detail.shop_score IS NOT NULL
  GROUP BY
    shop_detail.industry
)

SELECT
  sm.shop_id,
  sm.shop_name,
  sm.industry_category,
  sm.risk_score,
  im.avg_risk_score,
  im.p75_risk_score,
  CASE
    WHEN sm.risk_score > im.p75_risk_score THEN 'Top 25% (High Risk)'
    ELSE 'Normal Range'
  END as risk_score_percentile,
  im.shop_count as industry_peer_count
FROM
  shop_metrics sm
  JOIN industry_metrics im ON sm.industry_category = im.industry_category
```

## Shop Activity Timeline

This query creates a comprehensive timeline of significant shop events:

```sql
WITH shop_tickets AS (
  SELECT
    subjectable_id as shop_id,
    created_at as event_date,
    CONCAT('ticket_', latest_report_type) as event_type,
    CONCAT('Ticket created: ', trust_platform_ticket_id, ' - ', latest_report_type, ' - Priority: ', rule_group) as event_description
  FROM
    `shopify-dw.risk.trust_platform_tickets_summary_v1`
  WHERE
    subjectable_id = 12345
    AND created_at >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 180 DAY)
),

shop_ticket_actions AS (
  SELECT
    t.subjectable_id as shop_id,
    a.created_at as event_date,
    'ticket_action' as event_type,
    CONCAT('Ticket action: ', t.trust_platform_ticket_id, ' - ', a.action_type) as event_description
  FROM
    `shopify-dw.risk.trust_platform_actions_summary_v1` a
    JOIN `shopify-dw.risk.trust_platform_tickets_summary_v1` t ON a.ticket_id = t.trust_platform_ticket_id
  WHERE
    t.subjectable_id = 12345
    AND a.created_at >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 180 DAY)
),

shop_ticket_escalations AS (
  SELECT
    t.subjectable_id as shop_id,
    e.created_at as event_date,
    'ticket_escalation' as event_type,
    CONCAT('Ticket escalated: ', t.trust_platform_ticket_id, ' - Reason: ', e.escalation_reason) as event_description
  FROM
    `shopify-dw.risk.trust_platform_ticket_escalations_v1` e
    JOIN `shopify-dw.risk.trust_platform_tickets_summary_v1` t ON e.ticket_id = t.trust_platform_ticket_id
  WHERE
    t.subjectable_id = 12345
    AND e.created_at >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 180 DAY)
),

all_events AS (
  SELECT * FROM shop_tickets
  UNION ALL SELECT * FROM shop_ticket_actions
  UNION ALL SELECT * FROM shop_ticket_escalations
)

SELECT
  event_date,
  event_type,
  event_description
FROM
  all_events
ORDER BY
  event_date DESC
```

## Notes and Best Practices 📝

- These queries have been validated to ensure they use existing tables and proper data types
- For TIMESTAMP fields, always use `TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL X DAY)` instead of `DATE_SUB(CURRENT_DATE(), INTERVAL X DAY)`
- For DATE fields, always use `DATE_SUB(CURRENT_DATE(), INTERVAL X DAY)`
- To use these queries for different shops, replace the shop_id value (currently set to 12345) with your target shop ID
- The key tables for shop analysis are:
  - `shopify-dw.mart_core_operate.identity_account_shop_summary_current` - Primary shop information
  - `shopify-dw.risk.trust_platform_tickets_summary_v1` - Ticket information
- Adjust date ranges based on the shop's age and processing history
- Always use the recommended tables for tickets (`shopify-dw.risk.trust_platform_*`)

Feel free to contribute new full queries or improve the existing ones. Together, we can build a comprehensive resource for shop analysis.

Happy analyzing! 🔎