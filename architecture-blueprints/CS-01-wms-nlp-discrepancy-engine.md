# Case Study 1: Predictive Inbound Slotting & NLP-Driven Discrepancy Resolution Engine
**Domain:** Global Retail Supply Chain & WMS 
**Company:** Walmart Global Tech  
**Timeline:** August 2021 &ndash; December 2023  
**Architect:** Sidharth Mazumdar  
**Status:** In Production  

---

## 1. Business Context & Problem Statement
Between 2021 and 2023, Walmart's North American logistics distribution centers processed tens of thousands of freight trailers and inbound pallets daily. Discrepancies between Advanced Shipping Notices (ASNs), Electronic Data Interchange (EDI) feeds (ANSI X12 856), and physically unloaded dock inventory caused widespread operational bottlenecks.

Key operational hurdles in the 2021–2023 enterprise environment:
* **Unstructured Bill of Lading (BOL) & Manifest Friction:** Receiving associates encountered non-standard scanned manifests and handwritten carrier exception slips (e.g., damaged cartons, missing pallet tags, partial cross-dock overrides).
* **Manual Resolution Delays:** Discrepancy resolution required manual cross-referencing across terminal screens and relational inventory tables, taking **15 to 45 minutes per shipment** and compounding trailer yard congestion.
* **Enterprise Security & Tech Constraints:** Commercial generative LLM APIs were either unreleased or strictly prohibited under enterprise infosec policies. All machine learning, entity extraction, and semantic matching models had to be fine-tuned, audited, and hosted internally within private Azure/Walmart Cloud clusters using proven neural architectures (fine-tuned RoBERTa, DistilBERT, LayoutLMv2, and ONNX Runtime).

---

## 2. Target Architectural Metrics & KPIs
* **Resolution Speed:** Cut trailer intake discrepancy triage and resolution time from **25+ minutes to <45 seconds**.
* **Semantic Extraction Accuracy:** Achieve **$\ge 96.5\%$ F1-score** on entity extraction (SKU, PO number, quantity variance, damage codes) from noisy OCR manifests.
* **System Throughput:** Sustain intake peak volumes exceeding **35,000 shipment events/hour** across 40+ regional distribution hubs.
* **Deterministic Traceability:** 100% auditable discrepancy journals integrated with core WMS ledgers with zero unverified generative hallucinations.

---

## 3. C4 Architectural Model

### Level 1: System Context Diagram (C4-L1)
```mermaid
graph TD
    DockWorker["Receiving Dock Associate / Yard Lead"]
    EDI_Feeds["Carrier EDI Systems<br/>(EDI 856 ASN / EDI 214)"]

    subgraph Walmart_WMS["Walmart Supply Chain Cloud Platform (2021-2023)"]
        Gateway["Enterprise Ingress Gateway (Apigee)<br/>[mTLS, OAuth2, Rate Limiting]"]
        IntakeEngine["Intelligent Inbound Discrepancy Engine<br/>(Python 3.9, FastAPI, Triton Inference Server)"]
        CoreWMS["Core WMS Enterprise Services<br/>(Java 11, Spring Boot 2.7, Apache Kafka)"]
        YardDB["Enterprise Inventory & Yard Store<br/>(Azure Cosmos DB / Cassandra / DB2)"]
    end

    DockWorker -->|"Scan Manifest / Report Issue (Zebra TC77 Android)"| Gateway
    EDI_Feeds -->|"B2B Asynchronous Feeds"| Gateway
    Gateway --> IntakeEngine
    IntakeEngine -->|"Publish Reconciled Event"| CoreWMS
    CoreWMS --> YardDB
```

### Level 2: Container Diagram (C4-L2)
```mermaid
graph TD
    Client["Handheld Terminal (Zebra Android)"] -->|HTTPS / REST| APIGW["API Gateway (Apigee Enterprise)"]
    
    subgraph WMS_AI_Mesh["Private ML Inference Namespace (Azure AKS)"]
        IngressSvc["Manifest Ingress & Sanitizer<br/>[FastAPI / Python 3.9]"]
        OCRParser["Document Normalizer & OCR Pipeline<br/>[Tesseract OCR + LayoutLMv2]"]
        TransformerModel["Fine-Tuned RoBERTa Token Classifier<br/>[Triton Inference Server / ONNX Runtime INT8]"]
        SemanticSearch["Semantic Error Matcher<br/>[Milvus Vector DB v2.2 / HNSW Index]"]
        RulesEngine["Deterministic Resolution Engine<br/>[Java 11 / Spring Boot 2.7 Worker]"]
    end

    subgraph Storage_Layer["Enterprise Persistence Tier"]
        RedisCache[("Redis 6.2 Cluster<br/>(ASN Manifest Cache & Query Embeddings)")]
        KafkaCluster[("Apache Kafka 3.2 Cluster<br/>(Topic: wms.inbound.discrepancies)")]
        WMS_DB[("Core WMS Database<br/>(Cassandra & DB2)")]
    end

    APIGW --> IngressSvc
    IngressSvc --> OCRParser
    OCRParser --> TransformerModel
    TransformerModel -->|"Extract Dense Embeddings (768-d)"| SemanticSearch
    SemanticSearch -->|"Match Known Discrepancy Patterns"| RedisCache
    SemanticSearch --> RulesEngine
    RulesEngine -->|"Publish Verified Event"| KafkaCluster
    KafkaCluster --> WMS_DB
```

