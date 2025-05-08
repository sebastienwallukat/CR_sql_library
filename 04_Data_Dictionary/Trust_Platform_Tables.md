# Trust Platform Tables

This document provides comprehensive information about Trust Platform tables used for credit risk analysis at Shopify.

## Overview

Trust Platform tables contain data about risk-related investigations, tickets, and actions taken to mitigate risk. These tables are essential for understanding risk escalations, case management, and the history of risk-related actions taken on merchant accounts.

## Primary Tables

### shopify-dw.risk.trust_platform_tickets_summary_v1

**Description**:  
This is the primary table for Trust Platform tickets data, providing a comprehensive view of risk investigations and actions. It includes details about ticket creation, status, risk level, and outcomes.

**Update Frequency**: Hourly

**Primary Keys**: `ticket_id`

**Schema**:

| Field Name | Type | Description |
|------------|------|-------------|
| `ticket_id` | STRING | Unique identifier for the ticket |
| `shop_id` | INTEGER | Shop ID associated with the ticket |
| `created_at` | TIMESTAMP | When the ticket was created |
| `updated_at` | TIMESTAMP | When the ticket was last updated |
| `closed_at` | TIMESTAMP | When the ticket was closed (if applicable) |
| `ticket_type` | STRING | Type of ticket (e.g., 'risk_review', 'chargeback_investigation') |
| `ticket_status` | STRING | Current status (e.g., 'OPEN', 'CLOSED', 'PENDING') |
| `risk_level` | STRING | Assessed risk level (e.g., 'HIGH', 'MEDIUM', 'LOW') |
| `trust_subject` | STRING | Subject area for the ticket |
| `trust_sub_category` | STRING | More specific categorization |
| `resolution` | STRING | Final resolution of the ticket |
| `resolution_reason` | STRING | Reason for the resolution |
| `creator` | STRING | Who or what created the ticket |
| `assignee` | STRING | Person or team assigned to the ticket |
| `integration_type` | STRING | Integration that created the ticket (if applicable) |
| `action_taken` | STRING | Actions taken as a result of the ticket |
| `metadata` | RECORD | Additional metadata about the ticket (nested record) |

**Usage Notes**:
- Critical table for tracking risk investigations and actions
- `ticket_status` is essential for filtering active vs. resolved cases
- `risk_level` indicates severity and prioritization
- `action_taken` shows outcomes like account disablement, reserve application, etc.
- The table is partitioned by `created_at` - include date filters for better performance

**Example Query**:
```sql
-- Find recent high-risk tickets with actions taken
SELECT 
  ticket_id,
  shop_id,
  created_at,
  ticket_type,
  risk_level,
  ticket_status,
  action_taken
FROM `shopify-dw.risk.trust_platform_tickets_summary_v1`
WHERE created_at >= DATE_SUB(CURRENT_DATE(), INTERVAL 30 DAY)
  AND risk_level = 'HIGH'
  AND action_taken IS NOT NULL
ORDER BY created_at DESC
```

### shopify-dw.risk.risk_reviews_summary_v1

**Description**:  
Contains detailed information about risk reviews conducted on merchant accounts. Risk reviews are specific types of investigations that assess merchant risk profiles.

**Update Frequency**: Daily

**Primary Keys**: `review_id`

**Foreign Keys**: 
- `shop_id` references `shopify-dw.shopify.shops.id`
- `ticket_id` references `shopify-dw.risk.trust_platform_tickets_summary_v1.ticket_id`

**Schema**:

| Field Name | Type | Description |
|------------|------|-------------|
| `review_id` | STRING | Unique identifier for the risk review |
| `ticket_id` | STRING | Associated Trust Platform ticket ID |
| `shop_id` | INTEGER | Shop ID being reviewed |
| `created_at` | TIMESTAMP | When the review was created |
| `updated_at` | TIMESTAMP | When the review was last updated |
| `completed_at` | TIMESTAMP | When the review was completed |
| `review_type` | STRING | Type of risk review |
| `review_status` | STRING | Status of the review |
| `risk_level` | STRING | Risk level determined by the review |
| `reviewer` | STRING | Person who conducted the review |
| `review_reason` | STRING | Reason for conducting the review |
| `business_model` | STRING | Business model of the merchant |
| `review_outcome` | STRING | Outcome of the review |
| `action_taken` | STRING | Actions taken as a result of the review |
| `notes` | STRING | Additional notes about the review |

