# System Design: Keyword Search over Posts

## Requirements
- Search posts by keyword.
- Support required keywords (`AND`) and excluded keywords (`NOT`).
- Rank results by keyword frequency or a defined relevance score.
- Support post creation, updates, and deletions.
- Assumptions: case-insensitive token search, near-real-time indexing, cursor-based pagination.

## High-Level Architecture
- **API service:** validates requests, parses queries, enforces authorization.
- **PostgreSQL:** source of truth for post content and versions.
- **Transactional outbox:** records change events atomically with database updates.
- **Kafka (or equivalent):** transports post-change events.
- **Indexing workers:** consume events and synchronize the search index.
- **Elasticsearch:** inverted index, Boolean search, scoring, sharding, and replicas.
- **Search coordinator/API:** merges results, applies pagination, and returns post data.

## Data Structures and Storage
- **Post document:** `post_id`, `author_id`, `content`, `created_at`, `version`.
- **Inverted index:** term dictionary maps each keyword to a postings list of matching documents.
- **Postings:** typically include document ID and term frequency; positions can support phrase search.
- **Forward index (conceptual):** maps post ID to its old terms and frequencies; useful for understanding or implementing manual index updates.
- With Elasticsearch, its internal inverted index is managed by Elasticsearch; the application normally does not maintain a separate forward index just for search updates.

## Indexing / Write Path
1. Save the post change in PostgreSQL.
2. Insert a `PostCreated`, `PostUpdated`, or `PostDeleted` event into the outbox in the **same database transaction**.
3. An outbox relay or CDC tool (e.g., Debezium) publishes committed events to Kafka.
4. Indexing workers consume events and update/delete the Elasticsearch document.
5. Elasticsearch refresh makes changes searchable; indexing is generally near-real-time, not immediately consistent.

## Transactional Outbox
- Prevents the dual-write failure where the database commits but event publication fails.
- The post update and outbox event commit together or roll back together.
- Relay retries publication; duplicate delivery is possible.
- Make consumers idempotent using event IDs and post versions.
- Ignore stale/out-of-order versions; use durable retries and monitor queue lag.

## Updating a Post
- In a custom index: read old terms/frequencies from the forward index, compute new terms/frequencies, remove obsolete postings, update changed frequencies, add new postings, then save the new forward-index record.
- In Elasticsearch: submit the updated document using a stable document ID; Elasticsearch maintains its internal index structures.
- For deletes, delete or tombstone the document and ensure deleted content cannot be returned while cleanup is pending.
- Use version checks to prevent stale events from overwriting newer content.

## Query Path
1. Parse and normalize the query.
2. Translate required terms into `must` clauses and excluded terms into `must_not` clauses.
3. Execute the query on Elasticsearch shards.
4. Filter by authorization/visibility before a result counts toward top K.
5. Score eligible matches and retrieve top candidates.
6. Merge shard results, apply deterministic ordering, and return results with a cursor.

## Boolean Matching
- Required terms: intersect their postings lists.
- Excluded terms: subtract documents matching any excluded term.
- For a custom index, begin with the shortest required postings list to reduce candidate work.
- Apply exclusions **before final pagination**; otherwise pages may contain too few results.
- If there are only excluded terms, define explicit product behavior (often require at least one positive term).

## Ranking
- Simple raw-frequency score: `score(document, query) = sum(term_frequency(term, document))` for required query terms.
- Excluded terms determine eligibility; they do not add to the score.
- Elasticsearch's default `_score` is relevance scoring (typically BM25), **not necessarily raw occurrence count**.
- If exact raw frequency ranking is required, implement and benchmark a deliberate scoring strategy; do not assume default `_score` satisfies it.
- Use a bounded min-heap of size K in a custom implementation; use ranking-aware pruning (e.g., WAND/MaxScore) when appropriate.
- Use deterministic tie-breaking, e.g. `score DESC, post_id ASC`.

## Distributed Search and Pagination
- Start with document-based sharding; each shard maintains a local inverted index.
- Each shard returns enough top candidates to guarantee the global top K; coordinator merges shard results.
- Use replicas for availability and load distribution.
- Prefer cursor-based pagination (`search_after`) over deep offsets.
- For stable pagination, include deterministic sort fields and consider a point-in-time/search snapshot when consistency across pages matters.

## Scale and Reliability
- Compress postings and use efficient index segments.
- Optimize common terms and expensive queries; use skip data and ranking-aware pruning where supported.
- Monitor p95/p99 search latency, indexing lag, queue backlog, shard health, errors, and stale-result rates.
- Use retries, dead-letter handling where appropriate, replayable events, and periodic reconciliation.
- Cache only where correctness and authorization semantics remain safe.
- Size storage based on document count, terms per document, postings overhead, compression, replicas, and index metadata.

## Important Interview Trade-offs
- **Elasticsearch vs. custom engine:** prefer Elasticsearch for production unless requirements justify building a search engine.
- **Freshness vs. latency:** asynchronous indexing improves write latency but allows brief search staleness.
- **Raw frequency vs. relevance:** clarify whether the requirement is literal term count or relevance ranking.
- **Document sharding vs. term sharding:** document sharding is a straightforward starting point; it fans queries out to shards.
- **Correctness:** updates, deletions, authorization, duplicates, and out-of-order events must be handled explicitly.

## 2-Minute Summary
- Store authoritative posts in PostgreSQL.
- Use a transactional outbox to reliably publish post-change events to Kafka.
- Index posts asynchronously in Elasticsearch.
- Use Boolean queries for required and excluded terms.
- Use explicit scoring if raw keyword frequency is required; default Elasticsearch relevance is not identical to raw frequency.
- Let Elasticsearch manage its internal inverted index.
- Handle retries with idempotency and post versions.
- Merge ranked shard results and use cursor-based pagination.
- Monitor indexing lag, query latency, shard health, and result correctness.
