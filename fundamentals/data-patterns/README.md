# Data Patterns for System Design

## Database Selection

### SQL vs NoSQL Decision Tree

```
┌─────────────────────────────────────────────────────────────┐
│              Do you need ACID transactions?                  │
│                          │                                   │
│            ┌─────────────┼─────────────┐                    │
│            ▼                           ▼                     │
│           YES                          NO                    │
│            │                           │                     │
│     ┌──────▼──────┐          ┌─────────▼─────────┐          │
│     │    SQL      │          │ What's your data  │          │
│     │ PostgreSQL  │          │     model?        │          │
│     │   MySQL     │          └─────────┬─────────┘          │
│     └─────────────┘                    │                     │
│                              ┌─────────┼─────────┐          │
│                              ▼         ▼         ▼          │
│                          Key-Value  Document  Wide-Column   │
│                           Redis    MongoDB    Cassandra     │
│                          DynamoDB  CouchDB   ScyllaDB       │
└─────────────────────────────────────────────────────────────┘
```

### Database Types Comparison

| Type | Examples | Best For | Avoid When |
|------|----------|----------|------------|
| **Relational (SQL)** | PostgreSQL, MySQL | Complex queries, transactions, structured data | Massive scale, flexible schema |
| **Document** | MongoDB, CouchDB | Flexible schema, JSON data, rapid development | Complex joins, strong consistency |
| **Key-Value** | Redis, DynamoDB | Caching, sessions, simple lookups | Complex queries, relationships |
| **Wide-Column** | Cassandra, HBase | Time-series, write-heavy, distributed | Complex queries, strong consistency |
| **Graph** | Neo4j, Neptune | Relationships, social networks | Simple CRUD, tabular data |
| **Time-Series** | InfluxDB, TimescaleDB | Metrics, IoT, logs | General-purpose storage |
| **Search** | Elasticsearch, Solr | Full-text search, analytics | Primary data store |

---

## Data Modeling Patterns

### Normalization vs Denormalization

```
NORMALIZED (3NF):
┌──────────┐     ┌──────────┐     ┌──────────┐
│  Users   │     │  Orders  │     │ Products │
├──────────┤     ├──────────┤     ├──────────┤
│ id       │◄───┐│ id       │┌───►│ id       │
│ name     │    ││ user_id  ├┘    │ name     │
│ email    │    │├ product_id     │ price    │
└──────────┘    │└──────────┘     └──────────┘
                │
Benefits: No redundancy, easy updates
Drawback: Expensive joins at scale

DENORMALIZED:
┌─────────────────────────────────────┐
│            OrderDetails             │
├─────────────────────────────────────┤
│ order_id                            │
│ user_name (copied from Users)       │
│ user_email (copied from Users)      │
│ product_name (copied from Products) │
│ product_price (copied from Products)│
│ quantity                            │
│ total                               │
└─────────────────────────────────────┘

Benefits: Fast reads, no joins
Drawback: Data duplication, update complexity
```

### Embedding vs Referencing (Document DBs)

```
EMBEDDING (Denormalized):
{
  "post_id": "123",
  "title": "System Design",
  "comments": [
    {"user": "Alice", "text": "Great post!"},
    {"user": "Bob", "text": "Thanks!"}
  ]
}

✓ Use when: Data accessed together, 1:few relationship
✗ Avoid when: Large/unbounded arrays, independent access

REFERENCING (Normalized):
// Posts collection
{"post_id": "123", "title": "System Design"}

// Comments collection
{"comment_id": "1", "post_id": "123", "user": "Alice", "text": "Great!"}
{"comment_id": "2", "post_id": "123", "user": "Bob", "text": "Thanks!"}

✓ Use when: 1:many relationship, independent updates
✗ Avoid when: Always accessed together
```

---

## Storage Patterns

### Hot/Warm/Cold Storage

```
┌─────────────────────────────────────────────────────────────┐
│  HOT (Frequently Accessed)                                   │
│  • Recent data (last 7 days)                                │
│  • Storage: SSD, In-memory (Redis)                          │
│  • Cost: $$$                                                │
├─────────────────────────────────────────────────────────────┤
│  WARM (Occasionally Accessed)                                │
│  • Recent history (last 30-90 days)                         │
│  • Storage: Standard SSD/HDD                                │
│  • Cost: $$                                                 │
├─────────────────────────────────────────────────────────────┤
│  COLD (Rarely Accessed)                                      │
│  • Archives, compliance data                                 │
│  • Storage: S3 Glacier, Archive tiers                       │
│  • Cost: $                                                  │
└─────────────────────────────────────────────────────────────┘

Implementation:
• TTL-based migration
• Access pattern monitoring
• Automated tiering policies
```

