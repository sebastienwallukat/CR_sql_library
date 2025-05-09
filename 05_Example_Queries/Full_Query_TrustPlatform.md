# Full Query - TrustPlatform 🔍

Welcome to the TrustPlatform section of our SQL Query Resource Center. This page contains comprehensive end-to-end queries for analyzing the TrustPlatform system.

## Recommended Datasets

Always use these tables for Trust Platform analysis:
- `shopify-dw.risk.trust_platform_tickets_summary_v1` - Primary tickets table
- `shopify-dw.risk.trust_platform_actions_summary_v1` - Actions taken on tickets
- `shopify-dw.risk.trust_platform_ticket_escalations_v1` - Ticket escalation events

## Comprehensive Ticket Analysis

This query provides a complete analysis of tickets, actions, and escalations:

```sql
WITH ticket_metrics AS (
  SELECT
    t.subjectable_id AS shop_id,
    t.trust_platform_ticket_id,
    t.latest_report_type AS ticket_type,
    t.rule_group AS priority,
    t.status,
    t.created_at,
    t.latest_trust_platform_user_email AS assignee,
    TIMESTAMP_DIFF(CURRENT_TIMESTAMP(), t.created_at, HOUR) as hours_open,
    CASE
      WHEN t.status = 'closed' THEN TIMESTAMP_DIFF(t.latest_action_at, t.created_at, HOUR)
      ELSE NULL
    END as resolution_time_hours
  FROM
    `shopify-dw.risk.trust_platform_tickets_summary_v1` t
  WHERE
    t.created_at >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 90 DAY)
),

ticket_actions AS (
  SELECT
    a.actionable_id AS ticket_id,
    COUNT(*) as action_count,
    ARRAY_AGG(a.action_type ORDER BY a.created_at) as actions_taken,
    MIN(a.created_at) as first_action_time,
    MAX(a.created_at) as last_action_time
  FROM
    `shopify-dw.risk.trust_platform_actions_summary_v1` a
  JOIN
    ticket_metrics t ON a.actionable_id = t.trust_platform_ticket_id
  WHERE
    a.actionable_type = 'Ticket'
  GROUP BY
    a.actionable_id
),

ticket_escalations AS (
  SELECT
    e.from_trust_platform_ticket_id AS ticket_id,
    COUNT(*) as escalation_count,
    ARRAY_AGG(e.escalation_type ORDER BY e.escalated_at) as escalation_reasons,
    MIN(e.escalated_at) as first_escalation_time
  FROM
    `shopify-dw.risk.trust_platform_ticket_escalations_v1` e
  JOIN
    ticket_metrics t ON e.from_trust_platform_ticket_id = t.trust_platform_ticket_id
  GROUP BY
    e.from_trust_platform_ticket_id
)

SELECT
  t.*,
  a.action_count,
  a.actions_taken,
  a.first_action_time,
  a.last_action_time,
  TIMESTAMP_DIFF(a.first_action_time, t.created_at, MINUTE) as minutes_to_first_action,
  e.escalation_count,
  e.escalation_reasons,
  e.first_escalation_time,
  CASE
    WHEN e.first_escalation_time IS NOT NULL 
    THEN TIMESTAMP_DIFF(e.first_escalation_time, t.created_at, HOUR)
    ELSE NULL
  END as hours_to_escalation
FROM
  ticket_metrics t
LEFT JOIN
  ticket_actions a ON t.trust_platform_ticket_id = a.ticket_id
LEFT JOIN
  ticket_escalations e ON t.trust_platform_ticket_id = e.ticket_id
ORDER BY
  t.created_at DESC
```

## Ticket Performance by Assignee

```sql
WITH assignee_metrics AS (
  SELECT
    t.latest_trust_platform_user_email AS assignee,
    COUNT(*) as total_tickets,
    COUNTIF(t.status = 'closed') as closed_tickets,
    COUNTIF(t.status = 'open' OR t.status = 'in_progress') as open_tickets,
    AVG(CASE WHEN t.status = 'closed' THEN TIMESTAMP_DIFF(t.latest_action_at, t.created_at, HOUR) END) as avg_resolution_hours,
    APPROX_QUANTILES(CASE WHEN t.status = 'closed' THEN TIMESTAMP_DIFF(t.latest_action_at, t.created_at, HOUR) END, 2)[OFFSET(1)] as median_resolution_hours
  FROM
    `shopify-dw.risk.trust_platform_tickets_summary_v1` t
  WHERE
    t.created_at >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 90 DAY)
    AND t.latest_trust_platform_user_email IS NOT NULL
  GROUP BY
    t.latest_trust_platform_user_email
),

assignee_actions AS (
  SELECT
    t.latest_trust_platform_user_email AS assignee,
    COUNT(a.trust_platform_action_id) as total_actions,
    COUNT(a.trust_platform_action_id) / COUNT(DISTINCT t.trust_platform_ticket_id) as actions_per_ticket
  FROM
    `shopify-dw.risk.trust_platform_tickets_summary_v1` t
  JOIN
    `shopify-dw.risk.trust_platform_actions_summary_v1` a ON t.trust_platform_ticket_id = a.actionable_id
  WHERE
    t.created_at >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 90 DAY)
    AND t.latest_trust_platform_user_email IS NOT NULL
    AND a.actionable_type = 'Ticket'
  GROUP BY
    t.latest_trust_platform_user_email
)

SELECT
  m.assignee,
  m.total_tickets,
  m.closed_tickets,
  m.open_tickets,
  m.closed_tickets / m.total_tickets as ticket_close_rate,
  m.avg_resolution_hours,
  m.median_resolution_hours,
  a.total_actions,
  a.actions_per_ticket
FROM
  assignee_metrics m
JOIN
  assignee_actions a ON m.assignee = a.assignee
ORDER BY
  m.total_tickets DESC
```

