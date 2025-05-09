# TrustPlatform Tickets Queries 🎫

Welcome to the TrustPlatform Tickets section of our SQL Query Resource Center. This page contains datasets and queries for analyzing tickets and case management within Shopify's Trust Platform.

## Table of Contents
- [Primary Datasets](#primary-datasets)
- [Legacy Datasets](#legacy-datasets-not-recommended)
- [Common Queries](#common-queries)
  - [Ticket Volume by Status](#ticket-volume-by-status)
  - [Ticket Resolution Time Analysis](#ticket-resolution-time-analysis)
  - [Actions Taken Analysis](#actions-taken-analysis)
  - [Escalation Analysis](#escalation-analysis)
  - [Assignee Workload Analysis](#assignee-workload-analysis)
- [Notes and Best Practices](#notes-and-best-practices)

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
  status,                                                   -- Current ticket status
  COUNT(*) as ticket_count,                                 -- Total number of tickets
  COUNTIF(DATE(created_at) = CURRENT_DATE()) as created_today,  -- Tickets created today
  COUNTIF(DATE(latest_action_at) = CURRENT_DATE()) as updated_today,  -- Tickets updated today
  AVG(TIMESTAMP_DIFF(latest_action_at, created_at, HOUR)) as avg_hours_open  -- Average time tickets remain open
FROM
  `shopify-dw.risk.trust_platform_tickets_summary_v1`
WHERE
  created_at >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 30 DAY)  -- Only analyze last 30 days
GROUP BY
  status
ORDER BY
  ticket_count DESC
```

#### Description
This query provides a breakdown of current tickets by their status, showing the total count, how many were created or updated today, and the average time tickets have been open. It filters for only the last 30 days of data, making it useful for monitoring current workload and team performance.

#### Example
This query returns a table showing each status (solved, new, pending, open, on_hold), with counts and timing metrics. Team leads can use this to monitor daily ticket influx and resolution rates.

### Ticket Resolution Time Analysis

```sql
WITH closed_tickets AS (
  SELECT
    trust_platform_ticket_id as ticket_id,                  -- Unique ticket identifier
    created_at,                                             -- When the ticket was created
    CASE
      WHEN status = 'solved' THEN latest_action_at          -- Consider latest action time as closure time
      ELSE NULL
    END as closed_at,
    latest_trust_platform_user_email as assignee,           -- Person assigned to ticket
    latest_report_type as ticket_type,                      -- Type of report/issue
    CASE
      WHEN rule_group = 'HIGH' THEN 'high'                  -- Converting rule group to priority level
      WHEN rule_group = 'MEDIUM' THEN 'medium'
      WHEN rule_group = 'LOW' THEN 'low'
      ELSE 'normal'
    END as priority
  FROM
    `shopify-dw.risk.trust_platform_tickets_summary_v1`
  WHERE
    created_at >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 90 DAY)  -- Last 90 days of data
    AND status = 'solved'                                   -- Only include solved tickets
)

SELECT
  ticket_type,                                              -- Type of report/issue
  priority,                                                 -- Priority level
  COUNT(*) as ticket_count,                                 -- Number of tickets
  AVG(TIMESTAMP_DIFF(closed_at, created_at, HOUR)) as avg_resolution_hours,  -- Average time to resolution
  MIN(TIMESTAMP_DIFF(closed_at, created_at, HOUR)) as min_resolution_hours,  -- Minimum time to resolution
  MAX(TIMESTAMP_DIFF(closed_at, created_at, HOUR)) as max_resolution_hours,  -- Maximum time to resolution
  APPROX_QUANTILES(TIMESTAMP_DIFF(closed_at, created_at, HOUR), 2)[OFFSET(1)] as median_resolution_hours  -- Median time to resolution
FROM
  closed_tickets
GROUP BY
  ticket_type, priority
ORDER BY
  ticket_type, priority
```

#### Description
This query analyzes the resolution time for closed tickets by ticket type and priority. It helps identify which types of issues take longer to resolve and whether prioritization is working effectively. The query first creates a CTE to identify closed tickets and their key attributes, then calculates various statistics about resolution time.

#### Example
Results show resolution metrics for different ticket types (account_opening_fraud, account_takeover, etc.) broken down by priority level, revealing patterns in how quickly different issues are handled and which might need attention to improve resolution time.

### Actions Taken Analysis

```sql
SELECT
  a.action_type,                                            -- Type of action taken
  COUNT(*) as action_count,                                 -- Total count of this action
  COUNT(DISTINCT a.actionable_id) as tickets_affected,      -- Number of unique tickets affected
  AVG(TIMESTAMP_DIFF(a.created_at, t.created_at, HOUR)) as avg_hours_to_action  -- Average time from ticket creation to action
FROM 
  `shopify-dw.risk.trust_platform_actions_summary_v1` AS a   -- Actions table
  JOIN `shopify-dw.risk.trust_platform_tickets_summary_v1` AS t  -- Tickets table
    ON a.actionable_id = t.trust_platform_ticket_id AND a.actionable_type = 'Ticket'  -- Joining conditions
WHERE
  DATE(a.created_at) >= DATE_SUB(CURRENT_DATE(), INTERVAL 90 DAY)  -- Actions in last 90 days
  AND t.created_at >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 180 DAY)  -- Tickets from last 180 days
GROUP BY
  a.action_type
ORDER BY
  action_count DESC                                         -- Most common actions first
```

#### Description
This query examines the types of actions taken on tickets, showing which actions are most common and how quickly they are performed after a ticket is created. It joins the actions table with the tickets table to connect actions to their tickets and calculate time-based metrics.

#### Example
Results might show that "terminate_fraud" is the most common action taken, affecting a large number of tickets and typically performed quickly after ticket creation, while other actions like "email_merchant" might occur less frequently and take longer on average.

### Escalation Analysis

```sql
SELECT
  e.escalation_type,                                        -- Type of escalation
  COUNT(*) as escalation_count,                             -- Total count of this escalation type
  COUNT(DISTINCT e.from_trust_platform_ticket_id) as tickets_escalated,  -- Number of tickets escalated
  AVG(TIMESTAMP_DIFF(e.escalated_at, t.created_at, HOUR)) as avg_hours_to_escalation  -- Average time to escalation
FROM 
  `shopify-dw.risk.trust_platform_ticket_escalations_v1` AS e  -- Escalations table
  JOIN `shopify-dw.risk.trust_platform_tickets_summary_v1` AS t  -- Tickets table
    ON e.from_trust_platform_ticket_id = t.trust_platform_ticket_id  -- Join condition
WHERE
  DATE(e.escalated_at) >= DATE_SUB(CURRENT_DATE(), INTERVAL 90 DAY)  -- Escalations in last 90 days
  AND t.created_at >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 180 DAY)  -- Tickets from last 180 days
GROUP BY
  e.escalation_type
ORDER BY
  escalation_count DESC                                     -- Most common escalation types first
```

#### Description
This query analyzes ticket escalations, showing which types of escalations occur most frequently and how long it typically takes for a ticket to be escalated after creation. This helps identify patterns in the escalation process and potential areas for improvement in initial ticket routing.

#### Example
Results typically show internal moves as the most common escalation type, followed by support escalations and linked tickets, with varying average times to escalation depending on the type.

### Assignee Workload Analysis

```sql
SELECT
  latest_trust_platform_user_email as assignee,             -- User assigned to ticket
  COUNT(*) as total_tickets,                                -- Total tickets assigned to user
  COUNTIF(status = 'open') as open_tickets,                 -- Currently open tickets
  COUNTIF(status = 'pending') as in_progress_tickets,       -- Tickets in progress
  COUNTIF(status = 'on_hold') as waiting_tickets,           -- Tickets on hold
  COUNTIF(status = 'solved' AND DATE(latest_action_at) = CURRENT_DATE()) as closed_today,  -- Tickets closed today
  AVG(CASE WHEN status = 'solved' 
      THEN TIMESTAMP_DIFF(latest_action_at, created_at, HOUR) 
      ELSE TIMESTAMP_DIFF(CURRENT_TIMESTAMP(), created_at, HOUR) 
    END) as avg_handling_time                               -- Average handling time
FROM
  `shopify-dw.risk.trust_platform_tickets_summary_v1`
WHERE
  latest_trust_platform_user_email IS NOT NULL              -- Only tickets with assignees
  AND created_at >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 30 DAY)  -- Last 30 days
GROUP BY
  latest_trust_platform_user_email
ORDER BY
  total_tickets DESC                                        -- Assignees with most tickets first
```

#### Description
This query provides an overview of ticket workload by assignee, showing how many tickets each person is handling, their current status distribution, and average handling time. This is useful for team leads to monitor workload distribution and identify potential bottlenecks or capacity issues.

#### Example
Results show a breakdown of each team member's current workload, including total tickets assigned, how many are in various states, and their average handling time. This can help identify if certain team members are overloaded or if there are differences in processing efficiency.

## Notes and Best Practices 📝

- Always filter by `created_at` when querying `trust_platform_tickets_summary_v1` as it's a partitioned table
- The Trust Platform tickets use different status values than traditional support tickets ('new', 'open', 'pending', 'on_hold', 'solved')
- Join between tickets and actions using `trust_platform_ticket_id = actionable_id` AND `actionable_type = 'Ticket'`
- For escalation analysis, use `from_trust_platform_ticket_id` to track the source ticket
- When analyzing resolution times, use `latest_action_at` as the effective closure timestamp
- Consider timezone differences when analyzing time-based metrics
- For large-scale analysis, consider limiting your date ranges to improve query performance

Happy ticket analyzing! 🧐 