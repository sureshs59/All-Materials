# Spring Boot Kafka Application: Complete Guide

**Project:** Real-Time Claims Processing System with Kafka Pub/Sub  
**Framework:** Spring Boot 3.x + Spring Kafka  
**Pattern:** Producer/Consumer (Pub/Sub)  
**Dummy Data:** Claims events  

---

## Project Structure

```
kafka-spring-boot-app/
├── pom.xml
├── docker-compose.yml
├── src/
│   ├── main/
│   │   ├── java/com/carefirst/kafka/
│   │   │   ├── KafkaSpringBootApplication.java
│   │   │   ├── config/
│   │   │   │   └── KafkaConfig.java
│   │   │   ├── model/
│   │   │   │   └── ClaimEvent.java
│   │   │   ├── producer/
│   │   │   │   └── ClaimsProducer.java
│   │   │   ├── consumer/
│   │   │   │   └── ClaimsConsumer.java
│   │   │   └── controller/
│   │   │       └── KafkaController.java
│   │   └── resources/
│   │       └── application.yml
│   └── test/
│       └── java/com/carefirst/kafka/
│           └── KafkaIntegrationTest.java
└── README.md
```

---

## Step 1: Create pom.xml

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 
                             http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <groupId>com.carefirst</groupId>
    <artifactId>kafka-spring-boot-app</artifactId>
    <version>1.0.0</version>
    <packaging>jar</packaging>

    <name>Kafka Spring Boot Application</name>
    <description>Claims Processing with Kafka Pub/Sub</description>

    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>3.0.5</version>
        <relativePath/>
    </parent>

    <properties>
        <java.version>17</java.version>
        <maven.compiler.source>17</maven.compiler.source>
        <maven.compiler.target>17</maven.compiler.target>
        <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
    </properties>

    <dependencies>
        <!-- Spring Boot Web Starter -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>

        <!-- Spring Kafka -->
        <dependency>
            <groupId>org.springframework.kafka</groupId>
            <artifactId>spring-kafka</artifactId>
        </dependency>

        <!-- Lombok for @Data, @Slf4j annotations -->
        <dependency>
            <groupId>org.projectlombok</groupId>
            <artifactId>lombok</artifactId>
            <optional>true</optional>
        </dependency>

        <!-- Jackson for JSON serialization -->
        <dependency>
            <groupId>com.fasterxml.jackson.core</groupId>
            <artifactId>jackson-databind</artifactId>
        </dependency>

        <!-- Spring Boot DevTools -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-devtools</artifactId>
            <scope>runtime</scope>
            <optional>true</optional>
        </dependency>

        <!-- Testing -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-test</artifactId>
            <scope>test</scope>
        </dependency>

        <dependency>
            <groupId>org.springframework.kafka</groupId>
            <artifactId>spring-kafka-test</artifactId>
            <scope>test</scope>
        </dependency>
    </dependencies>

    <build>
        <plugins>
            <plugin>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-maven-plugin</artifactId>
                <configuration>
                    <excludes>
                        <exclude>
                            <groupId>org.projectlombok</groupId>
                            <artifactId>lombok</artifactId>
                        </exclude>
                    </excludes>
                </configuration>
            </plugin>
        </plugins>
    </build>

</project>
```

---

## Step 2: Create docker-compose.yml

```yaml
version: '3.8'

services:
  zookeeper:
    image: confluentinc/cp-zookeeper:7.4.0
    environment:
      ZOOKEEPER_CLIENT_PORT: 2181
      ZOOKEEPER_TICK_TIME: 2000
    ports:
      - "2181:2181"
    networks:
      - kafka-network

  kafka:
    image: confluentinc/cp-kafka:7.4.0
    depends_on:
      - zookeeper
    ports:
      - "9092:9092"
      - "9101:9101"
    environment:
      KAFKA_BROKER_ID: 1
      KAFKA_ZOOKEEPER_CONNECT: zookeeper:2181
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://kafka:29092,PLAINTEXT_HOST://localhost:9092
      KAFKA_LISTENER_SECURITY_PROTOCOL_MAP: PLAINTEXT:PLAINTEXT,PLAINTEXT_HOST:PLAINTEXT
      KAFKA_INTER_BROKER_LISTENER_NAME: PLAINTEXT
      KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 1
      KAFKA_TRANSACTION_STATE_LOG_MIN_ISR: 1
      KAFKA_TRANSACTION_STATE_LOG_REPLICATION_FACTOR: 1
      KAFKA_AUTO_CREATE_TOPICS_ENABLE: 'true'
    networks:
      - kafka-network

