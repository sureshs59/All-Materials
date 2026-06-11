# Spring Boot + Kafka Application - Quick Start Guide

**Project Type:** Real-Time Claims Processing with Kafka Producer/Consumer  
**Technologies:** Spring Boot 3.x, Apache Kafka, Docker  
**Pattern:** Pub/Sub (Publish/Subscribe)  
**Status:** ✅ Ready to Run Locally  

---

## 🚀 Quick Start (5 Minutes)

### Step 1: Start Kafka (Docker)

```bash
# Clone/create project directory
mkdir kafka-spring-boot-app
cd kafka-spring-boot-app

# Create docker-compose.yml (provided in guide)
# Run Kafka and Zookeeper
docker-compose up -d

# Verify running
docker ps
```

### Step 2: Build Spring Boot Application

```bash
# Clone pom.xml from guide, create project structure
mvn clean install

# Run application
mvn spring-boot:run

# Or after building:
java -jar target/kafka-spring-boot-app-1.0.0.jar
```

### Step 3: Send Test Messages

```bash
# Send 10 dummy claims
curl -X POST http://localhost:8080/api/kafka/send-claims?count=10

# Watch the logs - you'll see:
# - Publishing claim event
# - Received claim from partition
# - Successfully processed claim
```

✅ **Done!** Your Kafka pub/sub is working!

---

## 📋 File-by-File Reference

### 1. **pom.xml** (Maven Dependencies)
- Spring Boot Starter Web
- Spring Kafka
- Lombok (for @Data, @Slf4j)
- Jackson (JSON serialization)
- Test dependencies

**Size:** ~350 lines  
**Purpose:** Manage project dependencies and build

---

### 2. **docker-compose.yml** (Kafka Infrastructure)
- Zookeeper service (coordination)
- Kafka broker service (single node)
- Network configuration
- Auto-topic creation

**Size:** ~50 lines  
**Purpose:** Local Kafka cluster setup

**Commands:**
```bash
docker-compose up -d      # Start services
docker-compose down       # Stop services
docker logs kafka         # View Kafka logs
```

---

### 3. **application.yml** (Spring Configuration)
- Kafka bootstrap servers
- Producer settings (acks=all, compression, batching)
- Consumer settings (auto-commit, offsets)
- Topic names (claims-events, processed-claims)
- Logging configuration

**Size:** ~50 lines  
**Purpose:** Externalized configuration for easy environment switching

---

### 4. **ClaimEvent.java** (Model/DTO)
- Fields: claimId, memberId, amount, status, timestamp, eventType, description
- Lombok annotations (@Data, @Builder, @NoArgsConstructor, @AllArgsConstructor)
- createDummy() method for test data
- JSON serialization with @JsonProperty

**Size:** ~60 lines  
**Purpose:** Data model for claims messages

```java
ClaimEvent event = ClaimEvent.createDummy(1);
// Creates: CLAIM-[timestamp]-1, MEMBER-1, amount: 110.00, etc.
```

---

### 5. **KafkaConfig.java** (Kafka Configuration)
- Producer Factory (KafkaTemplate bean)
- Consumer Factory (Listener Container)
- Topic creation (NewTopic beans)
- Serialization/Deserialization settings
- Error handling

**Size:** ~200 lines  
**Purpose:** Spring Kafka beans and configuration

**Key Settings:**
```
Producer: acks=all, idempotence=true, compression=snappy
Consumer: auto-commit=true, group-id=claims-processor-group
```

---

### 6. **ClaimsProducer.java** (Message Publisher)
- sendClaim() - Send single claim (async)
- sendMultipleClaims(int count) - Send batch of claims
- sendClaimSync() - Blocking send (for testing)
- Message header configuration
- Success/failure callbacks

**Size:** ~120 lines  
**Purpose:** Publish claims events to Kafka topic

```java
claimsProducer.sendClaim(claimEvent);
claimsProducer.sendMultipleClaims(10);
```

---

