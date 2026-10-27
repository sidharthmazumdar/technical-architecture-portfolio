# Case Study 2: Full-Stack Cloud Architecture & Distributed Booking Engine Overhaul
**Client:** High-Growth Experiential Travel Company (Leading Adventure & Group Travel Platform)  
**Engineering Firm:** Quaapton AI Solutions Pvt. Ltd.  
**Timeline:** Calendar Year 2025  
**Role:** Co-Founder & Chief Technology Officer (CTO)  
**Status:** In Production  

---

## 1. Business Context & Problem Statement
In 2025, the experiential travel company expanded operations across multi-country group expeditions, dynamic seasonal campaigns, and multi-currency payment splits. The existing monolithic backend, originally hosted on legacy virtual servers, could no longer handle explosive mobile traffic.

Critical technical bottlenecks identified in early 2025:
* **Monolithic Thread Exhaustion:** Standard thread-per-request architectures on legacy runtimes locked up during synchronized promotional flash sales, producing cascading HTTP 504 Gateway Timeouts.
* **Inventory Race Conditions & Overbooking:** High concurrency on limited-capacity expeditions (e.g., Arctic tours, specialized treks) caused database record locking and double-bookings.
* **Slow Edge Asset Delivery:** Heavy media catalogs resulted in mobile First Contentful Paint (FCP) exceeding **3.8 seconds**, hurting marketing conversion rates.

---

## 2. Target Architectural Metrics & KPIs
* **Core Web Vitals:** Reduce FCP from **3.8s to <1.0s**; sub-1.5s total page load globally.
* **Concurrency & Scale:** Support **30,000 concurrent active users** and **1,800 checkout requests/minute** during flash sales without service degradation.
* **Zero Overbooking Guarantee:** Eliminate booking concurrency race conditions using distributed locking and asynchronous reservation queues.
* **Resilience:** Multi-AZ AWS EKS deployment achieving 99.98% availability with automated container recovery.

---

## 3. C4 Architectural Model

### Level 1: System Context Diagram (C4-L1)
```mermaid
graph TD
    Traveler["Traveler / Customer<br/>(Mobile & Web)"]
    Admin["Operations & Trip Leaders"]

    subgraph Platform_Boundary["Travel Platform Cloud Architecture (2025)"]
        Edge["Cloudflare Enterprise CDN & WAF<br/>[Edge Workers, HTTP/3, Dynamic Caching]"]
        CoreBackend["Microservices Mesh (AWS EKS 1.30)<br/>[Java 21 Virtual Threads + Go 1.23 Services]"]
        PaymentEngine["Payment Gateway Integration Service<br/>[Stripe, Razorpay, PayU Webhooks]"]
        NotificationEngine["Asynchronous Notification Engine<br/>[WhatsApp API, SendGrid, Amazon SNS]"]
    end

    ExternalPG["Payment Gateways (Razorpay / Stripe)"]
    GDS_ERP["Inventory & Logistics Management"]

    Traveler -->|"Browse & Book (HTTPS / HTTP3)"| Edge
    Edge --> CoreBackend
    CoreBackend --> PaymentEngine
    PaymentEngine --> ExternalPG
    CoreBackend --> NotificationEngine
    Admin -->|"Admin Console (RBAC / OIDC)"| CoreBackend
    CoreBackend -.->|"Inventory Reconciliation"| GDS_ERP
```

### Level 2: Container Diagram (C4-L2)
```mermaid
graph TD
    Client["Client Browser / Mobile App"] -->|HTTPS / HTTP/3| CF["Cloudflare Enterprise Edge & WAF"]
    CF -->|Static Assets & WebP Images| S3["AWS S3 + CloudFront Edge"]
    CF -->|Dynamic API Traffic| ALB["AWS Application Load Balancer"]

    subgraph AWS_EKS_Cluster["AWS EKS (Kubernetes 1.30 Cluster)"]
        BookingService["Booking & Reservation Engine<br/>[Java 21 / Spring Boot 3.3 / Virtual Threads]"]
        CatalogService["Itinerary & Catalog Read Service<br/>[Go 1.23 / High-Throughput Read Cache]"]
        PaymentService["Payment & Settlement Worker<br/>[Java 21 / Spring Boot 3.3]"]
        SearchService["Dynamic Filter & Search Service<br/>[Node.js 22 / Amazon OpenSearch]"]
    end

    subgraph Data_Tier["Multi-AZ Data Tier"]
        AuroraMaster[("Amazon Aurora PostgreSQL 16<br/>(Primary Transactional Writer)")]
        AuroraReplica[("Amazon Aurora Read Replicas<br/>(Autoscaling 2-5 Read Nodes)")]
        RedisCluster[("Redis 7.2 Cluster<br/>(Redlock Distributed Locks & State)")]
        KafkaCluster[("Amazon MSK (Apache Kafka 3.8)<br/>(Event Streaming Backbone)")]
    end

    ALB --> BookingService
    ALB --> CatalogService
    ALB --> SearchService

    CatalogService --> RedisCluster
    CatalogService --> AuroraReplica
    
    BookingService -->|"Redlock Lock Acquisition"| RedisCluster
    BookingService --> AuroraMaster
    BookingService -->|"Publish booking.reserved"| KafkaCluster
    
    KafkaCluster --> PaymentService
    PaymentService --> ExternalPG["Payment Gateway Webhooks"]
    PaymentService --> AuroraMaster
```