## Rule Triggering Analysis

This query provides insights into which rules are generating tickets and their metadata:

```sql
SELECT 
  t.subjectable_id AS shop_id,
  t.trust_platform_ticket_id,
  t.latest_report_type AS ticket_type,
  t.rule_group AS priority,
  t.status,
  t.created_at AS ticket_created_at,
  a.action_type,
  a.created_at AS action_created_at,
  -- Analyze metadata from actions to understand rule context
  JSON_EXTRACT_SCALAR(a.metadata, '$.rule_identifier') AS rule_identifier,
  JSON_EXTRACT_SCALAR(a.metadata, '$.trigger_reason') AS trigger_reason,
  JSON_EXTRACT_SCALAR(a.metadata, '$.risk_score') AS risk_score
FROM 
  `shopify-dw.risk.trust_platform_tickets_summary_v1` AS t
JOIN 
  `shopify-dw.risk.trust_platform_actions_summary_v1` AS a
  ON t.trust_platform_ticket_id = a.actionable_id
WHERE
  t.created_at >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 90 DAY)
  AND a.actionable_type = 'Ticket'
  AND JSON_EXTRACT_SCALAR(a.metadata, '$.rule_identifier') IS NOT NULL
ORDER BY
  t.created_at DESC
LIMIT 100
```

## Notes and Best Practices 📝

- Always use the recommended TrustPlatform tables listed above
- These tables provide the most reliable and comprehensive data
- For large date ranges, consider adding more specific filters
- When analyzing assignee performance, account for ticket priority and complexity
- Use the ticket action history to understand workflow patterns
- When querying rule metadata, use the JSON extraction functions to parse the metadata fields
- **Important**: All temporal fields in these tables are TIMESTAMP type, so always use TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL X DAY) for date comparisons

### Temporal Fields Documentation

| Field Name | Data Type | Comparison Function | Example |
|------------|-----------|---------------------|---------|
| created_at | TIMESTAMP | TIMESTAMP_SUB() | `WHERE created_at >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 7 DAY)` |
| first_action_at | TIMESTAMP | TIMESTAMP_SUB() | `WHERE first_action_at >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 7 DAY)` |
| latest_action_at | TIMESTAMP | TIMESTAMP_SUB() | `WHERE latest_action_at >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 7 DAY)` |
| escalated_at | TIMESTAMP | TIMESTAMP_SUB() | `WHERE escalated_at >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 7 DAY)` |

Feel free to contribute new full queries or improve the existing ones. Together, we can build a comprehensive resource for TrustPlatform analysis.

Happy analyzing! 🔎

## Look into rules metadata
```sql
SELECT 
  r.shop_id,
  r.detection_identifier as rule_name,
  r.report_group,
  r.report_type,
  t.trust_platform_ticket_id AS ticket_id,
  t.status,
  t.created_at AS ticket_created_at,
  r.monitoring_metadata,-- << this will pull all the metadata at once so if you want it to be separated then
  json_extract_scalar(monitoring_metadata,'$.bad_reviews') AS name_of_metadata --<< this way, you can pull the data one by one, easier to read 
FROM 
  `sdp-prd-cti-data.base.base__sensitive_reports` AS r
  JOIN `sdp-prd-cti-data.base.base__tickets` AS t
    ON r.report_id = t.reportable_id
WHERE
  t.created_at >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 180 DAY)
  AND r.detection_identifier = 'Event::Shop10kTo50kThreshold'-- << you need to use the detection_identifier of the rule 
```

# Trust Platform Query Examples

These example queries demonstrate how to work with Trust Platform data for credit risk analysis. All queries have been validated with MCP to ensure they execute correctly.

## High-Risk Tickets in the Last 30 Days

This query retrieves high-risk tickets created in the last 30 days with actions taken.

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
WHERE created_at >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 30 DAY)  -- Note TIMESTAMP_SUB for TIMESTAMP column
  AND rule_group = 'HIGH'
  AND latest_action_type IS NOT NULL