networks:
  kafka-network:
    driver: bridge
```

**How to run:**
```bash
docker-compose up -d

# Verify Kafka is running
docker ps
docker logs kafka

# Stop services
docker-compose down
```

---

## Step 3: Create Model Class (ClaimEvent.java)

```java
package com.carefirst.kafka.model;

import lombok.AllArgsConstructor;
import lombok.Builder;
import lombok.Data;
import lombok.NoArgsConstructor;
import com.fasterxml.jackson.annotation.JsonProperty;

@Data
@NoArgsConstructor
@AllArgsConstructor
@Builder
public class ClaimEvent {
    
    @JsonProperty("claim_id")
    private String claimId;
    
    @JsonProperty("member_id")
    private String memberId;
    
    @JsonProperty("amount")
    private double amount;
    
    @JsonProperty("status")
    private String status;
    
    @JsonProperty("timestamp")
    private long timestamp;
    
    @JsonProperty("event_type")
    private String eventType;
    
    @JsonProperty("description")
    private String description;
    
    /**
     * Create a dummy claim event
     */
    public static ClaimEvent createDummy(int index) {
        return ClaimEvent.builder()
            .claimId("CLAIM-" + System.currentTimeMillis() + "-" + index)
            .memberId("MEMBER-" + (index % 10))
            .amount(100.00 + (index * 10))
            .status("SUBMITTED")
            .timestamp(System.currentTimeMillis())
            .eventType("CLAIM_SUBMITTED")
            .description("Claim submitted for member " + (index % 10))
            .build();
    }
}
```

---

## Step 4: Create Kafka Configuration (KafkaConfig.java)

```java
package com.carefirst.kafka.config;

import org.apache.kafka.clients.admin.AdminClientConfig;
import org.apache.kafka.clients.admin.NewTopic;
import org.apache.kafka.clients.consumer.ConsumerConfig;
import org.apache.kafka.clients.producer.ProducerConfig;
import org.apache.kafka.common.serialization.StringDeserializer;
import org.apache.kafka.common.serialization.StringSerializer;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.kafka.annotation.EnableKafka;
import org.springframework.kafka.config.ConcurrentKafkaListenerContainerFactory;
import org.springframework.kafka.config.KafkaListenerContainerFactory;
import org.springframework.kafka.config.TopicBuilder;
import org.springframework.kafka.core.*;
import org.springframework.kafka.listener.ConcurrentMessageListenerContainer;
import org.springframework.kafka.support.serializer.JsonDeserializer;
import org.springframework.kafka.support.serializer.JsonSerializer;
import com.carefirst.kafka.model.ClaimEvent;

import java.util.HashMap;
import java.util.Map;

@Configuration
@EnableKafka
public class KafkaConfig {

    @Value("${spring.kafka.bootstrap-servers}")
    private String bootstrapServers;

    @Value("${kafka.topic.claims}")
    private String claimsTopic;

    @Value("${kafka.topic.claims-processed}")
    private String processedClaimsTopic;

    /**
     * Kafka Admin - Create Topics
     */
    @Bean
    public KafkaAdmin admin() {
        Map<String, Object> configs = new HashMap<>();
        configs.put(AdminClientConfig.BOOTSTRAP_SERVERS_CONFIG, bootstrapServers);
        return new KafkaAdmin(configs);
    }

    /**
     * Create claims-events topic
     */
    @Bean
    public NewTopic claimsEventsTopic() {
        return TopicBuilder.name(claimsTopic)
            .partitions(3)
            .replicas(1)
            .build();
    }

    /**
     * Create processed-claims topic
     */
    @Bean
    public NewTopic processedClaimsTopic() {
        return TopicBuilder.name(processedClaimsTopic)
            .partitions(3)
            .replicas(1)
            .build();
    }

