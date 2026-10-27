# ADR-002: Event-Driven Microservices & CQRS vs. Monolithic Architecture
- **Status:** Accepted  
- **Date:** 2025 
- **Deciders:** Sidharth Mazumdar (CTO), Platform Engineering Team  
- **Technical Context:** Core backend overhaul for high-growth travel company handling high-concurrency booking and inventory management (2025).  

---

## Context and Problem Statement
In early 2025, the experiential travel company's monolithic web service struggled with promotional flash sales. Browsing traffic consumed database connection pools, causing checkout transactions to time out and generating inventory race conditions (double bookings on limited-capacity tours).

## Decision Drivers
* Complete isolation of high-volume read traffic (itinerary browsing) from critical write operations (slot reservation).
* Zero overbooking errors under concurrent checkout spikes (>1,500 bookings/min).
* Independent scalability of domain services (Catalog, Booking, Payments, Notifications).

## Considered Options
* **Option 1:** Vertical scaling of monolithic application servers and database instances.
* **Option 2:** Microservices with Command Query Responsibility Segregation (CQRS), Java 21 Virtual Threads, and Apache Kafka.
* **Option 3:** Modular Monolith with separate Read/Write database replicas.

## Decision Outcome
**Chosen Option:** **Option 2: Microservices with CQRS, Distributed Locking (Redlock), Java 21, and Apache Kafka**.  

### Implementation Strategy
* **Read Path (Query):** Catalog and Itinerary searches are served through autoscaling Go/Node.js services backed by Redis in-memory caches and Aurora Read Replicas.
* **Write Path (Command):** Booking transactions are processed by a dedicated Java 21 Spring Boot 3.3 service utilizing Virtual Threads. Redisson distributed locking gates inventory availability. Once reserved, an immutable `booking.reserved` event is published to Apache Kafka.
* **Asynchronous Settlement:** Payment capture services process webhooks independently and trigger confirmation events via Kafka consumer groups.

## Pros and Cons of the Options
* *Option 1 (Vertical Scaling):* High cloud cost; single point of failure; does not resolve lock contention on shared database rows.
* *Option 2 (Microservices with CQRS - Chosen):* Total fault isolation; read spikes cannot crash checkouts; zero overbooking via atomic distributed locking; horizontal scalability per service.
* *Option 3 (Modular Monolith):* Simpler than full microservices, but runtime memory leaks or crashes in media rendering still risk taking down transactional checkout flows.

## Rollback Strategy
Services are deployed on AWS EKS using blue-green deployments via ArgoCD. If schema or consumer regression occurs, automated rollback of the container image is executed within 30 seconds.