### Write-Ahead Logging (WAL)

```
┌─────────────────────────────────────────────────────────────┐
│  1. Write to WAL (sequential, fast)                         │
│  2. Acknowledge to client                                    │
│  3. Apply to main storage (async)                           │
│                                                              │
│  ┌────────┐    1. Write    ┌───────┐                        │
│  │ Client │───────────────►│  WAL  │                        │
│  └────────┘                └───┬───┘                        │
│       ▲                        │ 3. Async apply             │
│       │ 2. Ack                 ▼                            │
│       │                   ┌─────────┐                       │
│       └───────────────────│  Data   │                       │
│                           │  Store  │                       │
│                           └─────────┘                       │
│                                                              │
│  Benefits: Durability + Performance                          │
│  Used by: PostgreSQL, MySQL, Kafka                          │
└─────────────────────────────────────────────────────────────┘
```

### Log-Structured Merge Trees (LSM)

```
┌─────────────────────────────────────────────────────────────┐
│                    LSM Tree Structure                        │
│                                                              │
│  ┌──────────────┐                                           │
│  │   MemTable   │◄──── Writes (in-memory, sorted)           │
│  └──────┬───────┘                                           │
│         │ Flush when full                                    │
│         ▼                                                    │
│  ┌──────────────┐                                           │
│  │   Level 0    │  SSTable files                            │
│  │  (immutable) │                                           │
│  └──────┬───────┘                                           │
│         │ Compaction                                         │
│         ▼                                                    │
│  ┌──────────────┐                                           │
│  │   Level 1    │  Merged, sorted SSTables                  │
│  └──────┬───────┘                                           │
│         │                                                    │
│         ▼                                                    │
│  ┌──────────────┐                                           │
│  │   Level N    │                                           │
│  └──────────────┘                                           │
│                                                              │
│  Write: O(1) - append to MemTable                           │
│  Read: O(log n) - check each level                          │
│  Used by: Cassandra, RocksDB, LevelDB                       │
└─────────────────────────────────────────────────────────────┘
```

---

## Data Replication Patterns

### Synchronous vs Asynchronous

```
SYNCHRONOUS REPLICATION:
┌────────┐    Write    ┌─────────┐    Sync    ┌─────────┐
│ Client │────────────►│ Primary │───────────►│ Replica │
└────────┘             └─────────┘            └────┬────┘
     ▲                                             │
     └─────────────── Ack (after replica confirms)─┘

+ Strong consistency
- Higher latency
- Availability risk if replica down

ASYNCHRONOUS REPLICATION:
┌────────┐    Write    ┌─────────┐
│ Client │────────────►│ Primary │
└────────┘             └────┬────┘
     ▲                      │ Ack (immediate)
     │                      │
     └──────────────────────┘
                            │ Async
                            ▼
                       ┌─────────┐
                       │ Replica │
                       └─────────┘

+ Low latency
+ High availability
- Potential data loss (replication lag)
```

### Multi-Master Replication

```
┌──────────┐           ┌──────────┐
│ Master 1 │◄─────────►│ Master 2 │
│ (US)     │           │ (EU)     │
└──────────┘           └──────────┘

Conflict Resolution Strategies:
1. Last Write Wins (LWW) - timestamp based
2. Application-level resolution
3. CRDTs (Conflict-free Replicated Data Types)
4. Manual resolution

Use case: Global applications needing low latency writes
```

---

## Data Partitioning Strategies

### Partition Key Selection

```
GOOD PARTITION KEY:
• High cardinality (many unique values)
• Even distribution
• Query-aligned (most queries include it)

Examples:
✓ user_id - unique per user, even distribution
✓ order_id - unique per order
✓ timestamp + region - for time-series with geographic spread

BAD PARTITION KEY:
✗ country - uneven (US >> Luxembourg)
✗ status - low cardinality (active/inactive)
✗ created_date only - hot partition for today
```

### Handling Hot Partitions

```
Problem: Celebrity user with millions of followers
         All requests hit same partition

Solutions:

1. Add Random Suffix:
   partition_key = user_id + random(0-9)
   Scatter writes across 10 partitions
   Reads must query all 10 and merge

2. Separate Hot/Cold:
   Regular users → normal sharding
   Celebrities → dedicated partition with more resources

3. Caching Layer:
   Cache hot data in Redis
   Offload reads from database
```