    /**
     * Producer Configuration
     */
    @Bean
    public ProducerFactory<String, ClaimEvent> producerFactory() {
        Map<String, Object> configProps = new HashMap<>();
        
        configProps.put(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG, bootstrapServers);
        configProps.put(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG, StringSerializer.class);
        configProps.put(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG, JsonSerializer.class);
        
        // Reliability settings
        configProps.put(ProducerConfig.ACKS_CONFIG, "all");              // Wait for all replicas
        configProps.put(ProducerConfig.RETRIES_CONFIG, 3);              // Retry 3 times
        configProps.put(ProducerConfig.ENABLE_IDEMPOTENCE_CONFIG, true); // Exactly-once
        
        // Performance settings
        configProps.put(ProducerConfig.BATCH_SIZE_CONFIG, 16384);        // 16KB batches
        configProps.put(ProducerConfig.LINGER_MS_CONFIG, 100);          // Wait 100ms for batch
        configProps.put(ProducerConfig.COMPRESSION_TYPE_CONFIG, "snappy"); // Compression
        
        return new DefaultKafkaProducerFactory<>(configProps);
    }

    @Bean
    public KafkaTemplate<String, ClaimEvent> kafkaTemplate() {
        return new KafkaTemplate<>(producerFactory());
    }

    /**
     * Consumer Configuration
     */
    @Bean
    public ConsumerFactory<String, ClaimEvent> consumerFactory() {
        Map<String, Object> configProps = new HashMap<>();
        
        configProps.put(ConsumerConfig.BOOTSTRAP_SERVERS_CONFIG, bootstrapServers);
        configProps.put(ConsumerConfig.GROUP_ID_CONFIG, "claims-processor-group");
        configProps.put(ConsumerConfig.KEY_DESERIALIZER_CLASS_CONFIG, StringDeserializer.class);
        configProps.put(ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG, JsonDeserializer.class);
        configProps.put(JsonDeserializer.VALUE_DEFAULT_TYPE, ClaimEvent.class.getName());
        configProps.put(JsonDeserializer.TRUSTED_PACKAGES, "*");
        
        // Offset management
        configProps.put(ConsumerConfig.AUTO_OFFSET_RESET_CONFIG, "earliest"); // Start from beginning
        configProps.put(ConsumerConfig.ENABLE_AUTO_COMMIT_CONFIG, true);     // Auto commit offsets
        configProps.put(ConsumerConfig.AUTO_COMMIT_INTERVAL_MS_CONFIG, 1000);// Commit every 1 second
        
        // Performance
        configProps.put(ConsumerConfig.MAX_POLL_RECORDS_CONFIG, 100);        // Process 100 at a time
        configProps.put(ConsumerConfig.SESSION_TIMEOUT_MS_CONFIG, 30000);    // 30 second timeout
        
        return new DefaultKafkaConsumerFactory<>(configProps);
    }

    @Bean
    public KafkaListenerContainerFactory<ConcurrentMessageListenerContainer<String, ClaimEvent>> 
        kafkaListenerContainerFactory() {
        ConcurrentKafkaListenerContainerFactory<String, ClaimEvent> factory = 
            new ConcurrentKafkaListenerContainerFactory<>();
        factory.setCommonErrorHandler(kafkaErrorHandler());
        factory.setConcurrency(3); // 3 consumer threads
        factory.getContainerProperties().setPollTimeout(3000); // Poll timeout 3 seconds
        factory.setConsumerFactory(consumerFactory());
        return factory;
    }

    /**
     * Error handling for consumers
     */
    @Bean
    public org.springframework.kafka.listener.CommonErrorHandler kafkaErrorHandler() {
        return new org.springframework.kafka.listener.DefaultErrorHandler();
    }
}
```

---

## Step 5: Create Producer (ClaimsProducer.java)

```java
package com.carefirst.kafka.producer;

import com.carefirst.kafka.model.ClaimEvent;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.kafka.core.KafkaTemplate;
import org.springframework.kafka.support.KafkaHeaders;
import org.springframework.messaging.Message;
import org.springframework.messaging.support.MessageBuilder;
import org.springframework.stereotype.Service;

import java.util.concurrent.CompletableFuture;

@Slf4j
@Service
@RequiredArgsConstructor
public class ClaimsProducer {

    private final KafkaTemplate<String, ClaimEvent> kafkaTemplate;

    @Value("${kafka.topic.claims}")
    private String claimsTopic;

