---
name: smart-school-api-gateway-architecture
description: Authoritative specification for the Enterprise API Gateway and Edge Routing Architecture based on empirical research (Neelan, 2025). Enforces Layer-7 North-South traffic ingress, Netty non-blocking filter pipelines, JWT perimeter authentication, Redis token bucket rate limiting, BFF (Backend-For-Frontend) facades, request composition, and Service Mesh coexistence. Trigger on: "api gateway", "edge gateway", "reverse proxy vs gateway", "rate limiting", "request composition", "bff pattern", "zuul", "spring cloud gateway", "kong", "north south traffic".
---

# Enterprise API Gateway & Edge Routing Architecture
## Smart School Enterprise Microservices Platform (Ingress Layer 7)

### Executive Architectural Blueprint

Synthesizing foundational research from Neelan (2025) (*"The Evolving Role of API Gateways in Scalable Microservices Architecture"*), this skill establishes the authoritative engineering standard for the **API Gateway and Edge Ingress Layer** across the Smart School microservices cluster.

The API Gateway acts as the single, highly-available, policy-driven reverse perimeter hub (`:8080` / `gateway-service`) positioned at the boundary between external clients (Web Portals, iOS/Android Mobile Apps, POS Terminals, Third-party Webhooks) and internal sovereign microservices (`:8081` through `:8087`).

```
┌────────────────────────────────────────────────────────────────────────────────────────────────────────┐
│                                       INGRESS TRAFFIC ARCHITECTURE                                     │
├────────────────────────────────────────────────────────────────────────────────────────────────────────┤
│  [ Web Portals ]     [ Mobile Clients (iOS/Android) ]     [ Canteen/Fee POS ]     [ Third-party API ] │
└─────────┬───────────────────────────┬───────────────────────────────┬──────────────────────┬───────────┘
          │ (HTTPS / TLS 1.3)         │ (HTTPS / TLS 1.3)             │                      │
          ▼                           ▼                               ▼                      ▼
┌────────────────────────────────────────────────────────────────────────────────────────────────────────┐
│                        AWS NETWORK LOAD BALANCER / MULTI-AZ ELB (SSL Termination)                      │
└───────────────────────────────────────────────────┬────────────────────────────────────────────────────┘
                                                    │
                                                    ▼
┌────────────────────────────────────────────────────────────────────────────────────────────────────────┐
│                         ENTERPRISE API GATEWAY CLUSTER (Spring Cloud Gateway / Netty)                  │
│                                           Port :8080 / Context: /api/v1/*                              │
│                                                                                                        │
│  ┌────────────────────────┐  ┌───────────────────────────┐  ┌───────────────────────────────────────┐  │
│  │ 1. Inbound Filters     │  │ 2. Routing & Mediation    │  │ 3. Outbound Filters                   │  │
│  │  - SSL / CORS Policy   │  │  - Dynamic Path Matching  │  │  - Response Compression (Gzip/Brotli) │  │
│  │  - JWT Claims Validate │  │  - Service Discovery (K8s)│  │  - Global Header Sanitization         │  │
│  │  - Redis Token Bucket  │  │  - Protocol Translation   │  │  - Telemetry & Latency Logging        │  │
│  │  - Request Sanitize    │  │  - Request Composition    │  │  - Distributed Tracing (OTel)         │  │
│  └────────────────────────┘  └───────────────────────────┘  └───────────────────────────────────────┘  │
└──────────────────────────┬──────────────────────┬───────────────────────┬──────────────────────────────┘
                           │                      │                       │
      ┌────────────────────┼──────────────────────┴───────────────────────┼────────────────────┐
      │ (Internal HTTP/REST / gRPC)                                       │                    │
      ▼                                           ▼                       ▼                    ▼
┌─────────────────────────┐  ┌──────────────────────────┐  ┌─────────────────────────┐  ┌─────────────┐
│ 1. auth-tenant-service  │  │ 2. academic-core-service  │  │ 3. attendance-msg-service│  │ 4-7 Services│
│        (:8081)          │  │         (:8082)           │  │         (:8083)         │  │ (:8084-8087)│
└─────────────────────────┘  └──────────────────────────┘  └─────────────────────────┘  └─────────────┘
```

---

## 1. Gateway vs. Reverse Proxy vs. Service Mesh (Clear Separation of Concerns)

According to Neelan (2025), modern architectures must strictly distinguish between these three tiers:

| Dimension | Simple Reverse Proxy (NGINX / HAProxy) | Enterprise API Gateway (Spring Cloud Gateway / Kong) | Service Mesh (Istio / Envoy Sidecars) |
| :--- | :--- | :--- | :--- |
| **Primary Scope** | Network & Transport Layer (L4/L7) | Application & API Layer (L7) | Intra-cluster Service Infrastructure |
| **Traffic Direction** | **North–South** (Generic web forwarding) | **North–South** (Client-to-Service Ingress) | **East–West** (Internal Service-to-Service) |
| **API Awareness** | Low (dumb URL and path forwarding) | High (JWT scopes, API versions, payloads) | High at network level, agnostic to business payload |
| **Security Mechanism**| Basic SSL termination, basic IP blocking | OAuth2/JWT validation, API Keys, Tenant isolation | Mutual TLS (mTLS), SPIFFE/SPIRE service identities |
| **Traffic Control** | Connection limits, round-robin | Token Bucket / Leaky Bucket rate limits, quotas | Retries, circuit breakers, traffic shifting, canary |
| **Observability** | Access logs, connection counters | API consumer analytics, quota metrics, latency | Fine-grained distributed tracing (Jaeger, OTel) |

> **Architectural Rule:** The Smart School platform deploys the API Gateway exclusively for **North–South** client ingress. Downstream microservices do not call each other through the API Gateway; internal communication must use asynchronous Kafka events or direct East-West service discovery.

---

## 2. The 5 Core Responsibilities Implemented

### 2.1. Request Routing & Dynamic Service Discovery
* The Gateway maintains routes for all 7 sovereign microservices:
  * `/api/v1/auth/**`, `/api/v1/tenants/**` $\to$ `auth-tenant-service:8081`
  * `/api/v1/academic/**`, `/api/v1/students/**` $\to$ `academic-core-service:8082`
  * `/api/v1/attend/**`, `/api/v1/notify/**` $\to$ `attendance-msg-service:8083`
  * `/api/v1/finance/**`, `/api/v1/fees/**` $\to$ `finance-ledger-service:8084`
  * `/api/v1/exams/**`, `/api/v1/marks/**` $\to$ `assessment-exam-service:8085`
  * `/api/v1/staff/**`, `/api/v1/payroll/**` $\to$ `workforce-hr-service:8086`
  * `/api/v1/transit/**`, `/api/v1/library/**`, `/api/v1/visitors/**` $\to$ `operations-hub-service:8087`
* **Path Rewriting:** Strips internal versioning prefixes or prefixes tenant headers before forwarding downstream.

### 2.2. Perimeter Authentication & Authorization
* Validates incoming `Authorization: Bearer <JWT>` tokens using public keys cached in-memory (rotated via JWKS).
* Decodes claims: `tenant_id`, `user_id`, `role`, and `permissions`.
* Enforces role-based route guards at the edge. If unauthenticated or forbidden, rejects immediately with `401 Unauthorized` or `403 Forbidden` without consuming internal service CPU cycles.
* Passes verified claims to downstream services via internal headers:
  * `X-Tenant-ID: <tenant-uuid>`
  * `X-User-ID: <user-uuid>`
  * `X-User-Role: <SUPER_ADMIN|ADMIN|TEACHER|STUDENT|PARENT|ACCOUNTANT>`

### 2.3. Distributed Rate Limiting & Throttling
* Prevents DDoS, brute-force login attempts, and resource exhaustion during peak bursts (e.g. 08:00 AM attendance scan or fee deadline).
* **Algorithm:** Redis-backed **Token Bucket** algorithm (`KeyResolver` based on Client IP and Tenant ID).
* **Tiers:**
  * Public Auth Endpoints (`/api/v1/auth/login`): 10 requests / minute / IP.
  * Standard Student/Parent APIs: 120 requests / minute / user.
  * Administrative Endpoints: 300 requests / minute / user.
  * Internal Webhooks / POS: 1,000 requests / minute / API key.

