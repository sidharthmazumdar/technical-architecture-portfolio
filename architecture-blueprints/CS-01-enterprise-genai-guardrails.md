# Case Study 1: Enterprise GenAI Guardrail & Semantic Cache Architecture
**Domain:** Global Retail Supply Chain & WMS (Fortune 1 Scale &mdash; Walmart Global Tech)  
**Architect:** Sidharth Mazumdar  
**Status:** In Production  

---

## 1. Business Context & Problem Statement
At global retail scale (serving hundreds of regional distribution centers across North America), frontline operations teams, logistics dispatchers, and warehouse supervisors generated over **120,000 repetitive operational queries daily** regarding warehouse inventory variances, SOP compliance, inbound SKU status, and carrier schedules. 

Direct routing of natural language queries to commercial LLMs (Azure OpenAI GPT-4) introduced significant operational risks:
* **High Inference Costs:** Token consumption approached $6,500/day ($195k+/month) with linear cost growth.
* **Unpredictable Latency:** p99 response times fluctuated between **1,800ms and 3,400ms**, causing unacceptable delays for handheld terminal operators.
* **Hallucination & Compliance Risks:** Inadvertent leakage of internal inventory pricing, PII, and non-deterministic answers on critical warehouse hazardous material procedures.

---

## 2. Target Architectural Metrics & KPIs
* **Latency:** Reduce p99 query response time from **>2,200ms to <120ms** for repetitive and semantically equivalent queries.
* **Cost Reduction:** Cut model inference expenditure by **$\ge 40\%$** within 90 days.
* **Guardrail Determinism:** 100% masking of sensitive PII (employee IDs, proprietary vendor SKU costs) and zero unauthorized SOP overrides.
* **High Availability:** 99.95% system uptime across multi-region Azure cloud infrastructure.

---

## 3. C4 Architectural Model

### Level 1: System Context Diagram (C4-L1)
```mermaid
graph TD
    User["Warehouse Operator / Logistics Lead"]
    
    subgraph Enterprise_Boundary["Enterprise Retail Platform Boundary"]
        Gateway["Enterprise API Gateway (Kong / Envoy)<br/>[mTLS, Rate Limiting, JWT Auth]"]
        GenAI_Platform["GenAI Semantic Cache & Guardrail Platform<br/>[FastAPI, LangGraph, Python]"]
        WMS["Core WMS & Inventory Core<br/>[Java Spring Boot Services]"]
        VectorDB["Enterprise Knowledge Vector DB<br/>[Qdrant / Milvus Cluster]"]
    end
    
    LLM_Cloud["Azure OpenAI / LLM Endpoints<br/>[GPT-4o / Claude 3.5 Sonnet]"]

    User -->|"HTTPS / REST / WebSocket"| Gateway
    Gateway -->|"Internal Service Mesh"| GenAI_Platform
    GenAI_Platform -->|"Query Master State"| WMS
    GenAI_Platform -->|"RAG Hybrid Retrieval"| VectorDB
    GenAI_Platform -->|"Inference on Cache Miss"| LLM_Cloud
```

### Level 2: Container Diagram (C4-L2)
```mermaid
graph TD
    Client["Handheld Terminal / Web App"] -->|HTTPS| Kong["API Gateway (Envoy / Kong)"]
    
    subgraph GenAI_Runtime["GenAI Orchestration Cluster (Azure AKS)"]
        IngressService["Ingress & Auth Controller<br/>[FastAPI]"]
        PIIFilter["PII Sanitizer & Prompt Guardrail<br/>[Presidio + Regex Rules]"]
        SemanticCache["Semantic Cache Engine<br/>[Redis Enterprise Vector Search]"]
        Orchestrator["Agentic Workflow Engine<br/>[LangGraph Runtime]"]
        HallucinationEngine["Response Validator & Grounding Evaluator<br/>[G-Eval Guardrail]"]
    end

    subgraph Data_Storage["Enterprise Tier Data Stores"]
        RedisCluster[("Redis Vector DB<br/>Cosine Similarity Threshold: 0.94")]
        KnowledgeDB[("Qdrant Vector DB<br/>Chunked SOPs & Manuals")]
        AuditLog[("Kafka Event Bus<br/>Topic: ai.audit.telemetry")]
    end

    subgraph Cloud_Models["External LLM Infrastructure"]
        AzureLLM["Azure OpenAI Dedicated Cluster<br/>(Private Endpoints)"]
    end

    Kong --> IngressService
    IngressService --> PIIFilter
    PIIFilter --> SemanticCache
    SemanticCache -->|"Vector KNN Search"| RedisCluster
    SemanticCache -->|"Cache Hit (p99 < 25ms)"| IngressService
    SemanticCache -->|"Cache Miss"| Orchestrator
    Orchestrator -->|"Fetch Embeddings & SOPs"| KnowledgeDB
    Orchestrator -->|"Execute Guarded Prompt"| AzureLLM
    AzureLLM --> HallucinationEngine
    HallucinationEngine --> SemanticCache
    HallucinationEngine --> IngressService
    IngressService -.->|"Async Telemetry"| AuditLog
```