    /**
     * Send claim event to Kafka
     */
    public void sendClaim(ClaimEvent claimEvent) {
        log.info("Publishing claim event: {}", claimEvent.getClaimId());
        
        try {
            // Build message with headers
            Message<ClaimEvent> message = MessageBuilder
                .withPayload(claimEvent)
                .setHeader(KafkaHeaders.TOPIC, claimsTopic)
                .setHeader(KafkaHeaders.MESSAGE_KEY, claimEvent.getClaimId())
                .setHeader("correlation-id", claimEvent.getClaimId())
                .setHeader("timestamp", System.currentTimeMillis())
                .build();

            // Send asynchronously
            kafkaTemplate.send(message)
                .whenComplete((result, ex) -> {
                    if (ex == null) {
                        log.info("Claim {} sent successfully to partition {} offset {}",
                            claimEvent.getClaimId(),
                            result.getRecordMetadata().partition(),
                            result.getRecordMetadata().offset());
                    } else {
                        log.error("Failed to send claim {}: {}", 
                            claimEvent.getClaimId(), ex.getMessage(), ex);
                    }
                });

        } catch (Exception e) {
            log.error("Error publishing claim: {}", e.getMessage(), e);
            throw new RuntimeException("Failed to publish claim", e);
        }
    }

    /**
     * Send multiple claims
     */
    public void sendMultipleClaims(int count) {
        log.info("Publishing {} claim events", count);
        
        for (int i = 0; i < count; i++) {
            ClaimEvent claim = ClaimEvent.createDummy(i);
            sendClaim(claim);
            
            // Small delay between sends (optional)
            try {
                Thread.sleep(100);
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
                log.error("Thread interrupted", e);
            }
        }
    }

    /**
     * Send claim synchronously (for testing)
     */
    public void sendClaimSync(ClaimEvent claimEvent) throws Exception {
        log.info("Sending claim synchronously: {}", claimEvent.getClaimId());
        
        kafkaTemplate.send(claimsTopic, claimEvent.getClaimId(), claimEvent)
            .get(); // Blocking call
        
        log.info("Claim sent synchronously");
    }
}
```

---

## Step 6: Create Consumer (ClaimsConsumer.java)

```java
package com.carefirst.kafka.consumer;

import com.carefirst.kafka.model.ClaimEvent;
import lombok.extern.slf4j.Slf4j;
import org.springframework.kafka.annotation.KafkaListener;
import org.springframework.kafka.support.Acknowledgment;
import org.springframework.kafka.support.KafkaHeaders;
import org.springframework.messaging.handler.annotation.Header;
import org.springframework.messaging.handler.annotation.Payload;
import org.springframework.stereotype.Service;

@Slf4j
@Service
public class ClaimsConsumer {

    /**
     * Listen to claims-events topic
     * This will be called for each message automatically
     */
    @KafkaListener(
        topics = "${kafka.topic.claims}",
        groupId = "claims-processor-group",
        concurrency = "3"  // 3 parallel consumers
    )
    public void consumeClaim(
        @Payload ClaimEvent claimEvent,
        @Header(KafkaHeaders.RECEIVED_PARTITION_ID) int partition,
        @Header(KafkaHeaders.OFFSET) long offset) {
        
        log.info("Received claim from partition {} offset {}: {}",
            partition, offset, claimEvent.getClaimId());
        log.info("Claim details - Member: {}, Amount: {}, Status: {}",
            claimEvent.getMemberId(),
            claimEvent.getAmount(),
            claimEvent.getStatus());

        try {
            // Process claim
            processClaimEvent(claimEvent);
            
            log.info("Successfully processed claim: {}", claimEvent.getClaimId());
            
        } catch (Exception e) {
            log.error("Error processing claim {}: {}",
                claimEvent.getClaimId(), e.getMessage(), e);
            // In production, send to dead letter queue
        }
    }

    /**
     * Alternative listener with manual acknowledgment
     */
    @KafkaListener(
        topics = "${kafka.topic.claims}",
        groupId = "claims-manual-ack-group",
        containerFactory = "kafkaListenerContainerFactory"
    )
    public void consumeClaimWithAck(
        @Payload ClaimEvent claimEvent,
        @Header(KafkaHeaders.RECEIVED_PARTITION_ID) int partition,
        @Header(KafkaHeaders.OFFSET) long offset,
        Acknowledgment acknowledgment) {
        
        log.info("Processing claim with manual ack: {}", claimEvent.getClaimId());
        
        try {
            // Process claim
            processClaimEvent(claimEvent);
            
            // Commit offset manually after successful processing
            if (acknowledgment != null) {
                acknowledgment.acknowledge();
                log.info("Offset committed for claim: {}", claimEvent.getClaimId());
            }
            
        } catch (Exception e) {
            log.error("Failed to process claim: {}", e.getMessage());
            // Don't acknowledge - will retry
        }
    }

