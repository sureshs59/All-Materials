# MongoDB & NoSQL Complete Guide

> A comprehensive reference for MongoDB commands and NoSQL concepts — covering Database, Collection, CRUD, Query, Aggregation, Index, Schema Design, and Admin operations.

---

## Table of Contents

1. [NoSQL Core Concepts](#1-nosql-core-concepts)
2. [Database Commands](#2-database-commands)
3. [Collection Commands](#3-collection-commands)
4. [CRUD Operations](#4-crud-operations)
5. [Query Operators](#5-query-operators)
6. [Aggregation Pipeline](#6-aggregation-pipeline)
7. [Index Commands](#7-index-commands)
8. [Schema Design](#8-schema-design)
9. [Admin & Performance](#9-admin--performance)

---

## 1. NoSQL Core Concepts

### What is NoSQL?

NoSQL = Not Only SQL. It refers to a family of database systems that store and retrieve data differently from traditional relational databases.

```
// MySQL (Relational)             MongoDB (Document)
// ─────────────────────          ──────────────────
// Database          →            Database
// Table             →            Collection
// Row               →            Document (JSON/BSON)
// Column            →            Field
// JOIN              →            Embedded docs / $lookup
// Schema is fixed   →            Schema is flexible
// ACID by default   →            ACID from v4.0+
```

MongoDB stores data as BSON documents:

```javascript
{
  _id: ObjectId("64a1b2c3d4e5f6789"),
  account_no: "ACC001",
  name: "John Doe",
  balance: 5000.00,
  address: {                    // embedded document
    city: "Detroit",
    state: "MI"
  },
  transactions: [               // embedded array
    { amount: -200, date: ISODate("2026-01-15") },
    { amount: 500,  date: ISODate("2026-01-20") }
  ]
}
```

> The biggest mindset shift from SQL: you model data for how you READ it, not just for normalization. Embed related data that is always accessed together.

---

### The 4 Types of NoSQL Databases

**1. Document Stores** (MongoDB, CouchDB, Firestore)
- Data stored as JSON/BSON documents
- Best for: user profiles, product catalogs, banking records

**2. Key-Value Stores** (Redis, DynamoDB)
- Simple key → value pairs
- Best for: caching, sessions, real-time leaderboards

**3. Column-Family** (Apache Cassandra, HBase)
- Data stored in columns rather than rows
- Best for: time-series data, IoT, write-heavy workloads

**4. Graph Databases** (Neo4j, Amazon Neptune)
- Data stored as nodes and relationships
- Best for: fraud detection, social networks, recommendation engines

```javascript
// Document Store example (MongoDB)
db.accounts.insertOne({ name: "John", balance: 5000 })

// Key-Value example (Redis)
SET session:user123 "{ token: 'abc', expires: 3600 }"

// Column-Family example (Cassandra)
INSERT INTO transactions (id, amount, ts) VALUES (uuid(), 500.00, now());

// Graph example (Neo4j Cypher)
MATCH (a:Account)-[:TRANSFERS_TO]->(b:Account) RETURN a, b
```

> For a Spring Boot banking application: MongoDB for account/customer data, Redis for session caching, Cassandra for high-volume transaction logs.

---

### SQL vs MongoDB Cheat Sheet

| SQL Concept | MongoDB Equivalent |
|---|---|
| `CREATE TABLE` | `db.createCollection()` |
| `INSERT INTO` | `insertOne() / insertMany()` |
| `SELECT` | `find()` |
| `UPDATE` | `updateOne() / updateMany()` |
| `DELETE` | `deleteOne() / deleteMany()` |
| `WHERE` | Query filter `{ field: value }` |
| `JOIN` | `$lookup` in aggregation |
| `GROUP BY` | `$group` in aggregation |
| `ORDER BY` | `.sort()` |
| `LIMIT / OFFSET` | `.limit()` / `.skip()` |
| `INDEX` | `createIndex()` |
| `STORED PROCEDURE` | Aggregation pipeline / `$function` |
| `TRANSACTION` | `session.startTransaction()` |

---

## 2. Database Commands

### show dbs / use

```javascript
// Show all databases
show dbs

// Switch to (or create) a database
use bank_db

// Show current database
db

// Drop current database
db.dropDatabase()

// Get database stats
db.stats()
```

> MongoDB creates a database lazily — it only appears in `show dbs` once you insert the first document.

---

### Connection String (Spring Boot)

```properties
# application.properties — basic local
spring.data.mongodb.uri=mongodb://localhost:27017/bank_db

# With authentication
spring.data.mongodb.uri=mongodb://user:password@localhost:27017/bank_db

# MongoDB Atlas (cloud)
spring.data.mongodb.uri=mongodb+srv://user:pass@cluster.mongodb.net/bank_db

# Replica set (production — required for transactions)
spring.data.mongodb.uri=mongodb://host1:27017,host2:27017,host3:27017/bank_db?replicaSet=rs0
```

```java
// Java MongoClient (programmatic)
MongoClient client = MongoClients.create("mongodb://localhost:27017");
MongoDatabase db = client.getDatabase("bank_db");
```

> Always use connection pooling in production. Spring Boot auto-configures this when you add `spring-boot-starter-data-mongodb`.

---

## 3. Collection Commands

### createCollection

```javascript
// Create collection explicitly
db.createCollection("accounts")

// Capped collection (fixed size, FIFO — great for audit logs)
db.createCollection("audit_logs", {
  capped: true,
  size: 10485760,   // 10MB max
  max: 100000       // max 100k documents
})

// Create with schema validation
db.createCollection("accounts", {
  validator: {
    $jsonSchema: {
      bsonType: "object",
      required: ["account_no", "name", "balance"],
      properties: {
        account_no: { bsonType: "string" },
        balance:    { bsonType: "double", minimum: 0 },
        status:     { enum: ["ACTIVE", "INACTIVE", "CLOSED"] }
      }
    }
  }
})

// List all collections
show collections
db.getCollectionNames()
```

> Schema validation is great for banking apps — enforce data integrity at the DB level even though MongoDB is schema-flexible.

---

### drop / rename / stats

```javascript
// Drop a collection (all data deleted — permanent)
db.accounts.drop()

// Rename a collection
db.accounts.renameCollection("bank_accounts")

// Copy collection to another DB
db.accounts.aggregate([{ $out: { db: "archive_db", coll: "accounts_2025" } }])

// Collection stats
db.accounts.stats()
db.accounts.countDocuments()
db.accounts.estimatedDocumentCount()
```

---

## 4. CRUD Operations

### insertOne / insertMany

```javascript
// Insert a single document
db.accounts.insertOne({
  account_no: "ACC001",
  name: "John Doe",
  balance: 5000.00,
  status: "ACTIVE",
  branch: "Detroit",
  created_at: new Date(),
  address: { city: "Detroit", state: "MI", zip: "48201" }
})

// Insert multiple documents
db.accounts.insertMany([
  { account_no: "ACC002", name: "Jane Smith",  balance: 10000.00, status: "ACTIVE" },
  { account_no: "ACC003", name: "Bob Johnson", balance: 2500.00,  status: "ACTIVE" },
  { account_no: "ACC004", name: "Alice Brown", balance: 7500.00,  status: "INACTIVE" }
])
```

> MongoDB auto-generates a unique `_id` (ObjectId) for every document. You can supply your own `_id` if you have a natural unique key like `account_no`.

---

### find / findOne

```javascript
// Find all documents
db.accounts.find().pretty()

// Find with filter
db.accounts.find({ status: "ACTIVE" })
db.accounts.find({ status: "ACTIVE", branch: "Detroit" })

// Find one document
db.accounts.findOne({ account_no: "ACC001" })

// Project specific fields (1=include, 0=exclude)
db.accounts.find(
  { status: "ACTIVE" },
  { name: 1, balance: 1, _id: 0 }
)

// Sort, limit, skip (pagination)
db.accounts.find({ status: "ACTIVE" })
  .sort({ balance: -1 })    // -1 = DESC, 1 = ASC
  .limit(10)
  .skip(20)

// Count documents
db.accounts.countDocuments({ status: "ACTIVE" })
```

> Always project only the fields you need in production — returning full documents wastes bandwidth, especially with embedded arrays.

---

### updateOne / updateMany

```javascript
// Update a single document
db.accounts.updateOne(
  { account_no: "ACC001" },             // filter
  { $set: { balance: 6000.00 } }        // update
)

// Update multiple documents
db.accounts.updateMany(
  { status: "INACTIVE" },
  { $set: { status: "CLOSED", closed_at: new Date() } }
)

// Increment a field — atomic balance adjustment
db.accounts.updateOne(
  { account_no: "ACC001" },
  { $inc: { balance: -500.00 } }        // debit 500
)

// Push to an array (add a transaction)
db.accounts.updateOne(
  { account_no: "ACC001" },
  { $push: { transactions: { amount: 500, date: new Date(), type: "CREDIT" } } }
)

// Upsert (insert if not exists, update if exists)
db.accounts.updateOne(
  { account_no: "ACC999" },
  { $set: { name: "New User", balance: 0 } },
  { upsert: true }
)
```

**Update operators:**

| Operator | Description |
|---|---|
| `$set` | Set a field value |
| `$unset` | Remove a field |
| `$inc` | Increment a numeric field |
| `$mul` | Multiply a numeric field |
| `$push` | Add element to array |
| `$pull` | Remove element from array |
| `$addToSet` | Add to array only if unique |
| `$pop` | Remove first or last array element |
| `$rename` | Rename a field |
| `$currentDate` | Set field to current date |

> `$inc` is atomic — use it for balance changes instead of read-then-write to avoid race conditions in banking transactions.

---

### deleteOne / deleteMany

```javascript
// Delete a single document
db.accounts.deleteOne({ account_no: "ACC001" })

// Delete multiple documents
db.accounts.deleteMany({ status: "CLOSED" })

// Find and delete (returns the deleted document)
db.accounts.findOneAndDelete(
  { account_no: "ACC001" },
  { projection: { name: 1, balance: 1 } }
)

// Soft delete pattern (recommended for banking)
db.accounts.updateOne(
  { account_no: "ACC001" },
  { $set: { deleted: true, deleted_at: new Date() } }
)
```

> In banking NEVER hard-delete financial records — use soft delete (add a `deleted: true` flag) to maintain audit trails and comply with regulations.

---

### replaceOne / findOneAndUpdate

```javascript
// Replace entire document
db.accounts.replaceOne(
  { account_no: "ACC001" },
  {
    account_no: "ACC001",
    name: "John Doe Updated",
    balance: 7000.00,
    status: "ACTIVE",
    updated_at: new Date()
  }
)

// Find, update, and return in one atomic operation
db.accounts.findOneAndUpdate(
  { account_no: "ACC001", balance: { $gte: 500 } },
  { $inc: { balance: -500 } },
  {
    returnDocument: "after",   // return updated document
    projection: { balance: 1, account_no: 1 }
  }
)
```

> `findOneAndUpdate` is atomic — perfect for banking debit/credit operations where you need to check balance AND deduct in one step without a race condition.

---

## 5. Query Operators

### Comparison Operators

```javascript
// $eq, $ne, $gt, $gte, $lt, $lte
db.accounts.find({ balance: { $gte: 1000, $lte: 10000 } })
db.accounts.find({ status: { $ne: "CLOSED" } })
db.accounts.find({ balance: { $gt: 5000 } })

// $in, $nin (match values in array)
db.accounts.find({ status: { $in: ["ACTIVE", "INACTIVE"] } })
db.accounts.find({ branch: { $nin: ["Detroit", "Chicago"] } })

// $exists (check if field exists)
db.accounts.find({ email: { $exists: true } })
db.accounts.find({ deleted_at: { $exists: false } })

// $type (check field type)
db.accounts.find({ balance: { $type: "double" } })

// $regex (pattern matching)
db.accounts.find({ name: { $regex: /^John/i } })
db.accounts.find({ email: { $regex: /@bank\.com$/ } })
```

> `$regex` with a leading wildcard (e.g. `/.*john/`) cannot use an index. Anchor patterns to the start of the string when possible.

---

### Logical Operators

```javascript
// $and (all conditions must match)
db.accounts.find({
  $and: [
    { balance: { $gte: 1000 } },
    { status: "ACTIVE" },
    { branch: "Detroit" }
  ]
})

// $or (at least one must match)
db.accounts.find({
  $or: [
    { balance: { $gt: 10000 } },
    { status: "VIP" }
  ]
})

// $not
db.accounts.find({
  balance: { $not: { $lt: 500 } }
})

// $nor (none must match)
db.accounts.find({
  $nor: [{ status: "CLOSED" }, { deleted: true }]
})

// Combining logical operators
db.accounts.find({
  status: "ACTIVE",
  $or: [
    { balance: { $gte: 10000 } },
    { account_type: "PREMIUM" }
  ]
})
```

---

### Array Operators

```javascript
// Find docs where array contains a value
db.accounts.find({ tags: "premium" })

// $all (array must contain ALL values)
db.accounts.find({ tags: { $all: ["premium", "verified"] } })

// $size (array has exact length)
db.accounts.find({ transactions: { $size: 0 } })  // no transactions

// $elemMatch (element in array matches conditions)
db.accounts.find({
  transactions: {
    $elemMatch: { amount: { $gt: 1000 }, type: "CREDIT" }
  }
})

// Query nested field in array
db.accounts.find({ "transactions.type": "DEBIT" })
db.accounts.find({ "transactions.amount": { $gt: 500 } })

// $push with $sort and $slice (capped sorted array)
db.accounts.updateOne(
  { account_no: "ACC001" },
  {
    $push: {
      recent_transactions: {
        $each: [{ amount: 200, date: new Date() }],
        $sort: { date: -1 },
        $slice: 10        // keep only last 10
      }
    }
  }
)
```

---

### Date Queries

```javascript
// Find documents created today
db.transactions.find({
  created_at: {
    $gte: new Date(new Date().setHours(0,0,0,0)),
    $lt:  new Date(new Date().setHours(23,59,59,999))
  }
})

// Last 30 days
db.transactions.find({
  created_at: {
    $gte: new Date(Date.now() - 30 * 24 * 60 * 60 * 1000)
  }
})

// Date range
db.transactions.find({
  created_at: {
    $gte: ISODate("2026-01-01T00:00:00Z"),
    $lt:  ISODate("2026-02-01T00:00:00Z")
  }
})
```

> Always store dates as ISODate/Date objects, never as strings. String date comparison gives wrong results for ranges.

---

## 6. Aggregation Pipeline

The aggregation pipeline processes documents through multiple stages — each stage transforms the documents.

### $match / $project

```javascript
// $match filters documents (like WHERE in SQL)
db.accounts.aggregate([
  { $match: { status: "ACTIVE", balance: { $gte: 1000 } } }
])

// $project reshapes output (like SELECT in SQL)
db.accounts.aggregate([
  { $match: { status: "ACTIVE" } },
  { $project: {
    account_no: 1,
    name: 1,
    balance: 1,
    _id: 0,
    balance_tier: {
      $switch: {
        branches: [
          { case: { $gte: ["$balance", 10000] }, then: "Platinum" },
          { case: { $gte: ["$balance",  5000] }, then: "Gold" },
          { case: { $gte: ["$balance",  1000] }, then: "Silver" }
        ],
        default: "Standard"
      }
    }
  }}
])
```

> Put `$match` as early as possible in the pipeline — it reduces the number of documents processed by subsequent stages.

---

### $group / $sort / $limit

```javascript
// $group (like GROUP BY in SQL)
db.accounts.aggregate([
  { $group: {
    _id: "$status",
    total_accounts: { $sum: 1 },
    total_balance:  { $sum: "$balance" },
    avg_balance:    { $avg: "$balance" },
    max_balance:    { $max: "$balance" },
    min_balance:    { $min: "$balance" }
  }},
  { $sort: { total_balance: -1 } }
])

// Group by branch — top 5
db.accounts.aggregate([
  { $match: { status: "ACTIVE" } },
  { $group: {
    _id: "$branch",
    account_count: { $sum: 1 },
    avg_balance:   { $avg: "$balance" }
  }},
  { $sort: { avg_balance: -1 } },
  { $limit: 5 }
])
```

**Accumulator operators:** `$sum`, `$avg`, `$min`, `$max`, `$first`, `$last`, `$push`, `$addToSet`, `$count`

> `$group` with `_id: null` groups ALL documents into one — useful for computing totals across the entire collection.

---

### $lookup (JOIN)

```javascript
// $lookup = LEFT JOIN in SQL
db.accounts.aggregate([
  {
    $lookup: {
      from: "customers",          // collection to join
      localField: "customer_id",  // field in accounts
      foreignField: "_id",        // field in customers
      as: "customer_info"         // output array field name
    }
  },
  { $unwind: "$customer_info" },  // flatten the array
  { $project: {
    account_no: 1,
    balance: 1,
    "customer_info.name": 1,
    "customer_info.email": 1
  }}
])

// $lookup with pipeline (more control — get last 5 transactions)
db.accounts.aggregate([
  {
    $lookup: {
      from: "transactions",
      let: { acc_id: "$_id" },
      pipeline: [
        { $match: { $expr: { $eq: ["$account_id", "$$acc_id"] } } },
        { $sort: { created_at: -1 } },
        { $limit: 5 }
      ],
      as: "recent_transactions"
    }
  }
])
```

> `$lookup` is expensive. In MongoDB you often avoid JOINs by embedding related data. Only use `$lookup` when data genuinely lives in separate collections.

---

### $unwind / $addFields

```javascript
// $unwind deconstructs an array into separate documents
db.accounts.aggregate([
  { $unwind: "$transactions" },
  { $match: { "transactions.amount": { $gt: 1000 } } },
  { $project: {
    account_no: 1,
    "transactions.amount": 1,
    "transactions.date": 1
  }}
])

// $unwind with preserveNullAndEmptyArrays
db.accounts.aggregate([
  { $unwind: { path: "$transactions", preserveNullAndEmptyArrays: true } }
])

// $addFields adds computed fields
db.accounts.aggregate([
  { $addFields: {
    balance_in_cents: { $multiply: ["$balance", 100] },
    is_premium:       { $gte: ["$balance", 10000] },
    account_age_days: {
      $dateDiff: {
        startDate: "$created_at",
        endDate:   "$$NOW",
        unit:      "day"
      }
    }
  }}
])
```

---

### $facet / $bucket

```javascript
// $facet runs multiple pipelines in parallel (great for dashboards)
db.accounts.aggregate([
  { $facet: {
    "by_status": [
      { $group: { _id: "$status", count: { $sum: 1 } } }
    ],
    "by_branch": [
      { $group: { _id: "$branch", total_balance: { $sum: "$balance" } } },
      { $sort: { total_balance: -1 } }
    ],
    "balance_stats": [
      { $group: {
        _id: null,
        avg: { $avg: "$balance" },
        total: { $sum: "$balance" },
        max: { $max: "$balance" }
      }}
    ]
  }}
])

// $bucket groups documents into ranges
db.accounts.aggregate([
  { $bucket: {
    groupBy: "$balance",
    boundaries: [0, 1000, 5000, 10000, 50000],
    default: "50000+",
    output: {
      count: { $sum: 1 },
      avg_balance: { $avg: "$balance" }
    }
  }}
])
```

> `$facet` is perfect for dashboard queries — get multiple aggregations in a single round trip to the database.

---

## 7. Index Commands

### createIndex

```javascript
// Single field index
db.accounts.createIndex({ status: 1 })          // 1 = ascending
db.accounts.createIndex({ balance: -1 })         // -1 = descending

// Compound index
db.accounts.createIndex({ status: 1, balance: -1 })

// Unique index
db.accounts.createIndex({ account_no: 1 }, { unique: true })

// Text index (full-text search)
db.accounts.createIndex({ name: "text", description: "text" })
db.accounts.find({ $text: { $search: "John Detroit" } })

// TTL index (auto-expire documents after N seconds)
db.sessions.createIndex(
  { created_at: 1 },
  { expireAfterSeconds: 3600 }   // expire after 1 hour
)

// Partial index (only index matching documents)
db.accounts.createIndex(
  { balance: 1 },
  { partialFilterExpression: { status: "ACTIVE" } }
)

// Sparse index (skip documents missing the field)
db.accounts.createIndex({ email: 1 }, { sparse: true })
```

> TTL indexes are perfect for session tokens and OTP codes in banking apps — MongoDB auto-deletes expired documents for you.

---

### getIndexes / dropIndex / explain

```javascript
// List all indexes on a collection
db.accounts.getIndexes()

// Drop a specific index
db.accounts.dropIndex("status_1")
db.accounts.dropIndex({ status: 1 })

// Drop all indexes (keeps _id index)
db.accounts.dropIndexes()

// Analyze query performance
db.accounts.find({ status: "ACTIVE" }).explain("executionStats")

// Key fields in explain() output:
// executionStats.totalDocsExamined  — lower is better
// executionStats.totalKeysExamined  — index keys scanned
// executionStats.executionTimeMillis — query time in ms
// winningPlan.stage == "IXSCAN"    — index used (GOOD)
// winningPlan.stage == "COLLSCAN"  — full scan (BAD)

// Force a specific index
db.accounts.find({ status: "ACTIVE" })
  .hint({ status: 1 })
  .explain("executionStats")
```

> `COLLSCAN` means MongoDB scanned every document. If you see this on a large collection, add an index on the query field immediately.

---

## 8. Schema Design

### Embedding vs Referencing

```javascript
// EMBEDDING (denormalized) — store related data together
// Use when: data always accessed together, 1-to-few relationship
{
  _id: ObjectId(),
  account_no: "ACC001",
  name: "John Doe",
  address: {                    // embed — always needed with account
    street: "123 Main St",
    city: "Detroit",
    state: "MI"
  },
  last_5_transactions: [        // embed — capped array
    { amount: 500, type: "CREDIT", date: ISODate("2026-06-01") },
    { amount: -200, type: "DEBIT", date: ISODate("2026-06-05") }
  ]
}

// REFERENCING (normalized) — store _id, lookup separately
// Use when: data grows unboundedly or shared across documents
// accounts collection:
{ _id: ObjectId("aaa"), account_no: "ACC001", customer_id: ObjectId("ccc") }

// customers collection:
{ _id: ObjectId("ccc"), name: "John Doe", email: "john@example.com" }

// transactions collection (high volume — always separate):
{ _id: ObjectId(), account_id: ObjectId("aaa"), amount: 500, date: new Date() }
```

**Decision guide:**

| Criteria | Embed | Reference |
|---|---|---|
| Access pattern | Always together | Independently |
| Relationship | 1-to-few | 1-to-many / many-to-many |
| Data growth | Bounded | Unbounded |
| Update frequency | Rarely | Frequently |
| Document size | Small additions | Large additions |

> Rule of thumb: embed if you always need it together (address, preferences). Reference if it grows unboundedly (transactions) or is shared across multiple documents.

---

### Schema Validation

```javascript
// Add or update validation on existing collection
db.runCommand({
  collMod: "accounts",
  validator: {
    $jsonSchema: {
      bsonType: "object",
      required: ["account_no", "name", "balance", "status", "created_at"],
      properties: {
        account_no: {
          bsonType: "string",
          description: "must be a string and is required"
        },
        balance: {
          bsonType: "double",
          minimum: 0,
          description: "balance cannot be negative"
        },
        status: {
          enum: ["ACTIVE", "INACTIVE", "CLOSED"],
          description: "must be one of the allowed values"
        },
        email: {
          bsonType: "string",
          pattern: "^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\\.[a-zA-Z]{2,}$"
        }
      }
    }
  },
  validationLevel: "strict",    // strict | moderate
  validationAction: "error"     // error | warn
})
```

---

### Spring Data MongoDB

```java
// @Document model class
@Document(collection = "accounts")
public class Account {
    @Id
    private String id;

    @Field("account_no")
    @Indexed(unique = true)
    private String accountNo;

    @NotBlank
    private String name;

    @Min(0)
    private Double balance;

    private String status;
    private Address address;       // embedded document
    private Date createdAt;
}

// Repository interface
public interface AccountRepository extends MongoRepository<Account, String> {
    List<Account> findByStatus(String status);
    List<Account> findByBalanceGreaterThan(Double amount);
    Optional<Account> findByAccountNo(String accountNo);

    @Query("{ 'balance': { $gte: ?0, $lte: ?1 } }")
    List<Account> findByBalanceRange(Double min, Double max);
}

// MongoTemplate for complex queries
@Autowired MongoTemplate mongoTemplate;

Query query = new Query(Criteria.where("status").is("ACTIVE")
    .and("balance").gte(1000));
query.with(Sort.by(Sort.Direction.DESC, "balance"));
List<Account> accounts = mongoTemplate.find(query, Account.class);
```

> Use `MongoRepository` for simple CRUD. Use `MongoTemplate` for complex aggregations and custom queries.

---

## 9. Admin & Performance

### ACID Transactions

```javascript
// MongoDB ACID transactions (requires replica set — v4.0+)
const session = db.getMongo().startSession()
session.startTransaction()

try {
  const accounts = session.getDatabase("bank_db").accounts

  // Debit source account
  accounts.updateOne(
    { account_no: "ACC001", balance: { $gte: 500 } },
    { $inc: { balance: -500 } },
    { session }
  )

  // Credit destination account
  accounts.updateOne(
    { account_no: "ACC002" },
    { $inc: { balance: 500 } },
    { session }
  )

  // Insert transaction record
  session.getDatabase("bank_db").transactions.insertOne({
    from: "ACC001", to: "ACC002",
    amount: 500, date: new Date(), status: "COMPLETED"
  }, { session })

  session.commitTransaction()
} catch (err) {
  session.abortTransaction()
  throw err
} finally {
  session.endSession()
}
```

> Multi-document transactions require a replica set — even in development. Run `mongod --replSet rs0` locally. Use atomic single-document operations where possible to avoid transaction overhead.

---

### $out / $merge (Write Aggregation Results)

```javascript
// $out — write results to a new collection
db.transactions.aggregate([
  { $match: { created_at: { $gte: ISODate("2026-01-01") } } },
  { $group: {
    _id: "$account_id",
    total_debits:  { $sum: { $cond: [{ $lt: ["$amount", 0] }, "$amount", 0] } },
    total_credits: { $sum: { $cond: [{ $gt: ["$amount", 0] }, "$amount", 0] } },
    txn_count: { $sum: 1 }
  }},
  { $out: "monthly_summaries" }
])

// $merge — merge results into existing collection
db.transactions.aggregate([
  { $group: { _id: "$account_id", total: { $sum: "$amount" } } },
  { $merge: {
    into: "account_summaries",
    on: "_id",
    whenMatched: "merge",
    whenNotMatched: "insert"
  }}
])
```

---

### Backup / Restore

```bash
# Export entire database
mongodump --db bank_db --out /backup/

# Export specific collection
mongodump --db bank_db --collection accounts --out /backup/

# Export as JSON
mongoexport --db bank_db --collection accounts --out accounts.json

# Export as CSV
mongoexport --db bank_db --collection accounts \
  --type csv --fields account_no,name,balance \
  --out accounts.csv

# Restore database
mongorestore --db bank_db /backup/bank_db/

# Restore specific collection
mongorestore --db bank_db --collection accounts \
  /backup/bank_db/accounts.bson

# Import JSON
mongoimport --db bank_db --collection accounts \
  --file accounts.json --jsonArray
```

> Schedule daily `mongodump` backups. For Atlas, use Atlas Backup for point-in-time recovery. Always test restores — an untested backup is not a backup.

---

### Performance & Monitoring

```javascript
// Find slow running operations (>5 seconds)
db.currentOp({ "active": true, "secs_running": { $gt: 5 } })

// Kill a slow operation
db.killOp(12345)   // pass opid from currentOp

// Enable profiler (log queries slower than 100ms)
db.setProfilingLevel(1, { slowms: 100 })
db.setProfilingLevel(2)       // log ALL queries
db.setProfilingLevel(0)       // profiling OFF

// View slow query log
db.system.profile.find().sort({ ts: -1 }).limit(10).pretty()

// Server and collection stats
db.serverStatus()
db.serverStatus().connections
db.serverStatus().opcounters
db.accounts.stats()
db.accounts.stats().size
db.accounts.stats().totalIndexSize

// Compact a collection (reclaim disk space)
db.runCommand({ compact: "accounts" })
```

> Run `db.setProfilingLevel(1, {slowms: 100})` in production to catch slow queries. Review `system.profile` regularly.

---

## Quick Reference Summary

| Category | Key Operations |
|---|---|
| NoSQL Core | Document, Key-Value, Column-Family, Graph types |
| Database | `use`, `show dbs`, `db.dropDatabase()`, `db.stats()` |
| Collection | `createCollection`, `drop`, `renameCollection`, `stats` |
| Insert | `insertOne`, `insertMany` |
| Read | `find`, `findOne`, `.sort()`, `.limit()`, `.skip()` |
| Update | `updateOne`, `updateMany`, `$set`, `$inc`, `$push`, upsert |
| Delete | `deleteOne`, `deleteMany`, `findOneAndDelete`, soft delete |
| Query | `$eq/$gt/$lt`, `$in`, `$regex`, `$and/$or`, `$elemMatch` |
| Aggregation | `$match`, `$group`, `$project`, `$lookup`, `$unwind`, `$facet` |
| Index | `createIndex`, `dropIndex`, `explain()`, TTL, text, partial |
| Schema | Embedding vs Referencing, validation, Spring Data annotations |
| Admin | Transactions, `$out/$merge`, `mongodump`, profiler |

---

*Generated for Java Spring Boot Banking Application Reference — MongoDB 7.0+*
