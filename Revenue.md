# Revenue Metrics Queries 📊

Welcome to the Revenue Metrics section of our SQL Query Resource Center. This page contains datasets and queries for tracking revenue and performance indicators.

## Datasets 📁

### `sdp-prd-cti-data.finance.shop_revenue_daily`

This dataset provides daily snapshots of revenue metrics by shop, including processing fees, subscription fees, and other revenue streams.

🔗 [Access on Dataplex](https://console.cloud.google.com/dataplex/projects/sdp-prd-cti-data/locations/us/entryGroups/@bigquery/entries/path_to_dataset)

### `sdp-prd-cti-data.finance.revenue_transactions`

This dataset contains individual revenue transactions with detailed categorization.

🔗 [Access on Dataplex](https://console.cloud.google.com/dataplex/projects/sdp-prd-cti-data/locations/us/entryGroups/@bigquery/entries/path_to_dataset)

## Common Queries 💻

### Revenue by Category

```sql
SELECT
  revenue_category,
  DATE_TRUNC(transaction_date, MONTH) as month,
  SUM(amount_usd) as total_revenue,
  COUNT(DISTINCT shop_id) as unique_shops,
  SUM(amount_usd) / COUNT(DISTINCT shop_id) as revenue_per_shop
FROM
  `sdp-prd-cti-data.finance.revenue_transactions`
WHERE
  transaction_date >= DATE_SUB(CURRENT_DATE(), INTERVAL 12 MONTH)
GROUP BY
  revenue_category, month
ORDER BY
  month DESC, total_revenue DESC
```

### Top Revenue Generating Merchants

```sql
SELECT
  shop_id,
  SUM(CASE WHEN revenue_category = 'processing_fee' THEN amount_usd ELSE 0 END) as processing_fee_revenue,
  SUM(CASE WHEN revenue_category = 'subscription_fee' THEN amount_usd ELSE 0 END) as subscription_fee_revenue,
  SUM(CASE WHEN revenue_category = 'add_on_services' THEN amount_usd ELSE 0 END) as add_on_services_revenue,
  SUM(amount_usd) as total_revenue
FROM
  `sdp-prd-cti-data.finance.revenue_transactions`
WHERE
  transaction_date >= DATE_SUB(CURRENT_DATE(), INTERVAL 3 MONTH)
GROUP BY
  shop_id
ORDER BY
  total_revenue DESC
LIMIT 100
```

### Revenue Growth Analysis

```sql
WITH monthly_revenue AS (
  SELECT
    shop_id,
    DATE_TRUNC(transaction_date, MONTH) as month,
    SUM(amount_usd) as monthly_revenue
  FROM
    `sdp-prd-cti-data.finance.revenue_transactions`
  WHERE
    transaction_date >= DATE_SUB(CURRENT_DATE(), INTERVAL 12 MONTH)
  GROUP BY
    shop_id, month
)

SELECT
  shop_id,
  month,
  monthly_revenue,
  LAG(monthly_revenue) OVER (PARTITION BY shop_id ORDER BY month) as prev_month_revenue,
  (monthly_revenue - LAG(monthly_revenue) OVER (PARTITION BY shop_id ORDER BY month)) 
    / NULLIF(LAG(monthly_revenue) OVER (PARTITION BY shop_id ORDER BY month), 0) as mom_growth_rate
FROM
  monthly_revenue
WHERE
  shop_id IN (
    SELECT shop_id
    FROM `sdp-prd-cti-data.finance.revenue_transactions`
    WHERE transaction_date >= DATE_SUB(CURRENT_DATE(), INTERVAL 12 MONTH)
    GROUP BY shop_id
    HAVING SUM(amount_usd) > 10000
  )
ORDER BY
  mom_growth_rate DESC
```

### Revenue vs Risk Analysis

```sql
WITH shop_revenue AS (
  SELECT
    shop_id,
    SUM(amount_usd) as total_revenue
  FROM
    `sdp-prd-cti-data.finance.revenue_transactions`
  WHERE
    transaction_date >= DATE_SUB(CURRENT_DATE(), INTERVAL 6 MONTH)
  GROUP BY
    shop_id
),

shop_risk AS (
  SELECT
    shop_id,
    AVG(risk_score) as avg_risk_score,
    SUM(loss_amount_usd) as total_losses
  FROM
    `sdp-prd-cti-data.risk.shop_risk_profile`
  WHERE
    profile_date >= DATE_SUB(CURRENT_DATE(), INTERVAL 6 MONTH)
  GROUP BY
    shop_id
)

SELECT
  r.shop_id,
  r.total_revenue,
  s.avg_risk_score,
  s.total_losses,
  s.total_losses / NULLIF(r.total_revenue, 0) as loss_to_revenue_ratio
FROM
  shop_revenue r
  JOIN shop_risk s ON r.shop_id = s.shop_id
WHERE
  r.total_revenue > 5000
ORDER BY
  loss_to_revenue_ratio DESC
```

## Revenue Categories Reference 📋

Our revenue categories include:

1. **Processing Fees** - Revenue from transaction processing
2. **Subscription Fees** - Regular subscription payments for platform access
3. **Add-On Services** - Revenue from additional services and features
4. **Interest Income** - Revenue from interest on held funds
5. **Other Revenue** - Miscellaneous revenue sources

## Notes and Best Practices 📝

- Consider seasonality when analyzing revenue trends
- Segment merchants by size and industry for more accurate comparisons
- Track revenue alongside risk metrics for a complete view of merchant health
- Monitor revenue diversification to identify growth opportunities

Feel free to contribute new queries or improve the existing ones. Together, we can build a comprehensive resource for revenue analysis.

Happy revenue analyzing! 💹 