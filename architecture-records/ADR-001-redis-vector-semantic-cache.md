# ADR-001: Redis Semantic Vector Caching vs. Raw LLM Invocations
- **Status:** Accepted  
- **Date:** 2023-04-12  
- **Deciders:** Sidharth Mazumdar (Lead Architect), Cloud & AI Platform Engineering Board  
- **Technical Context:** High-throughput enterprise GenAI applications (Walmart scale) handling 100k+ repetitive operational queries daily.  

---

## Context and Problem Statement
Frontline logistics workers across hundreds of distribution hubs submit high volumes of operational queries daily. Over 40% of queries are semantically identical (e.g., *"How to handle damaged hazardous battery pallet?"* vs. *"SOP for cracked battery package"*). 

Querying Azure OpenAI GPT-4 directly on every invocation results in:
1. Significant operational costs ($0.03-$0.06 per query).
2. Latency spikes exceeding 2,500ms, causing UI bottlenecks.
3. Rapid exhaustion of Azure OpenAI provisioned throughput units (PTUs).

## Decision Drivers
* Reduce operational inference expenditure by at least 35%.
* Maintain p99 latency < 50ms for previously resolved queries.
* Prevent false-positive semantic matches that could deliver incorrect safety procedures.
* Low infrastructure complexity and operational overhead.

## Considered Options
* **Option 1:** Raw LLM Invocations with Standard HTTP In-Memory Hashing (SHA-256).
* **Option 2:** Distributed Redis Enterprise with Vector Search (HNSW Indexing on Embeddings).
* **Option 3:** Dedicated Vector Database (Pinecone / Milvus) as a standalone cache tier.

## Decision Outcome
**Chosen Option:** **Option 2: Redis Enterprise with Vector Search**.  
Redis provides sub-millisecond in-memory read speeds combined with native vector similarity indexing (`FT.SEARCH` using HNSW). 

### Evaluation & Cutoff Benchmark
* Inbound queries are converted into 384-dimensional dense vectors using a lightweight local embedding model or `text-embedding-3-small`.
* A strict **Cosine Similarity threshold of $\ge 0.94$** was established based on empirical evaluation across 5,000 historical test queries.
* Any match $\ge 0.94$ is returned instantly from the Redis cache.
* Matches $< 0.94$ trigger a cache miss, proceeding through the LangGraph RAG pipeline, with the result written back to Redis with a 7-day sliding TTL.

## Pros and Cons of the Options

### Option 1: Standard SHA-256 Text Hashing
* *Pros:* Simple to implement; zero embedding overhead.
* *Cons:* Negligible cache-hit ratio (<3.8%) because natural language phrasing almost never matches verbatim.

### Option 2: Redis Enterprise Vector Search (Chosen)
* *Pros:* Sub-25ms response times on cache hits; proven clustering and failover; native TTL support for automatic cache invalidation; dual-use as session store.
* *Cons:* Requires vector embedding generation on the ingress path (adds ~12ms on misses).

### Option 3: Dedicated Standalone Vector DB
* *Pros:* High scalability for billions of vectors.
* *Cons:* Excessive architectural complexity; higher operational cost; higher query latency compared to Redis in-memory storage for working-set sizes under 10 million items.

## Rollback & Fallback Strategy
If Redis vector search experiences cluster degradation or index latency exceeds 40ms, the API Gateway circuit breaker automatically flips to direct RAG execution, bypassing the cache tier without dropping traffic.
