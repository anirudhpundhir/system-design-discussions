# Design Principles for Distributed Systems

## Core Design Principles

### 1. Keep It Simple (KISS)

```
Start with the simplest design that meets requirements.
Add complexity only when justified by scale or features.

❌ Day 1: "We need Kubernetes, microservices, event sourcing..."
✅ Day 1: "A monolith with good boundaries will work for now"
```

### 2. Design for Failure

Everything fails. Design assuming components will fail.

```
┌─────────────────────────────────────────────────────────┐
│  Assume:                                                 │
│  • Servers crash                                         │
│  • Networks partition                                    │
│  • Disks fail                                           │
│  • Deployments have bugs                                │
│  • Third-party services go down                         │
│                                                          │
│  Design:                                                 │
│  • Redundancy at every layer                            │
│  • Graceful degradation                                 │
│  • Circuit breakers                                     │
│  • Retry with exponential backoff                       │
│  • Health checks and auto-recovery                      │
└─────────────────────────────────────────────────────────┘
```

### 3. Horizontal over Vertical Scaling

```
Vertical Scaling (Scale Up):
└─ Buy bigger machines
└─ Has upper limits
└─ Single point of failure
└─ Expensive at scale

Horizontal Scaling (Scale Out):
└─ Add more machines
└─ Nearly unlimited
└─ Built-in redundancy
└─ More cost-effective
└─ Requires stateless design
```

### 4. Stateless Services

```
Stateful Service:
┌─────────┐    ┌─────────┐
│ Request │───►│ Server  │  Server holds user state
└─────────┘    │ (state) │  Hard to scale, failover breaks
               └─────────┘

Stateless Service:
┌─────────┐    ┌─────────┐
│ Request │───►│ Server  │◄──── State in external store
└─────────┘    │ (logic) │      (Redis, DB)
               └────┬────┘
                    │
               ┌────▼────┐
               │  State  │  Easy to scale, failover works
               │  Store  │
               └─────────┘
```

### 5. Design for Observability

```
The Three Pillars:

METRICS          LOGS              TRACES
├─ Counters      ├─ Structured     ├─ Request ID
├─ Gauges        ├─ Searchable     ├─ Span tracking
├─ Histograms    ├─ Sampled        ├─ Service map
└─ Dashboards    └─ Aggregated     └─ Latency breakdown

Without observability, you're flying blind at scale.
```

---

## Distributed Systems Principles

### 1. Single Source of Truth

Every piece of data has one authoritative source.

```
✅ Good:
User profile → User Service DB
Order data → Order Service DB
Product catalog → Catalog Service DB

❌ Bad:
User profile copied to 5 different services
Each service modifies its copy independently
Data becomes inconsistent
```

### 2. Idempotency

Operations can be safely retried without side effects.

```
POST /api/transfer
{
    "idempotency_key": "txn-abc-123",  ← Key makes it safe
    "from": "account_1",
    "to": "account_2",
    "amount": 100
}

First call: Executes transfer, stores key
Retry calls: Returns same result, no duplicate transfer
```

### 3. Event-Driven Architecture

Decouple services through events.

```
Tight Coupling (Synchronous):
Order ──► Inventory ──► Shipping ──► Email
         (waits)       (waits)       (waits)
If any fails, entire flow fails.

Loose Coupling (Event-Driven):
Order ──► Event Bus ──► Inventory (async)
                   ├──► Shipping (async)
                   └──► Email (async)
Services operate independently.
```

### 4. Circuit Breaker Pattern

Prevent cascade failures.

```
States:
┌────────┐   failures > threshold   ┌────────┐
│ CLOSED │ ──────────────────────► │  OPEN  │
│(normal)│                          │(reject)│
└────────┘                          └───┬────┘
     ▲                                  │
     │        ┌───────────┐             │
     │        │HALF-OPEN  │◄────────────┘
     └────────│(test)     │  timeout expires
  success     └───────────┘
```

```python
# Pseudocode
if circuit.is_open():
    return fallback_response()

try:
    response = call_service()
    circuit.record_success()
    return response
except Exception:
    circuit.record_failure()
    if circuit.failures > threshold:
        circuit.open()
    return fallback_response()
```

### 5. Backpressure

Handle overload gracefully.

```
Without Backpressure:
Producer ───────────► Consumer
  1000/s                100/s
           Queue grows forever → OOM crash

With Backpressure:
Producer ───────────► Consumer
  1000/s                100/s
     ▲
     └── "Slow down!" signal

Options:
• Drop oldest messages
• Reject new requests (429 Too Many Requests)
• Throttle producer
• Scale consumers
```

---

## Data Design Principles

### 1. Normalize for Writes, Denormalize for Reads