    /**
     * Business logic to process claim
     */
    private void processClaimEvent(ClaimEvent claimEvent) {
        // Simulate processing
        log.debug("Processing claim: {}", claimEvent);
        
        // Example validations
        if (claimEvent.getAmount() <= 0) {
            throw new IllegalArgumentException("Invalid amount: " + claimEvent.getAmount());
        }
        
        // Simulate some processing delay
        try {
            Thread.sleep(100); // Simulate work
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
            throw new RuntimeException(e);
        }
        
        log.info("Claim {} processed successfully", claimEvent.getClaimId());
    }
}
```

---

## Step 7: Create REST Controller (KafkaController.java)

```java
package com.carefirst.kafka.controller;

import com.carefirst.kafka.model.ClaimEvent;
import com.carefirst.kafka.producer.ClaimsProducer;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.http.HttpStatus;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;

import java.util.HashMap;
import java.util.Map;

@Slf4j
@RestController
@RequestMapping("/api/kafka")
@RequiredArgsConstructor
public class KafkaController {

    private final ClaimsProducer claimsProducer;

    /**
     * Health check endpoint
     */
    @GetMapping("/health")
    public ResponseEntity<Map<String, String>> health() {
        Map<String, String> response = new HashMap<>();
        response.put("status", "UP");
        response.put("message", "Kafka application is running");
        return ResponseEntity.ok(response);
    }

    /**
     * Send a single claim event
     */
    @PostMapping("/send-claim")
    public ResponseEntity<Map<String, String>> sendClaim(
        @RequestBody(required = false) ClaimEvent claimEvent) {
        
        try {
            // Use provided claim or create dummy
            if (claimEvent == null) {
                claimEvent = ClaimEvent.createDummy(1);
            }
            
            claimsProducer.sendClaim(claimEvent);
            
            Map<String, String> response = new HashMap<>();
            response.put("status", "SUCCESS");
            response.put("message", "Claim sent to Kafka");
            response.put("claimId", claimEvent.getClaimId());
            
            return ResponseEntity.ok(response);
            
        } catch (Exception e) {
            log.error("Error sending claim", e);
            
            Map<String, String> response = new HashMap<>();
            response.put("status", "ERROR");
            response.put("message", e.getMessage());
            
            return ResponseEntity.status(HttpStatus.INTERNAL_SERVER_ERROR).body(response);
        }
    }

    /**
     * Send multiple dummy claim events
     * Example: POST /api/kafka/send-claims?count=10
     */
    @PostMapping("/send-claims")
    public ResponseEntity<Map<String, Object>> sendMultipleClaims(
        @RequestParam(defaultValue = "10") int count) {
        
        try {
            log.info("Sending {} claims", count);
            
            claimsProducer.sendMultipleClaims(count);
            
            Map<String, Object> response = new HashMap<>();
            response.put("status", "SUCCESS");
            response.put("message", count + " claims sent to Kafka");
            response.put("count", count);
            response.put("timestamp", System.currentTimeMillis());
            
            return ResponseEntity.ok(response);
            
        } catch (Exception e) {
            log.error("Error sending claims", e);
            
            Map<String, Object> response = new HashMap<>();
            response.put("status", "ERROR");
            response.put("message", e.getMessage());
            
            return ResponseEntity.status(HttpStatus.INTERNAL_SERVER_ERROR).body(response);
        }
    }

    /**
     * Get application info
     */
    @GetMapping("/info")
    public ResponseEntity<Map<String, String>> info() {
        Map<String, String> response = new HashMap<>();
        response.put("application", "Kafka Spring Boot Claims Processor");
        response.put("version", "1.0.0");
        response.put("description", "Real-time claims processing with Kafka");
        response.put("endpoints", "POST /api/kafka/send-claim, POST /api/kafka/send-claims?count=10");
        return ResponseEntity.ok(response);
    }
}
```

---

## Step 8: Create application.yml

```yaml
spring:
  application:
    name: kafka-spring-boot-app
  
  kafka:
    bootstrap-servers: localhost:9092
    
    # Producer configuration
    producer:
      key-serializer: org.apache.kafka.common.serialization.StringSerializer
      value-serializer: org.springframework.kafka.support.serializer.JsonSerializer
      acks: all
      retries: 3
      batch-size: 16384
      linger-ms: 100
      compression-type: snappy
    
    # Consumer configuration
    consumer:
      bootstrap-servers: localhost:9092
      group-id: claims-processor-group
      key-deserializer: org.apache.kafka.common.serialization.StringDeserializer
      value-deserializer: org.springframework.kafka.support.serializer.JsonDeserializer
      auto-offset-reset: earliest
      enable-auto-commit: true
      auto-commit-interval-ms: 1000
      max-poll-records: 100
      properties:
        spring.json.trusted.packages: '*'

