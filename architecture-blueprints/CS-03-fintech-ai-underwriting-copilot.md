# Case Study 3: Event-Driven AI Credit Underwriting & Forensic Document Verification Copilot
**Domain:** Fintech / Commercial Lending & SME Credit  
**Engineering Firm:** Quaapton AI Solutions Pvt. Ltd.  
**Timeline:** 2026  
**Role:** Co-Founder & Chief Technology Officer (CTO)  
**Status:** In Production  

---

## 1. Business Context & Problem Statement
In 2026, a mid-market digital lending enterprise managing commercial debt facilities and working capital loans faced operational scaling limits. Their credit risk team manually appraised complex borrowing dossiers—incorporating 12 months of multi-bank PDF statements, GST tax returns, MCA filings, and audited balance sheets.

Key challenges in the 2026 landscape:
* **Underwriting Latency:** Manual forensic appraisal took **48 to 72 hours per case**, leading to customer drop-off toward automated lenders.
* **Subtle Financial Fraud:** Manual checks routinely missed circular fund routing (round-tripping between related entities), inflated month-end balances, and tampered PDF metadata.
* **AI Hallucination & Compliance Governance:** Pure generative LLMs frequently hallucinated complex mathematical ratios (e.g., Debt Service Coverage Ratio - DSCR). Furthermore, strict banking regulations (BaFin / RBI guidelines) mandated deterministic auditability, explainable reasoning traces, and strict customer data privacy.

---

## 2. Target Architectural Metrics & KPIs
* **Turnaround Time (TAT):** Cut financial appraisal memo preparation from **48 hours to <15 minutes**.
* **Data Extraction Precision:** Achieve **$\ge 99.0\%$ precision** on irregular, multi-column bank tables and scanned tax ledgers.
* **Fraud Detection:** Detect 100% of circular fund transfers, balance tampering, and PDF metadata anomalies.
* **Regulatory Governance:** Enforce deterministic computation sandboxes for financial math and provide an auditable reasoning trail with mandatory Human-In-The-Loop (HITL) sign-off.

---

## 3. C4 Architectural Model

### Level 1: System Context Diagram (C4-L1)
```mermaid
graph TD
    Borrower["Borrower / Commercial Broker"]
    CreditAnalyst["Senior Credit Underwriter (HITL)"]

    subgraph Lending_Platform["Lending Enterprise Platform (2026)"]
        APIGateway["Secure API Gateway<br/>[mTLS, Encrypted Payload, OIDC]"]
        DocIngestion["Document Ingestion & OCR Mesh<br/>[Python 3.11 / LayoutLMv3 / Marker OCR]"]
        EventStream["Kafka Event Backbone<br/>[Topics: docs.received, forensic.cleared, cam.ready]"]
        UnderwritingCopilot["Agentic Underwriting Copilot<br/>[LangGraph, MCP Tooling, Azure OpenAI Private]"]
        CoreLMS["Core Loan Management System (LMS)<br/>[Java 21 Enterprise Microservices]"]
    end

    Borrower -->|"Upload Bank & Tax Dossiers (PDF)"| APIGateway
    APIGateway --> DocIngestion
    DocIngestion --> EventStream
    EventStream --> UnderwritingCopilot
    UnderwritingCopilot --> CoreLMS
    CreditAnalyst -->|"Review CAM Report & Approve"| CoreLMS
```

### Level 2: Container Diagram (C4-L2)
```mermaid
graph TD
    Client["Underwriter Web Portal (Next.js)"] -->|HTTPS| Kong["API Gateway (OAuth2 / OIDC)"]
    
    subgraph Processing_Mesh["Kubernetes Processing Namespace (2026)"]
        UploadWorker["Document Ingestion Worker<br/>[FastAPI / Python 3.11 / S3 Presigned URLs]"]
        DocParser["Layout-Aware Table Extraction Engine<br/>[LayoutLMv3, PyMuPDF, Marker]"]
        ForensicEngine["Forensic Transaction & Fraud Detector<br/>[Python / Polars / Graph NetworkX Engine]"]
        AgenticCopilot["Underwriting Agentic Copilot<br/>[LangGraph Runtime / MCP Tools / Azure OpenAI]"]
        DeterministicSandbox["Deterministic Financial Ratio Sandbox<br/>[Python NumPy / Decimal Math Engine]"]
    end

    subgraph Storage_Tier["Encrypted Persistence Tier"]
        S3Vault[("AWS S3 Encrypted Vault<br/>(KMS Customer-Managed Keys)")]
        KafkaCluster[("Apache Kafka 3.8 Cluster")]
        PostgresAudit[("Amazon Aurora PostgreSQL 16<br/>(CAM Data, Lineage & Audit Logs)")]
        PolicyStore[("Qdrant Vector DB v1.9<br/>(Credit Policy Manuals & Risk Rules)")]
    end

    Kong --> UploadWorker
    UploadWorker --> S3Vault
    UploadWorker -->|"Emit doc.received"| KafkaCluster
    KafkaCluster --> DocParser
    DocParser -->|"Structured Ledger JSON"| ForensicEngine
    ForensicEngine -->|"Verified Financial Ledger"| AgenticCopilot
    AgenticCopilot --> DeterministicSandbox
    DeterministicSandbox --> AgenticCopilot
    AgenticCopilot --> PolicyStore
    AgenticCopilot --> PostgresAudit
    AgenticCopilot -->|"Emit cam.ready"| KafkaCluster
    KafkaCluster --> CoreLMS["Core LMS Platform"]
```