### Level 3: Component Diagram (C4-L3 - ML Inference & Extraction Pipeline)
```mermaid
graph LR
    subgraph Inference_Pipeline["Triton Inference & Extraction Pipeline"]
        Tokenizer["HuggingFace Fast Tokenizer<br/>(WordPiece / BPE)"]
        ONNX_Runtime["ONNX Runtime Engine<br/>(Quantized INT8 RoBERTa Model)"]
        NER_Head["Token Classification Head<br/>(SKU, PO, Qty, Damage Code)"]
        SimilarityMatcher["Cosine Distance Matcher<br/>(Milvus HNSW Index)"]
    end

    RawText["Cleaned Manifest Text"] --> Tokenizer
    Tokenizer --> ONNX_Runtime
    ONNX_Runtime --> NER_Head
    ONNX_Runtime --> SimilarityMatcher
    NER_Head --> StructuredJSON["Normalized Discrepancy Payload"]
    SimilarityMatcher --> StructuredJSON
```

### Level 4: Execution Sequence (Trailer Discrepancy Triage)
```mermaid
sequenceDiagram
    autonumber
    actor Associate as Dock Associate
    participant Handheld as Mobile App
    participant GW as API Gateway
    participant AI as AI Intake Service (Triton)
    participant Redis as Redis Cache
    participant Rules as WMS Resolution Rules
    participant Kafka as Enterprise Kafka
    participant WMS as WMS Core Database

    Associate->>Handheld: Scan Damaged Pallet + Enter Note ("Crushed bottom case, short 4 units on PO-8921")
    Handheld->>GW: POST /v1/inbound/discrepancy
    GW->>AI: Forward Payload (Text + Barcode)
    AI->>AI: Run Fine-Tuned RoBERTa NER (PO=8921, Qty=-4, Code=DAMAGED)
    AI->>Redis: Lookup Expected ASN Manifest (PO-8921)
    Redis-->>AI: Return Master Shipping Manifest Data
    AI->>AI: Reconcile Physical vs Expected Quantities
    AI->>Rules: Pass Normalized Discrepancy Schema
    Rules->>Rules: Evaluate Rule: Route to Inspection Bay vs Reject Carrier
    Rules-)Kafka: Emit "wms.pallet.discrepancy_resolved"
    Kafka->>WMS: Update Inventory Ledger & Yard Staging
    Rules-->>GW: Return Action Directive
    GW-->>Handheld: Display Action: "Affix Red Tag, Divert to Bay 14B" (Latency: 110ms)
```

---

## 4. Key Architectural Trade-Offs & Decisions (2021–2023)
1. **Fine-Tuned RoBERTa/ONNX vs. General Heuristics:** Handcrafted regex and fuzzy matching failed on messy OCR scans and dock worker abbreviations. Fine-tuning a domain-specific RoBERTa model quantized to INT8 on ONNX Runtime yielded **sub-25ms inference latency** on standard CPU instances without GPU cluster expense.
2. **Strict On-Prem / VPC Data Sovereignty:** To protect confidential supplier pricing and shipment volumes, all embedding generation and inference ran strictly inside private Azure tenant boundaries with zero external egress.

---

## 5. Measured Production Outcomes & Metrics
* **Triage Latency:** Reduced average shipment discrepancy handling time from **25 minutes to 38 seconds** (**>97% latency improvement**).
* **Dock Throughput:** Improved trailer turnaround times at pilot distribution hubs by **18%**, eliminating off-site yard staging demurrage penalties.
* **Extraction Precision:** Maintained **97.4% precision and 96.1% recall** on freight issue classifications across diverse North American carrier manifests.
* **Operational Stability:** Sustained peak Q4 holiday inventory volumes with **99.99% system availability**.
