# AI Tools for Credit Risk Analysis

This guide provides comprehensive information on using AI chat and assistance tools effectively for Credit Risk analysis tasks. It covers best practices, prompt engineering techniques, and specific examples for common Credit Risk scenarios.

## Table of Contents

- [Overview of AI Tools](#overview-of-ai-tools)
- [When to Use AI Tools](#when-to-use-ai-tools)
- [Prompt Engineering Guide](#prompt-engineering-guide)
- [Credit Risk-Specific Prompts](#credit-risk-specific-prompts)
- [Verification Best Practices](#verification-best-practices)
- [Hemingway Writing Assistant](#hemingway-writing-assistant)
- [Cursor](#cursor)
- [Limitations and Considerations](#limitations-and-considerations)
- [Recommended Tools](#recommended-tools)

## When to Use AI Tools

| Scenario | AI Tool Value | Examples |
|----------|--------------|----------|
| **Complex Query Development** | 🟢 High | Generating multi-CTE queries for complex chargeback analysis |
| **SQL Troubleshooting** | 🟢 High | Diagnosing and fixing query errors |
| **Documentation Creation** | 🟢 High | Creating table documentation based on schema |
| **Exploratory Analysis** | 🟡 Medium | Getting suggestions for data exploration angles |
| **Optimization Tasks** | 🟡 Medium | Improving query performance |
| **Critical Decision Making** | 🔴 Low | Setting reserve levels or making risk decisions |
| **Sensitive Data Handling** | 🔴 Low | Processing merchant PII information |

### Best Use Cases

1. **SQL Generation**: Generate SQL for complex tasks with multiple CTEs or window functions
2. **Explanation**: Understand complex queries or database schemas
3. **Documentation**: Draft or improve documentation of tables, queries, or processes
4. **Learning**: Understand new concepts or techniques related to credit risk analytics
5. **Brainstorming**: Generate ideas for new analytics approaches or metrics

### When Not to Use AI Tools

1. **Critical financial decisions**: Never rely solely on AI 
2. **Sensitive data handling**: Avoid sharing PII or sensitive merchant information
3. **Final verification**: Always manually verify generated queries before using in production
4. **Compliance matters**: Don't rely on AI for compliance or regulatory interpretations
5. **Outdated information**: Remember AI may not have the most current Shopify-specific knowledge

## Prompt Engineering Guide

### Prompt Structure Best Practices

1. **Context Setting**
   ```markdown
   I'm a Credit Risk analyst at Shopify analyzing merchant chargeback patterns. I'm using BigQuery to query these tables: [LIST TABLES]. I need to...
   ```

2. **Clear Objective**
   ```markdown
   Generate a SQL query that calculates the 30-day, 60-day, and 90-day chargeback rates for merchants with over $10,000 in GPV.
   ```

3. **Constraints and Requirements**
   ```markdown
   The query must:
   - Use CTEs for readability
   - Filter out test accounts
   - Avoid using self-joins for performance reasons
   - Include appropriate error handling with SAFE_DIVIDE
   ```

4. **Example Format or Structure**
   ```markdown
   Format the output as:
   - First CTE should handle GPV calculation
   - Second CTE should handle chargeback counts
   - Final query should join these with clear column names
   ```

5. **Level of Detail Needed**
   ```markdown
   Please include detailed comments explaining each step, particularly the window function logic.
   ```

### Iteration Techniques

1. **Step-by-Step Refinement**
   - Start with a basic version of your question
   - Review the response and identify areas for improvement
   - Ask for specific enhancements to the initial response

2. **Provide Examples**
   - Include snippets of existing code or queries that work well
   - Reference specific patterns your team uses
   - Show expected output format

3. **Clarify Misunderstandings**
   - If the AI misunderstands, clarify with more detail
   - Correct any assumptions about table structure
   - Provide schema details if needed

## Credit Risk-Specific Prompts

### Example 1: Chargeback Analysis Query

```sql
I need a SQL query for BigQuery to analyze chargeback patterns for high-risk merchants. 

Tables available:
- `sdp-prd-cti-data.intermediate.shop_chargeback_rates_current` (columns: shop_id, cb_count_30d, cb_rate_30d, etc.)
- `sdp-prd-cti-data.intermediate.shop_daily_order_fraud_facts` (columns: shop_id, date, daily_sp_paid_amount_usd, daily_sp_orders)
- `sdp-prd-cti-data.intermediate.shop_insights_shops` (columns: shop_id, shop_name, trust_battery, trust_battery_score)

Requirements:
1. Identify shops with chargeback rates over 1% in the last 30 days
2. Only include shops with at least $5,000 in GPV and at least 3 chargebacks
3. Include the Trust Battery score and category
4. Sort by chargeback rate descending
5. Format as a CTE-based query with clear comments

Please include comments explaining key logic.
```

### Example 2: Reserve Recommendation Logic

```markdown
I'm working on documentation for our reserve recommendation process. Can you help me explain the logic for calculating appropriate reserve levels based on merchant risk factors?

Key inputs to consider:
- Unfulfilled order exposure (from trust_platform_shop_risk_attributes_daily)
- Chargeback history (frequency, amount, reasons)
- Trust Battery score
- Industry risk level
- Failed transfer history

The explanation should:
1. Clearly describe how each factor contributes to risk
2. Explain the rationale for different reserve types (fixed vs. rolling)
3. Include considerations for setting appropriate reserve percentages
4. Note which factors are most predictive of future losses

The audience is new Credit Risk team members who understand basic risk concepts but need to learn our specific approach.
```

### Example 3: SQL Troubleshooting

```sql
I'm getting an error in this chargeback analysis query. Can you help identify the issue?

ERROR: Division by zero: SAFE_DIVIDE(SUM(cb_amount_usd_30d), SUM(gpv_usd_30d))

WITH shop_metrics AS (
  SELECT
    shop_id,
    cb_count_30d,
    cb_amount_usd_30d,
    gpv_usd_30d,
    SAFE_DIVIDE(cb_count_30d, NULLIF(transaction_count_30d, 0)) AS cb_rate_30d
  FROM
    `sdp-prd-cti-data.intermediate.shop_chargeback_rates_current`
  WHERE
    DATE(processed_at) = CURRENT_DATE() - 1
)

SELECT
  shop_id,
  cb_count_30d,
  gpv_usd_30d,
  cb_rate_30d,
  SAFE_DIVIDE(SUM(cb_amount_usd_30d), SUM(gpv_usd_30d)) AS overall_cb_value_rate
FROM
  shop_metrics
GROUP BY 1, 2, 3, 4
HAVING cb_count_30d > 0
ORDER BY cb_rate_30d DESC
LIMIT 100

I need to fix this error and ensure proper handling of potential zero values.
```

### Example 4: Data Exploration Guidance

```markdown
I'm exploring our chargeback and fraud data to identify new risk indicators. I have access to:

- Full chargeback history (reason codes, amounts, timing)
- Shop order history
- Trust Battery scores
- Merchant business models and industries

What are some non-obvious data patterns or relationships I should look for that might predict higher credit risk? Please suggest 5-7 specific data exploration angles, including what metrics to calculate and how they might indicate increased risk.
```

### Example 5: Documentation Template

```markdown
I need to create documentation for the table `sdp-prd-cti-data.intermediate.trust_platform_shop_risk_attributes_daily`.

The table contains daily shop risk metrics including:
- Unfulfilled order counts and values
- Chargeback counts
- Complaint metrics
- Risk scores

Please provide a template for comprehensive table documentation that includes:
1. Overview section
2. Schema documentation with column descriptions
3. Common usage patterns
4. Sample queries section
5. Best practices section

Format it in Markdown suitable for our knowledge base.
```

## Verification Best Practices

Always verify AI-generated content before using it in production:

1. **Test Generated SQL**
   - Run with limited rows or in development environment first
   - Verify results match expected patterns
   - Check performance (execution plan, optimization)

2. **Validate Logic**
   - Ensure business logic is correctly implemented
   - Compare results to existing validated queries
   - Check edge cases (nulls, zeroes, extreme values)

3. **Review for Security and Best Practices**
   - Ensure proper handling of sensitive data
   - Verify proper use of billing project
   - Check query follows team standards

4. **Peer Review Critical Work**
   - Have team members review complex generated queries
   - Document verification process for important analyses
   - Note any adjustments needed to AI output

## Hemingway Writing Assistant

Hemingway is a writing assistant tool that helps improve the clarity, conciseness, and readability of your documentation and communications.

### What Hemingway Does

Hemingway helps you:
- Eliminate unnecessary words and phrases
- Simplify complex sentences
- Replace jargon with clear language
- Identify passive voice and suggest active alternatives
- Highlight adverbs that can be replaced with stronger verbs

### When to Use Hemingway

| Use Case | Value |
|----------|-------|
| Documentation | 🟢 High - Creates clearer, more accessible documentation |
| Analysis Reports | 🟢 High - Makes complex findings easier to understand |
| Team Communications | 🟡 Medium - Improves clarity of important messages |
| Quick Messages | 🔴 Low - Not necessary for routine communications |

### Hemingway Best Practices

1. **Draft First, Edit Later**
   - Write your content completely before running it through Hemingway
   - Focus on getting your ideas down, then use Hemingway to refine

2. **Consider Context**
   - Some technical terms are necessary despite complexity
   - Balance readability with technical accuracy

3. **Use for Important Documents**
   - Prioritize using Hemingway for documentation that will be widely read
   - Focus on customer-facing or cross-team documentation

### Example: Before and After Hemingway

**Before:**

```markdown
The utilization of the chargeback analysis methodology that has been implemented by our team enables the identification of merchants that might potentially be exhibiting characteristics that could be indicative of elevated risk profiles based on a multitude of factors including, but not limited to, transaction patterns, chargeback frequencies, and various other metrics that have been determined to be of significance.
```

**After:**

```markdown
Our chargeback analysis method identifies high-risk merchants based on transaction patterns, chargeback frequencies, and other key metrics.
```

## Cursor

Cursor is an AI-powered code editor that enhances productivity when writing SQL queries and other code.

### What is Cursor?

Cursor is a code editor similar to VS Code but with integrated AI capabilities that can:
- Generate SQL queries based on natural language descriptions
- Explain existing code
- Suggest improvements and optimizations
- Help troubleshoot errors
- Complete code as you type

### Using Cursor for Credit Risk Analysis

1. **Query Development in Cursor**
   - Use Cursor's AI capabilities to draft complex queries
   - Leverage AI suggestions for query structure and optimization
   - Get explanations of complex SQL patterns

2. **Common Workflow**
   - Draft query in Cursor with AI assistance
   - Test and refine in Cursor
   - Validate through appropriate channels
   - Fix any issues identified
   - Execute final query through approved methods

### Cursor Best Practices

1. **Provide Clear Context**
   - Tell Cursor what you're trying to accomplish
   - Specify table names and structures
   - Mention any specific requirements or constraints

2. **Verify AI Suggestions**
   - Always review generated code before using
   - Check table and column references
   - Verify business logic is correctly implemented

3. **Iterative Refinement**
   - Start with simple queries and build complexity
   - Use Cursor to explain complex parts you don't understand
   - Ask for specific improvements to your code

## Limitations and Considerations

### Data Privacy and Security

- **Never share PII**: Don't paste queries containing sensitive merchant information
- **Sanitize examples**: Remove identifying information before sharing
- **Review outputs**: Check AI-generated content for accidental inclusion of sensitive data
- **Use internal tools**: Prefer Shopify-approved AI tools when available

### Accuracy Limitations

- **Data recency**: AI models may not have knowledge of recent Shopify table changes
- **Shopify-specific knowledge**: Generic AI may not understand Shopify's unique data structures
- **Complex domain knowledge**: AI may not fully understand credit risk nuances
- **Hallucinations**: AI can confidently present incorrect information

### Usage Guidelines

1. **Document AI usage**: Note when AI was used for significant work
2. **Maintain accountability**: You are responsible for AI-generated content
3. **Use appropriate tools**: Different AI tools have different strengths
4. **Balance efficiency with verification**: The time saved using AI should not be lost in extensive verification

## Recommended Tools

| Tool | Best Use Cases | Access Method | Shopify Approval Status |
|------|----------------|---------------|-------------------------|
| **SQLHelper** | SQL optimization, error correction | https://sql-helper.shopify.io/ | ✅ Approved |
| **Hemingway** | Documentation clarity and readability | https://chat.shopify.io/ (select Hemingway agent) | ✅ Approved |
| **Cursor** | AI-assisted code editing | Install via Okta | ✅ Approved |


---