# Comprehensive Guide: Spring Boot Microservices Architecture & Resilience4j Circuit Breaker

This document provides a reference architecture and practical implementation guide for building resilient, cloud-native Spring Boot microservices on Kubernetes (AWS EKS).

---

## 1. Spring Boot Microservices Reference Architecture

The architecture below illustrates the complete request lifecycle, edge routing, Kubernetes deployment, secret management, and cross-cutting concerns.

![Spring Boot Microservice Architecture](https://replicate.delivery/xpbkg/9o9Vj9S48e0Aey3r9IuR15l9k5s5u9p4H2n2F2F2F2F2F2F/output.png)

### 🔄 Architecture Component Breakdown

#### A. Edge Services & Request Entry
* **Clients:** Web browsers, mobile applications, and external client systems.
* **API Gateway (Spring Cloud Gateway):** Single entry point handling client routing, initial security validation, and rate-limiting.

#### B. Core Service Deployment Layer
* **Kubernetes Pods (AWS EKS):** Services (e.g., Order Service, Product Service, User Service) run as isolated containers across Kubernetes worker nodes.
* **Database-per-Service:** Each microservice owns its private, isolated data store to maintain loose coupling.
* **AWS Secrets Manager & IRSA:** Credentials are never hardcoded in code or images. Pods utilize IAM Roles for Service Accounts (IRSA) or EKS Pod Identity to fetch database passwords and secrets directly at runtime.

#### C. Control Plane & Registry
* **Service Registry (Consul / Eureka):** Maintains dynamic IP mappings so microservices can communicate directly without hardcoded addresses.
* **Centralized Configuration:** Config parameters are externalized and managed centrally via version-controlled systems (Git, AWS SSM).

#### D. Shared Cross-Cutting Concerns
* **Event-Driven Messaging (Apache Kafka):** Provides asynchronous messaging between microservices for decoupled communication workflows.
* **Distributed Observability:** Unified logging, metrics, and distributed tracing across boundaries.
* **Resilience4j:** Embedded fault-tolerance patterns (Circuit Breakers, Rate Limiters) to prevent service degradation from cascading.

---

## 2. Resilience4j Circuit Breaker Visual Guide & Mechanics

The Circuit Breaker pattern acts as a protective fuse for inter-service communication, cutting off traffic to failing downstream services to allow recovery.

![Resilience4j Circuit Breaker Visual Guide](https://replicate.delivery/xpbkg/7576185536915722526/output.jpg)

### 📊 Circuit Breaker State Machine

| State | Status | Behavior & Action |
| :--- | :--- | :--- |
| **1. CLOSED** | Normal Operation | Calls execute normally. Resilience4j calculates error and slow-call metrics over a sliding window. |
| **2. OPEN** | Failed / Short-Circuited | Downstream calls are blocked. System **fails fast**, throws `CallNotPermittedException`, and triggers the fallback method immediately. |
| **3. HALF-OPEN** | Recovery Testing | After a configured wait timer elapses, a limited number of test requests are permitted through to check if the downstream service has recovered. |

---

## 3. Practical Example: Order Service Calling Payment Service

### ⚙️ Scenario
An **Order Service** calls an external **Payment Service** to charge a customer during checkout.

1. **Normal Flow (CLOSED State):** Payment requests pass through normally.
2. **Outage Occurs:** The Payment Service database crashes, causing requests to time out or return HTTP `500` errors.
3. **Breaker Trips (OPEN State):** Once 50% of the last 10 requests fail, Resilience4j trips the circuit to `OPEN`. Subsequent payment attempts fail fast instantly without hammering the dying Payment Service. Users receive a graceful fallback: *"Payment system degraded. Order saved as PENDING."*
4. **Self-Healing (HALF-OPEN State):** After 10 seconds, Resilience4j permits 3 test requests to the Payment Service. If successful, the circuit resets to `CLOSED`.

---

## 4. Complete Spring Boot Code Implementation

### `pom.xml` Dependencies
```xml
<dependencies>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>
    <dependency>
        <groupId>io.github.resilience4j</groupId>
        <artifactId>resilience4j-spring-boot3</artifactId>
        <version>2.2.0</version>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-aop</artifactId>
    </dependency>
</dependencies>