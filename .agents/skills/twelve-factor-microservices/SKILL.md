---
name: twelve-factor-microservices
description: Authoritative 12-Factor App methodology standards mapped to Spring Boot 3.x, Spring Cloud, Kafka, and cloud-native microservices. Governs codebase discipline, externalized config, attached backing services, stateless concurrency, graceful disposability, and dev/prod parity.
---

# 12-Factor Microservice Architecture Specification

Authoritative implementation rules translating [The Twelve-Factor App](https://12factor.net/) methodology into production-grade Spring Boot 3.x and cloud-native microservices.

---

## 1. Factor I: Codebase (One Codebase, Many Deploys)
- **Rule**: Exactly one Git repository per microservice bounded context (or clean monorepo with independent build boundaries).
- **Enforcement**:
  - Never fork repositories for different deployment stages.
  - The same binary artifact is deployed to Local, Dev, Staging, and Production, varying only by runtime configuration.

## 2. Factor II: Dependencies (Explicitly Declare and Isolate)
- **Rule**: All third-party libraries and runtime plugins must be explicitly pinned and managed via Bill of Materials (BOM).
- **Implementation**:
  - `pom.xml` / `build.gradle.kts` imports `spring-boot-dependencies` and `spring-cloud-dependencies`.
  - Zero reliance on implicit system-wide shared libraries or host packages.

## 3. Factor III: Config (Store Config in the Environment)
- **Rule**: Strict separation of configuration from code.
- **Implementation**:
  - Store credentials, backing URLs, and feature flags strictly in environment variables.
  - Spring Boot `application.yml` maps env vars:
    ```yaml
    spring:
      datasource:
        url: ${DB_URL}
        username: ${DB_USERNAME}
        password: ${DB_PASSWORD}
    ```
  - Zero hardcoded passwords, tokens, or environment-specific URLs in Git.

## 4. Factor IV: Backing Services (Treat as Attached Resources)
- **Rule**: Databases, caches, message brokers, and mail servers are attached resources accessed via locator URLs.
- **Implementation**:
  - Swapping a local PostgreSQL database for an AWS RDS / Neon instance requires zero code changes—only a change to `${DB_URL}`.

## 5. Factor V: Build, Release, Run (Strict Stage Separation)
- **Rule**: Code transformation stages are strictly partitioned:
  1. **Build Stage**: Compiles code, runs tests, and produces an immutable OCI container image (Cloud Native Buildpacks: `./mvnw spring-boot:build-image`).
  2. **Release Stage**: Combines the build image with environment-specific config.
  3. **Run Stage**: Executes the release in target environment (Kubernetes / Tanzu / Docker).

## 6. Factor VI: Processes (Stateless and Share-Nothing)
- **Rule**: Microservices execute as stateless processes.
- **Implementation**:
  - Any persistent state must reside in an attached datastore (Postgres, Redis, Kafka).
  - Never rely on in-memory sticky sessions or local filesystem scratch files across requests.

## 7. Factor VII: Port Binding (Export Services via Port Binding)
- **Rule**: Services are self-contained and export their functionality by binding directly to an HTTP/TCP port.
- **Implementation**:
  - Embedded Tomcat/Netty listens directly on `SERVER_PORT` (default `8080`).
  - No external WebSphere, WebLogic, or Apache HTTPD container injection needed.

## 8. Factor VIII: Concurrency (Scale Out via the Process Model)
- **Rule**: Scale out horizontally by adding more process instances, supplemented by efficient internal concurrency.
- **Implementation**:
  - Horizontal scaling via Kubernetes Deployment replicas.
  - Vertical throughput enabled via Java 21 **Virtual Threads** (`spring.threads.virtual.enabled=true`).

## 9. Factor IX: Disposability (Fast Startup and Graceful Shutdown)
- **Rule**: Processes can be started or stopped at a moment's notice to facilitate elastic scaling and rapid deployment.
- **Implementation**:
  - Fast startup optimized with Spring AOT compilation.
  - Graceful termination enabled in `application.yml`:
    ```yaml
    server:
      shutdown: graceful
    spring:
      lifecycle:
        timeout-per-shutdown-phase: 30s
    ```
  - Finishes active in-flight requests and cleanly detaches Kafka consumers before process exit.

## 10. Factor X: Dev/Prod Parity (Keep Environments Similar)
- **Rule**: Keep development, staging, and production as similar as possible in code, backing services, and behavior.
- **Implementation**:
  - Avoid in-memory mocks (e.g. H2 database) for testing production behavior.
  - Use **Testcontainers** to spin up real PostgreSQL and Kafka instances during local testing.

## 11. Factor XI: Logs (Treat Logs as Event Streams)
- **Rule**: Apps never manage or route their own log files.
- **Implementation**:
  - Each running microservice writes an unbuffered stream of structured JSON log events directly to `stdout`.
  - The runtime environment (FluentBit, Promtail, Vector) captures stdout and ships to Loki, Elasticsearch, or Datadog.

## 12. Factor XII: Admin Processes (One-Off Management Tasks)
- **Rule**: Database migrations and one-off batch repairs run as distinct, ephemeral tasks alongside the regular application processes.
- **Implementation**:
  - Database schema evolution managed via **Liquibase** or **Flyway**.
  - Executed via Kubernetes init-containers, CI deployment steps, or Spring Boot command-line runners (`ApplicationRunner`).
