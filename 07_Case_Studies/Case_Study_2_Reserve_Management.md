# Case Study: Comprehensive Reserve Recommendation Analysis

## Overview

When assessing whether a merchant requires a reserve, the Credit Risk team needs to consider multiple risk dimensions:

1. Payment processing patterns
2. Chargeback history and reasons
3. Fulfillment performance and exposure
4. Business model and industry risk
5. Trust Battery signals
6. Customer complaint indicators

This case study walks through a comprehensive analysis process used to inform reserve decisions, based on verified data sources and methodologies.

## Business Problem

The Credit Risk team must determine:

1. Which merchants require reserves to mitigate potential losses
2. What type of reserve to implement (fixed, rolling, or time delay)
3. What percentage or amount to reserve
4. When reserve decisions should be reconsidered

## Verified Data Sources

This analysis combines data from multiple verified sources:

- **Payment Volume**: `sdp-prd-cti-data.intermediate.shop_daily_order_fraud_facts`
- **Unfulfilled Exposure**: `sdp-prd-cti-data.intermediate.trust_platform_shop_risk_attributes_daily`
- **Chargeback History**: `shopify-dw.money_products.chargebacks_summary`
- **Failed Transfers**: `shopify-dw.money_products.shopify_payments_balance_transactions`
- **Shop Information**:
  - `shopify-dw.mart_core_deliver.shop_back_office_summary_current` (Industry)
  - `shopify-dw.accounts_and_administration.shop_profile_current` (Geography)
  - `shopify-dw.accounts_and_administration.shop_billing_info_current` (Plan)
- **Trust Battery**: `sdp-prd-cti-data.intermediate.shop_insights_shops`
- **HVCR Model**: `sdp-prd-cti-data.intermediate.hvcr_predictions_v2`

## Analysis Approach

The reserve recommendation process follows these steps:

1. **Extract Payment Activity Data**:
   - Calculate recent GPV over the past 120 days
   - Identify payment processing patterns and velocity

2. **Measure Unfulfilled Exposure**:
   - Calculate total unfulfilled order value
   - Track unfulfilled order count trends

3. **Analyze Chargeback History**:
   - Identify dominant chargeback reasons
   - Measure chargeback velocity and trends

4. **Check Financial Stability**:
   - Review failed payment transfers
   - Assess payment method history

5. **Evaluate Merchant Profile**:
   - Analyze industry risk level
   - Consider geographic location
   - Review subscription plan type

6. **Assess Trust Signals**:
   - Extract Trust Battery score and category
   - Check identity verification status

7. **Incorporate Predictive Models**:
   - Use HVCR model scores and features
   - Analyze trends in key risk metrics

8. **Score Risk Dimensions**:
   - Apply verified risk scoring logic
   - Weight factors according to predictive strength

9. **Generate Recommendations**:
   - Determine appropriate reserve type
   - Calculate reserve percentage or amount
   - Set appropriate expiration timeline

## Reserve Recommendation Query Structure

The following is a simplified version of the verified query structure:

