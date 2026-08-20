# Comprehensive Guide: Spring Boot Microservices Architecture & Resilience4j Circuit Breaker

This document provides a reference architecture and practical implementation guide for building resilient, cloud-native Spring Boot microservices on Kubernetes (AWS EKS).

---

## 1. Spring Boot Microservices Reference Architecture

The architecture below illustrates the complete request lifecycle, edge routing, Kubernetes deployment, secret management, and cross-cutting concerns.

<img width="1024" height="559" alt="image" src="https://github.com/user-attachments/assets/d3354950-53e0-4a7d-a825-7af8acb6961b" />

Diagram Structure: The Lifecycle of a Request
Imagine a linear flow from left to right, showing how a request is handled.

Top Title: RESILIENCE4J CIRCUIT BREAKER: HOW IT WORKS

### Main Elements:

A large arrow labeled "CLIENT REQUEST" pointing into a central box.

The Central Box is labeled: "Resilience4j Circuit Breaker".

Two outgoing paths from the central box.

Layout:

State Transition Flowchart Inside the Central Box:
You should show the state transitions with simple logic.

### [ Start: CLOSED ] (Green Box)

Arrows going out: "Backend Call" (Solid arrow)

Arrow pointing back: "Failure Rate < Threshold"

Text Label: "All requests pass. Monitoring: send() / onError()."

A prominent arrow points from CLOSED to OPEN: "Failure Rate > Threshold" OR "Slow Call Rate > Threshold"

### [ OPEN ] (Red Box)

#### Arrow pointing down from it: "Requests FAILED FAST" (Solid arrow). Label: throws CallNotPermittedException

#### Arrow going to a separate side box: "Trigger FALLBACK" (Cache, Default Data).

#### A prominent arrow points from OPEN to HALF-OPEN after a time delay: "After WaitDurationInOpenState"

### [ HALF-OPEN ] (Amber Box)

#### Text Label: "Limited Test Requests Allowed."

#### Arrows going out: "Backend Call" (Solid arrow).

#### Arrow pointing from HALF-OPEN back to CLOSED: "Test Calls Succeed (Failure < Threshold)"

#### Arrow pointing from HALF-OPEN back to OPEN: "Test Calls Fail (Failure > Threshold)"

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
```
---

## 5. Resilience4j Configuration (application.yml)

### `application.yml`
```xml
resilience4j:
  circuitbreaker:
    instances:
      paymentServiceCB:
        slidingWindowType: COUNT_BASED
        slidingWindowSize: 10                   # Evaluate error rate over the last 10 calls
        minimumNumberOfCalls: 5                  # Must have at least 5 calls before evaluating
        failureRateThreshold: 50                 # Trip to OPEN if 50% or more calls fail
        slowCallDurationThreshold: 2s            # Calls taking longer than 2s count as slow
        slowCallRateThreshold: 50               # Trip if 50% of calls are slow
        waitDurationInOpenState: 10s            # Stay in OPEN for 10 seconds before HALF-OPEN
        permittedNumberOfCallsInHalfOpenState: 3 # Allow 3 trial requests in HALF-OPEN
        automaticTransitionFromOpenToHalfOpenEnabled: true
```
---

## 6. Service Layer with Circuit Breaker & Fallback (OrderService.java)
### `OrderService.java`
```Java

package com.example.orderservice.service;

import io.github.resilience4j.circuitbreaker.annotation.CircuitBreaker;
import org.springframework.stereotype.Service;
import org.springframework.web.client.RestTemplate;

@Service
public class OrderService {

    private final RestTemplate restTemplate = new RestTemplate();
    private static final String PAYMENT_SERVICE_URL = "http://localhost:8081/api/payment";

    /**
     * Executes remote payment call wrapped by Resilience4j Circuit Breaker.
     * Name matches the instance defined in application.yml.
     */
    @CircuitBreaker(name = "paymentServiceCB", fallbackMethod = "processPaymentFallback")
    public String processPayment(String orderId, double amount) {
        // Calling remote Payment Service
        return restTemplate.postForObject(
            PAYMENT_SERVICE_URL + "/charge?orderId=" + orderId + "&amount=" + amount, 
            null, 
            String.class
        );
    }

    /**
     * Fallback Method executed when:
     * 1. Remote service throws an Exception (e.g., 5xx, Timeout).
     * 2. Circuit Breaker is in OPEN state (throws CallNotPermittedException).
     */
    public String processPaymentFallback(String orderId, double amount, Throwable throwable) {
        System.err.println("Fallback triggered due to: " + throwable.getMessage());
        return "ORDER_PENDING: Payment Service is currently unavailable. Your order " 
                + orderId + " has been queued for processing.";
    }
}

```
---
## 7. REST Controller 
### `OrderController.java`
```Java

package com.example.orderservice.controller;

import com.example.orderservice.service.OrderService;
import org.springframework.web.bind.annotation.*;

@RestController
@RequestMapping("/api/orders")
public class OrderController {

    private final OrderService orderService;

    public OrderController(OrderService orderService) {
        this.orderService = orderService;
    }

    @PostMapping("/checkout")
    public String checkout(@RequestParam String orderId, @RequestParam double amount) {
        return orderService.processPayment(orderId, amount);
    }
}
```
