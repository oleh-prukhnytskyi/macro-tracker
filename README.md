# Macro Tracker

![Java](https://img.shields.io/badge/java-%23ED8B00.svg?style=for-the-badge&logo=openjdk&logoColor=white)
![Spring Boot](https://img.shields.io/badge/spring-%236DB33F.svg?style=for-the-badge&logo=spring&logoColor=white)
![Kubernetes](https://img.shields.io/badge/kubernetes-%23326ce5.svg?style=for-the-badge&logo=kubernetes&logoColor=white)
![Apache Kafka](https://img.shields.io/badge/Apache%20Kafka-000?style=for-the-badge&logo=apachekafka)
![PostgreSQL](https://img.shields.io/badge/postgres-%23316192.svg?style=for-the-badge&logo=postgresql&logoColor=white)
![Redis](https://img.shields.io/badge/redis-%23DD0031.svg?style=for-the-badge&logo=redis&logoColor=white)
![ElasticSearch](https://img.shields.io/badge/-ElasticSearch-005571?style=for-the-badge&logo=elasticsearch)
![Docker](https://img.shields.io/badge/docker-%230db7ed.svg?style=for-the-badge&logo=docker&logoColor=white)

***

[![License](https://img.shields.io/badge/license-Apache%202.0-blue?style=for-the-badge)](LICENSE)
[![Swagger](https://img.shields.io/badge/Swagger-API_Docs-85EA2D?style=for-the-badge&logo=swagger&logoColor=black)](https://macrotracker.uk/swagger-ui)
[![Docker Hub](https://img.shields.io/badge/Docker%20Hub-Image-blue?style=for-the-badge&logo=docker)](https://hub.docker.com/repositories/olehprukhnytskyi?search=macro-tracker)

**High-performance, distributed platform for nutrition tracking and calorie analysis.**

Unlike typical CRUD pet projects, Macro Tracker is engineered with **production-grade concerns** in mind: high availability, eventual consistency, distributed tracing, and fault tolerance. The system follows **12-Factor App** principles and is deployed on a bare-metal Kubernetes cluster.

## :rocket: Key Engineering Highlights

* **Transactional Outbox Pattern**: Solves the "Dual Write" problem to ensure eventual consistency between PostgreSQL and Kafka.
* **Performance Optimization**: Implements batch processing for heavy write operations and recursive deletion strategies to prevent database lock contention.
* **Resilience**: Rate limiting, retry policies, and idempotency mechanisms protect the system under high load.
* **Infrastructure as Code**: Fully automated CI/CD pipelines and Kubernetes manifests for reproducible deployments.
* **Elastic Scalability**: Implemented **Kubernetes HPA** to auto-scale pods based on real-time CPU/Memory utilization.
* **Traefik Ingress**: Manages external access and load balancing, decoupling routing logic from application code.
* **Bare-Metal Cluster**: Self-managed Kubernetes cluster running on **Alpine Linux** nodes, optimized for minimal resource footprint and hardened security.
* **Zero Trust Networking**: External access is secured via **Cloudflare Tunnels (Cloudflared)**. This eliminates the need for open firewall ports or static IPs, preventing DDoS attacks and hiding the origin server IP.

---

# :building_construction: Architecture & Infrastructure

![Architecture](/assets/architecture.svg)

### Core Components & Data Flow:
1.  **API Gateway (Spring Cloud Gateway)**: The entry point providing routing, **Redis-backed Rate Limiting**, and centralized JWT validation.
2.  **BFF Service (Backend for Frontend)**: Aggregates data from multiple downstream services (Intake, Goal) into optimized responses. Utilizes Reactive WebClient for non-blocking concurrent requests, reducing overall response time.
3.  **Event-Driven Core**: Asynchronous communication via **Apache Kafka** ensures service decoupling.
    * *Example*: When a user registers, an event is published. The **Notification Service** consumes it to send an email containing a verification code, without blocking the registration response.
4.  **Search Engine**: Food data is synced from MongoDB to **Elasticsearch** via **Monstache** (real-time oplog tailing) for high-performance fuzzy search.

---

# :brain: Engineering Decisions & Patterns

The architecture leverages a mix of **Enterprise Integration Patterns** and standard **GoF patterns** to ensure modularity and scalability.

### 1. Data Consistency & Reliability
* **Transactional Outbox (Polling Publisher)**: To guarantee message delivery without distributed transactions (2PC), changes are first written to an `outbox` table in the same transaction as business data. A background job (using **ShedLock**) asynchronously pushes these events to Kafka.
* **Idempotency Strategy**: Critical write operations are protected by a custom AOP aspect (`@Idempotent`). It uses Redis to store unique request keys, preventing duplicate processing during network retries or "double-click" scenarios.

### 2. High-Performance Data Processing
* **Smart Batch Deletion**: To prevent **Kafka Consumer rebalancing storms** and database lock escalation during massive data cleanup (e.g., GDPR user deletion):
    * The Intake Service processes deletions in small, controlled **batches (chunks)**.
    * If a user has extensive data, the service employs a **Recursive Event pattern**: it deletes a chunk, then republishes the deletion event to process the next chunk, keeping transactions short and the consumer alive.

### 3. Asynchronous Notification System
* **Decoupled Email Delivery**: The **Notification Service** operates independently of the User Service. This ensures that slow SMTP servers or network timeouts do not impact the latency of user-facing operations (like Registration or Password Reset). If the mail server is down, Kafka guarantees the message is persisted and retried.

### 4. Caching & Persistence Strategy
* **Polyglot Persistence**:
    * **PostgreSQL**: For relational data (Users, Intakes).
    * **MongoDB**: For unstructured catalog data (Food items).
    * **AWS S3**: For storing food images.
* **Redis Tuning**: Configured with `allkeys-lfu` (Least Frequently Used) eviction policy to ensure the most popular food items and active user sessions remain in cache under memory pressure.

### 5. Security
* **Asymmetric JWT Signing (RS256)**: Instead of using shared secrets (Symmetric encryption), the system implements public/private key cryptography using **RSA keys**.
    * The **User Service** acts as the Identity Provider (IdP) and holds the **Private Key** to sign tokens.
    * The **API Gateway** acts as an OAuth2 Resource Server and validates signatures using the **Public Key**.
* **JWKS Endpoint**: Public keys are exposed via the standard `/.well-known/jwks.json` endpoint. This allows zero-downtime key rotation and enables other services (Gateway) to dynamically fetch current verification keys without redeployment.
* **Social Login Integration**: Server-side validation of Google and Facebook tokens ensures that third-party identity claims are trustworthy before issuing an internal JWT.
* **Mitigated Confused Deputy Attacks**: Implemented strict OAuth2 Audience (`aud`) validation for Google/Facebook tokens. This ensures the backend accepts strictly those tokens generated for the official Macro Tracker application, preventing identity spoofing from malicious third-party apps.
* **Optimized Stateless Auth**: Instead of heavy session management, a custom **JWT Filter Chain** performs signature validation at the Gateway level (Edge Authentication), offloading downstream services.
* **Strategy Pattern for Social Login**: A flexible design that validates OAuth2 tokens (Google, Facebook) server-side, enforcing a strict trust boundary.
* **Secret Management**: Strict separation of config and code. All sensitive credentials (DB passwords, API keys, OAuth secrets) are managed via **Kubernetes Secrets** and injected into containers as environment variables, ensuring no sensitive data is exposed in the source code.

### 6. Advanced Traffic Control (Rate Limiting)
* **Hybrid Key Resolution Strategy**: Instead of simple IP-based limiting (which penalizes users behind shared NATs), the Gateway implements a custom logic:
    * **Authenticated Users**: Rate limits are bucketed by **User ID**. This ensures fair usage per account regardless of their IP.
    * **Anonymous Traffic**: Falls back to **IP-based** limiting to protect public endpoints (like `/auth/login`) from brute-force and DDoS attacks.
* **Tiered Throttling**: Limits are granularly tuned via `application.yml` based on endpoint sensitivity:
    * **Critical**: Low throughput for auth routes (Login/Register) to prevent credential stuffing.
    * **Standard**: High throughput for read-heavy operations (Food Search).

### 7. Quality Assurance
* **True Integration Testing**: Tests do not rely on mocks or embedded databases (H2). **Testcontainers** spin up real ephemeral instances of PostgreSQL, MongoDB, and Redis to guarantee production-like behavior.
* **RFC 7807 (Problem Details)**: All services return standardized JSON error responses, making client-side error handling predictable.
* **Shared Library**: Common DTOs, Utilities, and Exception Handling configurations are extracted into a separate Maven repository to strictly follow DRY.

---

# :bar_chart: Monitoring and Observability

* **OpenTelemetry Agents**: Attached to all pods for auto-instrumentation of traces and metrics.
* **Centralized & Structured Logging**: Services output logs in **JSON format** (via Logback/Logstash) for efficient parsing in the ELK Stack.
* **Health Checks**: **Spring Boot Actuator** exposes Liveness and Readiness probes, allowing K8s to automatically restart unhealthy pods or remove them from load balancing.
* **Distributed Tracing**: Full request visibility across microservices using Correlation IDs.

---

# :hammer_and_wrench: Technology Stack

| Category              | Technologies                                                                                                         |
|:----------------------|:---------------------------------------------------------------------------------------------------------------------|
| **Backend**           | Java 21, Spring Boot 3, Spring WebFlux (BFF), OpenFeign, MapStruct, Lombok, Actuator, Swagger, Maven, Logback, SLF4J |
| **Data & Storage**    | PostgreSQL, MongoDB, Redis, AWS S3, Liquibase                                                                        |
| **Messaging & Async** | Apache Kafka, Java Mail Sender, ShedLock (Distributed Locking)                                                       |
| **AI & Search**       | Google Gemini API, Elasticsearch, Monstache                                                                          |
| **DevOps & Infra**    | Kubernetes (K3s), Docker, GitHub Actions (CI/CD), Git Flow, HPA, Traefik, Cloudflare Tunnel, Alpine                  |
| **Observability**     | OpenTelemetry, ELK Stack (Elasticsearch, Logstash, Kibana), Uptime Monitor, Grafana                                  |
| **Testing**           | JUnit 5, Testcontainers, Mockito, MockMvc, Checkstyle                                                                |

---

# :package: Microservices Overview

| Service                  | Description                                                                 | Repository                                                                      |
|:-------------------------|:----------------------------------------------------------------------------|:--------------------------------------------------------------------------------|
| **API Gateway**          | Entry point, Rate Limiting (Token Bucket), Auth validation.                 | [Link](https://github.com/oleh-prukhnytskyi/macro-tracker-cloud-gateway)        |
| **BFF Service**          | Orchestrates parallel calls to downstream services; optimized for frontend. | [Link](https://github.com/oleh-prukhnytskyi/macro-tracker-bff-service)          |
| **User Service**         | User profile, Auth (JWT + Social), Goals management.                        | [Link](https://github.com/oleh-prukhnytskyi/macro-tracker-user-service)         |
| **Intake Service**       | High-throughput meal logging, Batch processing.                             | [Link](https://github.com/oleh-prukhnytskyi/macro-tracker-intake-service)       |
| **Food Service**         | Catalog management, AI enrichment (Gemini), Fuzzy Search.                   | [Link](https://github.com/oleh-prukhnytskyi/macro-tracker-food-service)         |
| **Notification Service** | Async email delivery (Email Verification, Password Reset) via Kafka events. | [Link](https://github.com/oleh-prukhnytskyi/macro-tracker-notification-service) |
| **Goal Service**         | Macro calculation engine based on user biometrics.                          | [Link](https://github.com/oleh-prukhnytskyi/macro-tracker-goal-service)         |
| **Common Lib**           | Shared DTOs, Exceptions (RFC 7807), AOP Aspects.                            | [Link](https://github.com/oleh-prukhnytskyi/macro-tracker-common)               |

| Resource       | Description                   | Link                                                                             |
|:---------------|:------------------------------|:---------------------------------------------------------------------------------|
| **Swagger UI** | Interactive API Documentation | [Open Docs](https://macrotracker.uk/swagger-ui)                                  |
| **Docker Hub** | Official Docker Images        | [View Images](https://hub.docker.com/repositories/olehprukhnytskyi?search=macro-tracker) |
> **Note:** For private endpoints, you need to authorize using the `Bearer <token>` scheme.

---

## :balance_scale: License

This project and all its associated microservices are distributed under the Apache License 2.0.

See the [LICENSE](LICENSE) file in each individual repository for details.