---

## Event Sourcing

```
Traditional CRUD:
┌──────────────────────────────────────┐
│ Account: { id: 1, balance: 100 }     │
│ UPDATE → { id: 1, balance: 150 }     │  History lost!
│ UPDATE → { id: 1, balance: 80 }      │
└──────────────────────────────────────┘

Event Sourcing:
┌──────────────────────────────────────┐
│ Event 1: AccountCreated(id=1)        │
│ Event 2: MoneyDeposited(50)          │
│ Event 3: MoneyWithdrawn(20)          │
│ Event 4: MoneyDeposited(100)         │
│ Event 5: MoneyWithdrawn(50)          │
│                                      │
│ Current State = Replay all events    │
│ Balance = 0 + 50 - 20 + 100 - 50 = 80│
└──────────────────────────────────────┘

Benefits:
• Complete audit trail
• Temporal queries ("balance on Jan 1?")
• Event replay for debugging
• Rebuild projections

Challenges:
• Storage growth
• Snapshot management
• Schema evolution
```

### CQRS (Command Query Responsibility Segregation)

```
┌──────────────────────────────────────────────────────────────┐
│                                                               │
│   Commands (Write)              Queries (Read)                │
│        │                              │                       │
│        ▼                              ▼                       │
│   ┌─────────┐                   ┌─────────┐                  │
│   │ Command │                   │  Query  │                  │
│   │ Handler │                   │ Handler │                  │
│   └────┬────┘                   └────┬────┘                  │
│        │                              │                       │
│        ▼                              ▼                       │
│   ┌─────────┐                   ┌─────────┐                  │
│   │  Write  │    Sync/Async     │  Read   │                  │
│   │  Model  │──────────────────►│  Model  │                  │
│   │(Events) │                   │(Views)  │                  │
│   └─────────┘                   └─────────┘                  │
│                                                               │
│   Write Model: Optimized for writes (normalized)              │
│   Read Model: Optimized for reads (denormalized)              │
└──────────────────────────────────────────────────────────────┘
```

---

## Data Migration Patterns

### Dual Write (Anti-pattern!)

```
❌ DON'T DO THIS:
┌─────────┐
│   App   │
└────┬────┘
     │
     ├────► Write to Old DB
     │
     └────► Write to New DB

Problems:
• No atomicity - one write may fail
• Race conditions
• Inconsistent data
```

### Change Data Capture (CDC)

```
✓ RECOMMENDED:
┌─────────┐     Write      ┌─────────┐
│   App   │───────────────►│ Old DB  │
└─────────┘                └────┬────┘
                                │ CDC (Debezium)
                                ▼
                          ┌─────────┐
                          │  Kafka  │
                          └────┬────┘
                                │
                                ▼
                          ┌─────────┐
                          │ New DB  │
                          └─────────┘

Benefits:
• Atomic - changes captured from DB log
• Reliable - Kafka provides durability
• Decoupled - DB doesn't know about migration
```

### Strangler Fig Pattern

```
Phase 1: All traffic to old system
┌────────┐     ┌──────────┐
│ Client │────►│ Old Sys  │
└────────┘     └──────────┘

Phase 2: Route some features to new
┌────────┐     ┌─────────┐
│ Client │────►│ Router  │
└────────┘     └────┬────┘
                    │
           ┌───────┴───────┐
           ▼               ▼
      ┌──────────┐   ┌──────────┐
      │ Old Sys  │   │ New Sys  │
      │(feature A)   │(feature B)│
      └──────────┘   └──────────┘

Phase 3: Complete migration
┌────────┐     ┌──────────┐
│ Client │────►│ New Sys  │
└────────┘     └──────────┘
```

---

## Summary: Data Pattern Selection

| Scenario | Pattern |
|----------|---------|
| Need ACID transactions | SQL Database |
| Flexible schema, rapid iteration | Document Store |
| Simple key-based lookups | Key-Value Store |
| High write throughput | LSM-based DB (Cassandra) |
| Complex relationships | Graph Database |
| Full-text search | Elasticsearch |
| Time-series data | Time-series DB |
| Audit requirements | Event Sourcing |
| Read/Write optimization | CQRS |
| Global low-latency | Multi-region replication |
| Cost optimization | Tiered storage |

---

## Further Reading
- [Designing Data-Intensive Applications](https://dataintensive.net/)
- [Database Internals](https://www.databass.dev/)
- [Martin Kleppmann's Blog](https://martin.kleppmann.com/)
