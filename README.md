Overview
This project focuses on identifying and fixing MongoDB performance issues in a simulated customer support ticketing system. The objective was to analyze slow queries, design efficient indexes, and replace inefficient search patterns with scalable alternatives.

The emphasis of this assignment is not only on improving execution time, but also on understanding why MongoDB behaves the way it does and how to design for real-world production systems.


Dataset
Database: DEXKOR
Collection: TICKETS
Total Documents: 50,000


Each ticket document contains:
Tenant information
Ticket status and priority
Subject, description, and tags
Agent assignment
Creation and update timestamps

Data Insertion Strategy
Documents were inserted in batches to efficiently load a large dataset and simulate realistic production ingestion behavior.

Part 1: Slow Query Analysis – Ticket Listing
Query
This query retrieves open tickets for a tenant, sorted by most recent creation time.

db.TICKETS.find({
  tenantId: "tenant_1",
  status: "open",
  createdAt: { $gte: ISODate("2025-01-01") }
})
.sort({ createdAt: -1 })
.limit(20)


Issue Observed (Before Indexing)
MongoDB performed a collection scan
All 50,000 documents were examined
Sorting was done in memory
Execution time increased with dataset size

Root Cause
No index existed to support the combination of filtering and sorting fields used in the query.


Optimization Applied
A compound index was created:

{ tenantId: 1, status: 1, createdAt: -1 }

Reasoning
Equality fields (tenantId, status) were placed first
Range and sort field (createdAt) was placed last
This allowed MongoDB to both filter and sort using the index

Result (After Indexing)
Query switched from COLLSCAN to IXSCAN
Documents examined dropped significantly
Sorting was handled by the index
Execution time improved

Part 2: Regex Search Performance Issue
Query
db.TICKETS.find({
  description: { $regex: "refund", $options: "i" }
})

Issue Observed
MongoDB performed a full collection scan
All 50,000 documents were examined
Execution time scaled linearly with dataset size

Explanation
Regex searches cannot efficiently leverage B-tree indexes, especially when patterns are not anchored. This makes them unsuitable for scalable text search in production systems.

Part 3: Full-Text Search Optimization
Text Index Created

To replace regex-based search, a native MongoDB text index was created on:
{
  subject: "text",
  description: "text",
  tags: "text"
}

Text Search Query
db.TICKETS.find(
  { $text: { $search: "refund delayed response" } },
  { score: { $meta: "textScore" } }
)
.sort({ score: { $meta: "textScore" } })

Execution Analysis
MongoDB used the TEXT_MATCH stage
The query leveraged the text index
Relevance scores were calculated using textScore
No collection scan was performed

Note on Result Size
In this dataset, all documents contained similar text values, resulting in many matches. While execution time remained noticeable, the key improvement is that the query is now index-driven, ranked, and scalable, which is critical for real-world workloads.

Part 4: Index Design Summary
Use Case	                     Index
Open tickets dashboard	       tenantId + status + createdAt
Agent workload view	           agentId + status
SLA escalation checks	         createdAt
Tag-based filtering	           tags
Full-text search	             subject + description + tags

Indexes were designed based on actual query patterns rather than indexing fields indiscriminately.

Key Learnings
Proper index field ordering has a major impact on query performance
Collection scans do not scale with growing datasets
Regex-based searches should be avoided for large-scale search
Native MongoDB text search provides indexed, ranked, and scalable search behavior
Performance optimization requires understanding both queries and data distribution

Conclusion
This assignment demonstrates a systematic approach to MongoDB performance tuning by identifying bottlenecks, applying targeted indexing strategies, and replacing inefficient search patterns with scalable alternatives. The focus was on designing solutions that would perform well under real-world production conditions
