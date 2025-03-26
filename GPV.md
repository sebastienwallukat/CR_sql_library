# GPV (Gross Processing Volume) Queries 📈

Welcome to the GPV section of our SQL Query Resource Center. This page contains datasets and queries for analyzing Gross Processing Volume metrics and growth patterns.

## Datasets 📁

### `sdp-prd-cti-data.intermediate.shop_gpv_daily_snapshot`

This dataset provides daily snapshots of GPV data, perfect for tracking processing volume trends over time.

🔗 [Access on Dataplex](https://console.cloud.google.com/dataplex/projects/sdp-prd-cti-data/locations/us/entryGroups/@bigquery/entries/path_to_dataset)

### `sdp-prd-cti-data.intermediate.shop_gpv_current`

This dataset contains the most recent GPV data for each shop, giving you an up-to-date view of processing volumes.

🔗 [Access on Dataplex](https://console.cloud.google.com/dataplex/projects/sdp-prd-cti-data/locations/us/entryGroups/@bigquery/entries/path_to_dataset)

## Common Queries 💻

### GPV Growth Rate by Shop

```sql
SELECT
  shop_id,
  DATE_TRUNC(transaction_date, MONTH) as month,
  SUM(gpv_amount_usd) as monthly_gpv,
  (SUM(gpv_amount_usd) - LAG(SUM(gpv_amount_usd)) OVER (PARTITION BY shop_id ORDER BY DATE_TRUNC(transaction_date, MONTH))) 
    / NULLIF(LAG(SUM(gpv_amount_usd)) OVER (PARTITION BY shop_id ORDER BY DATE_TRUNC(transaction_date, MONTH)), 0) as growth_rate
FROM
  `sdp-prd-cti-data.intermediate.shop_gpv_daily_snapshot`
WHERE
  transaction_date >= DATE_SUB(CURRENT_DATE(), INTERVAL 6 MONTH)
GROUP BY
  shop_id, month
ORDER BY
  shop_id, month
```

### High Growth Merchants Detection

```sql
WITH monthly_gpv AS (
  SELECT
    shop_id,
    DATE_TRUNC(transaction_date, MONTH) as month,
    SUM(gpv_amount_usd) as monthly_gpv
  FROM
    `sdp-prd-cti-data.intermediate.shop_gpv_daily_snapshot`
  WHERE
    transaction_date >= DATE_SUB(CURRENT_DATE(), INTERVAL 3 MONTH)
  GROUP BY
    shop_id, month
),

growth_metrics AS (
  SELECT
    shop_id,
    month,
    monthly_gpv,
    LAG(monthly_gpv) OVER (PARTITION BY shop_id ORDER BY month) as prev_month_gpv,
    (monthly_gpv - LAG(monthly_gpv) OVER (PARTITION BY shop_id ORDER BY month)) 
      / NULLIF(LAG(monthly_gpv) OVER (PARTITION BY shop_id ORDER BY month), 0) as growth_rate
  FROM
    monthly_gpv
)

SELECT
  shop_id,
  month,
  monthly_gpv,
  prev_month_gpv,
  growth_rate
FROM
  growth_metrics
WHERE
  growth_rate > 2.0  -- Identifying merchants with over 200% growth
  AND prev_month_gpv > 5000  -- With meaningful previous volume
ORDER BY
  growth_rate DESC
```

## Notes and Best Practices 📝

- When analyzing GPV, consider seasonality effects by comparing year-over-year rather than just month-over-month
- For new merchants, evaluate growth rate after at least 3 months of processing history
- Always consider currency fluctuations when analyzing cross-border merchants

Feel free to contribute new queries or improve the existing ones. Together, we can build a comprehensive resource for GPV analysis.

Happy data analyzing! 📊 