### Level 3: Component Diagram (C4-L3 - Semantic Cache & Guardrail Module)
```mermaid
graph LR
    subgraph Cache_Component["SemanticCacheService (Python / FastAPI)"]
        TextNormalizer["Text Normalizer<br/>(Lowercase, Lemmatize, Whitespace)"]
        EmbeddingWorker["Embedding Generator<br/>(text-embedding-3-small, batching)"]
        KNNMatcher["HNSW Vector Matcher<br/>(Cosine Sim >= 0.94)"]
        TTLManager["Dynamic TTL & Eviction Controller"]
    end

    subgraph Guardrail_Component["GuardrailEngine"]
        InputSanitizer["Input Sanitizer (Prompt Injection Detection)"]
        OutputScrubber["Output Grounding Evaluator"]
    end

    InputText["Inbound Query"] --> InputSanitizer
    InputSanitizer --> TextNormalizer
    TextNormalizer --> EmbeddingWorker
    EmbeddingWorker --> KNNMatcher
    KNNMatcher -->|"Similarity >= 0.94"| CachedResponse["Return Cached Output"]
    KNNMatcher -->|"Similarity < 0.94"| LLMPipeline["Route to LangGraph Agent"]
    LLMPipeline --> OutputScrubber
    OutputScrubber --> TTLManager
    TTLManager -->|"Async Write to Redis"| RedisCache[("Redis Vector Store")]
```

### Level 4: Execution Sequence (Cache Hit vs. Cache Miss)
```mermaid
sequenceDiagram
    autonumber
    actor Op as Warehouse Lead
    participant GW as API Gateway
    participant Guard as PII & Injection Guard
    participant SC as Semantic Cache (Redis)
    participant Agent as LangGraph Orchestrator
    participant LLM as Azure OpenAI
    participant Kafka as Audit Event Log

    Op->>GW: POST /api/v1/query (e.g. "What is protocol for damaged battery pallet?")
    GW->>Guard: Validate JWT & Sanitize Payload
    Guard->>Guard: Strip PII, Check Jailbreak / Prompt Injections
    Guard->>SC: Compute Vector & Query Nearest Neighbors (KNN=1)
    
    alt Cosine Similarity >= 0.94 (Semantic Cache Hit)
        SC-->>GW: Return Pre-Validated Answer (p99: 22ms)
        GW-->>Op: 200 OK + "X-Cache-Status: HIT"
        GW-)Kafka: Produce Audit Record (Cost: $0.000)
    else Cosine Similarity < 0.94 (Cache Miss)
        SC->>Agent: Trigger RAG Execution Workflow
        Agent->>Agent: Query Qdrant Vector Store for SOP chunks
        Agent->>LLM: Send Grounded Prompt (Private Link)
        LLM-->>Agent: Raw Generated Response
        Agent->>Guard: Evaluate Faithfulness & Hallucination Score
        Guard-->>SC: Store Question Vector + Answer (TTL: 7 Days)
        SC-->>GW: Formatted Verified Response (p99: 1,420ms)
        GW-->>Op: 200 OK + "X-Cache-Status: MISS"
        GW-)Kafka: Produce Audit Record (Tokens, Latency, Model ID)
    end
```

---

## 4. Key Architectural Trade-Offs & Decisions
1. **Semantic Similarity vs. Exact Hashing:** Exact MD5/SHA256 hashing yielded a negligible cache-hit ratio (<3.8%) due to natural phrasing variances across warehouse workers. Redis vector search with HNSW indexing and a tuned **0.94 cosine similarity cutoff** increased the valid cache-hit ratio to **48.2%** while eliminating false-positive matches.
2. **Private Endpoint Isolation:** In adherence to enterprise infosec standards, all Azure OpenAI instances were locked behind Azure Private Link endpoints, forbidding public internet egress.

---

## 5. Measured Production Outcomes & Metrics
* **Cache-Hit Ratio:** Stabilized at **46.8%** across 120,000+ daily queries.
* **Latency Improvements:** 
  * Cache-Hit p99 Latency: **24ms** (down from 2,400ms direct LLM).
  * Overall System p99 Latency: Reduced to **380ms** across all blended traffic.
* **Direct Cost Savings:** Model API invocation expenditure dropped from **$195,000/month to $104,000/month** (**46.6% monthly cost reduction**), saving ~$1.09M annualized.
* **Reliability:** Maintained **99.98% availability** during peak Q4 holiday retail surge.
