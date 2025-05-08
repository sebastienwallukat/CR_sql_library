# Trust Platform Tables

This document provides comprehensive information about Trust Platform tables used for credit risk analysis at Shopify.

## Overview

Trust Platform tables contain data about risk-related investigations, tickets, and actions taken to mitigate risk. These tables are essential for understanding risk escalations, case management, and the history of risk-related actions taken on merchant accounts.

## Primary Tables

### shopify-dw.risk.trust_platform_tickets_summary_v1

**Description**:  
This is the primary table for Trust Platform tickets data, providing a comprehensive view of risk investigations and actions. It includes details about ticket creation, status, risk level, and outcomes.

**Update Frequency**: Hourly

**Primary Keys**: `trust_platform_ticket_id`

**Schema**:

| Field Name | Type | Description |
|------------|------|-------------|
| `trust_platform_ticket_id` | INTEGER | Unique identifier for the ticket |
| `created_at` | TIMESTAMP | When the ticket was created |
| `status` | STRING | Current status (e.g., 'OPEN', 'CLOSED', 'PENDING') |
| `requester_id` | INTEGER | Unique ID of the Trust Platform Report requester |
| `latest_report_group` | STRING | Operational team for the report |
| `latest_report_type` | STRING | Type of ticket (e.g., 'risk_review', 'chargeback_investigation') |
| `email_report_group` | STRING | CTI program group based on email address |
| `latest_report_group_assigned_at` | TIMESTAMP | When the latest report group was assigned |
| `trust_platform_report_id` | INTEGER | ID of the associated report |
| `detection_identifier` | STRING | Identifier of what triggered a rule event |
| `is_ticket_assigned` | BOOLEAN | Whether the ticket is assigned to a user |
| `first_user_assigned_at` | TIMESTAMP | When the first user was assigned to the ticket |
| `first_trust_platform_user_email` | STRING | Email of first user assigned to the ticket |
| `latest_user_assigned_at` | TIMESTAMP | When the latest user was assigned to the ticket |
| `latest_trust_platform_user_email` | STRING | Email of latest user assigned to the ticket |
| `first_action_at` | TIMESTAMP | When the first action was taken on the ticket |
| `first_action_type` | STRING | Type of the first action taken |
| `first_trust_platform_action_id` | INTEGER | ID of the first action taken |
| `latest_action_at` | TIMESTAMP | When the latest action was taken |
| `latest_action_type` | STRING | Latest action taken as a result of the ticket |
| `latest_action_taken_user_email` | STRING | Email of the user who took the latest action |
| `latest_decision_mechanic` | STRING | Mechanism used to apply the decision |
| `latest_decision_classification` | STRING | Classification of decision status |
| `latest_ticket_solved_at` | TIMESTAMP | When the ticket was marked as solved |
| `subjectable_id` | INTEGER | Shop ID or other subject ID associated with the ticket |
| `subjectable_type` | STRING | Type of subject (e.g., 'Shop', 'Partner', 'ShopUser', 'IdentityAccount') |
| `trust_platform_signal_id` | STRING | ID of the signal that led to ticket creation |
| `source` | STRING | Source of the report |
| `tags` | ARRAY<STRING> | List of tags applied to the ticket |
| `first_email_sent_at` | TIMESTAMP | When the first email was sent to the subject |
| `time_to_first_action` | INTEGER | Minutes from ticket creation to first action |
| `time_to_latest_action` | INTEGER | Minutes from ticket creation to latest action |
| `time_from_first_assignment_to_first_action` | INTEGER | Minutes from first assignment to first action |
| `time_from_first_assignment_to_latest_ticket_solve` | INTEGER | Minutes from first assignment to ticket solve |
| `time_from_ticket_creation_to_latest_ticket_solve` | INTEGER | Minutes from ticket creation to ticket solve |
| `rule_group` | STRING | Risk level or group the rule belongs to (e.g., 'HIGH', 'MEDIUM', 'LOW') |
| `rule_subgroup` | STRING | More specific risk categorization |
| `rule_model_name` | STRING | Name of the rule model |
| `total_actions` | INTEGER | Total number of actions taken on the ticket |
| `meets_slo` | BOOLEAN | Whether the ticket met its assigned SLO |
| `has_completed_termination_task` | BOOLEAN | Whether the ticket has completed termination task |
| `has_completed_reserves_task` | BOOLEAN | Whether the ticket has completed reserves task |
| `has_completed_shopify_payments_rejected_task` | BOOLEAN | Whether the ticket has completed payments rejected task |
| `model_version` | STRING | Version of the data model |
| `_max_processed_at` | TIMESTAMP | Latest processing timestamp |

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

**Schema**:

