# Booked Losses Queries 📉

Welcome to the Booked Losses section of our SQL Query Resource Center. This page contains datasets and queries for loss tracking, classification, and trend analysis.

## Datasets 📁

### `sdp-prd-cti-data.finance.booked_losses`

This primary dataset contains detailed information on all booked losses, including categorization and attribution.

🔗 [Access on Dataplex](https://console.cloud.google.com/dataplex/projects/sdp-prd-cti-data/locations/us/entryGroups/@bigquery/entries/path_to_dataset)

### `sdp-prd-cti-data.finance.loss_recovery_attempts`

This dataset tracks recovery attempts and outcomes for booked losses.

🔗 [Access on Dataplex](https://console.cloud.google.com/dataplex/projects/sdp-prd-cti-data/locations/us/entryGroups/@bigquery/entries/path_to_dataset)

## Common Queries 💻

### Loss Distribution by Category

```sql
SELECT
  loss_category,
  COUNT(*) as loss_count,
  SUM(loss_amount_usd) as total_loss_amount,
  AVG(loss_amount_usd) as avg_loss_amount,
  MIN(loss_amount_usd) as min_loss_amount,
  MAX(loss_amount_usd) as max_loss_amount,
  STDDEV(loss_amount_usd) as std_dev_loss_amount
FROM
  `sdp-prd-cti-data.finance.booked_losses`
WHERE
  booking_date >= DATE_SUB(CURRENT_DATE(), INTERVAL 12 MONTH)
GROUP BY
  loss_category
ORDER BY
  total_loss_amount DESC
```

### Monthly Loss Trends

```sql
SELECT
  DATE_TRUNC(booking_date, MONTH) as month,
  loss_category,
  COUNT(*) as loss_count,
  SUM(loss_amount_usd) as total_loss_amount
FROM
  `sdp-prd-cti-data.finance.booked_losses`
WHERE
  booking_date >= DATE_SUB(CURRENT_DATE(), INTERVAL 24 MONTH)
GROUP BY
  month, loss_category
ORDER BY
  month DESC, total_loss_amount DESC
```

### Recovery Rate Analysis

```sql
WITH loss_data AS (
  SELECT
    l.loss_id,
    l.loss_amount_usd,
    l.booking_date,
    l.loss_category,
    COALESCE(SUM(r.recovery_amount_usd), 0) as recovered_amount,
    COALESCE(SUM(r.recovery_amount_usd) / NULLIF(l.loss_amount_usd, 0), 0) as recovery_rate
  FROM
    `sdp-prd-cti-data.finance.booked_losses` l
    LEFT JOIN `sdp-prd-cti-data.finance.loss_recovery_attempts` r
      ON l.loss_id = r.loss_id
  WHERE
    l.booking_date >= DATE_SUB(CURRENT_DATE(), INTERVAL 12 MONTH)
  GROUP BY
    l.loss_id, l.loss_amount_usd, l.booking_date, l.loss_category
)

SELECT
  loss_category,
  COUNT(*) as loss_count,
  SUM(loss_amount_usd) as total_loss_amount,
  SUM(recovered_amount) as total_recovered_amount,
  SUM(recovered_amount) / SUM(loss_amount_usd) as overall_recovery_rate,
  AVG(recovery_rate) as avg_recovery_rate
FROM
  loss_data
GROUP BY
  loss_category
ORDER BY
  overall_recovery_rate DESC
```

### Merchants with Highest Losses

```sql
SELECT
  shop_id,
  COUNT(*) as loss_count,
  SUM(loss_amount_usd) as total_loss_amount,
  MAX(booking_date) as most_recent_loss,
  MIN(booking_date) as first_loss,
  ARRAY_AGG(DISTINCT loss_category ORDER BY loss_category) as loss_categories
FROM
  `sdp-prd-cti-data.finance.booked_losses`
WHERE
  booking_date >= DATE_SUB(CURRENT_DATE(), INTERVAL 12 MONTH)
GROUP BY
  shop_id
HAVING
  COUNT(*) > 1
ORDER BY
  total_loss_amount DESC
LIMIT 100
```

## Loss Categories Reference 📋

Our loss categories include:

1. **Chargeback Losses** - Losses due to customer-initiated chargebacks
2. **Fraud Losses** - Losses from confirmed fraudulent transactions
3. **Credit Losses** - Losses from merchant bankruptcy or business closure
4. **Policy Losses** - Losses resulting from policy decisions or exceptions
5. **Operational Losses** - Losses from internal process failures

## Notes and Best Practices 📝

- Regular review of loss trends helps identify emerging risk patterns
- Always validate loss categorization for accurate reporting
- Consider recovery potential when analyzing gross loss figures
- Track time-to-recovery to optimize collection processes

Feel free to contribute new queries or improve the existing ones. Together, we can build a comprehensive resource for loss analysis and recovery optimization.

Happy loss analyzing! 🔍 