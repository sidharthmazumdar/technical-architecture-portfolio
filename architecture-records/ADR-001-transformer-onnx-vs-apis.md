# ADR-001: Fine-Tuned Transformer Models on ONNX Runtime vs. Cloud APIs
- **Status:** Accepted  
- **Date:** 2023  
- **Deciders:** Sidharth Mazumdar (Lead Systems Architect), Walmart Supply Chain Platform Team  
- **Technical Context:** High-throughput inbound trailer discrepancy resolution (Walmart WMS, 2021–2023).  

---

## Context and Problem Statement
In 2022, dock associates across 40+ regional distribution hubs needed to resolve thousands of manifest discrepancies daily. Commercial public generative APIs were either unreleased or strictly prohibited under enterprise infosec policies. The system needed high-accuracy entity extraction (PO numbers, SKU quantities, damaged carton classifications) directly on mobile terminals with p99 latency < 50ms.

## Decision Drivers
* Strict on-premise / private cloud data isolation (zero public cloud data leakage).
* Deterministic token classification and entity recognition.
* Low-latency CPU inference (<30ms) without dedicated GPU instances.
* High reliability under peak holiday intake surges.

## Considered Options
* **Option 1:** Heuristic Regex and Fuzzy Levenshtein String Matching.
* **Option 2:** Fine-Tuned Domain RoBERTa / DistilBERT quantized to INT8 on ONNX Runtime & Triton.
* **Option 3:** Cloud OCR Cognitive APIs (External Cloud Managed Services).

## Decision Outcome
**Chosen Option:** **Option 2: Fine-Tuned Domain RoBERTa deployed on ONNX Runtime and Triton Inference Server**.

### Implementation Strategy
* Annotated a domain dataset of 45,000 internal Bill of Lading manifests and dock notes.
* Fine-tuned a RoBERTa-base model for Named Entity Recognition (NER) and sentence-level discrepancy classification.
* Quantized weights to INT8 using ONNX Runtime, deploying containerized inference pods on Azure AKS.

## Pros and Cons of the Options
* *Option 1 (Regex/Fuzzy):* Fast, but brittle; failed on messy OCR scans, carrier shorthand, and OCR text distortions.
* *Option 2 (Fine-Tuned RoBERTa on ONNX - Chosen):* 97.4% precision; 22ms p99 inference latency on standard CPU pods; 100% private data isolation.
* *Option 3 (Cloud Cognitive APIs):* High external cost per transaction; potential data governance violations.

## Rollback Strategy
If Triton inference pods experience memory saturation or index latency exceeds 60ms, the system falls back to rule-based ASN lookup tables, queuing unstructured notes for manual supervisor review.
