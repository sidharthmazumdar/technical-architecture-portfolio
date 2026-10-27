# Engineering Playbook &mdash; Section 3: Quality Gates, FinOps & Security
**Author:** Sidharth Mazumdar  

---

## 1. Automated CI/CD Quality Gates
Delivery velocity requires automated, non-negotiable quality checks within CI/CD pipelines (GitLab CI, GitHub Actions, Jenkins):

```
[Code Push] ──> [Static Analysis & Secrets Check] ──> [Unit Tests (80%+)] ──> [Contract Testing (Pact)] ──> [Canary Deploy]
```

### Gate Requirements
1. **SonarQube Quality Gate:**
   * 0 Blocker or Critical bugs.
   * 0 Vulnerabilities or Security Hotspots.
   * Minimum **80% line and branch test coverage** on new code.
   * Code duplication $< 3.0\%$.
2. **Secret Scanning & Vulnerability Scanning:** Automated scanning (Trivy / Gitleaks) on every commit. Container images with unpatched CVEs (High/Critical) fail the build automatically.
3. **Consumer-Driven Contract Testing:** All inter-service REST and event contracts must be verified via Pact before merging to trunk.

---

## 2. FinOps & Cloud Cost Governance
* **Tagging Policy:** 100% of cloud resources (AWS / Azure / GCP) must bear mandatory tags: `Project`, `Owner`, `Environment` (`prod`, `staging`, `dev`), and `CostCenter`.
* **Dynamic Ephemeral Environments:** Non-production Kubernetes dev/staging namespaces automatically scale down to zero replicas during off-hours (20:00 to 07:00 local time) and weekends, saving up to 35% on cloud compute bills.
* **Storage Lifecycle Automation:** Application logs and S3 audit buckets enforce automated transition to Glacier/Cold storage after 30 days and hard expiration after 365 days.

---

## 3. Security, GDPR & European Regulatory Baseline
* **Principle of Least Privilege (PoLP):** Zero permanent IAM administrator access. Production cluster access is strictly gated via short-lived, role-based credentials (OIDC / Teleport / AWS IAM Identity Center).
* **GDPR Compliance (Right to Erasure & Auditability):**
  * Systems storing EU user information implement automated tombstoning protocols across primary databases and Kafka event partitions.
  * Personally Identifiable Information (PII) is encrypted at rest using envelope encryption (KMS) and masked in all log outputs.
* **Payment & Banking Standards (PSD2 / PCI-DSS):** Zero persistent storage of raw cardholder data (PAN/CVV); strict mTLS communication between payment microservices and third-party gateways.
