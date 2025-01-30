## Look into rules metadata
```
SELECT 
r.shop_id,
    r.detection_identifier as rule_name,
    r.report_group,
    r.report_type,
    t.ticket_id,
    t.status,
    t.created_at AS ticket_created_at,

    r.monitoring_metadata,-- << this will pull all the metadata at once so if you want it to be separated then

    json_extract_scalar(monitoring_metadata,'$.bad_reviews') AS name_of_metadata --<< this way, you can pull the data one by one, easier to read 
FROM 
  `sdp-prd-cti-data.base.base__sensitive_reports` AS r
  JOIN `sdp-prd-cti-data.base.base__tickets`  AS t
    ON r.report_id = t.reportable_id
WHERE
  DATE(t.created_at) >= date('2024-01-01')
  AND detection_identifier = 'Event::Shop10kTo50kThreshold'-- << you need to use the detection_identifier of the rule 
````