```sql
-- Reserve recommendation query with Trust Battery and HVCR features
WITH 
-- Calculate shop GPV over last 120 days
shop_gpv AS (
    SELECT
        shop_id,
        ROUND(SUM(f.daily_sp_paid_amount_usd), 4) AS total_gpv 
    FROM
        `sdp-prd-cti-data.intermediate.shop_daily_order_fraud_facts` AS f
    WHERE 
        DATE(date) >= DATE_SUB(CURRENT_DATE(), INTERVAL 120 DAY)
        AND CAST(shop_id AS STRING) = @input_shop_id
    GROUP BY shop_id
), 

-- Calculate unfulfilled order exposure
unfulfilled_120d_exposure AS (
    SELECT
        shop_id,
        SUM(net_unfulfilled_sp_order_count) AS unfulfilled_120d_count,
        ROUND(SUM(net_unfulfilled_sp_sales_usd), 4) AS total_unfulfilled_usd_120d
    FROM
        `sdp-prd-cti-data.intermediate.trust_platform_shop_risk_attributes_daily`
    WHERE 
        date >= current_date - interval '120' day  
        AND CAST(shop_id AS STRING) = @input_shop_id
    GROUP BY shop_id
),

-- Find dominant chargeback reason with tiebreaker logic
chargeback_dominate AS (
    SELECT 
        shop_id,
        reason as reasons,
        COUNT(*) AS occurrence_count
    FROM 
        `shopify-dw.money_products.chargebacks_summary`
    WHERE 
        CAST(shop_id AS STRING) = @input_shop_id
    GROUP BY 
        shop_id, reasons 
    QUALIFY 
        RANK() OVER (
            PARTITION BY shop_id 
            ORDER BY 
                occurrence_count DESC, 
                CASE -- Priorities for chargeback reasons (1 is highest)
                    WHEN reasons = 'product_not_received' THEN 1
                    WHEN reasons = 'product_unacceptable' THEN 2
                    WHEN reasons = 'credit_not_processed' THEN 3
                    WHEN reasons = 'fraudulent' THEN 4
                    WHEN reasons = 'general' THEN 5
                    WHEN reasons = 'noncompliant' THEN 6
                    WHEN reasons = 'customer_initiated' THEN 7
                    WHEN reasons = 'incorrect_account_details' THEN 8
                    WHEN reasons = 'unrecognized' THEN 9
                    WHEN reasons = 'subscription_canceled' THEN 10
                    ELSE 11
                END
        ) = 1
),

-- Check for failed transfers in last 30 days
failed_debits AS (
    SELECT
        shop_id,
        COUNT(balance_transaction_id) AS failed_transfer_count
    FROM
        `shopify-dw.money_products.shopify_payments_balance_transactions`
    WHERE 
        CAST(shop_id AS STRING) = @input_shop_id
        AND source_type = 'Payments::Transfer'
        AND transaction_type = 'transfer_failure'
        AND amount_usd < 0
        AND date(creation_date) >= CURRENT_DATE - 30  
    GROUP BY shop_id 
), 

-- Get shop industry information
shop_industry AS (
    SELECT 
        shop_id, 
        primary_industry
    FROM 
        `shopify-dw.mart_core_deliver.shop_back_office_summary_current`
    WHERE 
        CAST(shop_id AS STRING) = @input_shop_id
),

-- Get shop trust battery information
trust_battery AS (
    SELECT
        shop_id,
        trust_battery_score,
        trust_battery
    FROM 
        `sdp-prd-cti-data.intermediate.shop_insights_shops`
    WHERE 
        CAST(shop_id AS STRING) = @input_shop_id
),

-- Get latest HVCR prediction
hvcr_data AS (
    SELECT 
        shop_id,
        processed_at,
        score AS hvcr_score,
        features  -- JSON containing model features
    FROM 
        `sdp-prd-cti-data.intermediate.hvcr_predictions_v2`
    WHERE 
        processed_at = (
            SELECT MAX(processed_at) 
            FROM `sdp-prd-cti-data.intermediate.hvcr_predictions_v2` 
            WHERE CAST(shop_id AS STRING) = @input_shop_id
        )
)

-- Main query combining all risk factors
SELECT 
    -- Shop identifiers and basic info
    s.shop_id,
    sg.total_gpv AS total_gpv_last_120d,
    u.unfulfilled_120d_count,
    u.total_unfulfilled_usd_120d,
    COALESCE(c.reasons, "No Chargebacks") AS dominant_chargeback_reason,
    i.primary_industry,
    
    -- Trust Battery data
    t.trust_battery_score,
    t.trust_battery,
    
    -- HVCR model data
    h.hvcr_score,
    
    -- Risk scoring components
    CASE 
        WHEN i.primary_industry IN ('Gift Cards', 'Electronics', 'Apparel & Accessories') THEN 0.3
        WHEN i.primary_industry IN ('Health & Beauty', 'Home & Garden') THEN 0.15
        ELSE 0
    END AS industry_risk_score,
    
    CASE 
        WHEN fd.failed_transfer_count > 0 THEN 0.45
        ELSE 0
    END AS debits_risk_score,
    
    CASE
        WHEN c.reasons = 'product_not_received' THEN 0.45
        WHEN c.reasons = 'product_unacceptable' THEN 0.3
        WHEN c.reasons = 'credit_not_processed' THEN 0.2
        WHEN c.reasons = 'fraudulent' THEN 0.1
        ELSE 0.05
    END AS chargeback_risk_score,
    
    CASE
        WHEN t.trust_battery = 'low' THEN 0.2
        WHEN t.trust_battery = 'medium' THEN 0.1
        WHEN t.trust_battery = 'high' THEN -0.1
        WHEN t.trust_battery = 'max' THEN -0.2
        ELSE 0 -- For none/unspecified
    END AS trust_battery_score
    
FROM 
    shop_gpv sg
JOIN 
    shop_industry i ON sg.shop_id = i.shop_id
LEFT JOIN
    chargeback_dominate c ON sg.shop_id = c.shop_id
LEFT JOIN
    failed_debits fd ON sg.shop_id = fd.shop_id
LEFT JOIN 
    unfulfilled_120d_exposure u ON sg.shop_id = u.shop_id
LEFT JOIN
    trust_battery t ON sg.shop_id = t.shop_id
LEFT JOIN
    hvcr_data h ON sg.shop_id = h.shop_id
WHERE
    CAST(sg.shop_id AS STRING) = @input_shop_id
```