**Usage Notes**:
- More specific than general tickets for risk reviews
- Contains detailed assessment information
- `business_model` field is particularly relevant for credit risk analysis
- `action_taken` shows risk mitigation measures
- Often used in conjunction with the tickets table

**Example Query**:
```sql
-- Analyze risk review outcomes by business model
SELECT 
  business_model,
  COUNT(*) AS review_count,
  COUNTIF(review_outcome = 'APPROVED') AS approved_count,
  COUNTIF(review_outcome = 'REJECTED') AS rejected_count,
  COUNTIF(review_outcome = 'APPROVED') / COUNT(*) AS approval_rate
FROM `shopify-dw.risk.risk_reviews_summary_v1`
WHERE created_at >= DATE_SUB(CURRENT_DATE(), INTERVAL 90 DAY)
  AND business_model IS NOT NULL
GROUP BY business_model
ORDER BY review_count DESC
```

### shopify-dw.risk.trust_platform_actions_summary_v1

**Description**:  
This table records specific actions taken within the Trust Platform, such as account disablements, reserve applications, and other risk mitigation measures.

**Update Frequency**: Hourly

**Primary Keys**: `action_id`

**Foreign Keys**:
- `ticket_id` references `shopify-dw.risk.trust_platform_tickets_summary_v1.ticket_id`
- `shop_id` references `shopify-dw.shopify.shops.id`

**Schema**:

| Field Name | Type | Description |
|------------|------|-------------|
| `action_id` | STRING | Unique identifier for the action |
| `ticket_id` | STRING | Associated ticket ID |
| `shop_id` | INTEGER | Shop ID the action was taken on |
| `created_at` | TIMESTAMP | When the action was created |
| `executed_at` | TIMESTAMP | When the action was executed |
| `action_type` | STRING | Type of action taken |
| `action_status` | STRING | Status of the action |
| `creator` | STRING | Who or what created the action |
| `executor` | STRING | Who or what executed the action |
| `reason` | STRING | Reason for taking the action |
| `details` | RECORD | Additional details about the action (nested record) |
| `is_automated` | BOOLEAN | Whether the action was automated |

**Usage Notes**:
- Primary source for understanding risk mitigation actions
- `action_type` includes critical actions like:
  - `DISABLE_ACCOUNT`
  - `APPLY_RESERVE`
  - `MODIFY_RESERVE`
  - `REMOVE_RESERVE`
  - `BLOCK_PAYOUTS`
- `is_automated` helps distinguish between system-initiated and manual actions
- `details` contains action-specific information in a nested structure

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
WHERE created_at >= DATE_SUB(CURRENT_DATE(), INTERVAL 90 DAY)
GROUP BY action_type
ORDER BY action_count DESC
```

## Secondary Tables

### shopify-dw.risk.trust_platform_risk_signals_summary_v1

**Description**:  
Contains risk signals that triggered investigations or actions in the Trust Platform. Risk signals are indicators of potential risk detected by various systems.

**Update Frequency**: Daily

**Primary Keys**: `signal_id`

**Foreign Keys**:
- `shop_id` references `shopify-dw.shopify.shops.id`
- `ticket_id` references `shopify-dw.risk.trust_platform_tickets_summary_v1.ticket_id` (if associated)

**Schema**:

| Field Name | Type | Description |
|------------|------|-------------|
| `signal_id` | STRING | Unique identifier for the risk signal |
| `shop_id` | INTEGER | Shop ID the signal is related to |
| `ticket_id` | STRING | Associated ticket ID (if any) |
| `created_at` | TIMESTAMP | When the signal was created |
| `signal_type` | STRING | Type of risk signal |
| `signal_source` | STRING | Source system that generated the signal |
| `risk_level` | STRING | Risk level of the signal |
| `signal_details` | RECORD | Additional details about the signal |
| `led_to_action` | BOOLEAN | Whether the signal led to a risk action |
| `triggered_rules` | ARRAY | Rules that were triggered to generate the signal |

**Usage Notes**:
- Useful for understanding what triggered risk investigations
- `signal_type` and `signal_source` help categorize detection methods
- `led_to_action` indicates if the signal resulted in risk mitigation
- `triggered_rules` shows the specific rules that identified the risk
- Important for analyzing the effectiveness of risk detection

**Example Query**:
```sql
-- Analyze effectiveness of different signal types
SELECT 
  signal_type,
  signal_source,
  COUNT(*) AS signal_count,
  COUNTIF(led_to_action) AS action_count,
  COUNTIF(led_to_action) / COUNT(*) AS action_rate