### 2.4. Request Composition & Response Aggregation
* Solves the "Chatty Client" problem. For high-frequency aggregated screens (e.g., Student Dashboard Home), the Gateway invokes multiple services in parallel:
  * Call 1: `academic-core-service` (Student Profile & Current Classes)
  * Call 2: `attendance-msg-service` (Today's Attendance Status)
  * Call 3: `finance-ledger-service` (Pending Invoices & Due Fees)
  * Call 4: `assessment-exam-service` (Upcoming Exam Schedule)
* Aggregates the 4 JSON payloads non-blockingly and returns a single, unified `StudentDashboardSummaryDTO` in `< 120ms`.

### 2.5. Protocol Translation & Mediation
* Accepts standard REST/JSON from external Web and Mobile frontends.
* Transparently translates calls into internal high-performance protocols (e.g., gRPC for high-throughput biometric logs, or legacy SOAP for banking payment integrations).

---

## 3. Backend for Frontend (BFF) Pattern

To avoid bloating a single API contract with conflicting demands across client platforms, the Gateway adopts the **Backend for Frontend (BFF)** pattern:

```
                          ┌───────────────────────────┐
                          │    External API Gateway   │
                          │   (:8080 - Edge Ingress)  │
                          └─────────────┬─────────────┘
                                        │
           ┌────────────────────────────┼────────────────────────────┐
           ▼                            ▼                            ▼
┌───────────────────────┐    ┌───────────────────────┐    ┌───────────────────────┐
│       Web BFF         │    │      Mobile BFF       │    │     POS & IoT BFF     │
├───────────────────────┤    ├───────────────────────┤    ├───────────────────────┤
│ - Rich desktop tables │    │ - Compact payloads    │    │ - Ultra-low latency   │
│ - Full relational DTOs│    │ - Brotli compression  │    │ - Offline queue sync  │
│ - Complex filtering   │    │ - Stripped null keys  │    │ - Hardware token auth │
└───────────────────────┘    └───────────────────────┘    └───────────────────────┘
```

---

## 4. Operational Resilience & Eliminating the Single Point of Failure (SPOF)

As highlighted by Neelan (2025), a centralized gateway risks becoming a bottleneck or single point of failure if poorly architected. The following resilience directives are mandatory:

1. **Multi-AZ Stateless Deployment:**
   * Gateway instances must be strictly **stateless**. No session states are stored in JVM memory; all session or rate limit counters reside in AWS ElastiCache / Redis.
   * Run a minimum of 3 gateway instances distributed across distinct Availability Zones (AZ-a, AZ-b, AZ-c) behind an AWS Application Load Balancer.
2. **Circuit Breaking & Fallbacks:**
   * Integrate Spring Cloud Circuit Breaker / Resilience4j.
   * If a downstream microservice (e.g. `attendance-msg-service`) times out (default: 2,500ms), the Gateway trips the circuit and returns an immediate degraded fallback response without hanging client threads.
3. **Reactive Non-Blocking Engine:**
   * Gateway must be built on **Spring Cloud Gateway (Project Reactor + Netty)**, avoiding one-thread-per-connection thread exhaustion.
   * Concurrency is handled asynchronously via Netty event loops, achieving 50,000+ concurrent requests on minimal CPU footprint.

---

## 5. Standard Implementation Template (Spring Cloud Gateway Configuration)

```yaml
spring:
  cloud:
    gateway:
      routes:
        # Route 1: Auth & Tenant Service
        - id: auth-tenant-route
          uri: lb://auth-tenant-service
          predicates:
            - Path=/api/v1/auth/**, /api/v1/tenants/**
          filters:
            - name: RequestRateLimiter
              args:
                redis-rate-limiter.replenishRate: 20
                redis-rate-limiter.burstCapacity: 40
                key-resolver: "#{@ipKeyResolver}"

        # Route 2: Academic Core Service
        - id: academic-core-route
          uri: lb://academic-core-service
          predicates:
            - Path=/api/v1/academic/**, /api/v1/students/**
          filters:
            - name: JwtAuthenticationFilter
            - name: CircuitBreaker
              args:
                name: academicCircuitBreaker
                fallbackUri: forward:/fallback/academic-unavailable

        # Route 3: Finance & Ledger Service
        - id: finance-ledger-route
          uri: lb://finance-ledger-service
          predicates:
            - Path=/api/v1/finance/**, /api/v1/fees/**
          filters:
            - name: JwtAuthenticationFilter
            - name: StripPrefix=0
```

---

## 6. Verification & Quality Checklist

Before approving or deploying API Gateway changes, verify against the following checklist:

* [ ] **Statelessness Verified:** Gateway does not store in-memory HTTP sessions.
* [ ] **Perimeter Auth Active:** JWTs are verified and invalid tokens receive `401 Unauthorized` without reaching internal microservices.
* [ ] **Headers Propagated:** `X-Tenant-ID`, `X-User-ID`, and `X-User-Role` are passed to downstream microservices securely.
* [ ] **Rate Limiting Working:** Exceeding burst capacity triggers `429 Too Many Requests` with `Retry-After` header.
* [ ] **Timeouts & Circuit Breakers Set:** Downstream calls time out after 2.5s and activate fallback handlers.
* [ ] **Telemetry Ingested:** Distributed trace IDs (`traceparent` / OpenTelemetry) pass seamlessly from ingress to backend databases.
