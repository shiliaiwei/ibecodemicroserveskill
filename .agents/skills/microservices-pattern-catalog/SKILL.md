---
name: microservices-pattern-catalog
description: Authoritative microservices engineering catalog synthesized from microservices.io (Chris Richardson), Google Cloud SLO Monitoring & Microservices Architecture, and Google Cloud Microservices Demo (Online Boutique). Governs decomposition, transactional outbox, CQRS, sagas, service meshes, and SLO-driven observability.
---

# Enterprise Microservices Pattern Catalog & Google Cloud SLO Architecture

Synthesized from:
1. **[microservices.io](https://microservices.io/)** (Chris Richardson)
2. **[Google Cloud SLO Monitoring for Microservices](https://docs.cloud.google.com/stackdriver/docs/solutions/slo-monitoring/microservices)**
3. **[Google Cloud Platform Microservices Demo](https://github.com/googlecloudplatform/microservices-demo)** (Online Boutique)

---

## 1. Does a School Enterprise Microservices Project Match Industry Examples?

**YES, 100% Match.** A School Management Platform (Smart School) shares identical architectural domains with industry benchmarks:

| Industry Domain (E-Commerce / Streaming) | Smart School Enterprise Microservice Equivalent | Shared Architectural Challenge |
| :--- | :--- | :--- |
| **Product Catalog Service** | **Curriculum & Academic Catalog Service** | High read volume, cached syllabus/course taxonomy. |
| **Cart & Checkout Service** | **Fee Collection & Payment Checkout Service** | Distributed transactions, multi-channel payment gateway. |
| **Payment Service** | **Digital Payment & ABA PayWay Microservice** | Strict idempotency keys, bank webhook resilience. |
| **Recommendation Service** | **Student Performance & Risk Analytics Service** | Async batch scoring, predictive student alert engines. |
| **Email / Notification Service** | **Localized SMS & Multi-Channel Communicator** | High-velocity SMS/Telegram dispatch, Kafka buffered. |
| **Streaming / Realtime Gateway** | **Biometric Gate & QR Code Attendance Service** | Peak morning burst throughput (Sync-to-Async Bridge). |

---

## 2. The Comprehensive microservices.io Pattern Catalog

### A. Decomposition Patterns
1. **Decompose by Business Capability**: Organize services around business competencies (e.g. Admission, Billing, Academics, Library).
2. **Decompose by Subdomain (DDD)**: Decompose along Domain-Driven Design bounded contexts (Core, Supporting, Generic).
3. **Self-Contained Service**: A service handles requests without synchronously waiting for other services to respond.

### B. Data Management & Consistency Patterns
1. **Database per Service**: Each microservice must own and isolate its own database schema; other services must access data only via APIs or events.
2. **Transactional Outbox**: Save domain events to a local outbox table within the database transaction, published to Kafka by a message relay (polling or CDC Debezium).
3. **Saga Pattern (Choreography / Orchestration)**: Maintain data consistency across microservices using a sequence of local transactions with compensating actions on failure.
4. **CQRS (Command Query Responsibility Segregation)**: Segregate write commands (PostgreSQL) from complex multi-join read views (Elasticsearch/Read Models).
5. **API Composition**: An API Gateway or aggregator queries multiple microservices and aggregates the result for the client.
6. **Domain Event**: Emitted when a domain aggregate changes state (`StudentEnrolledEvent`, `FeePaidEvent`).

### C. Communication & Reliability Patterns
1. **API Gateway & Backend for Frontend (BFF)**: Single ingress entrypoint providing routing, rate-limiting, SSL termination, and client-tailored payloads (Web vs. Mobile).
2. **Idempotent Consumer**: Guarantee that duplicate messages received from event brokers produce identical results without duplicate side effects.
3. **Circuit Breaker & Retry**: Wrap synchronous HTTP/gRPC calls to trip open when downstream latency or error rates spike.
4. **Service Mesh & Sidecar**: Delegate mTLS, traffic splitting, and retries to Envoy/Istio sidecars without polluting application business logic.

---

## 3. Google Cloud SLO Monitoring & Site Reliability Engineering (SRE)

From [Google Cloud SLO Monitoring Guide](https://docs.cloud.google.com/stackdriver/docs/solutions/slo-monitoring/microservices):

### Core Definitions
* **SLI (Service Level Indicator)**: A quantifiable metric tracking service health (e.g. `% of HTTP requests returning non-5xx in < 300ms`).
* **SLO (Service Level Objective)**: Target reliability goal set by business & platform teams (e.g. `99.9% success rate over rolling 30-day window`).
* **Error Budget**: Allowed unreliability (`100% - SLO%`). When error budget is exhausted, releases freeze to focus on stability.

### Google Cloud Auto-Discovered Microservices Architecture
* Cloud Monitoring continuously analyzes metadata streams across GKE namespaces, Cloud Run services, and GKE workloads.
* Automatically synthesizes **Microservice Dashboards** linking:
  1. Service Metadata & Deployment Topology.
  2. Request Latency Distribution (P50, P95, P99).
  3. Real-time SLO Status & Error Budget Burn Rate.
  4. Distributed Spans & Correlated Error Logs.

---

## 4. Google Cloud Platform Microservices Demo (Online Boutique Architecture)

The reference architecture demonstrates how modern 12-tier cloud-first microservices operate:

```
                  ┌──────────────────────┐
                  │   Frontend (Web/UI)  │
                  └──────────┬───────────┘
                             │ (gRPC)
         ┌───────────────────┼────────────────────┐
         ▼                   ▼                    ▼
┌─────────────────┐ ┌─────────────────┐ ┌───────────────────┐
│ Cart Service    │ │ Product Catalog │ │ Currency Service  │
│ (Redis Cache)   │ │ (Read-Heavy)    │ │ (In-Memory Rates) │
└────────┬────────┘ └─────────────────┘ └───────────────────┘
         │
         ▼ (Checkout Flow)
┌─────────────────┐
│ Checkout Service│───────► Payment Service (Bank Auth)
└────────┬────────┘───────► Shipping Service (Tracking)
         │                ► Email Service (Confirmation)
         ▼
┌─────────────────┐
│ Recommendation  │
│ Service (ML)    │
└─────────────────┘
```

### Applied Rules for Smart School Enterprise
1. **gRPC / Internal Network Isolation**: Core inter-service queries communicate via low-latency binary gRPC or Kafka topics.
2. **Dedicated Caching**: High-velocity transient data (parent shopping cart, active gate scan tickets) resides in Redis.
3. **Multi-Tenant RLS Guarantee**: Every internal gRPC metadata header and Kafka message header must carry `tenant_id` and `branch_id`.