```
Normalized (good for writes):
┌─────────┐     ┌─────────┐     ┌─────────┐
│  User   │◄───►│  Order  │◄───►│ Product │
└─────────┘     └─────────┘     └─────────┘
• No data duplication
• Easy updates
• Complex queries (joins)

Denormalized (good for reads):
┌─────────────────────────────────────────────┐
│ Order (with embedded user and product info) │
└─────────────────────────────────────────────┘
• Data duplication
• Complex updates
• Simple, fast queries
```

### 2. Partition for Scale, Replicate for Availability

```
Partitioning (Sharding):
┌─────────┐
│ All Data│
└────┬────┘
     │ Split by user_id % 3
     ▼
┌────┴────┬─────────┬─────────┐
│ Shard 0 │ Shard 1 │ Shard 2 │
│(users   │(users   │(users   │
│ 0,3,6..)│ 1,4,7..)│ 2,5,8..)│
└─────────┴─────────┴─────────┘
• Distributes load
• Enables horizontal scaling

Replication:
┌─────────┐
│ Primary │
└────┬────┘
     │ Replicate
     ▼
┌────┴────┬─────────┐
│Replica 1│Replica 2│
└─────────┴─────────┘
• Improves read throughput
• Provides fault tolerance
```

### 3. Cache Strategically

```
Cache Hierarchy:
┌─────────────────────────────────────┐
│ Client Cache (Browser/App)          │ Fastest
├─────────────────────────────────────┤
│ CDN (Edge Cache)                    │
├─────────────────────────────────────┤
│ Application Cache (Redis)           │
├─────────────────────────────────────┤
│ Database Query Cache                │
├─────────────────────────────────────┤
│ Database                            │ Slowest
└─────────────────────────────────────┘

Caching Patterns:
• Cache-Aside: App manages cache
• Read-Through: Cache manages reads
• Write-Through: Cache manages writes
• Write-Behind: Async write to DB
```

---

## API Design Principles

### 1. Design for Backward Compatibility

```
✅ Safe changes:
• Add new optional fields
• Add new endpoints
• Add new response fields

❌ Breaking changes:
• Remove fields
• Change field types
• Rename fields
• Change endpoint paths

Handle with:
• API versioning (v1, v2)
• Feature flags
• Gradual rollout
```

### 2. Rate Limiting

Protect your system from abuse.

```
Rate Limiting Algorithms:

Token Bucket:
• Tokens added at fixed rate
• Requests consume tokens
• Allows bursts up to bucket size

Sliding Window:
• Count requests in time window
• Smooth rate limiting
• More memory intensive

Fixed Window:
• Reset counter each window
• Simple but allows burst at edges
```

### 3. Pagination

Never return unbounded lists.

```
Offset Pagination:
GET /users?offset=100&limit=20
• Simple but slow for large offsets
• Inconsistent during writes

Cursor Pagination:
GET /users?cursor=abc123&limit=20
• Consistent results
• Efficient for large datasets
• Cursor encodes position
```

---

## Security Principles

### 1. Defense in Depth

```
┌─────────────────────────────────────────┐
│ Layer 1: Network (Firewall, WAF)        │
├─────────────────────────────────────────┤
│ Layer 2: Authentication (OAuth, JWT)    │
├─────────────────────────────────────────┤
│ Layer 3: Authorization (RBAC, ABAC)     │
├─────────────────────────────────────────┤
│ Layer 4: Input Validation               │
├─────────────────────────────────────────┤
│ Layer 5: Encryption (TLS, at-rest)      │
├─────────────────────────────────────────┤
│ Layer 6: Audit Logging                  │
└─────────────────────────────────────────┘
```

### 2. Principle of Least Privilege

```
❌ Bad: Service has admin access to entire database
✅ Good: Service has read access to only tables it needs
```

### 3. Encrypt Everything

```
In Transit: TLS 1.3 everywhere
At Rest: AES-256 for stored data
Secrets: Use secret management (Vault, AWS Secrets Manager)
```

---

## Summary Table

| Principle | When to Apply | Trade-off |
|-----------|---------------|-----------|
| KISS | Always start here | May need refactor later |
| Design for Failure | Distributed systems | More complexity |
| Horizontal Scaling | Need to scale | Requires stateless design |
| Stateless Services | Scaling, reliability | External state management |
| Single Source of Truth | Data consistency | More network calls |
| Idempotency | Distributed operations | Need to track operations |
| Event-Driven | Decoupling services | Eventual consistency |
| Circuit Breaker | Service dependencies | Fallback logic needed |
| Backpressure | High throughput | May drop/reject requests |
| Rate Limiting | Public APIs | May block legitimate users |

---

## Further Reading
- [Designing Data-Intensive Applications](https://dataintensive.net/)
- [Release It!](https://pragprog.com/titles/mnee2/release-it-second-edition/)
- [Building Microservices](https://www.oreilly.com/library/view/building-microservices-2nd/9781492034018/)