# Custom Kafka topic names
kafka:
  topic:
    claims: claims-events
    claims-processed: processed-claims

# Server configuration
server:
  port: 8080
  servlet:
    context-path: /

# Logging
logging:
  level:
    root: INFO
    com.carefirst.kafka: DEBUG
    org.springframework.kafka: INFO
  pattern:
    console: "%d{yyyy-MM-dd HH:mm:ss} - %msg%n"
```

---

## Step 9: Create Main Application Class (KafkaSpringBootApplication.java)

```java
package com.carefirst.kafka;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.kafka.annotation.EnableKafka;

@SpringBootApplication
@EnableKafka
public class KafkaSpringBootApplication {

    public static void main(String[] args) {
        SpringApplication.run(KafkaSpringBootApplication.class, args);
    }
}
```

---

## Step 10: Running the Application

### Prerequisites
- Java 17+
- Maven 3.6+
- Docker & Docker Compose

### Start Kafka

```bash
# Navigate to project directory
cd kafka-spring-boot-app

# Start Kafka and Zookeeper
docker-compose up -d

# Verify Kafka is running
docker ps

# View logs
docker logs kafka
```

### Build and Run Application

```bash
# Build the application
mvn clean package

# Run the Spring Boot application
mvn spring-boot:run

# Or run the JAR directly
java -jar target/kafka-spring-boot-app-1.0.0.jar
```

### Check Application is Running

```bash
# Health check
curl http://localhost:8080/api/kafka/health

# Get application info
curl http://localhost:8080/api/kafka/info
```

---

## Step 11: Testing the Application

### Send a Single Claim

```bash
# Send single dummy claim
curl -X POST http://localhost:8080/api/kafka/send-claim

# Send single claim with custom data
curl -X POST http://localhost:8080/api/kafka/send-claim \
  -H "Content-Type: application/json" \
  -d '{
    "claim_id": "CLAIM-12345",
    "member_id": "MEMBER-999",
    "amount": 5000.00,
    "status": "SUBMITTED",
    "event_type": "CLAIM_SUBMITTED",
    "description": "Test claim"
  }'
```

### Send Multiple Claims

```bash
# Send 10 dummy claims
curl -X POST http://localhost:8080/api/kafka/send-claims?count=10

# Send 100 dummy claims
curl -X POST http://localhost:8080/api/kafka/send-claims?count=100
```

### View Application Logs

```bash
# Watch the application logs
# You should see:
# - Publishing claim event: CLAIM-xxx
# - Received claim from partition X offset Y
# - Successfully processed claim
```

### Monitor Kafka Topics

```bash
# List topics
docker exec -it kafka kafka-topics --list --bootstrap-server localhost:9092

# View topic details
docker exec -it kafka kafka-topics \
  --describe \
  --topic claims-events \
  --bootstrap-server localhost:9092

# Consume messages from beginning
docker exec -it kafka kafka-console-consumer \
  --topic claims-events \
  --bootstrap-server localhost:9092 \
  --from-beginning \
  --property print.key=true
```

---

## Application Flow Diagram

```
┌─────────────────────────────────────────────────────────────┐
│                  Spring Boot Application                    │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  REST API Endpoint                                           │
│  POST /api/kafka/send-claims?count=10                       │
│         │                                                    │
│         ↓                                                    │
│  ┌──────────────────────┐                                   │
│  │ KafkaController      │                                   │
│  │ Receives request     │                                   │
│  └──────────┬───────────┘                                   │
│             │                                                │
│             ↓                                                │
│  ┌──────────────────────┐                                   │
│  │ ClaimsProducer       │                                   │
│  │ Creates dummy claims │                                   │
│  │ Sends to Kafka       │                                   │
│  └──────────┬───────────┘                                   │
│             │                                                │
│             ↓                                                │
│  ┌─────────────────────────────────────┐                   │
│  │      KAFKA BROKER CLUSTER            │                   │
│  │  Topic: claims-events (3 partitions)│                   │
│  │  ├─ Partition 0                     │                   │
│  │  ├─ Partition 1                     │                   │
│  │  └─ Partition 2                     │                   │
│  └─────────────┬───────────────────────┘                   │
│                │                                             │
│                ↓                                             │
│  ┌──────────────────────────────────────┐                  │
│  │  ClaimsConsumer                      │                  │
│  │  Listens to topics (3 threads)       │                  │
│  │  Processes claims in real-time       │                  │
│  └──────────────────────────────────────┘                  │
│                                                              │
└─────────────────────────────────────────────────────────────┘