### Level 3: Component Diagram (C4-L3 - Booking Engine & Distributed Lock)
```mermaid
graph LR
    subgraph Booking_Microservice["Booking Service Components (Java 21 / Spring Boot 3.3)"]
        BookingAPI["Booking REST Controller<br/>(Project Loom Virtual Threads)"]
        LockManager["Distributed Lock Coordinator<br/>(Redisson Redlock on Redis 7.2)"]
        InventoryValidator["Inventory Capacity Validator"]
        StateEngine["Booking State Machine<br/>(PENDING -> RESERVED -> CONFIRMED)"]
        EventPublisher["Kafka 3.8 Event Publisher"]
    end

    BookingAPI --> LockManager
    LockManager -->|"Acquire Lease (10s TTL)"| RedisLock[("Redis 7.2 Cluster")]
    LockManager --> InventoryValidator
    InventoryValidator --> StateEngine
    StateEngine -->|"Write Lock"| DB[(Aurora PostgreSQL 16)]
    StateEngine --> EventPublisher
    EventPublisher -->|"Emit booking.reserved"| KafkaTopic[("Kafka: trip.reservations")]
```

### Level 4: Execution Sequence (Concurrent Booking & Settlement Flow)
```mermaid
sequenceDiagram
    autonumber
    actor Customer as Traveler
    participant Gateway as Cloudflare / ALB
    participant BookSvc as Booking Engine (Spring Boot 3.3)
    participant Lock as Redis 7.2 Distributed Lock
    participant DB as Aurora PostgreSQL 16
    participant Kafka as Amazon MSK (Kafka 3.8)
    participant PaySvc as Payment Worker
    participant PG as Payment Gateway (Stripe/Razorpay)

    Customer->>Gateway: POST /v1/trips/{id}/reserve (Slot: 1)
    Gateway->>BookSvc: Forward Request with Idempotency Key
    BookSvc->>Lock: Try Acquire Lock(trip_departure_id, timeout=3s)
    
    alt Lock Acquired
        BookSvc->>DB: Check Available Inventory & Hold Slot
        DB-->>BookSvc: Slot Allocated (Status: HELD_PENDING_PAYMENT, Expiry: 15m)
        BookSvc->>Lock: Release Lock
        BookSvc-)Kafka: Publish event: "booking.slot_reserved"
        BookSvc-->>Customer: 201 Created + Secure Checkout Session URL
        
        Customer->>PG: Complete Payment
        PG->>PaySvc: Webhook Callback (payment.succeeded)
        PaySvc->>Kafka: Publish event: "payment.verified"
        Kafka->>BookSvc: Consume "payment.verified"
        BookSvc->>DB: Transition Status -> CONFIRMED
        BookSvc-)Kafka: Publish event: "notification.dispatch_ticket"
    else Lock Contention / Sold Out
        BookSvc-->>Customer: 409 Conflict ("High demand: slot currently locked. Try again.")
    end
```

---

## 4. Key Architectural Trade-Offs & Decisions (2025)
1. **Java 21 Virtual Threads (Project Loom) vs. Reactive WebFlux:** Rather than rewriting services to reactive patterns (which introduces debugging overhead and stack trace loss), adopting Java 21 Virtual Threads on Spring Boot 3.3 allowed writing clean imperative code while scaling to **10,000+ concurrent requests per node** with near-zero memory footprint.
2. **CQRS with Read Replicas & Edge Caching:** Completely decoupled browse/search paths from checkout transactions. Read routes are served directly via Cloudflare edge workers, Redis caches, and Aurora read replicas, protecting writer DB capacity during flash campaigns.

---

## 5. Measured Production Outcomes & Metrics
* **Page Load Speed:** Average mobile page load decreased from **5.2s to 1.4s**; Google Lighthouse performance score improved from **41 to 94**.
* **Traffic Resilience:** Successfully supported flash marketing campaigns with **32,000 concurrent users** and **1,850 API requests/second** with **0% downtime**.
* **Overbooking Elimination:** Reduced booking race conditions and overbooking errors to **absolute zero (0%)**.
* **Conversion Impact:** Checkout drop-off decreased by **34%**, contributing to a **28% net booking increase across the first two quarters of 2025**.
