# Trust Platform Tables

This document provides comprehensive information about Trust Platform tables used for credit risk analysis at Shopify.

## Table of Contents
- [Overview](#overview)
- [Primary Tables](#primary-tables)
  - [trust_platform_tickets_summary_v1](#shopify-dwrisktrust_platform_tickets_summary_v1)
  - [trust_platform_actions_summary_v1](#shopify-dwrisktrust_platform_actions_summary_v1)
  - [trust_platform_ticket_escalations_v1](#shopify-dwrisktrust_platform_ticket_escalations_v1)
- [How to Join Trust Platform Tables](#how-to-join-trust-platform-tables)
  - [Joining Tickets with Actions](#joining-tickets-with-actions)
  - [Joining Tickets with Escalations](#joining-tickets-with-escalations)
  - [Joining Trust Platform Data with Chargeback Data](#joining-trust-platform-data-with-chargeback-data)
  - [Joining Trust Platform Data with Shop Status](#joining-trust-platform-data-with-shop-status)
- [Data Quality Considerations](#data-quality-considerations)
- [Temporal Field Handling](#temporal-field-handling)
- [Common Analysis Patterns](#common-analysis-patterns)
  - [Risk Action Impact Analysis](#risk-action-impact-analysis)
  - [Ticket Resolution Analysis](#ticket-resolution-analysis)
  - [Escalation Analysis](#escalation-analysis)

## Overview

Trust Platform tables contain data about risk-related investigations, tickets, and actions taken to mitigate risk. These tables are essential for understanding risk escalations, case management, and the history of risk-related actions taken on merchant accounts.

## Primary Tables

### shopify-dw.risk.trust_platform_tickets_summary_v1

**Description**:  
This is the primary table for Trust Platform tickets data, providing a comprehensive view of risk investigations and actions. It includes details about ticket creation, status, risk level, and outcomes.

**Update Frequency**: Hourly

**Primary Keys**: `trust_platform_ticket_id`

**Usage Notes**:
- Critical table for tracking risk investigations and actions
- `status` is essential for filtering active vs. resolved cases
- `rule_group` often indicates severity and prioritization
- `latest_action_type` shows outcomes like account disablement, reserve application, etc.
- The table is partitioned by `created_at` - include date filters for better performance

**Example Query**:
```sql
-- Find recent high-risk tickets with actions taken
SELECT 
  trust_platform_ticket_id,
  subjectable_id AS shop_id,
  created_at,
  latest_report_type AS ticket_type,
  rule_group AS risk_level,
  status AS ticket_status,
  latest_action_type AS action_taken
FROM `shopify-dw.risk.trust_platform_tickets_summary_v1`
WHERE created_at >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 30 DAY)
  AND rule_group = 'HIGH'
  AND latest_action_type IS NOT NULL
ORDER BY created_at DESC
```

### shopify-dw.risk.trust_platform_actions_summary_v1

**Description**:  
This table records specific actions taken within the Trust Platform, such as account disablements, reserve applications, and other risk mitigation measures.

**Update Frequency**: Hourly

**Primary Keys**: `trust_platform_action_id`

**Foreign Keys**:
- `actionable_id` (when `actionable_type` = 'Ticket') references `shopify-dw.risk.trust_platform_tickets_summary_v1.trust_platform_ticket_id`
- `subject_id` references `shopify-dw.shopify.shops.id` (when `subject_type` = 'Shop')

**Usage Notes**:
- Primary source for understanding risk mitigation actions
- `action_type` includes critical actions like:
  - `terminate_fraud`
  - `ach_banking_limit`
  - `enable_email_throttle`
  - `email_merchant`
  - `inconclusive`
- `is_automated` helps distinguish between system-initiated and manual actions
- `completed_tasks` contains action-specific information about tasks performed

**Example Query**:
```sql
-- Analyze action types and their frequency
SELECT 
  action_type,
  COUNT(*) AS action_count,
  COUNTIF(is_automated) AS automated_count,
  COUNTIF(NOT is_automated) AS manual_count,
  COUNTIF(is_automated) / COUNT(*) AS automation_rate
FROM `shopify-dw.risk.trust_platform_actions_summary_v1`
WHERE created_at >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 90 DAY)
GROUP BY action_type
ORDER BY action_count DESC
```

### shopify-dw.risk.trust_platform_ticket_escalations_v1

**Description**:  
This table contains information about escalated Trust Platform tickets, tracking when and how tickets are escalated between different teams. It captures details about both the source and destination tickets in the escalation process, with each ticket potentially being escalated multiple times.

**Update Frequency**: Daily

**Primary Keys**: `trust_platform_activity_id`

**Foreign Keys**:
- `from_trust_platform_ticket_id` references `shopify-dw.risk.trust_platform_tickets_summary_v1.trust_platform_ticket_id`
- `to_trust_platform_ticket_id` references `shopify-dw.risk.trust_platform_tickets_summary_v1.trust_platform_ticket_id` (when applicable)

**Usage Notes**:
- Tracks the flow of tickets between different teams
- Useful for understanding escalation patterns and workflows
- `escalation_type` indicates the nature of the escalation
- Not all tickets escalated to support are provided with an `external_ticket_id`
- Can be joined with the main tickets table to analyze escalation impact
- Helps in analyzing team workflows and collaboration patterns

**Example Query**:
```sql
-- Get counts of escalations by type and source report group
SELECT 
  escalation_type,
  from_report_group,
  COUNT(*) AS escalation_count,
  COUNT(DISTINCT from_trust_platform_ticket_id) AS unique_tickets_count,
  AVG(TIMESTAMP_DIFF(CURRENT_TIMESTAMP(), escalated_at, DAY)) AS avg_days_since_escalation
FROM `shopify-dw.risk.trust_platform_ticket_escalations_v1`
WHERE escalated_at >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 90 DAY)
GROUP BY escalation_type, from_report_group
ORDER BY escalation_count DESC
```

## How to Join Trust Platform Tables

### Joining Tickets with Actions

```sql
-- Join tickets with associated actions
SELECT 
  t.trust_platform_ticket_id,
  t.subjectable_id AS shop_id,
  t.created_at AS ticket_created_at,
  t.latest_report_type AS ticket_type,
  t.rule_group AS risk_level,
  a.trust_platform_action_id,
  a.action_type,
  a.created_at AS action_created_at,
  a.status AS action_status,
  a.is_automated
FROM `shopify-dw.risk.trust_platform_tickets_summary_v1` t
LEFT JOIN `shopify-dw.risk.trust_platform_actions_summary_v1` a
  ON t.trust_platform_ticket_id = a.actionable_id 
  AND a.actionable_type = 'Ticket'
WHERE t.created_at >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 90 DAY)
ORDER BY t.created_at DESC, a.created_at
```

**Description**:
This query combines ticket data with associated actions, allowing you to see what specific actions were taken for each ticket. The LEFT JOIN ensures all tickets are included, even those without actions. The results show ticket details alongside any actions that were taken, including timing and automation status.

**Example**:
This join would show, for instance, when a high-risk ticket was created and what specific action was taken (like account termination or email throttling), whether it was automated or manual, and the timing between ticket creation and action.

### Joining Tickets with Escalations

```sql
-- Join tickets with their escalation history
SELECT 
  t.trust_platform_ticket_id,
  t.subjectable_id AS shop_id,
  t.created_at AS ticket_created_at,
  t.latest_report_type AS ticket_type,
  t.rule_group AS risk_level,
  t.status AS ticket_status,
  e.escalated_at,
  e.escalation_type,
  e.from_report_group,
  e.to_report_group,
  e.to_external_ticket_id,
  TIMESTAMP_DIFF(e.escalated_at, t.created_at, HOUR) AS hours_to_escalation
FROM `shopify-dw.risk.trust_platform_tickets_summary_v1` t
JOIN `shopify-dw.risk.trust_platform_ticket_escalations_v1` e
  ON t.trust_platform_ticket_id = e.from_trust_platform_ticket_id
WHERE t.created_at >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 90 DAY)
ORDER BY t.created_at DESC, e.escalated_at
```

**Description**:
This query links tickets with their escalation history, showing how tickets move between teams. The INNER JOIN returns only tickets that have been escalated. The calculated field `hours_to_escalation` helps measure response time between ticket creation and escalation.

**Example**:
The results would show, for example, that a fraud ticket was escalated to the support team 3 hours after creation, with the specific escalation type and destination team clearly identified.

### Joining Trust Platform Data with Chargeback Data

```sql
-- Join chargebacks with related Trust Platform tickets
WITH chargeback_shops AS (
  SELECT 
    shop_id,
    COUNT(*) AS chargeback_count,
    SUM(chargeback_amount) AS chargeback_amount
  FROM `shopify-dw.money_products.chargebacks_summary`
  WHERE provider_chargeback_created_at >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 90 DAY)
  GROUP BY shop_id
  HAVING COUNT(*) >= 5
)

SELECT 
  c.shop_id,
  c.chargeback_count,
  c.chargeback_amount,
  COUNT(DISTINCT t.trust_platform_ticket_id) AS ticket_count,
  COUNTIF(t.status = 'OPEN') AS open_ticket_count,
  COUNTIF(a.action_type = 'terminate_fraud') AS account_disable_count,
  COUNTIF(a.has_completed_reserves_task) AS reserve_applied_count
FROM chargeback_shops c
LEFT JOIN `shopify-dw.risk.trust_platform_tickets_summary_v1` t
  ON c.shop_id = t.subjectable_id
  AND t.subjectable_type = 'Shop'
  AND t.created_at >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 90 DAY)
LEFT JOIN `shopify-dw.risk.trust_platform_actions_summary_v1` a
  ON t.trust_platform_ticket_id = a.actionable_id
  AND a.actionable_type = 'Ticket'
GROUP BY c.shop_id, c.chargeback_count, c.chargeback_amount
ORDER BY c.chargeback_count DESC
```

**Description**:
This query identifies shops with multiple chargebacks and correlates them with Trust Platform tickets and actions. The CTE first filters to shops with at least 5 chargebacks in the past 90 days. These shops are then joined with ticket and action data to analyze how Trust Platform has responded to potential risk.

**Example**:
The results would reveal shops with high chargeback activity and how many have tickets created, whether those tickets are still open, and what specific actions were taken (account disablement or reserve application).

### Joining Trust Platform Data with Shop Status

```sql
-- Analyze shops with Trust Platform tickets by current status
SELECT 
  s.shopify_payments_status,
  COUNT(DISTINCT t.subjectable_id) AS shop_count,
  COUNT(DISTINCT t.trust_platform_ticket_id) AS ticket_count,
  COUNT(DISTINCT CASE WHEN t.rule_group = 'HIGH' THEN t.trust_platform_ticket_id END) AS high_risk_ticket_count,
  COUNT(DISTINCT CASE WHEN a.has_completed_termination_task THEN a.trust_platform_action_id END) AS disable_action_count
FROM `shopify-dw.risk.trust_platform_tickets_summary_v1` t
LEFT JOIN `sdp-prd-cti-data.intermediate.shop_current_shopify_payments_status` s
  ON t.subjectable_id = s.shop_id
  AND t.subjectable_type = 'Shop'
LEFT JOIN `shopify-dw.risk.trust_platform_actions_summary_v1` a
  ON t.trust_platform_ticket_id = a.actionable_id
  AND a.actionable_type = 'Ticket'
WHERE t.created_at >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 90 DAY)
GROUP BY s.shopify_payments_status
ORDER BY shop_count DESC
```

**Description**:
This query segments Trust Platform tickets by the shop's current Shopify Payments status. It provides a breakdown of how many shops have tickets in each payment status category, how many tickets they have, and how many resulted in account disablement actions.

**Example**:
The results would show, for instance, that a significant number of tickets belong to shops with a "rejected - fraud" Shopify Payments status, or that shops with "active" status have fewer disablement actions relative to their ticket count.

## Data Quality Considerations

1. **Ticket Status Transitions**:
   - Tickets move through different statuses (OPEN, PENDING, CLOSED)
   - Always check the current status when analyzing active cases
   - Consider status transition timestamps for timing analysis

2. **Action Execution Status**:
   - Actions may be created but not yet executed
   - `created_at` and `updated_at` indicate the timing of the action
   - `status` provides the current state of the action

3. **Historical vs. Current View**:
   - Some tables provide historical snapshots while others show current state
   - Be careful when joining historical tickets with current shop status
   - Consider temporal relationships when analyzing cause and effect

4. **Deleted Tickets**:
   - Deleted tickets are removed from the output
   - Deletion updates aren't received until a full re-import occurs on the source
   - Tickets that have been deleted may still appear for up to one week

5. **Access Restrictions**:
   - Trust Platform data may have stricter access controls
   - Certain fields may be redacted or NULL based on permissions
   - Ensure you have appropriate access before analysis

## Temporal Field Handling

### Important Timestamp Fields

All time-based fields in Trust Platform tables are TIMESTAMP type, which require specific timestamp functions for comparison:

| Table Name | Timestamp Field | Data Type | Proper Comparison Example |
|------------|----------------|-----------|---------------------------|
| trust_platform_tickets_summary_v1 | created_at | TIMESTAMP | `WHERE created_at >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 30 DAY)` |
| trust_platform_tickets_summary_v1 | latest_ticket_solved_at | TIMESTAMP | `WHERE latest_ticket_solved_at IS NOT NULL` |
| trust_platform_actions_summary_v1 | created_at | TIMESTAMP | `WHERE created_at >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 90 DAY)` |
| trust_platform_actions_summary_v1 | updated_at | TIMESTAMP | `WHERE updated_at IS NOT NULL` |
| trust_platform_ticket_escalations_v1 | escalated_at | TIMESTAMP | `WHERE escalated_at >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 90 DAY)` |
| money_products.chargebacks_summary | provider_chargeback_created_at | TIMESTAMP | `WHERE provider_chargeback_created_at >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 90 DAY)` |

### Common Timestamp Functions

- **For filtering data within recent time periods**:
  ```sql
  -- Last 30 days of data (correct)
  WHERE created_at >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 30 DAY)
  
  -- Incorrect approach (will fail with type mismatch)
  WHERE created_at >= DATE_SUB(CURRENT_DATE(), INTERVAL 30 DAY)
  ```

- **For calculating time differences**:
  ```sql
  -- Hours between two timestamps (correct)
  TIMESTAMP_DIFF(end_timestamp, start_timestamp, HOUR) AS hours_elapsed
  
  -- Days between two timestamps (correct)
  TIMESTAMP_DIFF(end_timestamp, start_timestamp, DAY) AS days_elapsed
  ```

- **For comparing with DATE fields (when needed)**:
  ```sql
  -- Converting TIMESTAMP to DATE for comparison with DATE fields
  WHERE DATE(timestamp_field) = DATE_SUB(CURRENT_DATE(), INTERVAL 7 DAY)
  ```

- **For date ranges with TIMESTAMPs**:
  ```sql
  -- Date range with timestamps (correct)
  WHERE created_at BETWEEN 
    TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 30 DAY) AND
    CURRENT_TIMESTAMP()
  ```

### Best Practices

1. **Always verify column data types** before writing comparisons
2. **Use TIMESTAMP functions with TIMESTAMP columns**
3. **Use DATE functions with DATE columns**
4. If mixing types is necessary, explicitly convert using `DATE()` or `TIMESTAMP()`
5. For better query performance, prefer filtering on partitioned columns with appropriate functions

## Common Analysis Patterns

### Risk Action Impact Analysis

```sql
-- Analyze impact of reserve actions on chargeback rates
WITH reserve_actions AS (
  SELECT 
    subject_id AS shop_id,
    MIN(created_at) AS first_reserve_applied_at
  FROM `shopify-dw.risk.trust_platform_actions_summary_v1`
  WHERE has_completed_reserves_task = TRUE
    AND status = 'SUCCEEDED'
    AND subject_type = 'Shop'
  GROUP BY subject_id
),

daily_chargebacks AS (
  SELECT 
    c.shop_id,
    DATE(c.provider_chargeback_created_at) AS date,
    COUNT(*) AS daily_chargeback_count
  FROM `shopify-dw.money_products.chargebacks_summary` c
  JOIN reserve_actions r ON c.shop_id = r.shop_id
  WHERE c.provider_chargeback_created_at BETWEEN 
    TIMESTAMP_SUB(r.first_reserve_applied_at, INTERVAL 30 DAY) AND
    TIMESTAMP_ADD(r.first_reserve_applied_at, INTERVAL 30 DAY)
  GROUP BY c.shop_id, DATE(c.provider_chargeback_created_at)
)

SELECT 
  d.shop_id,
  r.first_reserve_applied_at,
  d.date,
  d.daily_chargeback_count,
  CASE
    WHEN d.date < DATE(r.first_reserve_applied_at) THEN 'Before Reserve'
    ELSE 'After Reserve'
  END AS period
FROM daily_chargebacks d
JOIN reserve_actions r ON d.shop_id = r.shop_id
ORDER BY d.shop_id, d.date
```

**Description**:
This analysis evaluates the impact of reserve actions on chargeback rates. It identifies when reserve actions were first applied to shops, then collects chargeback data for the 30 days before and after the reserve application. The results are categorized as "Before Reserve" or "After Reserve" to facilitate comparison.

**Example**:
This analysis would show whether applying a reserve to a merchant account successfully reduced the frequency of chargebacks, providing evidence of the effectiveness of this risk mitigation strategy.

### Ticket Resolution Analysis

```sql
-- Analyze ticket resolution times and outcomes
WITH ticket_lifecycle AS (
  SELECT 
    trust_platform_ticket_id,
    subjectable_id AS shop_id,
    latest_report_type AS ticket_type,
    rule_group AS risk_level,
    created_at,
    latest_ticket_solved_at AS closed_at,
    status AS ticket_status,
    CASE 
      WHEN status = 'CLOSED' THEN 'RESOLVED'
      ELSE status
    END AS resolution,
    TIMESTAMP_DIFF(latest_ticket_solved_at, created_at, HOUR) AS resolution_time_hours
  FROM `shopify-dw.risk.trust_platform_tickets_summary_v1`
  WHERE created_at >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 90 DAY)
    AND latest_ticket_solved_at IS NOT NULL
)

SELECT 
  ticket_type,
  risk_level,
  resolution,
  COUNT(*) AS ticket_count,
  AVG(resolution_time_hours) AS avg_resolution_time_hours,
  MIN(resolution_time_hours) AS min_resolution_time_hours,
  MAX(resolution_time_hours) AS max_resolution_time_hours,
  APPROX_QUANTILES(resolution_time_hours, 100)[OFFSET(50)] AS median_resolution_time_hours
FROM ticket_lifecycle
GROUP BY ticket_type, risk_level, resolution
ORDER BY ticket_type, risk_level, ticket_count DESC
```

**Description**:
This analysis examines ticket resolution patterns, focusing on the time it takes to resolve different types of tickets. It calculates various statistics about resolution time (average, minimum, maximum, median) broken down by ticket type, risk level, and resolution status.

**Example**:
The results would reveal, for instance, that high-risk fraud tickets take an average of X hours to resolve, while merchant verification tickets take Y hours, helping to identify potential bottlenecks in workflow processes.

### Escalation Analysis

```sql
-- Analyze escalation patterns and impact
WITH escalation_metrics AS (
  SELECT 
    t.trust_platform_ticket_id,
    t.subjectable_id AS shop_id,
    t.created_at AS ticket_created_at,
    t.latest_report_type AS ticket_type,
    t.rule_group AS risk_level,
    t.status AS ticket_status,
    MIN(e.escalated_at) AS first_escalation_at,
    COUNT(e.trust_platform_activity_id) AS escalation_count,
    ARRAY_AGG(DISTINCT e.escalation_type ORDER BY e.escalated_at) AS escalation_types,
    ARRAY_AGG(DISTINCT e.from_report_group ORDER BY e.escalated_at) AS source_groups,
    ARRAY_AGG(DISTINCT e.to_report_group ORDER BY e.escalated_at) AS destination_groups
  FROM `shopify-dw.risk.trust_platform_tickets_summary_v1` t
  LEFT JOIN `shopify-dw.risk.trust_platform_ticket_escalations_v1` e
    ON t.trust_platform_ticket_id = e.from_trust_platform_ticket_id
  WHERE t.created_at >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 90 DAY)
  GROUP BY t.trust_platform_ticket_id, t.subjectable_id, t.created_at, t.latest_report_type, t.rule_group, t.status
)

SELECT 
  ticket_type,
  risk_level,
  CASE 
    WHEN escalation_count > 0 THEN 'Escalated'
    ELSE 'Not Escalated'
  END AS escalation_status,
  COUNT(*) AS ticket_count,
  AVG(CASE WHEN first_escalation_at IS NOT NULL 
           THEN TIMESTAMP_DIFF(first_escalation_at, ticket_created_at, HOUR) 
           ELSE NULL END) AS avg_hours_to_first_escalation,
  AVG(escalation_count) AS avg_escalations_per_ticket,
  COUNTIF(ticket_status = 'CLOSED') AS closed_tickets,
  COUNTIF(ticket_status = 'CLOSED') / COUNT(*) AS closure_rate,
  AVG(CASE WHEN ticket_status = 'CLOSED' AND first_escalation_at IS NOT NULL
           THEN TIMESTAMP_DIFF(t.latest_ticket_solved_at, first_escalation_at, HOUR)
           ELSE NULL END) AS avg_hours_to_resolve_after_escalation
FROM escalation_metrics
LEFT JOIN `shopify-dw.risk.trust_platform_tickets_summary_v1` t 
  ON escalation_metrics.trust_platform_ticket_id = t.trust_platform_ticket_id
GROUP BY ticket_type, risk_level, escalation_status
ORDER BY ticket_type, risk_level, escalation_status
```

**Description**:
This comprehensive analysis examines how ticket escalations affect resolution patterns. It identifies if and when tickets are escalated, how many times they're escalated, and how this impacts resolution timing. The query calculates metrics like average time to first escalation, average escalations per ticket, and closure rates for escalated versus non-escalated tickets.

**Example**:
The results might show that tickets escalated between teams take longer to resolve but have higher closure rates, or that certain ticket types are escalated more frequently than others, helping to optimize workflow and team coordination.

---