### 7. **ClaimsConsumer.java** (Message Subscriber)
- @KafkaListener annotation for automatic consumption
- Partition and offset tracking
- Manual acknowledgment support
- Error handling
- Business logic (processClaimEvent)

**Size:** ~130 lines  
**Purpose:** Consume and process claims from Kafka

```java
@KafkaListener(topics = "claims-events", groupId = "claims-processor-group")
public void consumeClaim(ClaimEvent claimEvent, ...)
```

---

### 8. **KafkaController.java** (REST API)
- GET /api/kafka/health - Health check
- GET /api/kafka/info - Application info
- POST /api/kafka/send-claim - Send single claim
- POST /api/kafka/send-claims?count=10 - Send multiple

**Size:** ~150 lines  
**Purpose:** HTTP endpoints to interact with Kafka

```bash
curl POST http://localhost:8080/api/kafka/send-claims?count=10
```

---

### 9. **KafkaSpringBootApplication.java** (Main Class)
- @SpringBootApplication annotation
- @EnableKafka annotation
- main() method with SpringApplication.run()

**Size:** ~15 lines  
**Purpose:** Application entry point

---

## 🎯 Complete Architecture

```
┌─────────────────────────────────────────────────────┐
│         Spring Boot Application (Port 8080)         │
├─────────────────────────────────────────────────────┤
│                                                     │
│  REST Controller                                    │
│  ├─ POST /send-claim                               │
│  ├─ POST /send-claims?count=10                     │
│  └─ GET /health, /info                             │
│         ↓                                            │
│  Claims Producer                                    │
│  └─ Sends ClaimEvent to Kafka                      │
│                                                     │
└────────────────┬────────────────────────────────────┘
                 │
                 │ Produces to Topic: claims-events
                 ↓
┌─────────────────────────────────────────────────────┐
│         Kafka Broker (Port 9092)                    │
│  ┌───────────────────────────────────────────────┐  │
│  │ Topic: claims-events (3 partitions)           │  │
│  │ ├─ Partition 0: messages...                   │  │
│  │ ├─ Partition 1: messages...                   │  │
│  │ └─ Partition 2: messages...                   │  │
│  └───────────────────────────────────────────────┘  │
│                                                     │
│  Topic: processed-claims (backup)                  │
└────────────────┬────────────────────────────────────┘
                 │
                 │ Consumes from Topic: claims-events
                 ↓
┌─────────────────────────────────────────────────────┐
│         Spring Boot Consumer (Auto-running)         │
├─────────────────────────────────────────────────────┤
│                                                     │
│  Claims Consumer (3 threads)                        │
│  ├─ Thread 1: Process partition 0 messages        │
│  ├─ Thread 2: Process partition 1 messages        │
│  └─ Thread 3: Process partition 2 messages        │
│         ↓                                            │
│  Process Claim Event                                │
│  ├─ Validate claim                                  │
│  ├─ Log details                                     │
│  └─ Commit offset                                   │
│                                                     │
└─────────────────────────────────────────────────────┘

Message Flow:
1. User calls: POST /send-claims?count=10
2. Controller → Producer → Kafka (claims-events topic)
3. Messages stored in 3 partitions (round-robin)
4. Consumer automatically polls for messages
5. 3 threads process messages in parallel
6. Offsets committed after successful processing
```

---

## 🔄 Producer-Consumer Flow

### Sending a Claim (Producer)

```
REST API
  ↓
POST /api/kafka/send-claims?count=10
  ↓
KafkaController.sendMultipleClaims(10)
  ↓
Create 10 dummy ClaimEvent objects
  ├─ CLAIM-[timestamp]-0
  ├─ CLAIM-[timestamp]-1
  └─ CLAIM-[timestamp]-9
  ↓
ClaimsProducer.sendClaim(claimEvent)
  ├─ Serialize to JSON
  ├─ Key: claim.getClaimId()
  └─ Partition: hash(key) % 3
  ↓
KafkaTemplate.send()
  ├─ Network I/O
  ├─ Kafka broker receives
  └─ Message stored on disk
  ↓
Callback received
  ├─ Partition number
  ├─ Offset number
  └─ Log success
```

