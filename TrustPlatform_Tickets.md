# TrustPlatform Tickets Queries 🎫

Welcome to the TrustPlatform Tickets section of our SQL Query Resource Center. This page contains datasets and queries for analyzing tickets and case management.

## Datasets 📁

### `sdp-prd-cti-data.base.base__tickets`

This primary dataset contains all TrustPlatform tickets with their status, assignments, and key metadata.

🔗 [Access on Dataplex](https://console.cloud.google.com/dataplex/projects/sdp-prd-cti-data/locations/us/entryGroups/@bigquery/entries/path_to_dataset)

### `sdp-prd-cti-data.base.base__sensitive_reports`

This dataset contains the reports that may trigger ticket creation, including rule metadata.

🔗 [Access on Dataplex](https://console.cloud.google.com/dataplex/projects/sdp-prd-cti-data/locations/us/entryGroups/@bigquery/entries/path_to_dataset)

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
  `sdp-prd-cti-data.base.base__tickets`
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
    `sdp-prd-cti-data.base.base__tickets`
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

### Rule Effectiveness Analysis

```sql
SELECT
  r.detection_identifier as rule_name,
  r.report_group,
  r.report_type,
  COUNT(t.ticket_id) as tickets_generated,
  COUNTIF(t.status = 'closed' AND t.resolution_type = 'action_taken') as action_taken,
  COUNTIF(t.status = 'closed' AND t.resolution_type = 'false_positive') as false_positives,
  SAFE_DIVIDE(
    COUNTIF(t.status = 'closed' AND t.resolution_type = 'action_taken'),
    COUNTIF(t.status = 'closed')
  ) as precision_rate
FROM 
  `sdp-prd-cti-data.base.base__sensitive_reports` AS r
  JOIN `sdp-prd-cti-data.base.base__tickets` AS t
    ON r.report_id = t.reportable_id
WHERE
  DATE(t.created_at) >= DATE_SUB(CURRENT_DATE(), INTERVAL 90 DAY)
GROUP BY
  rule_name, report_group, report_type
HAVING
  tickets_generated > 10
ORDER BY
  precision_rate DESC
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
  `sdp-prd-cti-data.base.base__tickets`
WHERE
  assignee IS NOT NULL
  AND created_at >= DATE_SUB(CURRENT_DATE(), INTERVAL 30 DAY)
GROUP BY
  assignee
ORDER BY
  total_tickets DESC
```

## Rule Metadata Reference 📋

When working with rules, you can extract specific metadata using JSON functions:

```sql
SELECT 
  detection_identifier as rule_name,
  json_extract_scalar(monitoring_metadata,'$.parameter_name') AS extracted_parameter,
  COUNT(*) as occurrences
FROM 
  `sdp-prd-cti-data.base.base__sensitive_reports`
WHERE
  DATE(created_at) >= DATE_SUB(CURRENT_DATE(), INTERVAL 30 DAY)
  AND detection_identifier = 'Event::ExampleRuleName'
GROUP BY
  rule_name, extracted_parameter
ORDER BY
  occurrences DESC
```

## Notes and Best Practices 📝

- Use ticket resolution metrics to identify bottlenecks in your workflow
- Regularly analyze rule precision to improve detection effectiveness
- Track workload distribution to maintain team efficiency
- Consider time-of-day patterns when analyzing ticket volumes

Feel free to contribute new queries or improve the existing ones. Together, we can build a comprehensive resource for ticket analysis and workflow optimization.

Happy ticket analyzing! 🧐 