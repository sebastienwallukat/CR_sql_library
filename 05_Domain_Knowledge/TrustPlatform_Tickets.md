# TrustPlatform Tickets Queries 🎫

Welcome to the TrustPlatform Tickets section of our SQL Query Resource Center. This page contains datasets and queries for analyzing tickets and case management.

## Primary Datasets 📁

### `shopify-dw.risk.trust_platform_tickets_summary_v1`

This is the primary recommended dataset for all ticket analysis. It contains comprehensive ticket information with key metadata.

### `shopify-dw.risk.trust_platform_actions_summary_v1`

This dataset contains all actions taken on tickets, enabling detailed analysis of ticket workflows and resolutions.

### `shopify-dw.risk.trust_platform_ticket_escalations_v1`

This dataset captures all ticket escalation events, allowing for analysis of escalation patterns and effectiveness.

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
  COUNTIF(DATE(updated_at) = CURRENT_DATE()) as updated_today,
  AVG(TIMESTAMP_DIFF(updated_at, created_at, HOUR)) as avg_hours_open
FROM
  `shopify-dw.risk.trust_platform_tickets_summary_v1`
WHERE
  created_at >= DATE_SUB(CURRENT_DATE(), INTERVAL 30 DAY)
GROUP BY
  status
ORDER BY
  ticket_count DESC
```

### Ticket Resolution Time Analysis

```sql
WITH closed_tickets AS (
  SELECT
    ticket_id,
    created_at,
    CASE
      WHEN status = 'closed' THEN updated_at
      ELSE NULL
    END as closed_at,
    assignee,
    ticket_type,
    priority
  FROM
    `shopify-dw.risk.trust_platform_tickets_summary_v1`
  WHERE
    created_at >= DATE_SUB(CURRENT_DATE(), INTERVAL 90 DAY)
    AND status = 'closed'
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
  COUNT(DISTINCT a.ticket_id) as tickets_affected,
  AVG(TIMESTAMP_DIFF(a.created_at, t.created_at, HOUR)) as avg_hours_to_action
FROM 
  `shopify-dw.risk.trust_platform_actions_summary_v1` AS a
  JOIN `shopify-dw.risk.trust_platform_tickets_summary_v1` AS t
    ON a.ticket_id = t.ticket_id
WHERE
  DATE(a.created_at) >= DATE_SUB(CURRENT_DATE(), INTERVAL 90 DAY)
GROUP BY
  a.action_type
ORDER BY
  action_count DESC
```

### Escalation Analysis

```sql
SELECT
  e.escalation_reason,
  COUNT(*) as escalation_count,
  COUNT(DISTINCT e.ticket_id) as tickets_escalated,
  AVG(TIMESTAMP_DIFF(e.created_at, t.created_at, HOUR)) as avg_hours_to_escalation
FROM 
  `shopify-dw.risk.trust_platform_ticket_escalations_v1` AS e
  JOIN `shopify-dw.risk.trust_platform_tickets_summary_v1` AS t
    ON e.ticket_id = t.ticket_id
WHERE
  DATE(e.created_at) >= DATE_SUB(CURRENT_DATE(), INTERVAL 90 DAY)
GROUP BY
  e.escalation_reason
ORDER BY
  escalation_count DESC
```

### Assignee Workload Analysis

```sql
SELECT
  assignee,
  COUNT(*) as total_tickets,
  COUNTIF(status = 'open') as open_tickets,
  COUNTIF(status = 'in_progress') as in_progress_tickets,
  COUNTIF(status = 'waiting_for_info') as waiting_tickets,
  COUNTIF(status = 'closed' AND DATE(updated_at) = CURRENT_DATE()) as closed_today,
  AVG(CASE WHEN status = 'closed' 
      THEN TIMESTAMP_DIFF(updated_at, created_at, HOUR) 
      ELSE TIMESTAMP_DIFF(CURRENT_TIMESTAMP(), created_at, HOUR) 
    END) as avg_handling_time
FROM
  `shopify-dw.risk.trust_platform_tickets_summary_v1`
WHERE
  assignee IS NOT NULL
  AND created_at >= DATE_SUB(CURRENT_DATE(), INTERVAL 30 DAY)
GROUP BY
  assignee
ORDER BY
  total_tickets DESC
```

## Notes and Best Practices 📝

- Always use the recommended tables (`shopify-dw.risk.trust_platform_*`) as they provide the most reliable and comprehensive data
- Use ticket resolution metrics to identify bottlenecks in your workflow
- Regularly analyze action patterns to improve operational efficiency
- Track escalation reasons to identify recurring issues that may need process improvements
- Consider time-of-day patterns when analyzing ticket volumes

Feel free to contribute new queries or improve the existing ones. Together, we can build a comprehensive resource for ticket analysis and workflow optimization.

Happy ticket analyzing! 🧐 