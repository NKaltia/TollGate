# 0001. Database: PostgreSQL + pgvector

- Status: Accepted
- Date: 2026-11-24

## Context
Tollgate stores datasets, runs, trials, grader results, gateway traces and cache entries. The core operations are:
- statistical comparison of runs, which needs joins and grouping across runs, trials and test cases;
- correct budget accounting under concurrent requests, which needs transactions;
- similarity search over embeddings for the semantic cache;
- storage of traces whose content varies in shape.

## Options considered
1. **PostgreSQL + pgvector.** SQL for analysis; ACID transactions; vector search in the same database; JSONB for variable-shape data; already familiar to the author. Cons: vector search is less specialized than a dedicated vector database.
2. **MongoDB (+ Mongoose).** Flexible documents, a natural fit for traces; atomic update operators. Cons: joins and statistical queries are less natural; vector search relies on Atlas Vector Search, and the local option is unclear.
3. **PostgreSQL + MongoDB together.** Each used for its strength. Cons: two databases to run, back up and keep consistent; no measurable benefit at this scale.

## Decision
PostgreSQL with the pgvector extension, using JSONB columns for trace payloads.

## Consequences
- One database covers analysis, transactions and vector search.
- Schema changes need migrations (more discipline than a schemaless store).
- MongoDB experience, useful for the job market, is gained separately, not in this project.

## How we will know it was right
Comparison queries and cache lookups meet the latency targets. If vector search becomes a measured bottleneck, revisit.