### Consuming a Claim (Consumer)

```
Kafka Broker
  ├─ Topic: claims-events
  └─ 3 partitions (0, 1, 2)
  ↓
Consumer polling (every 100ms)
  ├─ Get messages from assigned partitions
  └─ ConsumerRecords<String, ClaimEvent>
  ↓
For each message:
  ├─ Deserialize JSON → ClaimEvent object
  ├─ Get metadata (partition, offset)
  └─ Call listener method
  ↓
@KafkaListener method
  ├─ consumeClaim(claimEvent, partition, offset)
  └─ Log: Received claim from partition X offset Y
  ↓
Business Logic
  ├─ processClaimEvent(claimEvent)
  ├─ Validate amount > 0
  ├─ Simulate processing (100ms)
  └─ Log: Successfully processed
  ↓
Offset Management
  ├─ Auto-commit: offset+1 every 1 second
  ├─ Or manual: acknowledgment.acknowledge()
  └─ Consumer group offset stored
```

---

## 📊 Expected Output

### Application Startup
```
2024-06-05 10:30:15.234  INFO 12345 --- [main] c.c.k.KafkaSpringBootApplication : 
  Starting KafkaSpringBootApplication
2024-06-05 10:30:16.789  INFO 12345 --- [main] o.s.b.w.e.tomcat.TomcatWebServer :
  Tomcat started on port 8080
2024-06-05 10:30:17.234  INFO 12345 --- [main] c.c.k.KafkaSpringBootApplication :
  Started KafkaSpringBootApplication
```

### Sending Claims
```
2024-06-05 10:31:20.100  INFO 12345 --- [main] c.c.k.producer.ClaimsProducer :
  Publishing 10 claim events
2024-06-05 10:31:20.150  INFO 12345 --- [main] c.c.k.producer.ClaimsProducer :
  Publishing claim event: CLAIM-1717579880150-0
2024-06-05 10:31:20.200  INFO 12345 --- [pool-X] c.c.k.producer.ClaimsProducer :
  Claim CLAIM-1717579880150-0 sent successfully to partition 1 offset 0
```

### Consuming Claims
```
2024-06-05 10:31:21.300  INFO 12345 --- [container-0] c.c.k.consumer.ClaimsConsumer :
  Received claim from partition 1 offset 0: CLAIM-1717579880150-0
2024-06-05 10:31:21.310  DEBUG 12345 --- [container-0] c.c.k.consumer.ClaimsConsumer :
  Processing claim: ClaimEvent(claimId=CLAIM-1717579880150-0, memberId=MEMBER-0, ...)
2024-06-05 10:31:21.420  INFO 12345 --- [container-0] c.c.k.consumer.ClaimsConsumer :
  Claim details - Member: MEMBER-0, Amount: 100.0, Status: SUBMITTED
2024-06-05 10:31:21.430  INFO 12345 --- [container-0] c.c.k.consumer.ClaimsConsumer :
  Successfully processed claim: CLAIM-1717579880150-0
```

---

## 🛠️ Common Operations

### Monitor Kafka Topics

```bash
# List all topics
docker exec kafka kafka-topics \
  --list --bootstrap-server localhost:9092

# Show topic details
docker exec kafka kafka-topics \
  --describe \
  --topic claims-events \
  --bootstrap-server localhost:9092

# Output:
# Topic: claims-events
# Partition: 0 Leader: 1 Replicas: [1]
# Partition: 1 Leader: 1 Replicas: [1]
# Partition: 2 Leader: 1 Replicas: [1]
```

### View Messages in Topic

```bash
# Read from beginning
docker exec kafka kafka-console-consumer \
  --topic claims-events \
  --bootstrap-server localhost:9092 \
  --from-beginning \
  --property print.key=true \
  --max-messages 5

# Output:
# CLAIM-1717579880150-0  {"claimId":"CLAIM-...", "memberId":"MEMBER-0", ...}
# CLAIM-1717579880250-1  {"claimId":"CLAIM-...", "memberId":"MEMBER-1", ...}
```