ORDER BY created_at DESC
```

## Chargeback-Related Tickets by Shop

This query combines Trust Platform data with chargeback data to find shops with high chargeback rates that also have trust platform tickets.

```sql

SELECT
  tp.subjectable_id AS shop_id,
  COUNT(DISTINCT tp.trust_platform_ticket_id) AS ticket_count,
  MAX(tp.created_at) AS most_recent_ticket_date,
  MAX(cb.chargeback_rate_90d) AS current_chargeback_rate_90d
FROM `shopify-dw.risk.trust_platform_tickets_summary_v1` tp
JOIN `sdp-prd-cti-data.intermediate.shop_chargeback_rates_current` cb
  ON tp.subjectable_id = cb.shop_id
WHERE tp.created_at >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 90 DAY)  -- TIMESTAMP field
  AND cb.chargeback_rate_90d > 0.01  -- Shops with >1% 90-day chargeback rate
  AND tp.subjectable_type = 'Shop'
GROUP BY tp.subjectable_id
HAVING ticket_count > 1
ORDER BY current_chargeback_rate_90d DESC
```

## Tickets with Reserve Configuration Changes

This query identifies tickets that resulted in reserve configuration changes.

```sql

WITH reserve_changes AS (
  SELECT
    shop_id,
    reserve_created_at,
    reserve_percentage,
    reserve_hold_duration
  FROM `sdp-prd-cti-data.base.base__shopify_payments_reserve_configurations`
  WHERE reserve_created_at >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 90 DAY)  -- TIMESTAMP field
    AND is_active = TRUE
)

SELECT
  tp.trust_platform_ticket_id,
  tp.subjectable_id AS shop_id,
  tp.created_at AS ticket_created_at,
  tp.latest_action_type,
  tp.rule_group,
  rc.reserve_created_at,
  rc.reserve_percentage,
  rc.reserve_hold_duration
FROM `shopify-dw.risk.trust_platform_tickets_summary_v1` tp
JOIN reserve_changes rc
  ON tp.subjectable_id = rc.shop_id
  AND tp.created_at <= rc.reserve_created_at  -- Ticket created before reserve change
  AND TIMESTAMP_ADD(tp.created_at, INTERVAL 1 DAY) >= rc.reserve_created_at  -- Reserve changed within 1 day of ticket
WHERE tp.subjectable_type = 'Shop'
  AND tp.created_at >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 90 DAY)  -- TIMESTAMP field
ORDER BY tp.created_at DESC
```

## Temporal Metrics by Report Group

This query analyzes the average time to first action by report group.

```sql

SELECT
  DATE_TRUNC(created_at, MONTH) AS month,  -- Truncate TIMESTAMP to month
  latest_report_group,
  COUNT(*) AS ticket_count,
  AVG(time_to_first_action) AS avg_minutes_to_first_action,
  SUM(CASE WHEN meets_slo = TRUE THEN 1 ELSE 0 END) AS slo_met_count,
  ROUND(SUM(CASE WHEN meets_slo = TRUE THEN 1 ELSE 0 END) / COUNT(*) * 100, 2) AS slo_met_percentage
FROM `shopify-dw.risk.trust_platform_tickets_summary_v1`
WHERE created_at >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 180 DAY)  -- TIMESTAMP field
  AND latest_report_group IS NOT NULL
GROUP BY 1, 2
ORDER BY 1 DESC, 2
```

## Combining with Financial Data

This query combines Trust Platform data with financial data to analyze tickets for shops with significant GPV.

```sql

SELECT
  tp.trust_platform_ticket_id,
  tp.subjectable_id AS shop_id,
  tp.created_at,
  tp.latest_report_type,
  tp.rule_group,
  gmv.gpv_usd_l90d,
  gmv.cumulative_gpv_usd
FROM `shopify-dw.risk.trust_platform_tickets_summary_v1` tp
JOIN `shopify-dw.finance.shop_gmv_current` gmv
  ON tp.subjectable_id = gmv.shop_id
WHERE tp.created_at >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 30 DAY)  -- TIMESTAMP field
  AND gmv.gpv_usd_l90d > 100000  -- $100k+ in last 90 days
  AND tp.subjectable_type = 'Shop'
ORDER BY gmv.gpv_usd_l90d DESC, tp.created_at DESC
```

## Important Notes on Temporal Fields

- `created_at` in `shopify-dw.risk.trust_platform_tickets_summary_v1` is a **TIMESTAMP** field
  - Use `TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL X DAY)` for filtering
- Date/timestamp mismatches are the most common query errors in BigQuery
- Always use the correct temporal function for the field type:
  - DATE fields: `DATE_SUB(CURRENT_DATE(), INTERVAL X DAY)`
  - TIMESTAMP fields: `TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL X DAY)`

For more information on handling temporal fields, see the [Date vs Timestamp Guide](../02_SQL_Guide/Date_vs_Timestamp_Guide.md).

---
