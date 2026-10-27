# Engineering Playbook &mdash; Section 2: Observability & SLA Standards
**Author:** Sidharth Mazumdar  

---

## 1. The Three Pillars of Observability
Systems under our architectural governance must implement comprehensive telemetry using **OpenTelemetry (OTel)** open standards, avoiding proprietary vendor lock-in.

### 1. Traces (Distributed Tracing)
* Every inbound HTTP/gRPC request is assigned a W3C-compliant `traceparent` at the API Gateway.
* The `trace_id` must be propagated across all asynchronous message headers (Kafka, JMS, RabbitMQ) and thread boundaries.
* Sub-system calls (database queries, cache lookups, external API invocations) must be captured as child spans.

### 2. Metrics (RED & USE Frameworks)
* **For Request-Driven Services (REST / gRPC / AI APIs) &mdash; RED Method:**
  * **Rate:** Requests per second (RPS).
  * **Errors:** Number of failed requests per second (broken down by HTTP 4xx vs 5xx).
  * **Duration:** Response latency distribution (tracked at p50, p90, p99, and p99.9).
* **For Infrastructure & Kubernetes Pods &mdash; USE Method:**
  * **Utilization:** CPU and Memory consumption percentages.
  * **Saturation:** Queue depths, thread pool backlog, socket wait times.
  * **Errors:** Pod eviction events, OOM kills, network dropped packets.

### 3. Structured Logging Standard
Unstructured plain-text logs are strictly prohibited in production runtimes. All services must emit structured JSON logs to `stdout`:

```json
{
  "timestamp": "2026-10-27T10:15:30.124Z",
  "level": "INFO",
  "service": "booking-service",
  "environment": "production",
  "trace_id": "4bf92f3577b34da6a3ce929d0e0e4736",
  "span_id": "00f067aa0ba902b7",
  "user_id": "usr_99812",
  "action": "reserve_slot",
  "status": "SUCCESS",
  "duration_ms": 34.2,
  "message": "Trip slot reserved successfully"
}
```

---

## 2. Production SLA & SLO Governance
* **Core Transactional APIs (Bookings, Payments):** 
  * SLO: 99.95% availability; p99 latency < 250ms.
  * Error Budget: 21.6 minutes of downtime per month.
* **AI Ingress & Model Inference:**
  * SLO: 99.9% availability; p99 latency < 500ms.
* **Alerting Protocol:** Alerts are triggered on **burn-rate consumption of the error budget**, not on transient point-in-time anomalies.
