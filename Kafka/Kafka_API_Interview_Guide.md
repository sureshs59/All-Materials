# Kafka API Interview Questions & Answers

**Author:** Technical Interview Preparation  
**Date:** June 2026  
**Level:** Beginner to Advanced  
**Focus:** Apache Kafka Producer/Consumer API, Real-World Scenarios

---

## Table of Contents

1. [Kafka Basics](#kafka-basics)
2. [Producer API Questions](#producer-api-questions)
3. [Consumer API Questions](#consumer-api-questions)
4. [Advanced Topics](#advanced-topics)
5. [Real-World Scenarios](#real-world-scenarios)
6. [Performance & Optimization](#performance--optimization)
7. [Security & Error Handling](#security--error-handling)
8. [Code Examples](#code-examples)

---

## Kafka Basics

### Q1: What is Apache Kafka and what problems does it solve?

**Answer:**

Apache Kafka is a **distributed streaming platform** built for high-throughput, low-latency publish-subscribe messaging. It's designed to handle real-time data feeds from multiple sources.

**Key Characteristics:**
- **High Throughput:** Handles millions of messages per second
- **Low Latency:** Real-time message delivery (sub-second)
- **Fault-Tolerant:** Replicates data across multiple brokers
- **Scalable:** Horizontally scalable across clusters
- **Persistent:** Messages stored on disk, not lost on restart
- **Decoupled:** Producers and consumers work independently

**Problems It Solves:**

| Problem | Kafka Solution |
|---------|---|
| Data silos | Unified data pipeline |
| Event streaming | Real-time event distribution |
| Microservice integration | Decoupled communication |
| Legacy system integration | Message-driven architecture |
| Real-time analytics | Streaming data processing |
| Data replication | Multi-datacenter sync |

**Use Cases:**
- Event sourcing
- Stream processing
- Log aggregation
- Metrics collection
- Real-time analytics
- Activity tracking
- IoT data ingestion

**Real-World Example (CareFirst):**
```
Medical Claims Processing Pipeline:
┌──────────────────────────────────────────────────────┐
│ Claims Submission System                             │
│ (Produces: claim.submitted events)                   │
└─────────────┬────────────────────────────────────────┘
              │
              ↓
┌──────────────────────────────────────────────────────┐
│ Kafka Cluster (Topic: claims-events)                 │
│ (Partitions: 10, Replication: 3)                     │
└───────┬──────────────────┬─────────────────┬─────────┘
        │                  │                 │
        ↓                  ↓                 ↓
   Claims          Eligibility Check   Billing System
   Processor       Service              (Produces bills)
   (Consumer 1)    (Consumer 2)
   
Benefits:
✓ Decoupled systems (each can scale independently)
✓ Real-time processing (sub-second latency)
✓ Fault tolerance (replicated data)
✓ Historical replay (stored for 7 days)
✓ Multiple consumers (claims, eligibility, billing)
```

---

### Q2: What is a Topic, Partition, and Broker in Kafka?

**Answer:**

**Topic:**
- Logical channel for data of a specific type
- Like a database table for streaming data
- Producers send messages to topics
- Consumers read from topics
- Example: `claims-submitted`, `user-events`, `sensor-data`

**Partition:**
- Topics are divided into partitions for parallelism
- Each partition is an ordered, immutable sequence of messages
- Messages within a partition have a unique **offset** (position)
- Different partitions can be processed in parallel by different consumers
- Example: Topic `claims-submitted` with 5 partitions (0-4)

```
Topic: claims-submitted (5 partitions)

Partition 0:  [msg0] → [msg5] → [msg10] → [msg15] → ...
              Offset: 0      1       2       3

Partition 1:  [msg1] → [msg6] → [msg11] → ...
              Offset: 0      1       2

Partition 2:  [msg2] → [msg7] → [msg12] → ...
              Offset: 0      1       2

Partition 3:  [msg3] → [msg8] → [msg13] → ...
Partition 4:  [msg4] → [msg9] → [msg14] → ...
```

**Broker:**
- Individual Kafka server in a cluster
- Manages partitions and stores messages
- Typically 3+ brokers in production (for fault tolerance)
- Each partition has a leader broker and replica brokers

```
Kafka Cluster (3 Brokers)

Broker 1:  [Partition 0 - Leader]  [Partition 1 - Replica]
Broker 2:  [Partition 1 - Leader]  [Partition 2 - Replica]
Broker 3:  [Partition 2 - Leader]  [Partition 0 - Replica]

Leader: Handles all reads/writes for a partition
Replica: Backup copy of partition data
```

**Key Relationship:**
```
Cluster
├─ Broker 1
├─ Broker 2
└─ Broker 3
   └─ Topic (claims-submitted)
      ├─ Partition 0 (offset: 0-999)
      ├─ Partition 1 (offset: 0-1050)
      ├─ Partition 2 (offset: 0-890)
      ├─ Partition 3 (offset: 0-1200)
      └─ Partition 4 (offset: 0-1100)
```

---

### Q3: What is an Offset and Consumer Group?

**Answer:**

**Offset:**
- Integer ID representing the position in a partition
- Each message in a partition has a unique offset
- Starts at 0 for the first message
- Consumers track which offset they've read up to

```
Partition 0 with Offsets:
┌──────────────────────────────────────────┐
│ Offset │ Message      │ Timestamp         │
├──────────────────────────────────────────┤
│ 0      │ claim_1001   │ 2026-06-05 10:00 │
│ 1      │ claim_1002   │ 2026-06-05 10:01 │
│ 2      │ claim_1003   │ 2026-06-05 10:02 │
│ 3      │ claim_1004   │ 2026-06-05 10:03 │
│ 4      │ claim_1005   │ 2026-06-05 10:04 │
└──────────────────────────────────────────┘

Consumer A reads up to offset 3 (current position: 4)
Consumer B reads up to offset 1 (current position: 2)
```

**Consumer Group:**
- Set of consumers that work together to consume from a topic
- Each partition is consumed by only ONE consumer in a group
- Enables parallel processing and load balancing
- Example: Consumer Group `claims-processing` with 3 consumers

```
Topic: claims-submitted (5 partitions)
Consumer Group: claims-processing (3 consumers)

Partition Assignment:
├─ Consumer 1: Partitions 0, 1
├─ Consumer 2: Partitions 2, 3
└─ Consumer 3: Partition 4

Benefits:
✓ Parallel processing (each consumer owns partitions)
✓ Automatic rebalancing (if consumer goes down)
✓ Load balancing across consumers
✓ Exactly-once semantics (per partition)

Multiple Consumer Groups on Same Topic:
┌─────────────────────────────────┐
│ Topic: claims-submitted         │
├─────────────────────────────────┤
│ Consumer Group 1: Processing    │
│ Consumer Group 2: Analytics     │
│ Consumer Group 3: Compliance    │
└─────────────────────────────────┘

Each group independently tracks offsets
Multiple independent data flows from same topic
```

---

## Producer API Questions

### Q4: What is a Kafka Producer and what does it do?

**Answer:**

A Kafka Producer is an **application/service that sends messages to Kafka topics**. It's responsible for:

1. **Message Creation** — Prepare data to send
2. **Serialization** — Convert objects to bytes
3. **Partitioning** — Determine which partition receives message
4. **Batching** — Group messages for efficiency
5. **Compression** — Reduce message size
6. **Sending** — Transmit to broker
7. **Acknowledgment** — Confirm delivery

**Producer Configuration:**

| Configuration | Purpose | Example |
|---|---|---|
| `bootstrap.servers` | Broker addresses | `localhost:9092` |
| `key.serializer` | Key serialization | `StringSerializer` |
| `value.serializer` | Value serialization | `JsonSerializer` |
| `acks` | Acknowledgment level | `all` (most reliable) |
| `retries` | Failed send attempts | `3` |
| `batch.size` | Message batch size | `16384` bytes |
| `linger.ms` | Wait time before send | `10` ms |
| `compression.type` | Compression algorithm | `snappy` |

**Producer Workflow:**

```
Application Code
       │
       ├─ Create ProducerRecord
       │  (topic, key, value)
       │
       ↓
Serializer
       │
       ├─ Serialize key
       ├─ Serialize value
       │
       ↓
Partitioner
       │
       ├─ Select partition:
       │  - If key provided: hash(key) % num_partitions
       │  - If no key: round-robin
       │
       ↓
Accumulator (Batching)
       │
       ├─ Group messages
       ├─ Compress if needed
       │
       ↓
Send to Broker
       │
       ├─ Network I/O
       │
       ↓
Broker Acknowledgment
       │
       ├─ acks=0: No wait
       ├─ acks=1: Leader ack
       ├─ acks=all: All replicas ack
       │
       ↓
Callback/Future
       │
       └─ Success or Exception
```

**Real-World Example (CareFirst Claims):**

```java
// Produce claim events to Kafka
ProducerRecord<String, ClaimEvent> record = 
    new ProducerRecord<>(
        "claims-submitted",              // topic
        claim.getClaimId(),              // key (partition by claim ID)
        new ClaimEvent(claim)            // value
    );

producer.send(record, new Callback() {
    public void onCompletion(RecordMetadata metadata, 
                            Exception exception) {
        if (exception == null) {
            logger.info("Claim sent to partition {} offset {}", 
                       metadata.partition(), 
                       metadata.offset());
        } else {
            logger.error("Failed to send claim", exception);
        }
    }
});
```

---

### Q5: Explain Producer Acknowledgment (acks) and its trade-offs

**Answer:**

The `acks` configuration determines how many brokers must acknowledge receipt before the producer considers a message sent successfully.

**Three Options:**

| acks | Behavior | Reliability | Latency | Use Case |
|---|---|---|---|---|
| **0** | Fire and forget | Low (data loss possible) | Fastest | Non-critical logs |
| **1** | Leader only | Medium (single broker failure = loss) | Medium | Most applications |
| **all** | All replicas | High (fault-tolerant) | Slowest | Critical data (payments, claims) |

**Detailed Comparison:**

```
acks=0 (No Acknowledgment)
┌──────────────────────────────────────────┐
│ Producer                                  │
│ send() ────────────────────────────► Broker
│         └─ Returns immediately            │
│            (before broker processes)      │
└──────────────────────────────────────────┘

Risk: Broker might crash before writing to disk
Use: Non-critical metrics, logs
Throughput: ~1M messages/second
Example: Application logs, analytics

acks=1 (Leader Acknowledgment)
┌──────────────────────────────────────────┐
│ Producer                                  │
│ send() ────────────────────────► Broker 1 (Leader)
│         ◄────── Acknowledges    │
│         └─ Returns after ack     │
│            (message on disk)     │
│                               │
│                        ┌──────────────┐
│                        │ Replica Brokers
│                        │ (async copy)
│                        └──────────────┘
└──────────────────────────────────────────┘

Risk: Broker crash before replica sync = loss
Use: Most use cases
Throughput: ~500K messages/second
Example: User events, claims (moderate durability)

acks=all (All Replicas)
┌──────────────────────────────────────────┐
│ Producer                                  │
│ send() ────────────────────────► Broker 1 (Leader)
│                                  │
│                     ┌────────────┼────────────┐
│                     ↓            ↓            ↓
│                  Broker 2     Broker 3     Broker 4
│                  (Replica)   (Replica)    (Replica)
│                     │            │            │
│                     └────────────┼────────────┘
│                                  │
│ Returns ◄─ All acked ──────────┘
│
└──────────────────────────────────────────┘

Risk: None (full replication)
Guarantee: Message persists even with broker failures
Use: Critical data
Throughput: ~100K messages/second
Example: Financial transactions, medical claims, legal docs

Configuration:
acks=all + min.insync.replicas=2 (requires 2+ replicas acknowledge)
```

**Real-World Scenario (CareFirst):**

```java
// Configuration for Claims (critical data)
Properties props = new Properties();
props.put("bootstrap.servers", "kafka-broker-1:9092");
props.put("acks", "all");                    // All replicas must ack
props.put("min.insync.replicas", 2);         // Min 2 replicas ack
props.put("retries", 3);                     // Retry failed sends
props.put("compression.type", "snappy");     // Compress
props.put("batch.size", 32768);              // 32KB batches
props.put("linger.ms", 100);                 // Wait 100ms for batch

KafkaProducer<String, ClaimEvent> producer = 
    new KafkaProducer<>(props, 
                        new StringSerializer(),
                        new JsonSerializer<>());

// Send claim with high reliability
Future<RecordMetadata> future = producer.send(
    new ProducerRecord<>("claims-submitted", 
                        claimId, 
                        claimEvent),
    (metadata, exception) -> {
        if (exception != null) {
            // acks=all + retries ensure we know about failures
            logger.error("Critical: Claim send failed after retries", exception);
            // Take action: retry, alert, dead letter queue
        }
    }
);

// Wait for completion (synchronous for critical messages)
try {
    RecordMetadata metadata = future.get();
    logger.info("Claim {} written to partition {} offset {}", 
               claimId, 
               metadata.partition(), 
               metadata.offset());
} catch (Exception e) {
    logger.error("Claim processing failed", e);
    // Handle failure appropriately
}
```

---

### Q6: What is Message Key and Partition Selection in Producers?

**Answer:**

**Message Key:**
- Optional field in ProducerRecord
- Used to determine which partition receives the message
- All messages with same key go to same partition (ordering guarantee)
- If no key: round-robin across partitions

**Partition Selection Logic:**

```java
// If Key is NULL
partition = nextPartition(num_partitions)  // Round-robin

// If Key is NOT NULL
partition = hash(key) % num_partitions
// All messages with same key → Same partition

Example:
Key: "CLAIM-123" → hash("CLAIM-123") = 47
47 % 5 partitions = 2
Message goes to Partition 2

Key: "CLAIM-125" → hash("CLAIM-125") = 51
51 % 5 partitions = 1
Message goes to Partition 1

Key: "CLAIM-456" → hash("CLAIM-456") = 47
47 % 5 partitions = 2
Message goes to Partition 2 (same as CLAIM-123)
```

**Practical Implications:**

| Key Usage | Behavior | Use Case |
|---|---|---|
| **WITH Key** | Ordered by key | User events (same user, same partition) |
| **WITH Key** | Load balanced | Event sourcing (entity ID as key) |
| **WITH Key** | Partition locked | State tracking (order matters) |
| **NO Key** | Load balanced equally | Logs, metrics (order doesn't matter) |
| **NO Key** | Round-robin | Fire-and-forget messages |

**Real-World Examples:**

```java
// Example 1: CareFirst Claims (ordered by Claim ID)
ProducerRecord<String, ClaimEvent> record = 
    new ProducerRecord<>(
        "claims-submitted",
        claim.getClaimId(),           // KEY: "CLM-12345"
        claimEvent                    // VALUE: Full claim data
    );
// All updates for CLM-12345 go to same partition
// Ensures processing order: submitted → verified → approved

// Example 2: User Activity (ordered by User ID)
ProducerRecord<String, UserEvent> record = 
    new ProducerRecord<>(
        "user-events",
        userId,                       // KEY: "USER-999"
        userEvent                     // VALUE: Activity details
    );
// All events for USER-999 in order
// Important for: session tracking, state management

// Example 3: Sensor Data (no key, just measure)
ProducerRecord<String, SensorReading> record = 
    new ProducerRecord<>(
        "sensor-readings",
        null,                         // NO KEY: random partition
        sensorReading                 // VALUE: Temperature, pressure, etc.
    );
// High throughput, order doesn't matter
// Good for: metrics, logs, analytics

// Example 4: Custom Partitioner
class CustomPartitioner implements Partitioner {
    public int partition(String topic, Object key, byte[] keyBytes,
                        Object value, byte[] valueBytes,
                        Cluster cluster) {
        if (key == null) return 0;
        
        String keyStr = (String) key;
        if (keyStr.startsWith("PREMIUM_")) {
            return 0;  // Premium claims to partition 0
        } else if (keyStr.startsWith("STANDARD_")) {
            return 1;  // Standard claims to partition 1
        }
        return 2;      // Other to partition 2
    }
}
```

---

### Q7: Explain Idempotent Producer and Exactly-Once Semantics

**Answer:**

**Problem:**
Producers can fail after sending but before receiving acknowledgment, leading to duplicate messages.

```
Producer Failure Scenario:

Producer sends message → Broker writes → Broker sends ack → Producer crashes
                         ✓ Message saved    ✗ Ack never received

Producer retries → Message written again (DUPLICATE!)
```

**Idempotent Producer Solution:**

Kafka automatically deduplicates messages using:
1. **Producer ID (PID)** — Unique producer instance ID
2. **Sequence Number** — Counter for each message
3. **Deduplication Window** — ~5 minutes by default

```
Message 1: [PID: 5, Seq: 0] → Stored
Message 2: [PID: 5, Seq: 1] → Stored
Message 1: [PID: 5, Seq: 0] → DUPLICATE! → Rejected
Message 3: [PID: 5, Seq: 2] → Stored

Broker tracks: PID 5 received sequences 0, 1, 2
Duplicate 0 ignored automatically
```

**Configuration:**

```java
Properties props = new Properties();
props.put("enable.idempotence", true);  // Enable exactly-once
props.put("acks", "all");               // Required
props.put("retries", Integer.MAX_VALUE); // Unlimited
props.put("max.in.flight.requests.per.connection", 5); // Order preserved

KafkaProducer<String, String> producer = 
    new KafkaProducer<>(props);
```

**Exactly-Once Processing:**

```
Without Idempotence (At-Least-Once):
Send → Fail → Retry → DUPLICATE possible

With Idempotence (Exactly-Once):
Send → Fail → Retry → Broker deduplicates → No duplicate

Producer Side:
├─ enable.idempotence: true
├─ acks: all
├─ retries: unlimited
└─ max.in.flight.requests: 5 (maintain order)

Broker Side:
├─ Track Producer ID + Sequence Number
├─ Reject duplicates
└─ Guarantee single write per sequence

Semantic Guarantee:
├─ Exactly-once per message
├─ Per-partition guarantee
└─ Order preserved within partition
```

**Real-World Example (CareFirst Financial Transactions):**

```java
// Critical: Must process each transaction exactly once
Properties props = new Properties();
props.put("bootstrap.servers", "kafka:9092");
props.put("enable.idempotence", true);
props.put("acks", "all");
props.put("retries", Integer.MAX_VALUE);

KafkaProducer<String, Transaction> producer = 
    new KafkaProducer<>(props);

Transaction payment = new Transaction(
    transactionId,
    amount,
    "TRANSFER"
);

// Send payment event
producer.send(
    new ProducerRecord<>("financial-transactions", 
                         transactionId, 
                         payment)
);

// Even if network fails and we retry:
// Broker ensures exactly one write
// No double-charge!
```

**Important Notes:**
- ✅ Prevents **producer duplicates** (network/timeout issues)
- ❌ Does NOT prevent **consumer duplicates** (separate issue)
- ⚠️ Slight latency cost (tracking overhead)
- ✅ Default enabled in modern Kafka (3.0+)

---

## Consumer API Questions

### Q8: What is a Kafka Consumer and how does it work?

**Answer:**

A Kafka Consumer is an **application that reads messages from Kafka topics**. It processes streaming data and maintains state about progress.

**Consumer Workflow:**

```
1. Consumer Application
   ├─ Create KafkaConsumer
   ├─ Subscribe to topic(s)
   ├─ Poll for messages
   ├─ Process messages
   ├─ Commit offsets
   └─ Handle rebalancing

2. Consumer Group
   ├─ Multiple consumers work together
   ├─ Each partition consumed by one consumer
   ├─ Automatic rebalancing if consumer fails
   └─ Shared offset tracking
```

**Consumer Configuration:**

| Config | Purpose | Default | Example |
|---|---|---|---|
| `bootstrap.servers` | Broker addresses | N/A | `localhost:9092` |
| `group.id` | Consumer group | N/A | `claims-processor` |
| `key.deserializer` | Key format | N/A | `StringDeserializer` |
| `value.deserializer` | Value format | N/A | `JsonDeserializer` |
| `auto.offset.reset` | Starting offset | `latest` | `earliest`, `none` |
| `enable.auto.commit` | Offset management | `true` | `false` (manual) |
| `auto.commit.interval.ms` | Commit frequency | `5000` | `10000` |
| `session.timeout.ms` | Heartbeat timeout | `10000` | `30000` |
| `max.poll.records` | Batch size | `500` | `100` |

**Consumer Types:**

| Type | Configuration | When Starts | Use Case |
|---|---|---|---|
| **Earliest** | `earliest` | First message ever | Replay history, testing |
| **Latest** | `latest` | Most recent message | Real-time processing |
| **Last Committed** | Default | Last committed offset | Resume from checkpoint |

**Real-World Example (CareFirst):**

```java
Properties props = new Properties();
props.put("bootstrap.servers", "kafka:9092");
props.put("group.id", "claims-processor");
props.put("key.deserializer", "StringDeserializer");
props.put("value.deserializer", "JsonDeserializer");
props.put("auto.offset.reset", "earliest");
props.put("enable.auto.commit", false);  // Manual commit

KafkaConsumer<String, ClaimEvent> consumer = 
    new KafkaConsumer<>(props);

consumer.subscribe(Arrays.asList("claims-submitted"));

while (true) {
    // Poll for records (timeout: 100ms)
    ConsumerRecords<String, ClaimEvent> records = 
        consumer.poll(Duration.ofMillis(100));
    
    for (ConsumerRecord<String, ClaimEvent> record : records) {
        try {
            // Process claim
            processClaimEvent(record.value());
            
            // Commit offset after successful processing
            consumer.commitSync();  // Blocking commit
            
        } catch (Exception e) {
            logger.error("Failed to process claim", e);
            // Don't commit on failure (will retry)
        }
    }
}
```

---

### Q9: Explain Manual Offset Management vs Auto Commit

**Answer:**

Two strategies for tracking consumer progress:

**Auto Commit (Default):**
```
enable.auto.commit: true
auto.commit.interval.ms: 5000

Workflow:
├─ Consumer polls messages
├─ Application processes
├─ Every 5 seconds: Kafka auto-commits offset
└─ Offset stored in __consumer_offsets topic

Advantages:
✓ Simple (automatic)
✓ No code needed
✓ Less code complexity

Disadvantages:
✗ May lose messages (commit before processing done)
✗ May reprocess messages (processing fails after commit)
✗ No control over timing
```

**Manual Commit (Recommended):**
```
enable.auto.commit: false

Types of Manual Commit:

1. commitSync() - Blocking
   └─ Waits until offset committed
      ├─ Guaranteed commitment before continuing
      ├─ Slower
      └─ Use for: Critical data where order/delivery matters

2. commitAsync() - Non-blocking
   └─ Returns immediately, commits in background
      ├─ Faster
      ├─ May lose messages if consumer crashes
      └─ Use for: High throughput where some loss acceptable

3. commitAsync with Callback
   └─ Non-blocking with callback when complete
      ├─ Best of both worlds
      ├─ Fire-and-forget with notification
      └─ Use for: Most production scenarios
```

**Comparison:**

| Aspect | Auto Commit | commitSync() | commitAsync() |
|---|---|---|---|
| **Control** | None | Full | Full |
| **Speed** | Medium | Slow | Fast |
| **Reliability** | Low | High | Medium |
| **Code** | Simple | Verbose | Verbose |
| **Duplicate Risk** | High | Low | Medium |
| **Loss Risk** | High | Low | Low |

**Code Examples:**

```java
// AUTO COMMIT (Simple but risky)
props.put("enable.auto.commit", true);
props.put("auto.commit.interval.ms", 5000);

while (true) {
    ConsumerRecords<String, ClaimEvent> records = 
        consumer.poll(Duration.ofMillis(100));
    
    for (ConsumerRecord<String, ClaimEvent> record : records) {
        processClaimEvent(record.value());
        // Offset auto-committed every 5 seconds
        // PROBLEM: What if processing fails?
    }
}

// MANUAL COMMIT SYNC (Safe but slow)
props.put("enable.auto.commit", false);

while (true) {
    ConsumerRecords<String, ClaimEvent> records = 
        consumer.poll(Duration.ofMillis(100));
    
    for (ConsumerRecord<String, ClaimEvent> record : records) {
        try {
            processClaimEvent(record.value());
            consumer.commitSync();  // Blocks until committed
        } catch (Exception e) {
            logger.error("Failed to process", e);
            // Don't commit - will retry from same offset
        }
    }
}

// MANUAL COMMIT ASYNC (Fast and flexible)
props.put("enable.auto.commit", false);

while (true) {
    ConsumerRecords<String, ClaimEvent> records = 
        consumer.poll(Duration.ofMillis(100));
    
    for (ConsumerRecord<String, ClaimEvent> record : records) {
        try {
            processClaimEvent(record.value());
            
            // Async commit with callback
            consumer.commitAsync(new OffsetCommitCallback() {
                public void onComplete(Map<TopicPartition, OffsetAndMetadata> offsets,
                                      Exception exception) {
                    if (exception == null) {
                        logger.debug("Offset committed: {}", offsets);
                    } else {
                        logger.error("Commit failed", exception);
                        // Will retry on next poll
                    }
                }
            });
            
        } catch (Exception e) {
            logger.error("Failed to process", e);
        }
    }
}

// HYBRID APPROACH (Best practice)
while (true) {
    ConsumerRecords<String, ClaimEvent> records = 
        consumer.poll(Duration.ofMillis(100));
    
    for (ConsumerRecord<String, ClaimEvent> record : records) {
        try {
            // Process message
            ClaimResult result = processClaimEvent(record.value());
            
            if (result.isSuccess()) {
                // Async commit for processed messages
                consumer.commitAsync();
            } else {
                // Fail - don't commit, will retry
                logger.warn("Failed to process, will retry");
            }
            
        } catch (Exception e) {
            // Critical error - don't commit
            logger.error("Critical error", e);
        }
    }
    
    // Periodic sync commit as backup
    if (System.currentTimeMillis() % 60000 == 0) {
        try {
            consumer.commitSync();  // Every 60 seconds
        } catch (Exception e) {
            logger.error("Sync commit failed", e);
        }
    }
}
```

**Decision Guide:**

```
Simple, non-critical data?
└─ Use: Auto Commit

Critical financial data?
└─ Use: Commit Sync

High throughput, some loss OK?
└─ Use: Commit Async

Production standard?
└─ Use: Commit Async with Callback
```

---

### Q10: What is Consumer Rebalancing and how does it work?

**Answer:**

**What is Rebalancing?**

Process of redistributing partitions among consumers in a group when membership changes.

**When Rebalancing Happens:**

1. New consumer joins group
2. Consumer leaves/crashes
3. Consumer becomes unresponsive (heartbeat timeout)
4. Topic partition count changes

**Rebalancing Process:**

```
Initial State:
Topic: claims (3 partitions)
Group: processors (2 consumers)

├─ Consumer 1: Partitions 0, 1
└─ Consumer 2: Partition 2

Event: Consumer 3 joins
       ↓
Step 1: Stop consumption
       ├─ All consumers stop
       ├─ Commit current offsets
       └─ Session timeout starts

Step 2: Rebalance
       ├─ Assign new partition distribution
       ├─ Use configured strategy (RoundRobin, Range, Sticky)
       └─ Notify consumers of new assignments

Final State:
├─ Consumer 1: Partitions 0
├─ Consumer 2: Partition 1
└─ Consumer 3: Partition 2

Step 3: Resume
       ├─ Seek to committed offset
       ├─ Resume message processing
       └─ Back to normal
```

**Rebalancing Strategies:**

| Strategy | Distribution | Use Case |
|---|---|---|
| **Range** | Contiguous partitions | Simple, but uneven load |
| **RoundRobin** | Even distribution | Most cases |
| **Sticky** | Minimize movement | Reduce rebalancing cost |
| **Cooperative** | Incremental rebalance | Reduce stop-the-world time |

**Example Code:**

```java
Properties props = new Properties();
props.put("group.id", "claims-processor");
props.put("partition.assignment.strategy", 
          "org.apache.kafka.clients.consumer.RoundRobinAssignor");

// Listen for rebalancing events
consumer.subscribe(
    Arrays.asList("claims-submitted"),
    new ConsumerRebalanceListener() {
        @Override
        public void onPartitionsRevoked(Collection<TopicPartition> partitions) {
            logger.info("Revoked partitions: {}", partitions);
            // Clean up before losing partitions
            for (TopicPartition partition : partitions) {
                closePartitionResources(partition);
            }
            // Commit offsets before rebalance
            consumer.commitSync();
        }
        
        @Override
        public void onPartitionsAssigned(Collection<TopicPartition> partitions) {
            logger.info("Assigned partitions: {}", partitions);
            // Initialize for new partitions
            for (TopicPartition partition : partitions) {
                initializePartitionResources(partition);
            }
        }
    }
);
```

**Rebalancing Issues:**

```
Problem: Stop-the-World
├─ All consumers pause during rebalance
├─ Can take seconds to minutes
└─ No messages processed during rebalance

Solution: Cooperative Rebalancing
├─ Only affected consumers stop
├─ Others continue processing
└─ Reduces downtime

Configuration:
props.put("partition.assignment.strategy",
          "org.apache.kafka.clients.consumer.CooperativeStickyAssignor");
```

---

## Advanced Topics

### Q11: What is Schema Registry and why is it important?

**Answer:**

**Purpose:**
Schema Registry is a **centralized repository for managing data schemas** used in Kafka messages.

**Problem it Solves:**

```
Without Schema Registry:
├─ Producer: "Message has fields: name, age"
├─ Consumer 1: "I expect fields: name, email"
├─ Consumer 2: "I expect fields: id, status"
└─ Result: Deserialization errors, data loss

With Schema Registry:
├─ All producers/consumers use same schema
├─ Schema versioning and evolution
├─ Validation before sending
└─ Error detection early
```

**Key Features:**

```
1. Centralized Schema Storage
   └─ Single source of truth
   
2. Schema Versioning
   ├─ Version 1: {name, age}
   ├─ Version 2: {name, age, email}
   └─ Version 3: {name, age, email, phone}

3. Compatibility Checking
   ├─ BACKWARD: New consumer can read old data
   ├─ FORWARD: Old consumer can read new data
   └─ FULL: Both backward and forward

4. Serialization Integration
   ├─ Avro (column-oriented, efficient)
   ├─ Protobuf (typed, compact)
   └─ JSON (human-readable)
```

**Example Usage:**

```java
// Schema Definition (Avro)
{
  "type": "record",
  "name": "ClaimEvent",
  "fields": [
    {"name": "claimId", "type": "string"},
    {"name": "memberId", "type": "string"},
    {"name": "amount", "type": "double"},
    {"name": "status", "type": "string"},
    {"name": "timestamp", "type": "long"}
  ]
}

// Producer Code
Properties props = new Properties();
props.put("value.subject.name.strategy",
          "io.confluent.kafka.serializers.subject.RecordNameStrategy");

KafkaProducer<String, ClaimEvent> producer = 
    new KafkaProducer<>(props, 
                        new StringSerializer(),
                        new KafkaAvroSerializer()); // Auto-validates schema

// Send
producer.send(new ProducerRecord<>(
    "claims",
    claimId,
    new ClaimEvent(claimId, memberId, amount, status, timestamp)
));

// Consumer Code
KafkaConsumer<String, ClaimEvent> consumer = 
    new KafkaConsumer<>(props,
                        new StringDeserializer(),
                        new KafkaAvroDeserializer()); // Auto-deserializes with schema

consumer.subscribe(Arrays.asList("claims"));

for (ConsumerRecord<String, ClaimEvent> record : records) {
    ClaimEvent event = record.value();
    // Type-safe access guaranteed by schema
    processClaimEvent(event);
}
```

---

### Q12: Explain Exactly-Once Semantics (EOS) in Kafka

**Answer:**

**Challenge:**
In distributed systems, messages can be processed multiple times due to failures.

```
Scenario:
1. Message received by consumer
2. Message processed successfully
3. Consumer crashes before committing offset
4. Consumer restarts and reprocesses same message

Result: DUPLICATE PROCESSING
```

**Kafka EOS Solution:**

Kafka provides **Exactly-Once Semantics (EOS)** through atomic writes and transactions.

**How It Works:**

```
Configuration:
processing.guarantee: exactly_once_v2
isolation.level: read_committed

Mechanism:
├─ Producer: Idempotent producer enabled
├─ Consumer: Transactional read and offset commit
├─ Broker: Atomic multi-partition writes
└─ Result: No duplicates, no data loss
```

**Implementation:**

```java
// Producer: Exactly-once writes
Properties producerProps = new Properties();
producerProps.put("enable.idempotence", true);
producerProps.put("acks", "all");
producerProps.put("transactional.id", "claims-processor-1");

KafkaProducer<String, ClaimEvent> producer = 
    new KafkaProducer<>(producerProps);

// Begin transaction
producer.beginTransaction();

try {
    // All writes are atomic
    for (ClaimEvent event : events) {
        producer.send(new ProducerRecord<>(
            "processed-claims",
            event.getId(),
            event
        ));
    }
    
    // Commit all writes atomically
    producer.commitTransaction();
    
} catch (Exception e) {
    // All writes rolled back
    producer.abortTransaction();
    throw e;
}

// Consumer: Exactly-once reads
Properties consumerProps = new Properties();
consumerProps.put("isolation.level", "read_committed");
consumerProps.put("enable.auto.commit", false);

KafkaConsumer<String, ClaimEvent> consumer = 
    new KafkaConsumer<>(consumerProps);

while (true) {
    ConsumerRecords<String, ClaimEvent> records = 
        consumer.poll(Duration.ofMillis(100));
    
    Map<TopicPartition, OffsetAndMetadata> offsets = 
        new HashMap<>();
    
    try {
        for (ConsumerRecord<String, ClaimEvent> record : records) {
            // Process message
            processClaimEvent(record.value());
            
            // Track offset for atomic commit
            offsets.put(
                new TopicPartition(record.topic(), record.partition()),
                new OffsetAndMetadata(record.offset() + 1)
            );
        }
        
        // Atomic: Process + Offset commit together
        consumer.commitSync(offsets);
        
    } catch (Exception e) {
        // Don't commit offsets - will reprocess
        logger.error("Failed to process records", e);
    }
}
```

**Guarantees:**

```
Exactly-Once Semantics provides:

1. No Message Loss
   ├─ All messages written to brokers
   ├─ Replicated across brokers
   └─ Survive any single broker failure

2. No Message Duplication
   ├─ Idempotent producers prevent duplicates
   ├─ Deduplication window (~5 minutes)
   └─ Even with producer retries

3. Ordered Processing
   ├─ Messages within partition ordered
   ├─ Consumer processes in order
   └─ Single consumer per partition

4. Atomic State Management
   ├─ Data written atomically
   ├─ Offset committed atomically
   └─ All-or-nothing semantics
```

---

### Q13: What are Consumer Lags and how do you monitor them?

**Answer:**

**Consumer Lag Definition:**

The difference between the latest offset in a partition and the consumer's current offset.

```
Partition 0 Latest Offset: 1000
Consumer Current Offset: 975
─────────────────────────────
Consumer Lag: 25 messages

Higher lag = Consumer falling behind
Lag = 0 = Consumer caught up
```

**Monitoring Consumer Lag:**

```java
// Method 1: Programmatic Monitoring
public Map<TopicPartition, Long> getConsumerLag(KafkaConsumer<?, ?> consumer) {
    Map<TopicPartition, Long> lags = new HashMap<>();
    
    for (TopicPartition tp : consumer.assignment()) {
        // Get end offset (latest)
        consumer.seekToEnd(tp);
        long endOffset = consumer.position(tp);
        
        // Get committed offset (current position)
        OffsetAndMetadata committed = consumer.committed(tp);
        long currentOffset = committed != null ? 
            committed.offset() : 0;
        
        long lag = endOffset - currentOffset;
        lags.put(tp, lag);
        
        logger.info("Topic: {}, Partition: {}, Lag: {}",
                   tp.topic(), tp.partition(), lag);
    }
    
    return lags;
}

// Method 2: Kafka CLI Command
// ./kafka-consumer-groups.sh --bootstrap-server localhost:9092 \
//    --group claims-processor \
//    --describe

// Output:
// GROUP                TOPIC      PARTITION  LAG  CURRENT-OFFSET  LOG-END-OFFSET
// claims-processor     claims     0          50   950             1000
// claims-processor     claims     1          0    1000            1000
// claims-processor     claims     2          150  800             950
```

**Why Monitor Lag?**

```
Low/Zero Lag (Good):
├─ Consumer keeping up
├─ Real-time processing
└─ SLA targets met

High Lag (Warning):
├─ Consumer falling behind
├─ Messages queuing up
├─ Delays in processing
└─ May indicate consumer slow or broken

Growing Lag (Alarm):
├─ Consumer can't keep up with producer rate
├─ Exponential growth = problem
├─ Needs scaling or optimization
└─ May miss SLA
```

**Lag Monitoring Strategy:**

```java
// Continuous lag monitoring
ScheduledExecutorService executor = 
    Executors.newScheduledThreadPool(1);

executor.scheduleAtFixedRate(() -> {
    Map<TopicPartition, Long> lags = 
        getConsumerLag(consumer);
    
    lags.forEach((tp, lag) -> {
        // Store metrics
        metricsRegistry.meter("kafka.consumer.lag",
                             "topic", tp.topic(),
                             "partition", tp.partition())
                       .mark(lag);
        
        // Alert if lag too high
        if (lag > MAX_ACCEPTABLE_LAG) {
            alerting.sendAlert("High consumer lag detected: " + lag);
        }
    });
}, 0, 30, TimeUnit.SECONDS);
```

**Real-World Thresholds (CareFirst):**

```
Critical Data (Claims, Payments):
├─ Warning: Lag > 100 messages
├─ Alert: Lag > 500 messages
└─ Action: Scale consumer group

Non-Critical Data (Analytics, Logs):
├─ Warning: Lag > 10,000 messages
├─ Alert: Lag > 50,000 messages
└─ Action: Add consumer instances
```

---

## Real-World Scenarios

### Q14: Design a Kafka-based Event Sourcing System (CareFirst Claims)

**Answer:**

**Requirements:**
- Process medical claims in real-time
- Track all changes (audit trail)
- Support multiple consumers (processing, reporting, compliance)
- Guarantee no data loss
- Enable event replay

**Architecture:**

```
┌──────────────────────────────────────────────────┐
│          Claims Submission System                │
│      (e.g., Hospital Billing Portal)             │
└────────────────────┬─────────────────────────────┘
                     │
                     │ Submit Claim
                     ↓
┌──────────────────────────────────────────────────┐
│        Claims Event Stream (Kafka)               │
│                                                  │
│  Topic: claims-events                            │
│  Partitions: 10 (by ClaimID for ordering)       │
│  Replication: 3 (high durability)               │
│  Retention: 30 days (replay capability)         │
└────┬──────────────────┬───────────────┬──────────┘
     │                  │               │
     ↓                  ↓               ↓
┌─────────────┐ ┌──────────────┐ ┌──────────────┐
│   Claims    │ │  Reporting   │ │  Compliance  │
│ Processor   │ │  Engine      │ │  Audit      │
│ Consumer 1  │ │  Consumer 2  │ │  Consumer 3  │
└─────────────┘ └──────────────┘ └──────────────┘
     │                  │               │
     ↓                  ↓               ↓
  Database          Analytics DB    Audit Log
```

**Implementation:**

```java
// 1. Event Definition
public class ClaimEvent {
    public enum EventType {
        CLAIM_SUBMITTED,
        ELIGIBILITY_VERIFIED,
        APPROVED,
        REJECTED,
        PAYMENT_PROCESSED
    }
    
    private String claimId;
    private String memberId;
    private EventType eventType;
    private double amount;
    private String status;
    private long timestamp;
    private Map<String, Object> metadata;
}

// 2. Producer: Submit Claims
public class ClaimsProducer {
    private KafkaProducer<String, ClaimEvent> producer;
    
    public void submitClaim(Claim claim) {
        ClaimEvent event = new ClaimEvent();
        event.setClaimId(claim.getId());
        event.setMemberId(claim.getMemberId());
        event.setEventType(EventType.CLAIM_SUBMITTED);
        event.setAmount(claim.getAmount());
        event.setTimestamp(System.currentTimeMillis());
        
        ProducerRecord<String, ClaimEvent> record = 
            new ProducerRecord<>(
                "claims-events",
                claim.getId(),           // Key: ClaimID (ensures ordering)
                event
            );
        
        producer.send(record, (metadata, exception) -> {
            if (exception == null) {
                logger.info("Claim {} submitted to partition {}",
                           claim.getId(), metadata.partition());
            } else {
                logger.error("Failed to submit claim", exception);
                // Retry logic
            }
        });
    }
}

// 3. Consumer 1: Claims Processor
public class ClaimsProcessor {
    private KafkaConsumer<String, ClaimEvent> consumer;
    private ClaimsRepository claimsDb;
    
    public void process() {
        consumer.subscribe(Arrays.asList("claims-events"));
        
        while (true) {
            ConsumerRecords<String, ClaimEvent> records = 
                consumer.poll(Duration.ofMillis(100));
            
            for (ConsumerRecord<String, ClaimEvent> record : records) {
                ClaimEvent event = record.value();
                
                try {
                    if (event.getEventType() == EventType.CLAIM_SUBMITTED) {
                        // Verify eligibility
                        verifyEligibility(event);
                        
                        // Emit next event
                        emitEvent(event.getClaimId(), 
                                 EventType.ELIGIBILITY_VERIFIED);
                    }
                    
                    consumer.commitSync();
                    
                } catch (Exception e) {
                    logger.error("Failed to process claim", e);
                    // Don't commit - will retry
                }
            }
        }
    }
    
    private void emitEvent(String claimId, EventType type) {
        ClaimEvent nextEvent = new ClaimEvent();
        nextEvent.setClaimId(claimId);
        nextEvent.setEventType(type);
        // ... set other fields
        
        producer.send(new ProducerRecord<>(
            "claims-events",
            claimId,
            nextEvent
        ));
    }
}

// 4. Consumer 2: Reporting Engine
public class ReportingEngine {
    public void generateReports() {
        consumer.subscribe(Arrays.asList("claims-events"));
        
        while (true) {
            ConsumerRecords<String, ClaimEvent> records = 
                consumer.poll(Duration.ofMillis(100));
            
            for (ConsumerRecord<String, ClaimEvent> record : records) {
                ClaimEvent event = record.value();
                
                // Update analytics database
                updateMetrics(event);
                updateDashboard(event);
                
                consumer.commitAsync();
            }
        }
    }
}

// 5. Event Sourcing Benefits
/*
Complete Audit Trail:
├─ Every state change recorded
├─ Who changed it (user ID)
├─ When it changed (timestamp)
└─ Why it changed (reason/metadata)

Event Replay:
├─ Rebuild entire claim state from events
├─ Debug issues (what happened to claim X?)
├─ Time-travel (what was state at time T?)
└─ Test new processors against historical data

Multiple Consumers:
├─ Processor: Handles business logic
├─ Reporter: Generates metrics
├─ Auditor: Compliance tracking
└─ All independent, parallel processing
*/
```

---

### Q15: How do you handle error scenarios and build resilient systems?

**Answer:**

**Common Error Scenarios:**

```
1. Transient Failures
   ├─ Network timeout
   ├─ Broker temporarily unavailable
   └─ Solution: Retry with backoff

2. Permanent Failures
   ├─ Invalid message format
   ├─ Processing logic bug
   └─ Solution: Dead Letter Queue

3. Poison Pills
   ├─ Malformed message
   ├─ Crashes consumer on deserialization
   └─ Solution: Skip and log

4. Slow Consumers
   ├─ Consumer can't keep up
   ├─ Lag grows exponentially
   └─ Solution: Scale horizontally
```

**Error Handling Pattern:**

```java
public class ResilientKafkaConsumer {
    private KafkaConsumer<String, ClaimEvent> consumer;
    private KafkaProducer<String, DeadLetter> dlqProducer;
    private CircuitBreaker circuitBreaker;
    
    public void process() {
        consumer.subscribe(Arrays.asList("claims-events"));
        
        while (true) {
            try {
                ConsumerRecords<String, ClaimEvent> records = 
                    consumer.poll(Duration.ofMillis(100));
                
                for (ConsumerRecord<String, ClaimEvent> record : records) {
                    processWithRetry(record);
                }
                
            } catch (Exception e) {
                logger.error("Unhandled exception", e);
                handleFatalError(e);
            }
        }
    }
    
    private void processWithRetry(ConsumerRecord<String, ClaimEvent> record) {
        int maxRetries = 3;
        int retryCount = 0;
        long backoffMs = 100;
        
        while (retryCount < maxRetries) {
            try {
                // Check circuit breaker before processing
                if (circuitBreaker.isClosed()) {
                    processClaim(record.value());
                    consumer.commitSync();
                    circuitBreaker.recordSuccess();
                    return;
                } else {
                    // Circuit open - send to DLQ
                    sendToDeadLetterQueue(record, "Circuit breaker open");
                    consumer.commitSync();
                    return;
                }
                
            } catch (RetryableException e) {
                // Transient error - retry
                retryCount++;
                
                if (retryCount < maxRetries) {
                    logger.warn("Retrying record {}, attempt {}",
                               record.key(), retryCount);
                    
                    // Exponential backoff
                    Thread.sleep(backoffMs * (long) Math.pow(2, retryCount - 1));
                    circuitBreaker.recordFailure();
                    
                } else {
                    // Max retries exceeded
                    logger.error("Max retries exceeded for record {}", record.key());
                    sendToDeadLetterQueue(record, "Max retries exceeded");
                    consumer.commitSync();
                }
                
            } catch (NonRetryableException e) {
                // Permanent error - don't retry
                logger.error("Non-retryable error for record {}: {}",
                           record.key(), e.getMessage());
                
                sendToDeadLetterQueue(record, e.getMessage());
                consumer.commitSync();
                return;
                
            } catch (Exception e) {
                // Unknown error
                logger.error("Unknown error processing record", e);
                sendToDeadLetterQueue(record, e.getMessage());
                consumer.commitSync();
                return;
            }
        }
    }
    
    private void sendToDeadLetterQueue(
        ConsumerRecord<String, ClaimEvent> record,
        String reason) {
        
        DeadLetter dlq = new DeadLetter();
        dlq.setOriginalRecord(record);
        dlq.setReason(reason);
        dlq.setTimestamp(System.currentTimeMillis());
        
        dlqProducer.send(
            new ProducerRecord<>(
                "claims-events-dlq",
                record.key(),
                dlq
            ),
            (metadata, exception) -> {
                if (exception != null) {
                    logger.error("Failed to send to DLQ", exception);
                    // Escalate alert
                } else {
                    logger.info("Message sent to DLQ: {}", record.key());
                }
            }
        );
    }
    
    private void processClaim(ClaimEvent event) 
        throws RetryableException, NonRetryableException {
        
        try {
            // Validate message
            validateClaimEvent(event);
            
            // Check eligibility (may fail transiently)
            checkEligibility(event);
            
            // Process claim
            saveClaim(event);
            
        } catch (EligibilityServiceUnavailable e) {
            // Transient - service will come back
            throw new RetryableException(e);
            
        } catch (InvalidClaimException e) {
            // Permanent - invalid data
            throw new NonRetryableException(e);
            
        } catch (DatabaseException e) {
            // Likely transient - database may recover
            throw new RetryableException(e);
        }
    }
    
    private void handleFatalError(Exception e) {
        // Consumer crashed - need intervention
        logger.error("FATAL: Consumer crashed", e);
        
        // Send alert
        alerting.sendAlert("Claims consumer failed: " + e.getMessage());
        
        // Could trigger:
        // - Auto-restart
        // - Failover to backup
        // - Page on-call engineer
    }
}

// Circuit Breaker Pattern
public class CircuitBreaker {
    private enum State { CLOSED, OPEN, HALF_OPEN }
    
    private State state = State.CLOSED;
    private int failureCount = 0;
    private int failureThreshold = 5;
    private long lastFailureTime = 0;
    private long resetTimeout = 60000; // 1 minute
    
    public synchronized boolean isClosed() {
        if (state == State.CLOSED) {
            return true;
        }
        
        if (state == State.OPEN) {
            // Check if timeout elapsed
            if (System.currentTimeMillis() - lastFailureTime > resetTimeout) {
                state = State.HALF_OPEN;
                failureCount = 0;
                return true;
            }
            return false;
        }
        
        return true; // HALF_OPEN - try
    }
    
    public synchronized void recordSuccess() {
        if (state == State.HALF_OPEN) {
            state = State.CLOSED;
        }
        failureCount = 0;
    }
    
    public synchronized void recordFailure() {
        failureCount++;
        lastFailureTime = System.currentTimeMillis();
        
        if (failureCount >= failureThreshold) {
            state = State.OPEN;
            logger.warn("Circuit breaker opened");
        }
    }
}
```

**Error Handling Strategy:**

```
┌──────────────────────────────────────┐
│  Incoming Message                    │
└────────────┬─────────────────────────┘
             │
             ↓
      ┌─────────────┐
      │  Validate   │
      └──────┬──────┘
             │
      ┌──────▼──────┐
      │  Invalid?   │───→ Dead Letter Queue
      │             │    (Non-retryable)
      └──────┬──────┘
             │ Valid
             ↓
    ┌─────────────────┐
    │ Circuit Breaker │
    │  Check          │
    └────────┬────────┘
             │
    ┌────────▼─────────┐
    │  Closed?         │───→ Dead Letter Queue
    │                  │    (Circuit open)
    └────────┬─────────┘
             │ Closed
             ↓
    ┌─────────────────┐
    │  Process with   │
    │  Retries        │
    └────────┬────────┘
             │
    ┌────────▼──────────────┐
    │  Success?             │
    └────┬──────────────┬───┘
         │ Yes          │ No
         ↓              ↓
      Commit      Retryable?
      Offset      
                  ┌──────┬──────┐
                  │ Yes  │ No   │
                  ↓      ↓
                Retry  DLQ
```

---

## Performance & Optimization

### Q16: What are best practices for Kafka performance tuning?

**Answer:**

**Producer Tuning:**

```java
Properties props = new Properties();

// 1. Batching & Compression
props.put("batch.size", 32768);          // 32KB batches
props.put("linger.ms", 100);             // Wait 100ms for batch
props.put("compression.type", "snappy"); // Compress with snappy

// Trade-off: Larger batches = better throughput but higher latency
// linger.ms = wait time before sending (accumulate messages)

// 2. Idempotence (slight performance cost)
props.put("enable.idempotence", true);
props.put("acks", "all");

// 3. Async sending (for throughput)
producer.send(record, callback); // Non-blocking

// 4. Connection pooling
props.put("connections.max.idle.ms", 540000); // 9 minutes

// 5. Buffer tuning
props.put("buffer.memory", 33554432);    // 32MB total buffer
props.put("max.in.flight.requests.per.connection", 5); // Pipeline requests

// Result: ~1M messages/second throughput
```

**Consumer Tuning:**

```java
Properties props = new Properties();

// 1. Fetch Size
props.put("fetch.min.bytes", 1024);      // Min 1KB per fetch
props.put("fetch.max.wait.ms", 500);     // Max wait 500ms

// Larger fetch.min.bytes = fewer requests but higher latency
// fetch.max.wait.ms = broker waits for minimum data

// 2. Batch Processing
props.put("max.poll.records", 500);      // Process 500 at a time

// More records = higher throughput but more memory

// 3. Session Management
props.put("session.timeout.ms", 30000);   // 30 second timeout
props.put("heartbeat.interval.ms", 10000);// 10 second heartbeat

// Shorter timeouts = faster failure detection but more overhead

// 4. Offset Commit Strategy
props.put("enable.auto.commit", false);   // Manual for control
// Use commitAsync for throughput, commitSync for safety

// Result: ~500K messages/second throughput
```

**Broker Tuning:**

```
# Server Properties

# 1. Log Retention
log.retention.hours=168        # Keep 7 days
log.segment.bytes=1073741824   # 1GB segments

# 2. Replication
min.insync.replicas=2          # Minimum replicas for acks=all
num.replica.fetchers=4         # Parallel replication

# 3. Network Threads
num.network.threads=8          # Handle network requests
num.io.threads=8               # Handle disk I/O

# 4. Log Flush
log.flush.interval.messages=10000  # Flush every 10K messages
log.flush.interval.ms=1000         # Or every 1 second

# Result: Handle 100K+ messages/second per broker
```

**Monitoring Metrics:**

```java
public class KafkaMetricsMonitor {
    private MetricRegistry metrics;
    
    public void startMonitoring(KafkaProducer<?, ?> producer) {
        // Producer metrics
        metrics.gauge("producer.record-send-rate",
                     () -> producer.metrics()
                                  .values()
                                  .stream()
                                  .map(m -> m.metricName())
                                  .filter(n -> n.name().equals("record-send-rate"))
                                  .findFirst());
        
        metrics.gauge("producer.record-error-rate",
                     () -> getMetric(producer, "record-error-rate"));
        
        metrics.gauge("producer.batch-size-avg",
                     () -> getMetric(producer, "batch-size-avg"));
    }
    
    public void startMonitoring(KafkaConsumer<?, ?> consumer) {
        // Consumer metrics
        metrics.gauge("consumer.lag",
                     () -> getConsumerLag(consumer));
        
        metrics.gauge("consumer.records-consumed-rate",
                     () -> getMetric(consumer, "records-consumed-rate"));
    }
    
    // Thresholds
    private static final int WARN_CONSUMER_LAG = 1000;
    private static final int WARN_ERROR_RATE = 100; // per second
}
```

---

## Security & Error Handling

### Q17: How do you secure Kafka in production?

**Answer:**

**Security Layers:**

```
┌─────────────────────────────────────┐
│  1. Network Security                │
│  ├─ SASL Authentication             │
│  ├─ SSL/TLS Encryption              │
│  └─ IP Whitelisting                 │
├─────────────────────────────────────┤
│  2. Topic/Partition Security        │
│  ├─ ACLs (Access Control Lists)     │
│  ├─ User/Group permissions          │
│  └─ Admin controls                  │
├─────────────────────────────────────┤
│  3. Data Security                   │
│  ├─ Message encryption              │
│  ├─ Data at rest encryption         │
│  └─ Audit logging                   │
├─────────────────────────────────────┤
│  4. Operational Security            │
│  ├─ Monitoring/Alerting             │
│  ├─ Rate limiting                   │
│  └─ Dead letter queues              │
└─────────────────────────────────────┘
```

**Implementation:**

```properties
# Broker Configuration (server.properties)

# 1. SASL/SSL Security
listeners=SASL_SSL://0.0.0.0:9092
advertised.listeners=SASL_SSL://kafka-broker-1:9092
security.inter.broker.protocol=SASL_SSL
sasl.mechanism.inter.broker.protocol=PLAIN
sasl.enabled.mechanisms=PLAIN,SCRAM-SHA-256

# 2. SSL/TLS Certificates
ssl.keystore.location=/path/to/keystore.jks
ssl.keystore.password=keystore-password
ssl.key.password=key-password
ssl.truststore.location=/path/to/truststore.jks
ssl.truststore.password=truststore-password

# 3. ACLs
authorizer.class.name=kafka.security.authorizer.AclAuthorizer
super.users=User:kafka-admin

# 4. Audit Logging
log.message.format.version=2.6
```

```java
// Producer Configuration (Secure)
Properties props = new Properties();
props.put("bootstrap.servers", "kafka-broker-1:9092");
props.put("security.protocol", "SASL_SSL");
props.put("sasl.mechanism", "SCRAM-SHA-256");
props.put("sasl.jaas.config",
    "org.apache.kafka.common.security.scram.ScramLoginModule required " +
    "username=\"producer-user\" " +
    "password=\"producer-password\";");

// SSL/TLS
props.put("ssl.truststore.location", "/path/to/truststore.jks");
props.put("ssl.truststore.password", "password");

KafkaProducer<String, String> producer = 
    new KafkaProducer<>(props);
```

**ACL Management:**

```bash
# List ACLs
./kafka-acls.sh --bootstrap-server localhost:9092 \
  --list

# Grant produce permission
./kafka-acls.sh --bootstrap-server localhost:9092 \
  --create \
  --allow-principal User:producer-user \
  --operation Write \
  --topic claims-events

# Grant consume permission
./kafka-acls.sh --bootstrap-server localhost:9092 \
  --create \
  --allow-principal User:consumer-group \
  --operation Read \
  --topic claims-events \
  --group claims-processor-group
```

---

## Code Examples

### Complete Producer Example

```java
public class SecureClaimsProducer {
    public static void main(String[] args) {
        // Configuration
        Properties props = new Properties();
        props.put("bootstrap.servers", "kafka-broker-1:9092,kafka-broker-2:9092");
        props.put("key.serializer", StringSerializer.class.getName());
        props.put("value.serializer", JsonSerializer.class.getName());
        
        // Reliability
        props.put("acks", "all");
        props.put("retries", Integer.MAX_VALUE);
        props.put("enable.idempotence", true);
        
        // Performance
        props.put("batch.size", 32768);
        props.put("linger.ms", 100);
        props.put("compression.type", "snappy");
        
        // Security
        props.put("security.protocol", "SASL_SSL");
        props.put("sasl.mechanism", "SCRAM-SHA-256");
        props.put("sasl.jaas.config",
            "org.apache.kafka.common.security.scram.ScramLoginModule required " +
            "username=\"producer\" password=\"password\";");
        props.put("ssl.truststore.location", "/path/to/truststore.jks");
        props.put("ssl.truststore.password", "password");
        
        KafkaProducer<String, ClaimEvent> producer = 
            new KafkaProducer<>(props);
        
        // Send claims
        for (int i = 0; i < 100; i++) {
            ClaimEvent event = new ClaimEvent(
                "CLAIM-" + i,
                "MEMBER-" + (i % 10),
                1000.00,
                "SUBMITTED"
            );
            
            producer.send(
                new ProducerRecord<>("claims-events", 
                                   event.getClaimId(), 
                                   event),
                (metadata, exception) -> {
                    if (exception == null) {
                        System.out.println("Claim " + event.getClaimId() + 
                                         " sent to partition " + metadata.partition());
                    } else {
                        System.err.println("Failed to send claim: " + exception);
                    }
                }
            );
        }
        
        producer.flush();
        producer.close();
    }
}
```

---

## Summary: Quick Reference

| Topic | Key Takeaway |
|-------|--------------|
| **Kafka Basics** | Distributed streaming platform for real-time data |
| **Topics & Partitions** | Topics split into ordered partitions for parallelism |
| **Producer** | Send messages, partitioned by key, configurable reliability |
| **Consumer** | Read messages, commit offsets, handle rebalancing |
| **Exactly-Once** | Idempotent producers + manual offset management |
| **Error Handling** | Dead Letter Queues, retries with backoff, circuit breakers |
| **Performance** | Batch, compress, async for throughput |
| **Security** | SASL/SSL, ACLs, audit logging |
| **Monitoring** | Track consumer lag, error rates, throughput |

