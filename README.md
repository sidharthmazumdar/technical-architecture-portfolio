# Enterprise Systems & AI Architecture Portfolio
**Author:** Sidharth Mazumdar  
**Target Roles:** Principal Software Architect | Enterprise Architect | Head of Architecture / Engineering  
**Focus Areas:** Distributed Systems, Event-Driven Architecture, High-Throughput Java/Spring Boot Platforms, Production AI & Governance  
**Location:** Noida (NCR), India *(Relocating to Germany | Fast-Track EU Blue Card §18g / §81a AufenthG Eligible | Anabin H+ Recognized)*  

---

## Executive Overview
This portfolio documents production-grade architectural blueprints, Architectural Decision Records (ADRs), and technical governance standards developed across 14+ years of systems engineering. The architectures showcased here span Fortune 1 scale (Walmart Global Tech), Tier-1 capital markets and open banking (NatWest / RBS, European PSD2), and executive startup delivery as Co-Founder & CTO at Quaapton AI Solutions.

Every case study follows the **C4 Architectural Modeling Standard** (Context, Container, Component, Code/Sequence) with runnable Mermaid.js diagrams, concrete operational trade-offs, security compliance boundaries (GDPR, BaFin/PSD2), and FinOps optimization metrics, strictly reflecting the technology state-of-the-art at the time of execution.

---

## Repository Structure & Project Timelines

```text
├── README.md                                    # Executive summary, bio, and architectural principles
├── architecture-blueprints/
│   ├── CS-01-wms-nlp-discrepancy-engine.md     # Case Study 1: Inbound Slotting & NLP Engine (Walmart Scale, 2021–2023)
│   ├── CS-02-travel-platform-overhaul.md        # Case Study 2: Cloud Overhaul & Distributed Booking Engine (Travel Platform, 2025)
│   └── CS-03-fintech-ai-underwriting-copilot.md # Case Study 3: Forensic AI Underwriting Copilot & MCP Sandbox (Quaapton Finance, 2026)
├── architecture-records/
│   ├── ADR-001-transformer-onnx-vs-apis.md     # ADR 1: Fine-Tuned Transformer Models on ONNX vs. Cloud APIs (2022)
│   ├── ADR-002-event-driven-cqrs-booking.md    # ADR 2: Event-Driven Microservices, Java 21 Virtual Threads & CQRS (2025)
│   └── ADR-003-mcp-sandbox-vs-pure-llm.md      # ADR 3: Model Context Protocol (MCP) & Sandbox vs. Pure LLM Calculations (2026)
└── engineering-playbook/
    ├── 01-architecture-governance-rfcs.md      # Lightweight RFC lifecycle, ADR guidelines, and DDD boundaries
    ├── 02-observability-sla-standards.md       # OpenTelemetry, RED/USE metrics, structured logging, and APM
    └── 03-quality-gates-finops-security.md     # Trunk-based CI/CD, SonarQube gates, GDPR/FinOps compliance
```

---

## Architectural Principles & Core Tenets

1. **Era-Accurate Pragmatism over Trend Chasing:** Systems are designed using proven, secure technologies appropriate to their operating constraints and regulatory reality.
2. **Deterministic FinOps & Latency Governance:** AI and cloud capabilities must justify their unit economics. Model inference, vector storage, and cross-AZ data transfers must be gated by caching tiers, quantization optimizations, and budgeted p99 SLAs.
3. **Decoupled Asynchrony & Zero Cross-Domain Contamination:** Bounded contexts (DDD) maintain strict separation of concerns. Cross-domain data propagation is asynchronous via partitioned event streams (Kafka), preventing synchronous cascading failures.
4. **European Regulatory Compliance by Design:** Architectures handling EU customer or financial data strictly adhere to GDPR (automated right-to-erasure / tombstoning), BaFin/PSD2 security standards (mTLS, OIDC, strict audit logs), and the EU AI Act (human-in-the-loop auditability).