FROM `shopify-dw.risk.trust_platform_risk_signals_summary_v1`
WHERE created_at >= DATE_SUB(CURRENT_DATE(), INTERVAL 90 DAY)
GROUP BY signal_type, signal_source
ORDER BY signal_count DESC
```

### shopify-dw.risk.trust_platform_comments_summary_v1

**Description**:  
Contains comments and notes added to Trust Platform tickets, providing additional context and investigation details.

**Update Frequency**: Hourly

**Primary Keys**: `comment_id`

**Foreign Keys**:
- `ticket_id` references `shopify-dw.risk.trust_platform_tickets_summary_v1.ticket_id`

**Schema**:

| Field Name | Type | Description |
|------------|------|-------------|
| `comment_id` | STRING | Unique identifier for the comment |
| `ticket_id` | STRING | Associated ticket ID |
| `created_at` | TIMESTAMP | When the comment was created |
| `updated_at` | TIMESTAMP | When the comment was last updated |
| `creator` | STRING | Who created the comment |
| `content` | STRING | Text content of the comment |
| `is_internal` | BOOLEAN | Whether the comment is internal or customer-facing |
| `comment_type` | STRING | Type of comment |

**Usage Notes**:
- Provides qualitative information about investigations
- May contain sensitive information requiring proper permissions
- `is_internal` distinguishes between team notes and merchant communications
- Text analysis of comments can reveal common risk patterns
- Often used for detailed case review rather than aggregate analysis

**Example Query**:
```sql
-- Find tickets with extensive internal discussion
WITH comment_counts AS (
  SELECT 
    ticket_id,
    COUNT(*) AS total_comments,
    COUNTIF(is_internal) AS internal_comments
  FROM `shopify-dw.risk.trust_platform_comments_summary_v1`
  GROUP BY ticket_id
)

SELECT 
  c.ticket_id,
  t.created_at,
  t.ticket_type,
  t.risk_level,
  t.ticket_status,
  c.total_comments,
  c.internal_comments
FROM comment_counts c
JOIN `shopify-dw.risk.trust_platform_tickets_summary_v1` t
  ON c.ticket_id = t.ticket_id
WHERE c.total_comments > 5
ORDER BY c.total_comments DESC
LIMIT 100
```

### shopify-dw.risk.trust_platform_reason_categories_summary_v1

**Description**:  
Provides standardized categorization of reasons for Trust Platform tickets and actions. This helps with consistent analysis and reporting of risk trends.

**Update Frequency**: Daily

**Primary Keys**: `category_id`

**Schema**:

| Field Name | Type | Description |
|------------|------|-------------|
| `category_id` | STRING | Unique identifier for the category |
| `category_name` | STRING | Name of the category |
| `parent_category_id` | STRING | ID of the parent category (if hierarchical) |
| `category_level` | INTEGER | Level in the category hierarchy |
| `is_active` | BOOLEAN | Whether the category is currently active |
| `created_at` | TIMESTAMP | When the category was created |
| `updated_at` | TIMESTAMP | When the category was last updated |
| `description` | STRING | Description of what the category represents |

**Usage Notes**:
- Provides standardized risk reason taxonomy
- Used to classify tickets and actions consistently
- Hierarchical categories allow for different levels of granularity
- Important for trend analysis and reporting
- Should be joined with tickets and actions tables for categorization

**Example Query**:
```sql
-- Join tickets with reason categories
SELECT 
  t.ticket_id,
  t.created_at,
  t.ticket_type,
  c.category_name AS reason_category,
  t.ticket_status
FROM `shopify-dw.risk.trust_platform_tickets_summary_v1` t
JOIN `shopify-dw.risk.trust_platform_reason_categories_summary_v1` c
  ON t.metadata.reason_category_id = c.category_id
