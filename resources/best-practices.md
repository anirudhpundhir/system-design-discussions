# System Design Best Practices

A comprehensive guide to designing robust, scalable, and maintainable systems.

---

## Architecture Best Practices

### 1. Start Simple, Evolve Incrementally

```
❌ Day 1: "We need microservices, Kubernetes, event sourcing, CQRS..."
✅ Day 1: "Monolith with clear module boundaries"
✅ Day 100: "Extract high-traffic service"
✅ Day 365: "Microservices where it makes sense"
```

### 2. Design for Failure

| Component | Failure Mitigation |
|-----------|-------------------|
| Server | Auto-scaling groups, health checks |
| Database | Replication, automated failover |
| Network | Retry with backoff, circuit breakers |
| Region | Multi-region deployment |
| Deployment | Blue-green, canary releases |

### 3. Stateless Services

```
Stateful:
┌─────────────────────────────┐
│ Server A (has user session) │ ← If this dies, session lost
└─────────────────────────────┘

Stateless:
┌─────────────────────────────┐
│ Server A/B/C (any can serve)│
└──────────────┬──────────────┘
               │
         ┌─────▼─────┐
         │   Redis   │ ← Session stored externally
         └───────────┘
```

### 4. Loose Coupling, High Cohesion

```
Tight Coupling (Bad):
Service A ──sync──► Service B ──sync──► Service C
           If B fails, A fails

Loose Coupling (Good):
Service A ──► Queue ──► Service B
                   └──► Service C
              Services operate independently
```

---

## Data Best Practices

### 1. Single Source of Truth

```
✅ Good:
User Service owns user data
Other services query User Service or subscribe to events

❌ Bad:
User data copied to Order, Billing, Shipping services
Each service updates its copy independently → inconsistency
```

### 2. Database Per Service

```
┌─────────────┐    ┌─────────────┐    ┌─────────────┐
│ User Service│    │Order Service│    │Inventory Svc│
└──────┬──────┘    └──────┬──────┘    └──────┬──────┘
       │                  │                   │
  ┌────▼────┐        ┌────▼────┐        ┌────▼────┐
  │ User DB │        │Order DB │        │Inventory│
  └─────────┘        └─────────┘        └─────────┘

Benefits:
• Independent scaling
• Technology flexibility
• Failure isolation
```

### 3. Optimize Data Access Patterns

| Pattern | Optimization |
|---------|-------------|
| Read-heavy | Caching, read replicas |
| Write-heavy | Async writes, batching |
| Hot data | In-memory cache |
| Cold data | Archive to cheap storage |
| Time-series | Partitioning by time |

---

## API Best Practices

### 1. Design for Backward Compatibility

```
Safe Changes:
✅ Add new optional fields
✅ Add new endpoints
✅ Add new enum values

Breaking Changes (Avoid):
❌ Remove fields
❌ Change field types
❌ Rename endpoints
❌ Change required fields
```

### 2. Idempotency

```
POST /api/orders
{
    "idempotency_key": "order-12345-attempt-1",
    "items": [...]
}

First call: Creates order, returns order_id
Retry calls: Returns same order_id (no duplicate)
```

### 3. Pagination

```
❌ Bad: GET /users → Returns 1 million users

✅ Good: GET /users?cursor=abc123&limit=100
Response:
{
    "data": [...],
    "next_cursor": "def456"
}
```

### 4. Rate Limiting

```
Response Headers:
X-RateLimit-Limit: 1000
X-RateLimit-Remaining: 998
X-RateLimit-Reset: 1640000000

On limit exceeded:
HTTP 429 Too Many Requests
Retry-After: 60
```

---

## Performance Best Practices

### 1. Cache Strategically

```
Cache Hierarchy:
┌────────────────────────────────┐
│ L1: Browser/CDN (edge)        │ ~1-10ms
├────────────────────────────────┤
│ L2: Application Cache (Redis)  │ ~1-5ms
├────────────────────────────────┤
│ L3: Database Query Cache       │ ~10-50ms
├────────────────────────────────┤
│ L4: Database                   │ ~50-200ms
└────────────────────────────────┘
```

### 2. Async Everything Possible

```
Synchronous (Blocking):
User Request → Process A → Process B → Process C → Response
                                                    ↑
                                              User waits

Asynchronous:
User Request → Queue Job → Response (immediate)
               ↓
           Background: Process A → B → C → Notify user
```