### Check Consumer Group Status

```bash
# List consumer groups
docker exec kafka kafka-consumer-groups \
  --list \
  --bootstrap-server localhost:9092

# Describe group
docker exec kafka kafka-consumer-groups \
  --describe \
  --group claims-processor-group \
  --bootstrap-server localhost:9092

# Output:
# GROUP                    TOPIC         PARTITION  OFFSET  CURRENT-OFFSET  LAG
# claims-processor-group   claims-events 0          10      10              0
# claims-processor-group   claims-events 1          8       8               0
# claims-processor-group   claims-events 2          6       6               0
```

---

## 🎓 Learning Path

1. **Understand Kafka Basics**
   - Topics, partitions, brokers
   - Producers, consumers, offsets
   - Consumer groups

2. **Study the Code**
   - Read ClaimEvent.java (model)
   - Read KafkaConfig.java (configuration)
   - Read ClaimsProducer.java (sending)
   - Read ClaimsConsumer.java (receiving)

3. **Run Locally**
   - Start Kafka with docker-compose
   - Run Spring Boot application
   - Send test claims via REST API
   - Watch logs in console

4. **Modify and Experiment**
   - Change number of partitions
   - Modify claim event structure
   - Add more consumer groups
   - Implement error handling

5. **Advanced Topics**
   - Schema Registry
   - Exactly-once semantics
   - Dead letter queues
   - Consumer lag monitoring

---

## 🔍 Troubleshooting

| Problem | Solution |
|---------|----------|
| **Kafka connection refused** | Verify docker-compose up -d ran, wait 10s |
| **Port 9092 already in use** | Change docker-compose.yml port mapping |
| **Messages not appearing** | Check consumer group offset: kafka-consumer-groups --describe |
| **Spring Boot won't start** | Check port 8080 not in use, Java 17+ installed |
| **Lombok not working** | Ensure IDE annotation processing enabled |
| **Serialization error** | Verify ClaimEvent fields match JSON in request |

---

## 📚 All Files Provided

✅ Complete pom.xml with all dependencies  
✅ Docker-compose.yml for Kafka setup  
✅ All Java source files with complete implementation  
✅ application.yml with optimized Kafka settings  
✅ REST endpoints ready to use  
✅ Comprehensive logging  
✅ Error handling  
✅ Code comments explaining every section  

---

## ✨ Key Features

**Producer:**
- ✅ Asynchronous message sending
- ✅ Idempotent producer (exactly-once)
- ✅ Message compression (snappy)
- ✅ Batching for performance
- ✅ Custom serialization
- ✅ Callback handlers

**Consumer:**
- ✅ Automatic message consumption
- ✅ Parallel processing (3 threads)
- ✅ Offset tracking (auto-commit)
- ✅ Manual acknowledgment support
- ✅ Custom deserialization
- ✅ Error handling

**Operations:**
- ✅ REST API for testing
- ✅ Health check endpoint
- ✅ Dummy data generation
- ✅ Structured logging
- ✅ Configurable via application.yml
- ✅ Docker support

---

## 🚀 Next Steps

1. **Copy all code files** from the guide
2. **Create project structure** matching the paths
3. **Start Kafka** with docker-compose
4. **Build and run** the Spring Boot application
5. **Send test messages** via REST API
6. **Verify logs** showing producer/consumer flow
7. **Monitor Kafka** topics with CLI tools
8. **Extend functionality** (add database, error handling, etc.)

---

## 📖 File Locations

All files are documented in: **SpringBoot_Kafka_Complete_Guide.md**

Markdown file contains:
- Complete code for each file
- Detailed explanations
- Usage examples
- Troubleshooting tips
- Performance tuning
- Architecture diagrams

---

**Status:** ✅ Ready to Run  
**Time to Setup:** ~5 minutes  
**Lines of Code:** ~1,000+ (fully functional)  
**Difficulty:** Beginner to Intermediate  

Start building! 🎉

