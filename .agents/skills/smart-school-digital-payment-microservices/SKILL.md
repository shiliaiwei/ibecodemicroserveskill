---
name: smart-school-digital-payment-microservices
description: Authoritative architecture, transaction integrity, and implementation specifications for scalable Digital Payment & Financial Microservices based on empirical research by Mallikarjun Bellundagi (2022). Enforces autonomous service decomposition (API Gateway, Auth, Payment Processing, Transaction Ledger, Notifications), hybrid REST and Kafka messaging, dual-currency (KHR/USD) transaction rollbacks, distributed database isolation with Neon RLS, and Kubernetes container orchestration. Trigger on: "digital payment microservices", "payment processing", "transaction management", "payment gateway architecture", "fee checkout", "transaction integrity", "saga rollback", "bakong khqr payment".
---

# Digital Payment & Financial Microservices Architecture
## Smart School Enterprise Platform (Payment Engine Standard)

### Executive Architectural Blueprint

Synthesizing empirical research by Mallikarjun Bellundagi (2022) (*"Design and Implementation of Scalable Microservices Architecture for Digital Payment Systems"*, IJEETR), this skill defines the authoritative architecture for high-throughput, fault-tolerant digital payment processing across the Smart School ecosystem.

Traditional monolithic payment modules suffer from database locking, poor fault isolation, and catastrophic downtime during deadline payment bursts. By decomposing the payment lifecycle into autonomous, loosely coupled services using **Spring Boot 3, Kafka event streaming, and Docker/Kubernetes orchestration**, the architecture achieves:
* **+40% Higher Throughput** (140+ TPS vs. 100 TPS in monoliths)
* **43% Latency Reduction** (200ms vs. 350ms average response time)
* **99.5% High Availability** with complete fault isolation (notification failures never disrupt payment execution).

```
┌────────────────────────────────────────────────────────────────────────────────────────────────────────┐
│                              DIGITAL PAYMENT PROCESSING PIPELINE                                      │
├────────────────────────────────────────────────────────────────────────────────────────────────────────┤
│  [ Parent App / Student Portal ]       [ Touchscreen Canteen POS ]       [ Admin Finance Cashier Desk ]│
└─────────────────────────────────┬────────────────────────────────────────┬─────────────────────────────┘
                                  │ (HTTPS / TLS 1.3 - REST API Call)      │
                                  ▼                                        ▼
┌────────────────────────────────────────────────────────────────────────────────────────────────────────┐
│                                       API GATEWAY (:8080)                                              │
│               - Route Matching: /api/v1/payments/**, /api/v1/checkout/**                               │
│               - Perimeter Security & Token Validation                                                 │
│               - Rate Limiting: 20 req/min/user to prevent duplicate double-charge bursts               │
└───────────────────────────────────────────────────┬────────────────────────────────────────────────────┘
                                                    │
                                                    ▼
┌────────────────────────────────────────────────────────────────────────────────────────────────────────┐
│                               1. AUTHENTICATION SERVICE (:8081)                                        │
│               - Validates JWT bearer token, scopes, tenant tenancy & payer identity                    │
└───────────────────────────────────────────────────┬────────────────────────────────────────────────────┘
                                                    │
                                                    ▼
┌────────────────────────────────────────────────────────────────────────────────────────────────────────┐
│                         2. PAYMENT PROCESSING SERVICE (:8084 - Subsystem)                              │
│  - Payment payload validation (Dual-currency USD / KHR conversion & fee schedule match)               │
│  - Dispatch to External Payment Rail: Bakong KHQR / Stripe / Wing / ABA PayWay / Visa                 │
│  - Idempotency Key validation (Redis) to prevent duplicate card or QR charges                          │
└───────────────────────┬────────────────────────────────────────────────────────┬───────────────────────┘
                        │                                                        │
         (Synchronous)  │                                          (Asynchronous)│ Kafka Event:
                        ▼                                                        ▼ payment.completed.v1
┌────────────────────────────────────────┐             ┌─────────────────────────────────────────────────┐
│ 3. TRANSACTION MANAGEMENT & LEDGER     │             │ 4. NOTIFICATION & RECEIPT SERVICE (:8083)       │
│ - Neon PostgreSQL RLS Isolated DB      │             │ - Push DLT SMS, Telegram, WhatsApp alerts       │
│ - Immutable Double-Entry Ledger Entry  │             │ - Generate PDF Tax Receipt / Khmer Tax Invoice  │
│ - Saga Orchestration & Rollback Logic  │             │ - Failures isolated; never blocks the payment   │
└────────────────────────────────────────┘             └─────────────────────────────────────────────────┘
```

---

## 1. Decomposed Core Payment Components

According to Bellundagi (2022), a digital payment platform must segregate responsibilities into five autonomous components:

| Component | Responsibility | Failure Impact & Isolation Strategy |
| :--- | :--- | :--- |
| **API Gateway** | Perimeter entry, request routing, rate limiting, and SSL termination. | Redundant multi-AZ instances behind AWS ALB. |
| **Authentication Service** | User validation, tenant isolation, and session token verification. | Stateless verification via cached public keys (JWKS). |
| **Payment Processing Service** | Core business logic, external gateway dispatch, and currency conversion. | Isolated runtime; uses circuit breakers on third-party bank APIs. |
| **Transaction Management** | Double-entry transaction logging, ACID compliance, and compensation rollbacks. | Dedicated relational database with Row-Level Security (RLS). |
| **Notification Service** | Async SMS, push notifications, and receipt generation. | Decoupled via Kafka queues. If down, transactions still succeed! |

---

## 2. Hybrid Communication: REST vs. Kafka Asynchronous Messaging

To optimize system responsiveness while guaranteeing transaction integrity:

1. **Synchronous REST APIs (Client $\rightarrow$ Gateway $\rightarrow$ Payment Processing):**
   * Used for real-time payment submission where the user awaits an immediate checkout intent (e.g. generating a dynamic Bakong KHQR or submitting a credit card token).
   * Strict timeouts: External bank calls must time out within $3.5\text{ s}$.

2. **Asynchronous Kafka Event Streaming (Payment Processing $\rightarrow$ Downstream Consumers):**
   * Once the external payment rail returns `SUCCESS`, the Payment Service immediately emits a canonical event:
     ```json
     {
       "eventId": "evt_98f12a80-1234-4567-89ab-cdef01234567",
       "eventType": "payment.completed.v1",
       "timestamp": "2026-09-14T19:05:00Z",
       "tenantId": "kh-pnh-001",
       "payload": {
         "transactionId": "txn_8849102",
         "invoiceId": "inv_2026_09_0012",
         "payerId": "usr_student_442",
         "amountUSD": 150.00,
         "amountKHR": 615000.00,
         "exchangeRate": 4100.00,
         "paymentMethod": "BAKONG_KHQR",
         "idempotencyKey": "idem_ab12cd34ef56"
       }
     }
     ```
   * **Consumers:**
     * `finance-ledger-service`: Updates general ledger and marks student invoice as `PAID`.
     * `notification-msg-service`: Dispatches instant SMS / Telegram payment confirmation.
     * `academic-core-service`: Unlocks semester examination admit cards or class registration.

---

## 3. Transaction Integrity & Distributed Consistency (Saga Pattern)

Because microservices utilize isolated databases per service, distributed transactions cannot use traditional two-phase locking (2PC) due to latency and database locks:

* **Idempotency Safeguard:** Every payment request requires a unique `X-Idempotency-Key` header generated by the client frontend. The key is cached in Redis with a 120-second TTL to eliminate accidental double-charges.
* **Saga Orchestrator with Compensating Transactions:**
  * If the external payment succeeds, but downstream invoice allocation fails due to a system glitch, the Transaction Manager triggers a compensating action:
    1. Log transaction as `PENDING_RECONCILIATION`.
    2. Dispatch an automated refund or credit note.
    3. Alert the finance administrator via audit telemetry.

---

## 4. Empirical Benchmarks (Bellundagi 2022 vs. Monolith)

| Performance Metric | Monolithic Payment Architecture | Microservices Architecture (Bellundagi 2022) | Business Impact |
| :--- | :--- | :--- | :--- |
| **Throughput (TPS)** | 100 TPS | **140 TPS (+40%)** | Handles morning deadline rush without queue collapse |
| **Response Time** | 350 ms | **200 ms (-43%)** | Faster checkout experience on mobile & POS |
| **Availability** | 95.0% | **99.5%** | Minimizes lost tuition or canteen fee collections |
| **Fault Isolation** | Low (whole app crashes on notification fail)| **High (isolated to specific container)** | Notification delay does NOT block receipt issuance |
| **Scalability** | Limited (must scale entire monolith) | **High (scale only payment pod replicas)** | Optimizes cloud server costs on AWS/Azure |

---

## 5. Implementation Standard Checklist

When deploying or refactoring payment modules, ensure adherence to the following:

* [ ] **Idempotency Key Enforced:** Client requests pass unique idempotency key; repeated calls return cached response.
* [ ] **Dedicated Microservice Database:** Payment records and transactions live in their own schema/database with Neon RLS tenant isolation.
* [ ] **Zero Blocking Notifications:** SMS and email receipts are strictly consumed off Kafka; never call SMS APIs synchronously inside the payment loop.
* [ ] **External API Timeouts:** Third-party gateway calls wrapped in circuit breakers with $< 3.5\text{ s}$ timeout and auto-retry policies.
* [ ] **Kubernetes HPA Configured:** Horizontal Pod Autoscaler set to scale payment pods based on CPU $> 70\%$ or request rate $> 80\text{ req/sec}$.
