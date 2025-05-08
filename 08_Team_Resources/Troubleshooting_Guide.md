# Credit Risk Tools Troubleshooting Guide

## Available Information

The example files demonstrate the use of several SQL functions and patterns that are commonly used in Credit Risk analysis:

- `SAFE_DIVIDE()` function for handling division operations safely
- `COALESCE()` for handling null values
- Window functions such as `RANK()` and `ROW_NUMBER()`
- Complex joins between multiple tables
- Date manipulation functions
- JSON handling with `JSON_VALUE()`

## Information Required

### BigQuery Issues

- Question 1: What are the most common BigQuery issues faced by the Credit Risk team?
- Question 2: What are the recommended query time limits and best practices for optimization?
- Question 3: Are there specific project or dataset access issues the team often encounters?

### Looker Issues

- Question 1: What Looker explores are available to the Credit Risk team?
- Question 2: What are common issues when creating or loading Looker dashboards?
- Question 3: Are there team-specific visualization standards or practices?

### SQL Helper Issues

- Question 1: What SQL helper tools are used by the team?
- Question 2: What are common pitfalls when using these tools?
- Question 3: Are there team-specific patterns or templates that should be used?

### Data Inconsistency Issues

Some of the example files suggest potential data inconsistencies that may need to be addressed:

1. **Chargeback Metrics**: The example files calculate chargeback rates in different ways:
   - Using transaction counts (`COUNT(*)`)
   - Using monetary values (chargeback amount / transaction amount)

However, more information is needed:

- Question 1: What is the official definition of chargeback rate used by the team?
- Question 2: How should discrepancies between different rate calculations be reconciled?
- Question 3: Are there known issues with specific tables that analysts should be aware of?

### Access and Permissions

The example files reference various datasets that likely require specific permissions:

- `shopify-dw.money_products.*`
- `sdp-prd-cti-data.intermediate.*`
- `sdp-prd-cti-data.base.*`

More information is needed:

- Question 1: What is the process for requesting access to these datasets?
- Question 2: Are there specific roles or permissions required for Credit Risk analysis?
- Question 3: How should PII data be handled in queries and reports?

### Common Error Messages

- Question 1: What are the most frequent SQL errors encountered by the team?
- Question 2: Are there team-specific error messages in custom tools or dashboards?
- Question 3: What is the process for reporting and resolving errors?

## Troubleshooting Process

Based on the SQL patterns seen in the example files, here are some recommended troubleshooting steps for complex queries:

1. Break down complex CTEs to test them individually
2. Verify date ranges are appropriate for the tables being queried
3. Check join conditions to prevent unintended cross joins
4. Use `SAFE_DIVIDE()` to prevent division by zero errors
5. Use `COALESCE()` to handle potential null values

Additional information needed:

- Question 1: What is the team's standard troubleshooting process?
- Question 2: Are there specific testing methodologies for verifying query results?
- Question 3: What query output validation processes are in place?

---

*Note: This document requires input from the Credit Risk team to complete. The information above is based solely on available documentation and example files. Once completed, this guide will help team members troubleshoot common issues with tools and data.*

*Last Updated: May 2024* 