WHERE t.created_at >= DATE_SUB(CURRENT_DATE(), INTERVAL 30 DAY)
ORDER BY t.created_at DESC
```

## How to Join Trust Platform Tables

### Joining Tickets with Actions

```sql
-- Join tickets with associated actions
SELECT 
  t.ticket_id,
  t.shop_id,
  t.created_at AS ticket_created_at,
  t.ticket_type,
  t.risk_level,
  a.action_id,
  a.action_type,
  a.executed_at,
  a.action_status,
  a.is_automated
FROM `shopify-dw.risk.trust_platform_tickets_summary_v1` t
LEFT JOIN `shopify-dw.risk.trust_platform_actions_summary_v1` a
  ON t.ticket_id = a.ticket_id
WHERE t.created_at >= DATE_SUB(CURRENT_DATE(), INTERVAL 90 DAY)
ORDER BY t.created_at DESC, a.executed_at
```

### Joining Tickets with Risk Reviews

```sql
-- Join tickets with risk reviews
SELECT 
  t.ticket_id,
  t.shop_id,
  t.created_at AS ticket_created_at,
  t.risk_level AS ticket_risk_level,
  r.review_id,
  r.review_type,
  r.business_model,
  r.risk_level AS review_risk_level,
  r.review_outcome,
  r.action_taken
FROM `shopify-dw.risk.trust_platform_tickets_summary_v1` t
JOIN `shopify-dw.risk.risk_reviews_summary_v1` r
  ON t.ticket_id = r.ticket_id
WHERE t.created_at >= DATE_SUB(CURRENT_DATE(), INTERVAL 90 DAY)
  AND t.ticket_type = 'risk_review'
ORDER BY t.created_at DESC
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
  WHERE created_at >= DATE_SUB(CURRENT_DATE(), INTERVAL 90 DAY)
  GROUP BY shop_id
  HAVING COUNT(*) >= 5
)

SELECT 
  c.shop_id,
  c.chargeback_count,
  c.chargeback_amount,
  COUNT(DISTINCT t.ticket_id) AS ticket_count,
  COUNTIF(t.ticket_status = 'OPEN') AS open_ticket_count,
  COUNTIF(a.action_type = 'DISABLE_ACCOUNT') AS account_disable_count,
  COUNTIF(a.action_type = 'APPLY_RESERVE') AS reserve_applied_count
FROM chargeback_shops c
LEFT JOIN `shopify-dw.risk.trust_platform_tickets_summary_v1` t
  ON c.shop_id = t.shop_id
  AND t.created_at >= DATE_SUB(CURRENT_DATE(), INTERVAL 90 DAY)
LEFT JOIN `shopify-dw.risk.trust_platform_actions_summary_v1` a
  ON t.ticket_id = a.ticket_id
GROUP BY c.shop_id, c.chargeback_count, c.chargeback_amount
ORDER BY c.chargeback_count DESC
```

### Joining Trust Platform Data with Shop Status

```sql
-- Analyze shops with Trust Platform tickets by current status
SELECT 
  s.shopify_payments_status,
  COUNT(DISTINCT t.shop_id) AS shop_count,
  COUNT(DISTINCT t.ticket_id) AS ticket_count,
  COUNT(DISTINCT CASE WHEN t.risk_level = 'HIGH' THEN t.ticket_id END) AS high_risk_ticket_count,
  COUNT(DISTINCT CASE WHEN a.action_type = 'DISABLE_ACCOUNT' THEN a.action_id END) AS disable_action_count
FROM `shopify-dw.risk.trust_platform_tickets_summary_v1` t
LEFT JOIN `sdp-prd-cti-data.intermediate.shop_current_shopify_payments_status` s
  ON t.shop_id = s.shop_id
LEFT JOIN `shopify-dw.risk.trust_platform_actions_summary_v1` a
  ON t.ticket_id = a.ticket_id
WHERE t.created_at >= DATE_SUB(CURRENT_DATE(), INTERVAL 90 DAY)
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
   - `executed_at` may be NULL for pending actions
   - `action_status` provides the current state of the action

3. **Historical vs. Current View**:
   - Some tables provide historical snapshots while others show current state
   - Be careful when joining historical tickets with current shop status
   - Consider temporal relationships when analyzing cause and effect