### Level 3: Component Diagram (C4-L3 - Agentic Copilot & Ratio Sandbox)
```mermaid
graph LR
    subgraph Copilot_Module["Agentic Underwriting Copilot (2026)"]
        AgentRouter["LangGraph State Coordinator"]
        MCP_Client["Model Context Protocol (MCP) Client"]
        Sandbox["Deterministic Ratio Sandbox (NumPy/Polars)"]
        PolicyMatcher["Risk Policy Matcher (Qdrant Vector DB)"]
        CAM_Synthesizer["CAM Report Narrative Synthesizer"]
    end

    VerifiedJSON["Clean Ledger JSON"] --> AgentRouter
    AgentRouter --> MCP_Client
    MCP_Client -->|"Execute DSCR, Liquidity & Current Ratios"| Sandbox
    Sandbox -->|"Return Verified Math"| MCP_Client
    AgentRouter --> PolicyMatcher
    PolicyMatcher --> CAM_Synthesizer
    CAM_Synthesizer --> FinalCAM["Audited Credit Appraisal Memo (CAM JSON)"]
```

### Level 4: Execution Sequence (Document Analysis & Verification Pipeline)
```mermaid
sequenceDiagram
    autonumber
    actor Broker as Commercial Broker
    participant API as Ingestion Gateway
    participant S3 as S3 Encrypted Vault
    participant Kafka as Event Bus
    participant OCR as Layout-Aware Parser
    participant Forensic as Forensic Anomaly Engine
    participant Agent as LangGraph Underwriting Copilot
    participant Sandbox as Math Sandbox (Polars)
    participant LMS as Core LMS / Underwriter Portal

    Broker->>API: Upload 12 Months Bank Statements & Tax Returns
    API->>S3: Stream Encrypted Files (AES-256 KMS)
    API->>Kafka: Emit "doc.uploaded"
    Kafka->>OCR: Ingest Dossier
    OCR->>OCR: Extract Bounding-Box Tables & Transaction Rows
    OCR->>Kafka: Emit "doc.parsed_tables"
    
    Kafka->>Forensic: Ingest Transactions
    Forensic->>Forensic: Trace Counterparties & Detect Circular Round-Tripping
    Forensic->>Forensic: Verify PDF Font Consistency & End-of-Day Balances
    
    alt Forensic Fraud Detected
        Forensic->>Kafka: Emit "loan.fraud_flagged"
        Kafka->>LMS: Alert Underwriter (Red Warning Flag on Dashboard)
    else Financials Cleared
        Forensic->>Kafka: Emit "loan.forensics_cleared"
        Kafka->>Agent: Trigger Underwriting Copilot
        Agent->>Sandbox: Compute Financial Ratios (DSCR, Quick Ratio, Debt/EBITDA)
        Sandbox-->>Agent: Return Deterministic Exact Numbers
        Agent->>Agent: Query Qdrant for Credit Policy & Risk Guidelines
        Agent->>Agent: Synthesize Structured CAM Narrative & Justification
        Agent->>Kafka: Emit "cam.generated"
        Kafka->>LMS: Populate Underwriter Dashboard with CAM Report
    end
```

---

## 4. Key Architectural Trade-Offs & Decisions (2026)
1. **Model Context Protocol (MCP) & Sandbox vs. Pure LLM Calculations:** Commercial LLMs make arithmetic errors when computing complex corporate debt ratios. In this architecture, all arithmetic (DSCR, Current Ratio, Operating Cash Flows) is delegated via MCP tools to an isolated **deterministic Python sandbox**. The LLM receives verified mathematical outputs solely to synthesize qualitative risk narratives.
2. **Deterministic Data Privacy:** Private Azure OpenAI deployments were provisioned with enterprise zero-data-retention agreements, ensuring client financial filings are never utilized for model fine-tuning.

---

## 5. Measured Production Outcomes & Metrics
* **Appraisal TAT:** Average underwriting dossier preparation time dropped from **48 hours to 11 minutes** (**>95% time reduction**).
* **Extraction Precision:** Maintained **99.1% parsing accuracy** across 24 distinct banking statement and tax ledger formats.
* **Forensic Protection:** Uncovered **$3.2M in round-tripping and fraudulent transaction schemes** during the first two quarters of 2026 that had previously gone unnoticed in manual spot-checks.
* **Capacity Scaling:** Increased underwriter throughput by **4.5x**, allowing the lender to scale portfolio originations without adding operational headcount.
