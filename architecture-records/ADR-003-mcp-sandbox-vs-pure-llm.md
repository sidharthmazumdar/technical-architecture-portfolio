# ADR-003: Model Context Protocol (MCP) & Sandbox vs. Pure LLM Calculations
- **Status:** Accepted  
- **Date:** 2026 
- **Deciders:** Sidharth Mazumdar (CTO), Enterprise Risk & AI Architecture Board  
- **Technical Context:** AI Credit Underwriting Copilot (Fintech Commercial Lending Domain, 2026).  

---

## Context and Problem Statement
In automated commercial financial statement appraisal, extracting and analyzing financial records from scanned balance sheets, bank records, and tax filings requires zero mathematical error. In 2026, pure LLM generation still exhibits non-deterministic rounding and calculation errors on financial ratios (e.g., Debt Service Coverage Ratio - DSCR). 

Furthermore, banking regulations (BaFin / RBI) demand fully verifiable audit trails and explainability for credit decisions.

## Decision Drivers
* 100% mathematical precision (zero tolerance for calculated ratio hallucinations).
* 99%+ table extraction precision on degraded, scanned PDF ledgers.
* Transparent audit lineage for regulatory compliance.
* Strict enterprise data isolation (zero customer data retained for base-model training).

## Considered Options
* **Option 1:** Direct Multi-Modal LLM Ingestion (Passing raw PDF pages straight to LLM vision endpoints).
* **Option 2:** Layout-Aware OCR (LayoutLMv3/Marker) + Deterministic Code Sandbox (via Model Context Protocol) + LLM Qualitative Reasoning.
* **Option 3:** Proprietary Off-the-Shelf Document AI SaaS.

## Decision Outcome
**Chosen Option:** **Option 2: Layout-Aware Parsing + Deterministic Python Sandbox via MCP Tools + Guardrail LLM**.

### Architecture Workflow
1. **Document Parsing:** Scanned documents are parsed using LayoutLMv3 and Marker into structured bounding-box tables and normalized JSON.
2. **Deterministic Computation:** Mathematical calculations (DSCR, Quick Ratio, Debt/EBITDA, Average Monthly Balance) are executed strictly in an isolated Python/Polars sandbox via Model Context Protocol (MCP) tool execution.
3. **Qualitative Synthesis:** The verified mathematical ratios and parsed ledger summaries are provided to private Azure OpenAI instances to synthesize the Credit Appraisal Memo (CAM) narrative.
4. **Audit Validation:** Automated evaluators verify every claim in the generated text against source JSON before human underwriter sign-off.

## Pros and Cons of the Options
* *Option 1 (Direct Multi-Modal LLM):* Simple pipeline, but prone to math errors and token window limitations on 100-page bank statements.
* *Option 2 (Layout-Aware + Sandbox via MCP - Chosen):* 100% mathematical accuracy; transparent audit trail connecting calculations to source ledger cells; fully compliant with regulatory standards.
* *Option 3 (Commercial SaaS):* High per-page cost; poor handling of non-standard regional Indian and European tax ledgers; vendor lock-in.

## Rollback & Security Assurance
* Model instances deployed in private VPC endpoints with Customer-Managed Keys (AWS KMS).
* Enterprise zero-data-retention agreements guarantee banking confidentiality.
