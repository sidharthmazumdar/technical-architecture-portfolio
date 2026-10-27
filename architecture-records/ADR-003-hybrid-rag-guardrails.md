# ADR-003: Deterministic Hybrid RAG & Guardrails for Financial Documents
- **Status:** Accepted  
- **Date:** 2025-02-10  
- **Deciders:** Sidharth Mazumdar (CTO), Enterprise Risk & AI Steering Committee  
- **Technical Context:** AI Credit Underwriting Copilot (Quaapton Finance Domain).  

---

## Context and Problem Statement
In automated financial statement evaluation, extracting data from scanned balance sheets, bank records, and tax filings requires high precision. Pure LLM generation frequently suffers from mathematical hallucinations (e.g., miscalculating Debt Service Coverage Ratios) or formatting errors on unstandardized tables. 

Furthermore, banking regulatory bodies (such as BaFin / RBI) require fully explainable credit appraisals and verifiable data lineage.

## Decision Drivers
* Strict mathematical accuracy (0% tolerance for calculated ratio hallucinations).
* 98%+ table extraction precision on degraded scanned PDFs.
* Deterministic explainability for compliance audits.
* Strict data isolation (no customer data used for third-party model retraining).

## Considered Options
* **Option 1:** Direct Multi-Modal LLM Ingestion (Pass raw PDF images straight to GPT-4o / Claude 3.5).
* **Option 2:** Hybrid Pipeline: Layout-Aware OCR (LayoutLMv3/PaddleOCR) + Deterministic Code Sandbox + LLM Qualitative Reasoning.
* **Option 3:** Off-the-shelf Commercial Document AI SaaS (e.g., AWS Textract / Google Document AI).

## Decision Outcome
**Chosen Option:** **Option 2: Hybrid Extraction Pipeline with Deterministic Sandbox and Guardrail LLM**.

### Architecture Workflow
1. **Document Parsing:** Scanned documents are processed via layout-aware OCR models that output structured bounding-box tables as normalized JSON.
2. **Deterministic Computation:** Mathematical calculations (Debt Service Coverage Ratio - DSCR, Current Ratio, Average Monthly Balance - AMB, Cheque Bounce Ratios) are executed in an isolated Python/NumPy sandbox.
3. **Qualitative Synthesis:** The verified mathematical ratios and parsed ledger excerpts are passed into an Azure OpenAI instance to draft the Credit Appraisal Memo (CAM) narrative.
4. **Guardrail Validation:** An automated evaluator checks every generated claim against source financial JSON before human sign-off.

## Pros and Cons of the Options

### Option 1: Direct Multi-Modal LLM
* *Pros:* Simplest ingestion flow.
* *Cons:* Inconsistent mathematical calculations; risk of token truncation on 100-page bank statements; uncontrollable hallucination risks.

### Option 2: Hybrid Pipeline (Chosen)
* *Pros:* 100% mathematical precision; transparent audit trails linking calculations directly to specific ledger rows; regulatory compliance compliance.
* *Cons:* Requires custom maintenance of OCR bounding-box parsing models.

### Option 3: Standard Commercial Document AI SaaS
* *Pros:* Fast time-to-market.
* *Cons:* High per-page cost at scale; poor handling of non-standard regional Indian and European tax filing formats; vendor lock-in.

## Compliance & Security Assurance
* Model instances deployed within private VPC boundaries using Customer-Managed Encryption Keys (AWS KMS).
* Enterprise contractual agreements ensure zero data retention by model providers.
