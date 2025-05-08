# TrustPlatform Tickets Queries 🎫

Welcome to the TrustPlatform Tickets section of our SQL Query Resource Center. This page contains datasets and queries for analyzing tickets and case management.

## Primary Datasets 📁

### `shopify-dw.risk.trust_platform_tickets_summary_v1`

This is the primary recommended dataset for all ticket analysis. It contains comprehensive ticket information with key metadata including creation dates, statuses, report types, and ticket resolution details.

### `shopify-dw.risk.trust_platform_actions_summary_v1`

This dataset contains all actions taken on tickets, enabling detailed analysis of ticket workflows and resolutions, including action types, timestamps, and completion statuses.

### `shopify-dw.risk.trust_platform_ticket_escalations_v1`

This dataset captures all ticket escalation events, allowing for analysis of escalation patterns and effectiveness between teams.

## Legacy Datasets (Not Recommended)

### `sdp-prd-cti-data.base.base__tickets`

This older dataset is no longer recommended. Please use the primary datasets above for more reliable and comprehensive analysis.

### `sdp-prd-cti-data.base.base__sensitive_reports`

This dataset contains legacy reports data and should be replaced with the primary datasets listed above.

## Common Queries 💻

### Ticket Volume by Status

```sql
SELECT
  status,
  COUNT(*) as ticket_count,
  COUNTIF(DATE(created_at) = CURRENT_DATE()) as created_today,
  COUNTIF(DATE(latest_action_at) = CURRENT_DATE()) as updated_today,
  AVG(TIMESTAMP_DIFF(latest_action_at, created_at, HOUR)) as avg_hours_open
FROM
  `shopify-dw.risk.trust_platform_tickets_summary_v1`
WHERE
  created_at >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 30 DAY)
GROUP BY
  status
ORDER BY
  ticket_count DESC
```

### Ticket Resolution Time Analysis

```sql
WITH closed_tickets AS (
  SELECT
    trust_platform_ticket_id as ticket_id,
    created_at,
    CASE
      WHEN status = 'solved' THEN latest_action_at
      ELSE NULL
    END as closed_at,
    latest_trust_platform_user_email as assignee,
    latest_report_type as ticket_type,
    CASE
      WHEN rule_group = 'HIGH' THEN 'high'
      WHEN rule_group = 'MEDIUM' THEN 'medium'
      WHEN rule_group = 'LOW' THEN 'low'
      ELSE 'normal'
    END as priority
  FROM
    `shopify-dw.risk.trust_platform_tickets_summary_v1`
  WHERE
    created_at >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 90 DAY)
    AND status = 'solved'
)

SELECT
  ticket_type,
  priority,
  COUNT(*) as ticket_count,
  AVG(TIMESTAMP_DIFF(closed_at, created_at, HOUR)) as avg_resolution_hours,
  MIN(TIMESTAMP_DIFF(closed_at, created_at, HOUR)) as min_resolution_hours,
  MAX(TIMESTAMP_DIFF(closed_at, created_at, HOUR)) as max_resolution_hours,
  APPROX_QUANTILES(TIMESTAMP_DIFF(closed_at, created_at, HOUR), 2)[OFFSET(1)] as median_resolution_hours
FROM
  closed_tickets
GROUP BY
  ticket_type, priority
ORDER BY
  ticket_type, priority
```

### Actions Taken Analysis

```sql
SELECT
  a.action_type,
  COUNT(*) as action_count,
  COUNT(DISTINCT a.actionable_id) as tickets_affected,
  AVG(TIMESTAMP_DIFF(a.created_at, t.created_at, HOUR)) as avg_hours_to_action
FROM 
  `shopify-dw.risk.trust_platform_actions_summary_v1` AS a
  JOIN `shopify-dw.risk.trust_platform_tickets_summary_v1` AS t
    ON a.actionable_id = t.trust_platform_ticket_id AND a.actionable_type = 'Ticket'
WHERE
  DATE(a.created_at) >= DATE_SUB(CURRENT_DATE(), INTERVAL 90 DAY)
  AND t.created_at >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 180 DAY)
GROUP BY
  a.action_type
ORDER BY
  action_count DESC
```

### Escalation Analysis

```sql
SELECT
  e.escalation_type,
  COUNT(*) as escalation_count,
  COUNT(DISTINCT e.from_trust_platform_ticket_id) as tickets_escalated,
  AVG(TIMESTAMP_DIFF(e.escalated_at, t.created_at, HOUR)) as avg_hours_to_escalation
FROM 
  `shopify-dw.risk.trust_platform_ticket_escalations_v1` AS e
  JOIN `shopify-dw.risk.trust_platform_tickets_summary_v1` AS t
    ON e.from_trust_platform_ticket_id = t.trust_platform_ticket_id
WHERE
  DATE(e.escalated_at) >= DATE_SUB(CURRENT_DATE(), INTERVAL 90 DAY)
  AND t.created_at >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 180 DAY)
GROUP BY
  e.escalation_type
ORDER BY
  escalation_count DESC
```

### Assignee Workload Analysis

```sql
SELECT
  latest_trust_platform_user_email as assignee,
  COUNT(*) as total_tickets,
  COUNTIF(status = 'open') as open_tickets,
  COUNTIF(status = 'pending') as in_progress_tickets,
  COUNTIF(status = 'on_hold') as waiting_tickets,
  COUNTIF(status = 'solved' AND DATE(latest_action_at) = CURRENT_DATE()) as closed_today,
  AVG(CASE WHEN status = 'solved' 
      THEN TIMESTAMP_DIFF(latest_action_at, created_at, HOUR) 
      ELSE TIMESTAMP_DIFF(CURRENT_TIMESTAMP(), created_at, HOUR) 
    END) as avg_handling_time
FROM
  `shopify-dw.risk.trust_platform_tickets_summary_v1`
WHERE
  latest_trust_platform_user_email IS NOT NULL
  AND created_at >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 30 DAY)
GROUP BY
  latest_trust_platform_user_email
ORDER BY
  total_tickets DESC
```

## Notes and Best Practices 📝

- Always filter by `created_at` when querying `trust_platform_tickets_summary_v1` as it's a partitioned table
- The Trust Platform tickets use different status values than traditional support tickets ('new', 'open', 'pending', 'on_hold', 'solved')
- Join between tickets and actions using `trust_platform_ticket_id = actionable_id` AND `actionable_type = 'Ticket'`
- For escalation analysis, use `from_trust_platform_ticket_id` to track the source ticket
- When analyzing resolution times, use `latest_action_at` as the effective closure timestamp
- Consider timezone differences when analyzing time-based metrics

Feel free to contribute new queries or improve the existing ones. Together, we can build a comprehensive resource for ticket analysis and workflow optimization.

Happy ticket analyzing! 🧐 