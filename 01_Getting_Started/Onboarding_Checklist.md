# Credit Risk Team Onboarding Checklist

Welcome to the Credit Risk team! This checklist will guide you through the essential steps to get set up with the necessary tools, access, and resources to become productive quickly.

## Access Requirements

### BigQuery Access

- [ ] Request access to `shopify-dw` project in BigQuery
  - ⚠️ **Important**: Always use `shopify-dw` as the billing project for all queries and reports
  - Process: Submit a Clouddo request for BigQuery access with justification

### PII Data Access

- [ ] Claim the `sdp-pii` Clouddo permit for sensitive data access
  - Process: Submit a Clouddo request with manager approval
  - Review PII data handling guidelines before accessing sensitive data

### Service Accounts

- [ ] For Looker integration, submit a Helpdesk request for a Looker Service Account
  - Note: This requires Trust approval
  - Include justification for business need in your request

## Essential Tools Setup

- [ ] Install and configure [Google Cloud SDK](https://cloud.google.com/sdk/docs/install)
- [ ] Set up BigQuery access through the [Google Cloud Console](https://console.cloud.google.com/)
- [ ] Request access to Looker dashboards relevant to Credit Risk
- [ ] Set up Plex for data discovery
- [ ] Install SQL Helper for query assistance
- [ ] Set up AI Assistant tools for query development (optional)

## First Steps for New Team Members

1. [ ] Review the [Repository Navigation](./Repository_Navigation.md) guide
2. [ ] Read the [BigQuery Guide](../02_Tools/BigQuery_Guide.md) to understand our data environment
3. [ ] Explore the [Data Dictionary](../04_Data_Dictionary/Table_Overview.md) to familiarize yourself with key tables
4. [ ] Study the [SQL Best Practices](../03_SQL_Guide/SQL_Best_Practices.md) guide
5. [ ] Review at least one [Case Study](../07_Case_Studies/) to understand analysis workflows
6. [ ] Run your first query using one of the [Example Queries](../06_Example_Queries/)

## Key Resources to Review

- [Table Relationships](../04_Data_Dictionary/Table_Relationships.md) - Understanding how data is connected
- [Chargeback Analysis](../05_Domain_Knowledge/Chargeback.md) - Core domain knowledge
- [Risk Scoring](../05_Domain_Knowledge/Risk_Scoring.md) - Foundational risk concepts
- [SQL Guide](../03_SQL_Guide/SQL_Basics.md) - SQL fundamentals for credit risk analysis
- [Troubleshooting Guide](../08_Team_Resources/Troubleshooting_Guide.md) - Help with common issues

## Team Communication

- [ ] Join the following Slack channels:
  - `#credit-risk-team` - Main team channel
  - `#credit-risk-data` - Data-specific discussions
  - `#credit-risk-alerts` - Monitoring and alerts
- [ ] Schedule introduction meetings with key team members
- [ ] Review the [Collaboration Guide](../08_Team_Resources/Collaboration_Guide.md)

## First Week Goals

- [ ] Complete all access requests
- [ ] Run at least three example queries to validate your setup
- [ ] Create your first analysis with guidance from a team member
- [ ] Contribute to the Knowledge Base (fix a typo, add a comment, etc.)
- [ ] Attend team meetings and introduce yourself

## Questions?

If you have any questions during onboarding, please reach out to:
- Your direct manager
- The team's data specialist
- The #credit-risk-team Slack channel

---

*Last Updated: May 2024* 