| Field Name | Type | Description |
|------------|------|-------------|
| `trust_platform_action_id` | INTEGER | Unique identifier for the action |
| `created_at` | TIMESTAMP | When the action was created |
| `updated_at` | TIMESTAMP | When the action was last updated |
| `actionable_id` | INTEGER | ID of the entity this action is associated with |
| `actionable_type` | STRING | Type of entity (e.g., 'Ticket', 'BulkProcess') |
| `subject_id` | INTEGER | Shop ID or other subject the action was taken on |
| `subject_type` | STRING | Type of subject (e.g., 'Shop', 'Partner', 'ShopUser') |
| `is_automated` | BOOLEAN | Whether the action was automated |
| `classification` | STRING | Decision classification |
| `trust_platform_signal_id` | STRING | ID of the associated signal |
| `action_type` | STRING | Type of action taken |
| `trust_platform_user_id` | INTEGER | ID of the user who initiated the action |
| `trust_platform_user_email` | STRING | Email of user who created or executed the action |
| `status` | STRING | Status of the action |
| `action_configuration_id` | INTEGER | Identifier for action sequences |
| `trust_platform_decision_id` | INTEGER | ID of the associated decision |
| `decision_report_group` | STRING | Report group of the decision |
| `decision_report_type` | STRING | Report type of the decision |
| `decision_mechanic` | STRING | Mechanic used for the decision |
| `report_type` | STRING | Type of report associated with the action |
| `report_group` | STRING | Team/group responsible for the action |
| `has_completed_termination_task` | BOOLEAN | Whether the action includes a completed termination task |
| `has_completed_restoration_task` | BOOLEAN | Whether the action includes a completed restoration task |
| `has_completed_shopify_payments_rejected_task` | BOOLEAN | Whether the action includes a completed payments rejected task |
| `has_completed_shopify_payments_payouts_disabled_task` | BOOLEAN | Whether the action includes a completed payouts disabled task |
| `has_completed_charges_disabled_task` | BOOLEAN | Whether the action includes a completed charges disabled task |
| `has_completed_reserves_task` | BOOLEAN | Whether the action includes a completed reserves task |
| `has_completed_request_documentation_task` | BOOLEAN | Whether the action includes a completed documentation task |
| `has_completed_email_task` | BOOLEAN | Whether the action includes a completed email task |
| `completed_tasks` | REPEATED RECORD | Details of completed tasks |
| `message_template_ids` | REPEATED STRING | IDs of message templates used |
| `rule_group` | STRING | Group the rule belongs to |
| `rule_subgroup` | STRING | Subgroup the rule belongs to |
| `rule_model_name` | STRING | Name of the rule model |
| `model_version` | STRING | Version of the data model |
| `_max_processed_at` | TIMESTAMP | Latest processing timestamp |

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

**Schema**:

| Field Name | Type | Description |
|------------|------|-------------|
| `trust_platform_activity_id` | INTEGER | Unique identifier for the escalation activity |
| `from_trust_platform_ticket_id` | INTEGER | Source ticket ID being escalated |
| `escalated_at` | TIMESTAMP | When the escalation occurred |
| `escalation_type` | STRING | Type of escalation |
| `from_report_group` | STRING | Source team/group that escalated the ticket |
| `from_email_report_group` | STRING | Email group of the source team |
| `from_report_type` | STRING | Report type of the source ticket |
| `from_trust_platform_user_email` | STRING | Email of the user who escalated the ticket |
| `to_trust_platform_ticket_id` | INTEGER | Destination ticket ID (if applicable) |
| `to_external_ticket_id` | INTEGER | External ticket ID (e.g., for support tickets) |
| `to_report_group` | STRING | Destination team/group receiving the escalation |
| `to_report_type` | STRING | Report type of the destination ticket |
| `to_trust_platform_user_email` | STRING | Email of the user receiving the escalation |
| `model_version` | STRING | Version of the data model |
| `_max_processed_at` | TIMESTAMP | Latest processing timestamp |

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

### Joining Trust Platform Data with Chargeback Data

```sql
-- Join chargebacks with related Trust Platform tickets
WITH chargeback_shops AS (
  SELECT 
    shop_id,
    COUNT(*) AS chargeback_count,
    SUM(amount) AS chargeback_amount
  FROM `shopify-dw.money_products.chargebacks_summary`
  WHERE created_at >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 90 DAY)
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
LEFT JOIN `shopify-dw.risk.shop_current_shopify_payments_status` s
  ON t.subjectable_id = s.shop_id
  AND t.subjectable_type = 'Shop'
LEFT JOIN `shopify-dw.risk.trust_platform_actions_summary_v1` a
  ON t.trust_platform_ticket_id = a.actionable_id
  AND a.actionable_type = 'Ticket'
WHERE t.created_at >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 90 DAY)
GROUP BY s.shopify_payments_status
ORDER BY shop_count DESC
```

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
| money_products.chargebacks_summary | created_at | TIMESTAMP | `WHERE created_at >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 90 DAY)` |

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
    DATE(c.created_at) AS date,
    COUNT(*) AS daily_chargeback_count
  FROM `shopify-dw.money_products.chargebacks_summary` c
  JOIN reserve_actions r ON c.shop_id = r.shop_id
  WHERE c.created_at BETWEEN 
    TIMESTAMP_SUB(r.first_reserve_applied_at, INTERVAL 30 DAY) AND
    TIMESTAMP_ADD(r.first_reserve_applied_at, INTERVAL 30 DAY)
  GROUP BY c.shop_id, DATE(c.created_at)
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

---

*Last MCP Validation: 2024-05-06* 