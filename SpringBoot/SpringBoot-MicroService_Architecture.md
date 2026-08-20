# Spring Boot Microservice Application Architecture

A modern, production-grade Spring Boot microservices architecture designed for resilience, scalability, and security. This repository serves as a reference guide for building and deploying distributed cloud-native applications on Kubernetes (AWS EKS).

---

## 🏛 Architecture Diagram

Below is the simple visual flow showing client request routing, microservice isolation, dynamic secret injection, and shared cross-cutting concerns:


---

## 🔄 Request Flow & Layer Breakdown

### 1. Clients & APIs (Edge Services)
* **Clients:** Web browsers, mobile apps, and third-party external services initiate HTTP/REST requests.
* **API Gateway (Spring Cloud Gateway):** Acts as the single entry point to the system. Handles edge routing, request validation, authentication, and rate limiting to protect downstream services from traffic spikes.

### 2. Core Service Deployment (Kubernetes / AWS EKS)
* **Containerized Spring Boot Microservices:** Each domain service (e.g., Order Service, Product Service, User Service) is packaged into a container image and deployed as Kubernetes Pods across worker nodes.
* **Database-per-Service Pattern:** To ensure loose coupling and independent scalability, each microservice manages its own dedicated, isolated database.
* **Zero-Trust Credential Injection:** Credentials (database URLs, passwords, API keys) are **never hardcoded**. Instead, pods use **IAM Roles for Service Accounts (IRSA)** or **EKS Pod Identity** combined with AWS Secrets Manager to inject runtime configurations securely via environment variables.

### 3. Service Support & Control Plane
* **Service Registry (Consul / Eureka):** Enables automatic service discovery so microservices can locate and communicate with each other dynamically without hardcoded IP addresses.
* **Centralized Configuration:** Externalizes environment-specific properties across all services using centralized version-controlled repositories (e.g., Spring Cloud Config, Git, AWS SSM).

### 4. Shared Cross-Cutting Concerns
* **Resilience4j Integration:** Employs **Circuit Breakers** to fail fast during downstream outages, preventing cascading failures and providing graceful fallback responses.
* **Event-Driven Messaging (Apache Kafka):** Handles asynchronous inter-service communication and event streaming (e.g., Order Service publishing an `OrderCreated` event consumed by User/Notification services).
* **Distributed Observability:** Implements centralized logging, metrics collection, and distributed tracing (e.g., OpenTelemetry, Zipkin, Prometheus) to monitor requests end-to-end across service boundaries.

---

## 🚀 Key Benefits

| Benefit | Description |
| :--- | :--- |
| **Independent Deployment** | Services can be updated, tested, and deployed individually without affecting the whole system. |
| **Fault Tolerance** | Self-healing resilience patterns (circuit breakers, retries) contain failures to isolated components. |
| **Horizontal Scalability** | High-demand services can scale independently on Kubernetes based on CPU/Memory or custom metrics. |
| **Automated Security** | Pod-level IAM policies and dynamic secret fetching prevent credential exposure in code or container layers. |

---

## 🛠 Tech Stack

* **Framework:** Spring Boot, Spring Cloud (Gateway, Config, Eureka/Consul)
* **Resilience:** Resilience4j (Circuit Breaker, Rate Limiter)
* **Messaging:** Apache Kafka
* **Containerization & Orchestration:** Docker, Kubernetes, AWS EKS
* **Security & Secret Management:** AWS Secrets Manager, IAM Roles for Service Accounts (IRSA)
* **Database:** MySQL / PostgreSQL (Database-per-Service pattern)