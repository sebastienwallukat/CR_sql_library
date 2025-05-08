# Repository Navigation Guide

This guide will help you efficiently navigate the Credit Risk Team Knowledge Base repository to find the information you need.

## Repository Structure

The knowledge base is organized into the following main sections:

```
├── 01_Getting_Started/         # Onboarding and access information
├── 02_Tools/                   # Documentation for tools and platforms
├── 03_SQL_Guide/               # SQL best practices and patterns
├── 04_Data_Dictionary/         # Table documentation and relationships
├── 05_Domain_Knowledge/        # Domain-specific analysis techniques
├── 06_Example_Queries/         # Real-world SQL query examples
├── 07_Case_Studies/            # End-to-end analysis examples
└── 08_Team_Resources/          # Collaboration and team guidelines
```

## Finding Specific Information

### Looking for Table Information?

1. Start with [Table Overview](../04_Data_Dictionary/Table_Overview.md) for a comprehensive list of tables
2. Check [Table Relationships](../04_Data_Dictionary/Table_Relationships.md) to understand how tables connect
3. Visit specific domain table documentation:
   - [Chargeback Tables](../04_Data_Dictionary/Chargeback_Tables.md)
   - [Shop Tables](../04_Data_Dictionary/Shop_Tables.md)
   - [Risk Tables](../04_Data_Dictionary/Risk_Tables.md)
   - [Financial Tables](../04_Data_Dictionary/Financial_Tables.md)
   - [Trust Platform Tables](../04_Data_Dictionary/Trust_Platform_Tables.md)

### Need a Query Example?

1. Check [Full Query Examples](../06_Example_Queries/) for complete, end-to-end queries
2. Reference [Common Query Patterns](../06_Example_Queries/Common_Query_Patterns.md) for reusable snippets
3. Review domain-specific analyses in the [Domain Knowledge](../05_Domain_Knowledge/) section

### Have a Tool Question?

1. Visit the [Tools Documentation](../02_Tools/) directory
2. Find specific guides for:
   - [BigQuery Guide](../02_Tools/BigQuery_Guide.md)
   - [Plex Guide](../02_Tools/Plex_Guide.md)
   - [SQL Helper Guide](../02_Tools/SQL_Helper_Guide.md)
   - [AI Chat Tools](../02_Tools/AI_Chat_Tools.md)
   - [Looker Guide](../02_Tools/Looker_Guide.md)

### Learning About Credit Risk Domains?

1. Explore the [Domain Knowledge](../05_Domain_Knowledge/) directory
2. Key domains include:
   - [Chargeback Analysis](../05_Domain_Knowledge/Chargeback.md)
   - [GPV Analysis](../05_Domain_Knowledge/GPV.md)
   - [Risk Scoring](../05_Domain_Knowledge/Risk_Scoring.md)
   - [Reserves Management](../05_Domain_Knowledge/Reserves.md)
   - [Fraud Detection](../05_Domain_Knowledge/Fraud_Detection.md)

## Search Strategies

### GitHub Search

The most efficient way to find specific information is to use GitHub's search functionality:

1. Navigate to the repository in GitHub
2. Use the search bar at the top of the repository
3. Filter your search with keywords like:
   - `table:chargeback` to find chargeback table documentation
   - `query:risk score` to find risk scoring queries
   - `domain:fraud` to find fraud detection resources

### File Navigation

For browsing, follow this approach:

1. Start with the README.md at the repository root for an overview
2. Navigate to the relevant section based on your needs
3. Use the cross-references and links within documents to find related information

## Best Practices for Using the Knowledge Base

1. **Check document freshness**: Note the "Last Updated" date at the bottom of each document
2. **Verify query validity**: All queries should include a "Last Validated" date
3. **Contribute improvements**: If you find outdated information, submit updates
4. **Ask for help**: Use the team Slack channels for questions
5. **Start with examples**: For new analyses, start with an existing example and adapt it
6. **Reference sources**: When using information from the knowledge base, cite the source

## Contributing to the Knowledge Base

We encourage all team members to contribute to this knowledge base:

1. For small changes, edit directly in GitHub and submit a pull request
2. For larger contributions, follow the process in the [Collaboration Guide](../08_Team_Resources/Collaboration_Guide.md)
3. Always maintain consistent formatting and structure

## Need Help?

If you can't find what you're looking for:

1. Ask in the `#credit-risk-team` Slack channel
2. Check the [FAQ section](../08_Team_Resources/Troubleshooting_Guide.md)
3. Contact the knowledge base maintainer

---

*Last Updated: May 2024* 