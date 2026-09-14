---
name: spring-boot-microservice-resilience
description: Authoritative standards for building resilient Spring Boot 3.x microservices with Spring Cloud (Resilience4j Circuit Breakers, RateLimiter, Retry, Bulkhead), Spring Cloud Stream Kafka messaging, Virtual Threads (Loom), and Micrometer Tracing. Applies to enterprise microservice engineering and the Smart School pipeline.
---

# Spring Boot 3.x Microservices & Spring Cloud Resilience

Authoritative architecture guide based on [Spring Microservices](https://spring.io/microservices), Spring Boot 3.x, and Spring Cloud 2023+ (Resilience4j).

---

## 1. Core Architectural Pillars (Spring.io Reference)

1. **Spring Boot Microservices**: Standalone, modular, independently deployable services with minimal configuration and embedded runtimes.
2. **Spring Cloud Circuit Breaker (Resilience4j)**: Eliminates cascading downstream failures using Circuit Breakers, Retries, Rate Limiters, Timeouts, and Bulkheads.
3. **Spring Cloud Stream & Kafka**: Decoupled, asynchronous event-driven streaming with dead-letter queue (DLQ) support and canonical event payloads.
4. **Production Observability (Micrometer & Actuator)**: Distributed tracing (Micrometer Tracing + Brave / OpenTelemetry) and Prometheus metrics.
5. **Spring Boot 3 Virtual Threads**: Non-blocking concurrent I/O throughput with `spring.threads.virtual.enabled=true`.

---

## 2. Spring Cloud Circuit Breaker Implementation (Resilience4j)

### Maven Dependencies (`pom.xml`)
```xml
<dependencies>
    <!-- Spring Boot Actuator for health & metrics -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-actuator</artifactId>
    </dependency>

    <!-- Spring Cloud Circuit Breaker with Resilience4j -->
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-circuitbreaker-resilience4j</artifactId>
    </dependency>

    <!-- AOP support for @CircuitBreaker, @Retry, @RateLimiter -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-aop</artifactId>
    </dependency>
</dependencies>
```

### Configuration (`application.yml`)
```yaml
spring:
  threads:
    virtual:
      enabled: true

resilience4j:
  circuitbreaker:
    instances:
      feePaymentService:
        slidingWindowType: COUNT_BASED
        slidingWindowSize: 20
        minimumNumberOfCalls: 10
        failureRateThreshold: 50.0
        slowCallRateThreshold: 75.0
        slowCallDurationThreshold: 2s
        waitDurationInOpenState: 10s
        permittedNumberOfCallsInHalfOpenState: 5
        automaticTransitionFromOpenToHalfOpenEnabled: true
  retry:
    instances:
      feePaymentService:
        maxAttempts: 3
        waitDuration: 500ms
        enableExponentialBackoff: true
        exponentialBackoffMultiplier: 2
  ratelimiter:
    instances:
      feePaymentService:
        limitForPeriod: 100
        limitRefreshPeriod: 1s
        timeoutDuration: 200ms
```

### Resilient Service Pattern
```java
package com.smartschool.enterprise.service;

import io.github.resilience4j.circuitbreaker.annotation.CircuitBreaker;
import io.github.resilience4j.retry.annotation.Retry;
import io.github.resilience4j.ratelimiter.annotation.RateLimiter;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.stereotype.Service;

@Service
public class PaymentProcessingService {

    private static final Logger log = LoggerFactory.getLogger(PaymentProcessingService.class);
    private static final String SERVICE_NAME = "feePaymentService";

    @CircuitBreaker(name = SERVICE_NAME, fallbackMethod = "processPaymentFallback")
    @Retry(name = SERVICE_NAME)
    @RateLimiter(name = SERVICE_NAME)
    public PaymentResponse processTransaction(PaymentRequest request) {
        // External payment aggregator / downstream microservice invocation
        return paymentClient.executePayment(request);
    }

    // Fallback executed when circuit is OPEN or retries exhausted
    public PaymentResponse processPaymentFallback(PaymentRequest request, Throwable ex) {
        log.warn("Downstream payment failure: {}. Activating circuit fallback for student: {}", 
                 ex.getMessage(), request.studentId());
        return PaymentResponse.bufferedAccepted(
            request.transactionId(), 
            "Payment request queued for asynchronous retry."
        );
    }
}
```

---

## 3. Spring Cloud Stream & Event-Driven Resilience

### Configuration (`application.yml`)
```yaml
spring:
  cloud:
    stream:
      bindings:
        paymentEvents-out-0:
          destination: enterprise.school.payments
        paymentEvents-in-0:
          destination: enterprise.school.payments
          group: billing-processor-group
          consumer:
            max-attempts: 3
            back-off-initial-interval: 1000
            back-off-multiplier: 2.0
      kafka:
        bindings:
          paymentEvents-in-0:
            consumer:
              enable-dlq: true
              dlq-name: enterprise.school.payments.dlq
```

---

## 4. Integration with Smart School Enterprise Directives

* **Tenant & Branch Context**: Propagate `X-Branch-ID` and PostgreSQL RLS session variables across Circuit Breakers.
* **Sync-to-Async Bridge**: Use Circuit Breakers to detect gateway degradation and immediately return `HTTP 202 Accepted` to buffer events into Kafka.
* **Health & Metrics Monitoring**: Expose `/actuator/health` and `/actuator/circuitbreakerevents` to Prometheus and Kubernetes readiness probes.

---

## 5. Event-Driven Streaming: Spring Cloud Stream Microservices

Spring Cloud Stream provides functional programming models (`java.util.function.Function`, `Consumer`, `Supplier`) to decouple microservices from underlying messaging middleware (Kafka, RabbitMQ).

### Functional Producer / Consumer Code Pattern
```java
package com.smartschool.enterprise.stream;

import com.smartschool.enterprise.dto.EventEnvelope;
import com.smartschool.enterprise.dto.AttendanceEvent;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import java.util.function.Consumer;
import java.util.function.Function;

@Configuration
public class AttendanceStreamConfig {

    private static final Logger log = LoggerFactory.getLogger(AttendanceStreamConfig.class);

    // Stream Processor: Receives attendance, processes RLS & audits, yields enrichment
    @Bean
    public Function<EventEnvelope<AttendanceEvent>, EventEnvelope<AttendanceEvent>> processAttendance() {
        return event -> {
            log.info("Processing stream event for branch: {}", event.branchId());
            return event.withStatus("PROCESSED");
        };
    }

    // Stream Consumer: Consumes canonical events directly
    @Bean
    public Consumer<EventEnvelope<AttendanceEvent>> logAttendanceConsumer() {
        return event -> log.info("Audited event: id={}, student={}", event.eventId(), event.payload().studentId());
    };
}
```

---

## 6. What is VMware Tanzu Platform?

**VMware Tanzu Platform** (Broadcom) is an enterprise-grade **Private Cloud Platform as a Service (PaaS)** optimized for Spring Boot and modern agentic applications:
1. **Spring Enterprise Runtime**: Automated CVE security patches, Ahead-of-Time (AOT) compilation tuning, and native image optimizations.
2. **Dual Runtimes**: Deploys workloads across both **Cloud Foundry** (rapid developer PaaS experience) and **Kubernetes** (Tanzu Kubernetes Grid).
3. **AI & Agentic Gateway**: Built-in governance, security scanning, and rate-limiting for LLMs/AI models and private enterprise data.
4. **Platform Data Services**: Native, compliant managed data layers (PostgreSQL, MySQL, RabbitMQ) supporting event-driven architectures.

---

## 7. The 12-Factor App Discipline for Spring Boot 3.x Microservices

"Stealing like an artist" from [12factor.net](https://12factor.net/) to engineer production-ready, cloud-native enterprise microservices:

| Factor | 12-Factor Principle | Spring Boot 3.x & Spring Cloud Implementation |
| :--- | :--- | :--- |
| **I. Codebase** | One codebase tracked in VCS, many deploys | Single Git repo per microservice context or disciplined monorepo with semantic versioning. |
| **II. Dependencies** | Explicitly declare and isolate dependencies | Maven `pom.xml` / Gradle `build.gradle.kts` with fixed BOM (`spring-boot-dependencies`, `spring-cloud-dependencies`). |
| **III. Config** | Store config in the environment | Externalized configuration via `application.yml` referencing env vars (`${DB_URL}`, `${KAFKA_BOOTSTRAP_SERVERS}`). Zero hardcoded credentials. |
| **IV. Backing Services** | Treat backing services as attached resources | Databases (Neon Postgres), caches (Redis), and brokers (Kafka) accessed via connection URLs without code coupling. |
| **V. Build, Release, Run** | Strictly separate build and run stages | CI/CD pipeline builds immutable OCI container images with Spring Boot Buildpacks (`./mvnw spring-boot:build-image`). |
| **VI. Processes** | Execute as stateless processes | Zero local memory session sharing; state persisted to PostgreSQL or Redis. |
| **VII. Port Binding** | Export services via port binding | Embedded Tomcat listening directly on `SERVER_PORT` (default `8080`), bound to container ingress. |
| **VIII. Concurrency** | Scale out via the process model | Horizontal scaling of container replicas, paired with Java 21 **Virtual Threads** (`spring.threads.virtual.enabled=true`) for vertical concurrency. |
| **IX. Disposability** | Fast startup and graceful shutdown | Fast container boot with Spring AOT; `server.shutdown=graceful` to finish in-flight requests before SIGTERM termination. |
| **X. Dev/Prod Parity** | Keep dev, staging, prod as similar as possible | Use Testcontainers for local integration tests (Postgres, Kafka) to mirror production backing services exactly. |
| **XI. Logs** | Treat logs as event streams | Output unbuffered JSON/stdout logs via Logback/SLF4J, collected by FluentBit/Promtail to Loki or OpenTelemetry. |
| **XII. Admin Processes** | Run admin/management tasks as one-off processes | Database schema migrations via Liquibase/Flyway executed as Kubernetes init-containers or CI deployment hooks. |

---

## 8. Google Cloud Microservices Architecture Master Specification

Based on [Google Cloud Architecture Center: What is Microservices Architecture?](https://cloud.google.com/learn/what-is-microservices-architecture):

### 1. Microservices Architecture Defined
* An architectural style where a complex application is engineered as a collection of small, independently deployable, modular services.
* Each microservice runs its own unique process, manages its own isolated datastore, and communicates via lightweight network protocols (REST APIs or Event Streams).

### 2. Monolithic vs. Microservices Architecture
| Dimension | Monolithic Architecture | Microservices Architecture |
| :--- | :--- | :--- |
| **Coupling** | Tightly coupled single deployable unit. | Loosely coupled bounded contexts. |
| **Scaling** | Scaled as a monolith (resource intensive). | Scaled granularly per service workload. |
| **Failure Blast Radius** | Single bug/leak crashes whole application. | Isolated to failing service; graceful fallbacks. |
| **Deployment Agility** | Slow, coordinated multi-team release cycles. | Independent CI/CD deployments on demand. |

### 3. Industry Real-World Examples
* **E-commerce**: Independent services for product catalog, shopping cart, payment checkout, inventory management, and shipping tracking.
* **Streaming Media (Netflix, YouTube)**: Separate microservices for video encoding, recommendation engine, subscription auth, and global CDN delivery.
* **Financial Services & EdTech**: Decoupled modules for ledger accounting, tuition payment gateways, student enrollment, and biometric gate attendance.

### 4. What Microservices Architecture Is Used For
* Accelerating developer velocity and feature delivery.
* Legacy modernization (decomposing monoliths via the Strangler Fig pattern).
* Multi-cloud and hybrid deployments.

### 5. Core Microservices Design Patterns
1. **Observability (Metrics, Logs, Traces)**:
   * Tracking single correlation IDs across distributed service hops.
   * Telemetry via OpenTelemetry / Micrometer Tracing.
   * AI-assisted root-cause diagnosis and anomaly detection (e.g. Gemini Cloud Assist).
2. **Idempotency (Ensuring Reliability Under Retries)**:
   * Network transient failures cause duplicate client retries.
   * Services verify `idempotencyKey` / `trackingId` to guarantee identical business outcomes without duplicate billing or side-effects.
3. **Event-Driven Architecture (EDA)**:
   * Asynchronous publish/subscribe message brokers (Kafka, Google Pub/Sub).
   * Producers emit immutable state change events; consumers react independently without blocking the caller.

### 6. Cloud Native Infrastructure & Managed Services
* **Container Orchestration**: Google Kubernetes Engine (GKE) / Cloud Run.
* **API Gateway & Traffic Management**: Apigee / Cloud Load Balancing.
* **Distributed Messaging**: Cloud Pub/Sub / Apache Kafka.
* **Observability**: Google Cloud Operations Suite (Cloud Logging, Cloud Monitoring, Cloud Trace).