## Risk Scoring Dimensions

The Reserve Recommendation model incorporates multiple risk dimensions with varying weights:

### Business Model & Industry Risk
Higher risk categories include:
- Gift Cards (0.3)
- Electronics (0.3)
- Apparel & Accessories (0.3)

Medium risk categories include:
- Health & Beauty (0.15)
- Home & Garden (0.15)
- Toys & Games (0.15)

### Financial Stability Indicators
- Failed transfers in last 30 days (0.45)
- Chargeback reasons weighted by severity:
  - Product not received (0.45)
  - Product unacceptable (0.3)
  - Credit not processed (0.2)
  - Fraudulent (0.1)

### Trust Battery Scoring
- Low trust (-0.2)
- Medium trust (0.1)
- High trust (-0.1)
- Max trust (-0.2)

## Reserve Type Selection

Based on the combined risk scores and shop characteristics, the appropriate reserve type is determined:

1. **Fixed Reserve**:
   - Set specific dollar amount
   - Typically used for merchants with consistent, predictable risk
   - Often suitable for larger, established businesses

2. **Rolling Reserve**:
   - Holds percentage of each transaction
   - Released after specified period (typically 30-90 days)
   - Good for ongoing, variable risk

3. **Time Delay**:
   - Delays initial payouts
   - Typically used for new merchants or high-risk situations
   - Provides time to detect issues before releasing funds

## Reserve Implementation

After determining the appropriate reserve type and amount, the implementation requires:

1. **Create Configuration Entry**:
   ```sql
   -- Example implementation (schematic only)
   INSERT INTO `shopify-dw.money_products.shopify_payments_reserve_configurations_v1` 
   (shop_id, is_active, rolling_reserve_percentage, hold_duration, created_at)
   VALUES (@shop_id, TRUE, 0.15, 30, CURRENT_TIMESTAMP())
   ```

2. **Document Decision Rationale**:
   - Risk factors that led to the decision
   - Quantitative metrics supporting the decision
   - Expected timeline for reserve reconsideration

3. **Monitor Performance**:
   - Track key metrics (chargeback rate, fulfillment times)
   - Set up alerts for significant changes
   - Review reserve after 90 days of stable performance

## Key Performance Indicators

The success of a reserve implementation is measured by:

1. **Chargeback Coverage**:
   - Percentage of chargebacks covered by reserved funds
   - Target: >90% coverage

2. **Reserve Efficiency**:
   - Ratio of reserved amount to actual losses
   - Target: 2:1 to 3:1 ratio

3. **Merchant Impact**:
   - Change in order volume after reserve implementation
   - Target: <5% reduction

## Conclusion

This comprehensive reserve analysis approach allows the Credit Risk team to:

1. Make data-driven reserve decisions based on multiple risk dimensions
2. Select the most appropriate reserve type for each merchant's risk profile
3. Protect Shopify from financial loss while minimizing merchant impact
4. Continuously monitor and adjust reserves based on merchant behavior

By combining traditional risk metrics with advanced signals like Trust Battery and HVCR predictions, this process provides a forward-looking risk assessment that balances merchant experience with financial protection.

---

*Last Updated: May 2024* 