---
name: microservices-patterns-chris-richardson-2e
description: Authoritative microservices architecture and design patterns distilled from Chris Richardson's "Microservices Patterns, Second Edition (2E)". Covers Fast Flow Delivery, 4 Flavors of Coupling, Monolith vs Microservices evolution, Hexagonal Architecture, Asynchronous Messaging, Ordered Competing Receivers, Transactional Outbox, and Idempotent Consumers.
---

# Microservices Patterns (2nd Edition) Master Specification

Distilled directly from **Chris Richardson's *Microservices Patterns (Second Edition)***.

---

## 1. The Core Purpose: Architecture for Fast Flow

* **The Success Triangle**: High performance requires **DevOps + Team Topologies + Fast Flow Architecture**.
* **Fast Flow Definition**: Rapid, reliable, sustainable continuous delivery of small changes from code commit into production with rapid feedback.
* **The 5 Fast Flow Architectural Requirements**:
  1. **Modifiability**: High cohesion and loose design-time coupling to make changes safely without touching other services.
  2. **Evolvability**: Ability to update frameworks (e.g. Spring Boot 2 -> 3) or change languages independently.
  3. **Testability**: Fast-running automated tests runnable locally without spinning up the entire fleet of services.
  4. **Deployability**: Independent releases without coordinated, synchronous lock-step deployments.
  5. **Observability**: Rapid detection of runtime failures via health checks, distributed traces, and log aggregation.

---

## 2. The 4 Flavors of Coupling (Critical Architecture Concepts)

Loose coupling is not one-dimensional. Systems must minimize four distinct types of coupling:

| Coupling Flavor | Problem & Manifestation | Mitigation Strategy |
| :--- | :--- | :--- |
| **1. Design-time Coupling** | Changing Service A requires changing Service B (e.g., shared code, shared schemas). | High cohesion, Domain-Driven Design (DDD), aggregate boundaries, and stable interface contracts. |
| **2. Build-time Coupling** | Building Service A requires compiling or waiting on Service B's build pipeline. | Interface segregation, published client JARs / API specs (AsyncAPI / OpenAPI), no multi-module project locks. |
| **3. Runtime Coupling** | Service A cannot respond until Service B responds synchronously; latency & failures cascade. | Self-contained services, Asynchronous Messaging (Kafka/RabbitMQ), and local data replication. |
| **4. Infrastructure Coupling**| Services share the same database instance, queue broker, or physical disk. | Database-per-service pattern, tenant namespace isolation. |

---

## 3. Hexagonal Architecture Style (Ports and Adapters)

Every microservice must be structured internally following the Hexagonal style:
* **Inbound Ports**: Define the API operations implemented by the core business logic (Use Cases / Application Services).
* **Inbound Adapters**: Controllers, gRPC handlers, or Kafka message consumers that invoke inbound ports.
* **Core Business Logic**: Entities, Value Objects, and Domain Services completely decoupled from frameworks and databases.
* **Outbound Ports**: Interfaces defined by the core logic to interact with external systems (Repositories, Notification Senders).
* **Outbound Adapters**: Database implementations (JPA / RLS), HTTP clients (WebClient), and Kafka message producers.

---

## 4. Interprocess Communication (IPC) Patterns

### A. Synchronous Remote Procedure Invocation (RPI)
* Used when real-time read responses are mandatory (REST / gRPC).
* **Mandatory Safeguards**:
  * Set explicit timeouts on all HTTP/gRPC requests.
  * Wrap in **Circuit Breakers** and **Rate Limiters** (Resilience4j).
  * Never chain synchronous calls deep across 3+ services.

### B. Asynchronous Messaging Patterns (Preferred for Fast Flow)
1. **Ordered Competing Receivers Pattern**:
   * Scale consumers horizontally while maintaining strict order for related events.
   * Publish messages with a deterministic partition key (e.g. `{branchId}#{studentId}`). Kafka partitions guarantee order while allowing competing consumers across partitions.
2. **Transactional Outbox Pattern**:
   * Solves the dual-write problem (writing to database + publishing to message broker).
   * Persist business entity and an `OUTBOX` event table within the same ACID database transaction.
   * A message relay (Debezium CDC or polling publisher) reads the `OUTBOX` table and reliably publishes to Kafka.
3. **Idempotent Consumer Pattern**:
   * Message brokers guarantee "at-least-once" delivery. Duplicate messages are inevitable.
   * Consumers record processed `messageId` in a deduplication table or apply idempotent state mutations (`UPDATE ... WHERE status = 'PENDING'`).

---

## 5. Evolution Strategy: FTGO Case Study & Monolith-First Rule

* **The Monolith-First Approach**: For brand new greenfield domains with high domain ambiguity, start with a well-structured modular monolith.
* **Avoid Fine-Grained Hell (V2 Trap)**: Creating dozens of tiny microservices prematurely leads to excessive runtime coupling, distributed transaction nightmares, and team gridlock.
* **Strangler Fig Application**: Gradually extract cohesive bounded contexts into microservices only when independent deployability and organizational scaling justify the distributed overhead.