Message Flow:
1. REST API receives request: POST /send-claims?count=10
2. Controller calls ClaimsProducer.sendMultipleClaims(10)
3. Producer creates 10 dummy ClaimEvent objects
4. Each event serialized to JSON and sent to Kafka
5. Kafka broker receives and stores messages in claims-events topic
6. Messages distributed across 3 partitions
7. Consumer automatically processes messages from topic
8. Consumer prints logs: "Received claim...", "Successfully processed..."
```

---

## Expected Output

### When Starting Application

```
2024-06-05 10:30:15 - Started KafkaSpringBootApplication
2024-06-05 10:30:15 - Tomcat started on port 8080
2024-06-05 10:30:15 - Application startup complete
```

### When Sending Claims

```
2024-06-05 10:31:20 - Publishing 10 claim events
2024-06-05 10:31:20 - Publishing claim event: CLAIM-1717579880123-0
2024-06-05 10:31:20 - Claim CLAIM-1717579880123-0 sent successfully to partition 1 offset 0
2024-06-05 10:31:20 - Publishing claim event: CLAIM-1717579880223-1
2024-06-05 10:31:20 - Claim CLAIM-1717579880223-1 sent successfully to partition 0 offset 0
```

### When Consuming Claims

```
2024-06-05 10:31:21 - Received claim from partition 1 offset 0: CLAIM-1717579880123-0
2024-06-05 10:31:21 - Claim details - Member: MEMBER-0, Amount: 100.0, Status: SUBMITTED
2024-06-05 10:31:21 - Processing claim with manual ack: CLAIM-1717579880123-0
2024-06-05 10:31:21 - Successfully processed claim: CLAIM-1717579880123-0
```

---

## Cleanup

```bash
# Stop Kafka and Zookeeper
docker-compose down

# Remove volumes (optional)
docker-compose down -v

# Check stopped containers
docker ps -a
```

---

## Key Features Implemented

✅ **Producer**
- Asynchronous message sending
- Idempotent producer (exactly-once)
- Message compression (snappy)
- Batching for performance

✅ **Consumer**
- Automatic message consumption
- Manual acknowledgment support
- Parallel processing (3 threads)
- Error handling

✅ **Configuration**
- Externalized properties (application.yml)
- Kafka topics auto-created
- Producer/Consumer settings optimized

✅ **REST API**
- Send single claim: POST /api/kafka/send-claim
- Send multiple claims: POST /api/kafka/send-claims?count=10
- Health check: GET /api/kafka/health
- Application info: GET /api/kafka/info

✅ **Logging**
- Structured logging with Slf4j
- Partition and offset tracking
- Error logging

✅ **Docker Support**
- Docker Compose for Kafka infrastructure
- Easy local development
- Reproducible environment

---

## Troubleshooting

### Kafka Connection Error

```
Error: Connection to Kafka broker failed

Solution:
1. Verify Kafka is running: docker ps
2. Check Kafka logs: docker logs kafka
3. Verify bootstrap-servers: localhost:9092
4. Wait 10 seconds for Kafka to fully start
```

### Messages Not Appearing

```
Solution:
1. Check consumer group offset: 
   docker exec kafka kafka-consumer-groups --list --bootstrap-server localhost:9092

2. Reset consumer offset:
   docker exec kafka kafka-consumer-groups \
     --reset-offsets --to-earliest \
     --group claims-processor-group \
     --topic claims-events \
     --execute --bootstrap-server localhost:9092
```

### Port Already in Use

```
Solution:
# Change port in docker-compose.yml
ports:
  - "9093:9092"  # Change external port

# Update bootstrap-servers in application.yml
bootstrap-servers: localhost:9093
```