### 3. Connection Pooling

```
Without Pooling:
Each request → New connection → Close connection
Overhead: ~10-50ms per request

With Pooling:
Request → Get from pool → Return to pool
Overhead: ~1ms per request
```

---

## Reliability Best Practices

### 1. Circuit Breaker Pattern

```
┌─────────────────────────────────────────────────┐
│                Circuit Breaker                   │
│                                                  │
│  CLOSED ──failures──► OPEN ──timeout──► HALF-OPEN
│    ▲                                      │
│    └──────────success────────────────────┘
│                                                  │
│  CLOSED: Normal operation                        │
│  OPEN: Reject requests, return fallback          │
│  HALF-OPEN: Allow test request                   │
└─────────────────────────────────────────────────┘
```

### 2. Retry with Exponential Backoff

```python
def retry_with_backoff(func, max_retries=5):
    for attempt in range(max_retries):
        try:
            return func()
        except TransientError:
            wait_time = (2 ** attempt) + random.uniform(0, 1)
            time.sleep(wait_time)
    raise MaxRetriesExceeded()
```

### 3. Graceful Degradation

```
Full Service:
User → API → Recommendation Engine → Personalized results

Degraded (Recommendation down):
User → API → Fallback: Popular items
             ↓
        Log: "Recommendation service unavailable"
```

---

## Security Best Practices

### 1. Defense in Depth

```
Layer 1: Network (Firewall, WAF)
Layer 2: Authentication (OAuth, MFA)
Layer 3: Authorization (RBAC, ABAC)
Layer 4: Input Validation (Sanitize all input)
Layer 5: Encryption (TLS in transit, AES at rest)
Layer 6: Audit Logging (Track all access)
```

### 2. Principle of Least Privilege

```
❌ Service has: Admin access to all databases
✅ Service has: Read-only access to tables it needs
```

### 3. Secrets Management

```
❌ Secrets in code: password = "secret123"
❌ Secrets in env vars: export DB_PASSWORD=secret123

✅ Secrets in vault:
   vault kv get secret/database/password
   Auto-rotation, audit logging, access control
```

---

## Observability Best Practices

### 1. The Three Pillars

```
METRICS (What's happening?)
├── Request rate
├── Error rate
├── Latency percentiles
└── Resource utilization

LOGS (Why did it happen?)
├── Structured JSON
├── Request ID correlation
├── Log levels (INFO, WARN, ERROR)
└── Centralized aggregation

TRACES (Where did it happen?)
├── Distributed tracing
├── Service dependencies
├── Latency breakdown
└── Error propagation
```

### 2. Alert on Symptoms, Not Causes

```
❌ Bad Alert: "CPU > 80%"
   (May be normal, causes alert fatigue)

✅ Good Alert: "Error rate > 1% for 5 minutes"
   (User-facing impact, actionable)
```

---

## Deployment Best Practices

### 1. Blue-Green Deployment

```
Before:
[Production] ← All traffic

During:
[Blue (old)] ← All traffic
[Green (new)] ← Ready, tested

After:
[Blue (old)] ← 0% traffic
[Green (new)] ← All traffic (rollback if issues)
```

### 2. Canary Deployment

```
Phase 1: [Canary] ← 1% traffic (test)
         [Stable] ← 99% traffic

Phase 2: [Canary] ← 10% traffic (if healthy)
         [Stable] ← 90% traffic

Phase 3: [New Stable] ← 100% traffic
```

### 3. Feature Flags

```python
if feature_flags.is_enabled("new_checkout", user_id):
    return new_checkout_flow(request)
else:
    return old_checkout_flow(request)
```

---

## Summary Checklist

### Before Building
- [ ] Requirements clearly defined
- [ ] Scale estimates calculated
- [ ] Technology choices justified
- [ ] Trade-offs documented

### During Building
- [ ] Stateless services
- [ ] Idempotent operations
- [ ] Error handling everywhere
- [ ] Observability from start

### Before Production
- [ ] Load testing done
- [ ] Failure scenarios tested
- [ ] Monitoring and alerts set
- [ ] Runbooks prepared

### In Production
- [ ] Regular capacity review
- [ ] Chaos engineering
- [ ] Performance profiling
- [ ] Security audits

---

*Best practices evolve. Challenge them, adapt them, improve them.*