4. **Comment Text Analysis**:
   - Comments may contain unstructured data requiring text analysis
   - Comment content may include PII requiring proper permissions
   - Content may use inconsistent terminology requiring normalization

5. **Access Restrictions**:
   - Trust Platform data may have stricter access controls
   - Certain fields may be redacted or NULL based on permissions
   - Ensure you have appropriate access before analysis

## Common Analysis Patterns

### Risk Action Impact Analysis

```sql
-- Analyze impact of reserve actions on chargeback rates
WITH reserve_actions AS (
  SELECT 
    shop_id,
    MIN(executed_at) AS first_reserve_applied_at
  FROM `shopify-dw.risk.trust_platform_actions_summary_v1`
  WHERE action_type = 'APPLY_RESERVE'
    AND executed_at IS NOT NULL
    AND action_status = 'SUCCEEDED'
  GROUP BY shop_id
),

daily_chargebacks AS (
  SELECT 
    c.shop_id,
    DATE(c.created_at) AS date,
    COUNT(*) AS daily_chargeback_count
  FROM `shopify-dw.money_products.chargebacks_summary` c
  JOIN reserve_actions r ON c.shop_id = r.shop_id
  WHERE c.created_at BETWEEN 
    DATE_SUB(r.first_reserve_applied_at, INTERVAL 30 DAY) AND
    DATE_ADD(r.first_reserve_applied_at, INTERVAL 30 DAY)
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

### Risk Signal Effectiveness

```sql
-- Analyze how often different risk signals lead to actions
WITH signal_effectiveness AS (
  SELECT 
    s.signal_id,
    s.shop_id,
    s.signal_type,
    s.risk_level,
    s.created_at AS signal_created_at,
    MIN(t.created_at) AS first_ticket_created_at,
    MIN(a.executed_at) AS first_action_executed_at
  FROM `shopify-dw.risk.trust_platform_risk_signals_summary_v1` s
  LEFT JOIN `shopify-dw.risk.trust_platform_tickets_summary_v1` t
    ON s.shop_id = t.shop_id AND t.created_at >= s.created_at AND t.created_at <= DATE_ADD(s.created_at, INTERVAL 7 DAY)
  LEFT JOIN `shopify-dw.risk.trust_platform_actions_summary_v1` a
    ON t.ticket_id = a.ticket_id
  WHERE s.created_at >= DATE_SUB(CURRENT_DATE(), INTERVAL 90 DAY)
  GROUP BY s.signal_id, s.shop_id, s.signal_type, s.risk_level, s.created_at
)

SELECT 
  signal_type,
  risk_level,
  COUNT(*) AS signal_count,
  COUNTIF(first_ticket_created_at IS NOT NULL) AS led_to_ticket_count,
  COUNTIF(first_action_executed_at IS NOT NULL) AS led_to_action_count,
  COUNTIF(first_ticket_created_at IS NOT NULL) / COUNT(*) AS ticket_creation_rate,
  COUNTIF(first_action_executed_at IS NOT NULL) / COUNT(*) AS action_execution_rate,
  AVG(IF(first_ticket_created_at IS NOT NULL, 
         TIMESTAMP_DIFF(first_ticket_created_at, signal_created_at, HOUR), 
         NULL)) AS avg_hours_to_ticket,
  AVG(IF(first_action_executed_at IS NOT NULL, 
         TIMESTAMP_DIFF(first_action_executed_at, signal_created_at, HOUR), 
         NULL)) AS avg_hours_to_action
FROM signal_effectiveness
GROUP BY signal_type, risk_level
ORDER BY signal_count DESC, action_execution_rate DESC
```

### Ticket Resolution Analysis

```sql
-- Analyze ticket resolution times and outcomes
WITH ticket_lifecycle AS (
  SELECT 
    ticket_id,
    shop_id,
    ticket_type,
    risk_level,
    created_at,
    closed_at,
    ticket_status,
    resolution,
    TIMESTAMP_DIFF(closed_at, created_at, HOUR) AS resolution_time_hours
  FROM `shopify-dw.risk.trust_platform_tickets_summary_v1`
  WHERE created_at >= DATE_SUB(CURRENT_DATE(), INTERVAL 90 DAY)
    AND closed_at IS NOT NULL
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

---

*Last Updated: May 2024* 