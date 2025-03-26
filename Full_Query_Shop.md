# Full Query - Shop 🏪

Welcome to the Full Query - Shop section of our SQL Query Resource Center. This page contains comprehensive end-to-end queries for shop analysis, combining multiple data sources.

## Comprehensive Shop Risk Profile

This query provides a complete risk profile for a shop by joining multiple datasets:

```sql
WITH shop_base AS (
  SELECT
    s.shop_id,
    s.shop_name,
    s.created_at as shop_created_at,
    s.industry_category,
    s.country_code,
    s.risk_tier,
    DATE_DIFF(CURRENT_DATE(), s.created_at, DAY) as days_since_creation
  FROM
    `sdp-prd-cti-data.merchant.shop_details` s
  WHERE
    s.shop_id = @shop_id
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
    p.shop_id = @shop_id
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
    c.shop_id = @shop_id
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
    f.shop_id = @shop_id
    AND f.transaction_date >= DATE_SUB(CURRENT_DATE(), INTERVAL 90 DAY)
  GROUP BY
    f.shop_id
),

shop_tickets AS (
  SELECT
    t.shop_id,
    COUNT(*) as total_tickets,
    COUNTIF(t.status = 'open' OR t.status = 'in_progress') as open_tickets,
    COUNTIF(t.ticket_type = 'risk_review') as risk_review_tickets,
    COUNTIF(t.ticket_type = 'chargeback_dispute') as chargeback_dispute_tickets,
    COUNTIF(t.ticket_type = 'fraud_investigation') as fraud_investigation_tickets,
    MAX(t.created_at) as latest_ticket_date
  FROM
    `sdp-prd-cti-data.base.base__tickets` t
  WHERE
    t.shop_id = @shop_id
    AND t.created_at >= DATE_SUB(CURRENT_DATE(), INTERVAL 180 DAY)
  GROUP BY
    t.shop_id
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
    l.shop_id = @shop_id
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
    r.shop_id = @shop_id
    AND r.snapshot_date = (
      SELECT MAX(snapshot_date)
      FROM `sdp-prd-cti-data.merchant.shop_risk_indicators`
      WHERE shop_id = @shop_id
    )
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
  r.transaction_count_daily
FROM
  shop_base b
  LEFT JOIN shop_processing p ON b.shop_id = p.shop_id
  LEFT JOIN shop_chargeback c ON b.shop_id = c.shop_id
  LEFT JOIN shop_fraud f ON b.shop_id = f.shop_id
  LEFT JOIN shop_tickets t ON b.shop_id = t.shop_id
  LEFT JOIN shop_losses l ON b.shop_id = l.shop_id
  LEFT JOIN shop_risk_indicators r ON b.shop_id = r.shop_id
```

## Shop Risk Comparison with Industry Peers

This query compares a shop's risk metrics with peers in the same industry:

```sql
WITH shop_metrics AS (
  SELECT
    s.shop_id,
    s.shop_name,
    s.industry_category,
    c.chargeback_rate_30d,
    f.confirmed_fraud_count,
    r.risk_score,
    p.total_gpv_90d
  FROM
    `sdp-prd-cti-data.merchant.shop_details` s
    LEFT JOIN `sdp-prd-cti-data.intermediate.shop_chargeback_rates_current` c
      ON s.shop_id = c.shop_id
    LEFT JOIN (
      SELECT
        shop_id,
        COUNT(DISTINCT transaction_id) as confirmed_fraud_count
      FROM
        `sdp-prd-cti-data.fraud.transaction_fraud_flags`
      WHERE
        transaction_date >= DATE_SUB(CURRENT_DATE(), INTERVAL 90 DAY)
        AND is_fraud_confirmed = TRUE
      GROUP BY
        shop_id
    ) f ON s.shop_id = f.shop_id
    LEFT JOIN `sdp-prd-cti-data.merchant.shop_risk_indicators` r
      ON s.shop_id = r.shop_id
      AND r.snapshot_date = CURRENT_DATE()
    LEFT JOIN (
      SELECT
        shop_id,
        SUM(processing_volume_usd) as total_gpv_90d
      FROM
        `sdp-prd-cti-data.finance.shop_processing_daily`
      WHERE
        transaction_date >= DATE_SUB(CURRENT_DATE(), INTERVAL 90 DAY)
      GROUP BY
        shop_id
    ) p ON s.shop_id = p.shop_id
  WHERE
    s.shop_id = @shop_id
),

industry_metrics AS (
  SELECT
    s.industry_category,
    COUNT(DISTINCT s.shop_id) as shop_count,
    AVG(c.chargeback_rate_30d) as avg_chargeback_rate,
    APPROX_QUANTILES(c.chargeback_rate_30d, 100)[OFFSET(50)] as median_chargeback_rate,
    APPROX_QUANTILES(c.chargeback_rate_30d, 100)[OFFSET(75)] as p75_chargeback_rate,
    APPROX_QUANTILES(c.chargeback_rate_30d, 100)[OFFSET(90)] as p90_chargeback_rate,
    AVG(COALESCE(f.confirmed_fraud_count, 0)) as avg_fraud_count,
    APPROX_QUANTILES(COALESCE(f.confirmed_fraud_count, 0), 100)[OFFSET(90)] as p90_fraud_count,
    AVG(r.risk_score) as avg_risk_score,
    APPROX_QUANTILES(r.risk_score, 100)[OFFSET(75)] as p75_risk_score
  FROM
    `sdp-prd-cti-data.merchant.shop_details` s
    LEFT JOIN `sdp-prd-cti-data.intermediate.shop_chargeback_rates_current` c
      ON s.shop_id = c.shop_id
    LEFT JOIN (
      SELECT
        shop_id,
        COUNT(DISTINCT transaction_id) as confirmed_fraud_count
      FROM
        `sdp-prd-cti-data.fraud.transaction_fraud_flags`
      WHERE
        transaction_date >= DATE_SUB(CURRENT_DATE(), INTERVAL 90 DAY)
        AND is_fraud_confirmed = TRUE
      GROUP BY
        shop_id
    ) f ON s.shop_id = f.shop_id
    LEFT JOIN `sdp-prd-cti-data.merchant.shop_risk_indicators` r
      ON s.shop_id = r.shop_id
      AND r.snapshot_date = CURRENT_DATE()
    LEFT JOIN (
      SELECT
        shop_id,
        SUM(processing_volume_usd) as total_gpv_90d
      FROM
        `sdp-prd-cti-data.finance.shop_processing_daily`
      WHERE
        transaction_date >= DATE_SUB(CURRENT_DATE(), INTERVAL 90 DAY)
      GROUP BY
        shop_id
    ) p ON s.shop_id = p.shop_id
  WHERE
    p.total_gpv_90d > 10000
  GROUP BY
    s.industry_category
)

SELECT
  sm.shop_id,
  sm.shop_name,
  sm.industry_category,
  sm.chargeback_rate_30d,
  im.avg_chargeback_rate,
  im.median_chargeback_rate,
  im.p75_chargeback_rate,
  im.p90_chargeback_rate,
  CASE
    WHEN sm.chargeback_rate_30d > im.p90_chargeback_rate THEN 'Top 10% (High Risk)'
    WHEN sm.chargeback_rate_30d > im.p75_chargeback_rate THEN 'Top 25% (Elevated Risk)'
    WHEN sm.chargeback_rate_30d > im.median_chargeback_rate THEN 'Above Median'
    ELSE 'Below Median'
  END as chargeback_rate_percentile,
  sm.confirmed_fraud_count,
  im.avg_fraud_count,
  im.p90_fraud_count,
  CASE
    WHEN sm.confirmed_fraud_count > im.p90_fraud_count THEN 'Top 10% (High Risk)'
    ELSE 'Normal Range'
  END as fraud_count_percentile,
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
WITH shop_transactions AS (
  SELECT
    shop_id,
    transaction_date as event_date,
    'transaction' as event_type,
    CONCAT('Processed $', FORMAT('%,.2f', processing_volume_usd), ' with ', transaction_count, ' transactions') as event_description
  FROM
    `sdp-prd-cti-data.finance.shop_processing_daily`
  WHERE
    shop_id = @shop_id
    AND transaction_date >= DATE_SUB(CURRENT_DATE(), INTERVAL 180 DAY)
),

shop_chargebacks AS (
  SELECT
    shop_id,
    chargeback_date as event_date,
    'chargeback' as event_type,
    CONCAT('Chargeback received: $', FORMAT('%,.2f', chargeback_amount_usd), ' - ', chargeback_reason) as event_description
  FROM
    `sdp-prd-cti-data.finance.shop_chargebacks`
  WHERE
    shop_id = @shop_id
    AND chargeback_date >= DATE_SUB(CURRENT_DATE(), INTERVAL 180 DAY)
),

shop_tickets AS (
  SELECT
    shop_id,
    created_at as event_date,
    CONCAT('ticket_', ticket_type) as event_type,
    CONCAT('Ticket created: ', ticket_id, ' - ', ticket_type, ' - Priority: ', priority) as event_description
  FROM
    `sdp-prd-cti-data.base.base__tickets`
  WHERE
    shop_id = @shop_id
    AND created_at >= DATE_SUB(CURRENT_DATE(), INTERVAL 180 DAY)
),

shop_losses AS (
  SELECT
    shop_id,
    booking_date as event_date,
    'loss' as event_type,
    CONCAT('Loss booked: $', FORMAT('%,.2f', loss_amount_usd), ' - Category: ', loss_category) as event_description
  FROM
    `sdp-prd-cti-data.finance.booked_losses`
  WHERE
    shop_id = @shop_id
    AND booking_date >= DATE_SUB(CURRENT_DATE(), INTERVAL 180 DAY)
),

shop_risk_changes AS (
  SELECT
    shop_id,
    snapshot_date as event_date,
    'risk_change' as event_type,
    CONCAT('Risk score changed to ', CAST(risk_score as STRING), ' (prev: ', 
           CAST(LAG(risk_score) OVER (PARTITION BY shop_id ORDER BY snapshot_date) as STRING), ')') as event_description
  FROM
    `sdp-prd-cti-data.merchant.shop_risk_indicators`
  WHERE
    shop_id = @shop_id
    AND snapshot_date >= DATE_SUB(CURRENT_DATE(), INTERVAL 180 DAY)
    AND ABS(risk_score - LAG(risk_score) OVER (PARTITION BY shop_id ORDER BY snapshot_date)) > 0.1
),

all_events AS (
  SELECT * FROM shop_transactions
  UNION ALL SELECT * FROM shop_chargebacks
  UNION ALL SELECT * FROM shop_tickets
  UNION ALL SELECT * FROM shop_losses
  UNION ALL SELECT * FROM shop_risk_changes
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

- These queries can be resource-intensive due to joining multiple large datasets
- Consider using parameters to filter for specific shops before running
- For production use, these queries may need optimization or pre-aggregated tables
- Add custom filters based on your specific analysis needs
- Adjust date ranges based on the shop's age and processing history

Feel free to contribute new full queries or improve the existing ones. Together, we can build a comprehensive resource for shop analysis.

Happy analyzing! 🔎 