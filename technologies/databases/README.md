# Database Technologies

## Overview

Databases are the backbone of any system. Understanding when to use which type is crucial for system design.

---

## Database Categories

### Relational (SQL)

```
┌─────────────────────────────────────────────────────────────┐
│  PostgreSQL, MySQL, Oracle, SQL Server                       │
│                                                              │
│  ✓ ACID transactions                                        │
│  ✓ Complex queries (JOINs)                                  │
│  ✓ Strong consistency                                        │
│  ✓ Mature ecosystem                                          │
│                                                              │
│  Use when: Financial data, complex relationships,            │
│            transactional requirements                        │
└─────────────────────────────────────────────────────────────┘
```

### Document (NoSQL)

```
┌─────────────────────────────────────────────────────────────┐
│  MongoDB, CouchDB, DocumentDB                                │
│                                                              │
│  ✓ Flexible schema                                          │
│  ✓ JSON-native                                              │
│  ✓ Horizontal scaling                                        │
│  ✓ Developer friendly                                        │
│                                                              │
│  Use when: Rapid development, varying data structures,       │
│            content management                                │
└─────────────────────────────────────────────────────────────┘
```

### Key-Value (NoSQL)

```
┌─────────────────────────────────────────────────────────────┐
│  Redis, DynamoDB, etcd                                       │
│                                                              │
│  ✓ Simple model (key → value)                               │
│  ✓ Extremely fast                                            │
│  ✓ Easy to scale                                             │
│  ✓ Great for caching                                         │
│                                                              │
│  Use when: Sessions, caching, feature flags,                 │
│            simple lookups                                    │
└─────────────────────────────────────────────────────────────┘
```

### Wide-Column (NoSQL)

```
┌─────────────────────────────────────────────────────────────┐
│  Cassandra, HBase, ScyllaDB                                  │
│                                                              │
│  ✓ Write-optimized                                          │
│  ✓ Time-series friendly                                      │
│  ✓ Linear scalability                                        │
│  ✓ No single point of failure                               │
│                                                              │
│  Use when: High write throughput, time-series,               │
│            distributed across regions                        │
└─────────────────────────────────────────────────────────────┘
```

### Graph (NoSQL)

```
┌─────────────────────────────────────────────────────────────┐
│  Neo4j, Amazon Neptune, JanusGraph                           │
│                                                              │
│  ✓ Relationship-first model                                 │
│  ✓ Traversal queries fast                                    │
│  ✓ Natural for connected data                               │
│  ✓ Pattern matching                                          │
│                                                              │
│  Use when: Social networks, recommendations,                 │
│            fraud detection, knowledge graphs                 │
└─────────────────────────────────────────────────────────────┘
```

---

## Comparison Matrix

| Feature | PostgreSQL | MongoDB | Cassandra | Redis | Neo4j |
|---------|------------|---------|-----------|-------|-------|
| Consistency | Strong | Eventual/Strong | Eventual | Strong | Strong |
| Schema | Rigid | Flexible | Semi-flexible | Schema-less | Schema-optional |
| Scaling | Vertical | Horizontal | Horizontal | Horizontal | Vertical |
| Best for | OLTP | General | Writes | Caching | Relationships |
| Transactions | ACID | Multi-doc | Limited | Limited | ACID |
| Query Lang | SQL | MQL | CQL | Commands | Cypher |

---

## Decision Tree

```
Need ACID transactions?
├── YES → SQL (PostgreSQL)
└── NO
    │
    ├── Simple key-value lookups?
    │   └── YES → Redis/DynamoDB
    │
    ├── Complex relationships?
    │   └── YES → Neo4j
    │
    ├── High write throughput?
    │   └── YES → Cassandra
    │
    └── Flexible documents?
        └── YES → MongoDB
```

---

## Deep Dives (In this folder)

- `postgresql.md` - RDBMS deep dive
- `mongodb.md` - Document store patterns
- `cassandra.md` - Wide-column at scale
- `redis.md` - Caching and more
- `neo4j.md` - Graph database patterns
- `dynamodb.md` - AWS managed NoSQL
- `comparison.md` - Detailed comparisons

---

## Resources

- [Use The Index, Luke](https://use-the-index-luke.com/)
- [Database Internals Book](https://www.databass.dev/)
- [Jepsen Analyses](https://jepsen.io/analyses)
