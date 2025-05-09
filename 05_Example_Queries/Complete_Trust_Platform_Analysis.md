# Complete Trust Platform Analysis

This document provides comprehensive SQL queries for analyzing Trust Platform data, with a focus on ticket analysis, trends, and performance metrics.

## Introduction
This guide provides comprehensive end-to-end queries for analyzing the TrustPlatform system. These queries are designed to help credit risk analysts efficiently analyze tickets, actions, and escalations within the Trust Platform to identify risk patterns and make informed decisions.

## Table of Contents
- [Recommended Datasets](#recommended-datasets)
- [Comprehensive Ticket Analysis](#comprehensive-ticket-analysis)
- [Ticket Performance by Assignee](#ticket-performance-by-assignee)
- [Rule Triggering Analysis](#rule-triggering-analysis)
- [Notes and Best Practices](#notes-and-best-practices)
- [Additional Query Examples](#additional-query-examples)
  - [High-Risk Tickets](#high-risk-tickets-in-the-last-30-days)
  - [Chargeback-Related Tickets](#chargeback-related-tickets-by-shop)
  - [Reserve Configuration Changes](#tickets-with-reserve-configuration-changes)
  - [Temporal Metrics](#temporal-metrics-by-report-group)
  - [Financial Data Integration](#combining-with-financial-data)
- [Important Notes on Temporal Fields](#important-notes-on-temporal-fields)

## Recommended Datasets

Always use these tables for Trust Platform analysis:
- `shopify-dw.risk.trust_platform_tickets_summary_v1` - Primary tickets table
- `shopify-dw.risk.trust_platform_actions_summary_v1` - Actions taken on tickets
- `shopify-dw.risk.trust_platform_ticket_escalations_v1` - Ticket escalation events

## Comprehensive Ticket Analysis

This query provides a complete analysis of tickets, actions, and escalations:

```sql
-- Purpose: Analyze Trust Platform tickets with their actions and escalations
-- This query joins ticket data with actions and escalations to provide a comprehensive view

WITH ticket_metrics AS (
  SELECT
    t.subjectable_id AS shop_id,                      -- Shop ID associated with the ticket
    t.trust_platform_ticket_id,                       -- Unique identifier for the ticket
    t.latest_report_type AS ticket_type,              -- Type of report/issue
    t.rule_group AS priority,                         -- Priority level (e.g., HIGH, MEDIUM, LOW)
    t.status,                                         -- Current ticket status
    t.created_at,                                     -- When the ticket was created
    t.latest_trust_platform_user_email AS assignee,   -- Current assignee
    TIMESTAMP_DIFF(CURRENT_TIMESTAMP(), t.created_at, HOUR) as hours_open,  -- How long ticket has been open
    CASE
      WHEN t.status = 'closed' THEN TIMESTAMP_DIFF(t.latest_action_at, t.created_at, HOUR)
      ELSE NULL
    END as resolution_time_hours                      -- How long it took to resolve (if closed)
  FROM
    `shopify-dw.risk.trust_platform_tickets_summary_v1` t
  WHERE
    t.created_at >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 90 DAY)  -- Only recent tickets
),

ticket_actions AS (
  SELECT
    a.actionable_id AS ticket_id,
    COUNT(*) as action_count,                         -- Number of actions taken
    ARRAY_AGG(a.action_type ORDER BY a.created_at) as actions_taken,  -- List of all actions in chronological order
    MIN(a.created_at) as first_action_time,           -- When first action was taken
    MAX(a.created_at) as last_action_time             -- When last action was taken
  FROM
    `shopify-dw.risk.trust_platform_actions_summary_v1` a
  JOIN
    ticket_metrics t ON a.actionable_id = t.trust_platform_ticket_id
  WHERE
    a.actionable_type = 'Ticket'                      -- Only include ticket actions
  GROUP BY
    a.actionable_id
),

ticket_escalations AS (
  SELECT
    e.from_trust_platform_ticket_id AS ticket_id,
    COUNT(*) as escalation_count,                     -- Number of times ticket was escalated
    ARRAY_AGG(e.escalation_type ORDER BY e.escalated_at) as escalation_reasons,  -- List of escalation reasons
    MIN(e.escalated_at) as first_escalation_time      -- When ticket was first escalated
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
  TIMESTAMP_DIFF(a.first_action_time, t.created_at, MINUTE) as minutes_to_first_action,  -- Response time
  e.escalation_count,
  e.escalation_reasons,
  e.first_escalation_time,
  CASE
    WHEN e.first_escalation_time IS NOT NULL 
    THEN TIMESTAMP_DIFF(e.first_escalation_time, t.created_at, HOUR)
    ELSE NULL
  END as hours_to_escalation                          -- How long before first escalation
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

This query helps identify performance metrics for each assignee, including resolution times and action counts:

```sql
-- Purpose: Measure ticket handling performance by assignee
-- Analyzes resolution times, ticket volumes, and action efficiency

WITH assignee_metrics AS (
  SELECT
    t.latest_trust_platform_user_email AS assignee,
    COUNT(*) as total_tickets,                        -- Total tickets assigned
    COUNTIF(t.status = 'closed') as closed_tickets,   -- How many tickets were closed
    COUNTIF(t.status = 'open' OR t.status = 'in_progress') as open_tickets,  -- Current workload
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
    COUNT(a.trust_platform_action_id) / COUNT(DISTINCT t.trust_platform_ticket_id) as actions_per_ticket  -- Action efficiency
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
  m.closed_tickets / m.total_tickets as ticket_close_rate,  -- Percentage of assigned tickets closed
  m.avg_resolution_hours,
  m.median_resolution_hours,
  a.total_actions,
  a.actions_per_ticket                                -- Average actions needed per ticket
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
-- Purpose: Analyze which rules are triggering tickets and the associated context
-- Examines rule identifiers, trigger reasons, and risk scores from action metadata

SELECT 
  t.subjectable_id AS shop_id,
  t.trust_platform_ticket_id,
  t.latest_report_type AS ticket_type,
  t.rule_group AS priority,
  t.status,
  t.created_at AS ticket_created_at,
  a.action_type,
  a.created_at AS action_created_at,
  -- Extract important rule context from action metadata
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
  AND JSON_EXTRACT_SCALAR(a.metadata, '$.rule_identifier') IS NOT NULL  -- Only include rule-triggered actions
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

## Additional Query Examples

### High-Risk Tickets in the Last 30 Days

This query retrieves high-risk tickets created in the last 30 days with actions taken:

```sql
-- Purpose: Identify recent high-risk tickets that have had actions taken
-- Useful for monitoring the most critical tickets requiring review

SELECT 
  trust_platform_ticket_id,
  subjectable_id AS shop_id,
  created_at,
  latest_report_type AS ticket_type,
  rule_group AS risk_level,                           -- HIGH risk level filter
  status AS ticket_status,
  latest_action_type AS action_taken                  -- What action was taken
FROM `shopify-dw.risk.trust_platform_tickets_summary_v1`
WHERE created_at >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 30 DAY)  -- Recent tickets only
  AND rule_group = 'HIGH'                             -- Only high-risk tickets
  AND latest_action_type IS NOT NULL                  -- Only tickets with actions
ORDER BY created_at DESC
```

### Chargeback-Related Tickets by Shop

This query combines Trust Platform data with chargeback data to find shops with high chargeback rates that also have trust platform tickets:

```sql
-- Purpose: Identify shops with both high chargeback rates and Trust Platform tickets
-- Helps discover shops that may require closer monitoring or investigation

SELECT
  tp.subjectable_id AS shop_id,
  COUNT(DISTINCT tp.trust_platform_ticket_id) AS ticket_count,  -- Number of tickets
  MAX(tp.created_at) AS most_recent_ticket_date,      -- Last ticket creation date
  MAX(cb.chargeback_rate_90d) AS current_chargeback_rate_90d  -- Current chargeback rate
FROM `shopify-dw.risk.trust_platform_tickets_summary_v1` tp
JOIN `sdp-prd-cti-data.intermediate.shop_chargeback_rates_current` cb
  ON tp.subjectable_id = cb.shop_id
WHERE tp.created_at >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 90 DAY)
  AND cb.chargeback_rate_90d > 0.01                   -- Shops with >1% chargeback rate
  AND tp.subjectable_type = 'Shop'
GROUP BY tp.subjectable_id
HAVING ticket_count > 1                               -- Multiple tickets indicates higher risk
ORDER BY current_chargeback_rate_90d DESC             -- Sort by worst chargeback rate first
```

### Tickets with Reserve Configuration Changes

This query identifies tickets that resulted in reserve configuration changes:

```sql
-- Purpose: Find tickets that led to reserve configuration changes
-- Useful for tracking the impact of Trust Platform tickets on financial risk controls

WITH reserve_changes AS (
  SELECT
    shop_id,
    reserve_created_at,
    reserve_percentage,                               -- What percentage is being held
    reserve_hold_duration                             -- How long funds are held
  FROM `sdp-prd-cti-data.base.base__shopify_payments_reserve_configurations`
  WHERE reserve_created_at >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 90 DAY)
    AND is_active = TRUE                              -- Only current configurations
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
  AND tp.created_at <= rc.reserve_created_at          -- Ticket created before reserve change
  AND TIMESTAMP_ADD(tp.created_at, INTERVAL 1 DAY) >= rc.reserve_created_at  -- Reserve changed within 1 day of ticket
WHERE tp.subjectable_type = 'Shop'
  AND tp.created_at >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 90 DAY)
ORDER BY tp.created_at DESC
```

### Temporal Metrics by Report Group

This query analyzes the average time to first action by report group:

```sql
-- Purpose: Measure SLO performance by report group over time
-- Helps identify which report groups are meeting their response time targets

SELECT
  DATE_TRUNC(created_at, MONTH) AS month,             -- Group by month
  latest_report_group,
  COUNT(*) AS ticket_count,
  AVG(time_to_first_action) AS avg_minutes_to_first_action,  -- Average response time
  SUM(CASE WHEN meets_slo = TRUE THEN 1 ELSE 0 END) AS slo_met_count,  -- Number of tickets meeting SLO
  ROUND(SUM(CASE WHEN meets_slo = TRUE THEN 1 ELSE 0 END) / COUNT(*) * 100, 2) AS slo_met_percentage  -- SLO success rate
FROM `shopify-dw.risk.trust_platform_tickets_summary_v1`
WHERE created_at >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 180 DAY)
  AND latest_report_group IS NOT NULL
GROUP BY 1, 2
ORDER BY 1 DESC, 2                                    -- Sort by most recent month first
```

### Combining with Financial Data

This query combines Trust Platform data with financial data to analyze tickets for shops with significant GPV:

```sql
-- Purpose: Analyze Trust Platform tickets for high-value merchants
-- Identifies tickets that may have a large financial impact

SELECT
  tp.trust_platform_ticket_id,
  tp.subjectable_id AS shop_id,
  tp.created_at,
  tp.latest_report_type,
  tp.rule_group,
  gmv.gpv_usd_l90d,                                   -- 90-day processing volume
  gmv.cumulative_gpv_usd                              -- Lifetime processing volume
FROM `shopify-dw.risk.trust_platform_tickets_summary_v1` tp
JOIN `shopify-dw.finance.shop_gmv_current` gmv
  ON tp.subjectable_id = gmv.shop_id
WHERE tp.created_at >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 30 DAY)
  AND gmv.gpv_usd_l90d > 100000                       -- $100k+ in last 90 days
  AND tp.subjectable_type = 'Shop'
ORDER BY gmv.gpv_usd_l90d DESC, tp.created_at DESC    -- Highest volume shops first
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
