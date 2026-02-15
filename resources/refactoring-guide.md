# System Refactoring Guide

A practical guide to identifying design issues and refactoring existing systems safely.

---

## When to Refactor

### Warning Signs (Code Smells at Scale)

```
┌─────────────────────────────────────────────────────────────┐
│                    REFACTORING TRIGGERS                      │
├─────────────────────────────────────────────────────────────┤
│ Performance                                                  │
│ ├── Response times degrading over time                      │
│ ├── Database queries taking > 1 second                      │
│ ├── Cache hit rate < 80%                                    │
│ └── CPU/Memory always > 70%                                 │
├─────────────────────────────────────────────────────────────┤
│ Scalability                                                  │
│ ├── Can't add more servers due to state                     │
│ ├── Database is the bottleneck                              │
│ ├── Hitting infrastructure limits                           │
│ └── Traffic spikes cause failures                           │
├─────────────────────────────────────────────────────────────┤
│ Reliability                                                  │
│ ├── Frequent outages                                        │
│ ├── Single failures cause cascade                           │
│ ├── No graceful degradation                                 │
│ └── Recovery takes hours                                    │
├─────────────────────────────────────────────────────────────┤
│ Development Velocity                                         │
│ ├── Simple changes take weeks                               │
│ ├── High regression rate                                    │
│ ├── Fear of deploying                                       │
│ └── Onboarding takes months                                 │
└─────────────────────────────────────────────────────────────┘
```

---

## Refactoring Patterns

### 1. Strangler Fig Pattern

Replace system incrementally without big-bang rewrite.

```
Phase 1: Facade in front of legacy
┌─────────┐     ┌─────────┐     ┌─────────────┐
│ Client  │────►│ Facade  │────►│ Legacy Sys  │
└─────────┘     └─────────┘     └─────────────┘

Phase 2: Route some traffic to new
┌─────────┐     ┌─────────┐
│ Client  │────►│ Facade  │
└─────────┘     └────┬────┘
                     │
           ┌─────────┴─────────┐
           ▼                   ▼
    ┌─────────────┐     ┌─────────────┐
    │   Legacy    │     │   New Sys   │
    │ (feature A) │     │ (feature B) │
    └─────────────┘     └─────────────┘

Phase 3: Complete migration
┌─────────┐     ┌─────────┐     ┌─────────────┐
│ Client  │────►│ Facade  │────►│   New Sys   │
└─────────┘     └─────────┘     └─────────────┘
                                 (Legacy retired)
```

**Implementation Steps:**
1. Add API gateway/facade
2. Migrate one feature at a time
3. Route traffic gradually (canary)
4. Monitor and validate each step
5. Deprecate legacy components

---

### 2. Branch by Abstraction

Refactor internal components while maintaining functionality.

```python
# Step 1: Create abstraction
class PaymentProcessor(ABC):
    @abstractmethod
    def process(self, payment): pass

# Step 2: Wrap legacy implementation
class LegacyPaymentProcessor(PaymentProcessor):
    def process(self, payment):
        return old_payment_system.charge(payment)

# Step 3: Create new implementation
class NewPaymentProcessor(PaymentProcessor):
    def process(self, payment):
        return stripe_client.charge(payment)

# Step 4: Feature flag to switch
def get_processor(user_id):
    if feature_flags.is_enabled("new_payment", user_id):
        return NewPaymentProcessor()
    return LegacyPaymentProcessor()
```

---

### 3. Database Migration Patterns

#### Pattern A: Dual Write (Use with Caution)

```
┌─────────┐     ┌─────────┐     ┌─────────────┐
│   App   │────►│ Write   │────►│   Old DB    │
└─────────┘     │ Both    │────►│   New DB    │
                └─────────┘     └─────────────┘

Risks:
• One write may fail
• Transaction integrity
• Race conditions
```

#### Pattern B: Change Data Capture (Recommended)

```
┌─────────┐     ┌─────────┐     ┌─────────────┐
│   App   │────►│ Old DB  │────►│  Debezium   │
└─────────┘     └─────────┘     │  (CDC)      │
                                └──────┬──────┘
                                       │
                                       ▼
                                ┌─────────────┐
                                │    Kafka    │
                                └──────┬──────┘
                                       │
                                       ▼
                                ┌─────────────┐
                                │   New DB    │
                                └─────────────┘
```

#### Pattern C: Expand-Contract Migration

```
Phase 1: Expand (add new columns/tables)
┌─────────────────────────────────────┐
│ users                                │
│ ├── id                              │
│ ├── name                            │
│ ├── email_old (varchar)             │ ← Keep old
│ └── email_new (with validation)     │ ← Add new
└─────────────────────────────────────┘

Phase 2: Migrate data
UPDATE users SET email_new = email_old;

Phase 3: Contract (remove old)
ALTER TABLE users DROP COLUMN email_old;
ALTER TABLE users RENAME email_new TO email;
```

---

### 4. Monolith to Microservices

#### Step-by-Step Extraction

```
Step 1: Identify boundaries (Domain-Driven Design)
┌─────────────────────────────────────┐
│           Monolith                   │
│ ┌─────────┐ ┌─────────┐ ┌─────────┐ │
│ │  User   │ │  Order  │ │ Payment │ │ ← Bounded contexts
│ └─────────┘ └─────────┘ └─────────┘ │
└─────────────────────────────────────┘

Step 2: Create internal APIs between modules
┌─────────────────────────────────────┐
│           Monolith                   │
│ ┌─────────┐   API    ┌─────────┐   │
│ │  User   │◄────────►│  Order  │   │ ← Internal APIs
│ └─────────┘          └─────────┘   │
└─────────────────────────────────────┘

Step 3: Extract one service
┌─────────────────────┐    ┌─────────────┐
│      Monolith       │    │   Payment   │
│ ┌─────────┐ ┌─────┐ │◄──►│   Service   │
│ │  User   │ │Order│ │    └─────────────┘
│ └─────────┘ └─────┘ │
└─────────────────────┘

Step 4: Repeat until done
```

#### What to Extract First

| Priority | Characteristic |
|----------|---------------|
| High | Changes frequently, different team |
| High | Different scaling needs |
| High | Technology mismatch |
| Medium | Clear domain boundary |
| Low | Tightly coupled to core |
| Low | Low change frequency |

---

## Common Refactoring Scenarios

### Scenario 1: Database is the Bottleneck

**Symptoms:**
- Slow queries
- Connection pool exhausted
- Lock contention

**Solutions:**

```
┌─────────────────────────────────────────────────────────────┐
│                    BEFORE                                    │
│ ┌─────────┐     ┌─────────────────────────────┐             │
│ │   App   │────►│        Single DB            │             │
│ └─────────┘     │ (all reads + all writes)    │             │
│                 └─────────────────────────────┘             │
├─────────────────────────────────────────────────────────────┤
│                    AFTER                                     │
│                                                              │
│ Option 1: Read Replicas                                      │
│ ┌─────────┐     ┌──────────┐                                │
│ │   App   │────►│ Primary  │ (writes only)                  │
│ │         │     └──────────┘                                │
│ │         │────►│ Replica 1│ (reads)                        │
│ │         │────►│ Replica 2│ (reads)                        │
│ └─────────┘     └──────────┘                                │
│                                                              │
│ Option 2: Caching Layer                                      │
│ ┌─────────┐     ┌─────────┐     ┌─────────┐                │
│ │   App   │────►│  Redis  │────►│   DB    │                │
│ └─────────┘     │ (80% hit│     │         │                │
│                 └─────────┘     └─────────┘                │
│                                                              │
│ Option 3: Sharding                                           │
│ ┌─────────┐     ┌─────────┐                                 │
│ │   App   │────►│ Shard 0 │ (users A-M)                     │
│ │         │────►│ Shard 1 │ (users N-Z)                     │
│ └─────────┘     └─────────┘                                 │
└─────────────────────────────────────────────────────────────┘
```

---

### Scenario 2: Stateful to Stateless

**Symptoms:**
- Can't scale horizontally
- Sticky sessions required
- State lost on server restart

**Solution:**

```
BEFORE (Stateful):
┌─────────────────────────────────────────────────┐
│  Server 1        Server 2        Server 3       │
│ ┌─────────┐    ┌─────────┐    ┌─────────┐      │
│ │Sessions │    │Sessions │    │Sessions │      │
│ │for A,B  │    │for C,D  │    │for E,F  │      │
│ └─────────┘    └─────────┘    └─────────┘      │
│                                                 │
│ Problem: Server 1 dies → Users A,B lose state  │
└─────────────────────────────────────────────────┘

AFTER (Stateless):
┌─────────────────────────────────────────────────┐
│  Server 1        Server 2        Server 3       │
│ ┌─────────┐    ┌─────────┐    ┌─────────┐      │
│ │ No local│    │ No local│    │ No local│      │
│ │  state  │    │  state  │    │  state  │      │
│ └────┬────┘    └────┬────┘    └────┬────┘      │
│      │              │              │            │
│      └──────────────┼──────────────┘            │
│                     ▼                           │
│              ┌─────────────┐                    │
│              │   Redis     │ (all sessions)    │
│              │  Cluster    │                    │
│              └─────────────┘                    │
│                                                 │
│ Benefit: Any server can handle any request     │
└─────────────────────────────────────────────────┘
```

---

### Scenario 3: Synchronous to Asynchronous

**Symptoms:**
- Long response times
- Cascade failures
- Timeouts during peaks

**Solution:**

```
BEFORE (Synchronous):
┌─────────────────────────────────────────────────┐
│ User Request                                     │
│     │                                           │
│     ▼                                           │
│ ┌─────────┐                                     │
│ │ Process │                                     │
│ │  Order  │──► 500ms                            │
│ └────┬────┘                                     │
│      │                                          │
│ ┌────▼────┐                                     │
│ │ Charge  │                                     │
│ │ Payment │──► 800ms                            │
│ └────┬────┘                                     │
│      │                                          │
│ ┌────▼────┐                                     │
│ │  Send   │                                     │
│ │  Email  │──► 300ms                            │
│ └────┬────┘                                     │
│      │                                          │
│      ▼                                          │
│ Response ──────► Total: 1600ms                  │
└─────────────────────────────────────────────────┘

AFTER (Asynchronous):
┌─────────────────────────────────────────────────┐
│ User Request                                     │
│     │                                           │
│     ▼                                           │
│ ┌─────────┐     ┌─────────┐                    │
│ │ Process │────►│  Queue  │                    │
│ │  Order  │     └────┬────┘                    │
│ └────┬────┘          │                          │
│      │               ├──► Payment Worker        │
│      ▼               └──► Email Worker          │
│ Response ──► 200ms                              │
│                                                 │
│ Background:                                     │
│ Payment & Email processed asynchronously        │
│ User notified via WebSocket/push                │
└─────────────────────────────────────────────────┘
```

---

## Refactoring Safely

### 1. Parallel Run (Shadow Testing)

```
┌─────────┐
│ Request │
└────┬────┘
     │
     ├────────────────────────┐
     ▼                        ▼
┌─────────┐              ┌─────────┐
│ Current │              │   New   │
│ System  │              │ System  │
└────┬────┘              └────┬────┘
     │                        │
     ▼                        ▼
  Response              Compare Results
  to User               (log differences)
```

### 2. Feature Flags

```python
def process_order(order):
    if feature_flags.is_enabled("new_order_system", order.user_id):
        return new_order_service.process(order)
    else:
        return legacy_order_service.process(order)

# Gradual rollout:
# Day 1: 1% of users
# Day 3: 10% of users
# Day 7: 50% of users
# Day 14: 100% of users
```

### 3. Monitoring During Migration

```
Key Metrics to Watch:
├── Error rates (should stay same or improve)
├── Latency p50, p95, p99
├── Throughput
├── Resource utilization
├── Business metrics (conversion, etc.)
└── Customer complaints
```

---

## Refactoring Decision Framework

```
Should We Refactor?

1. Is there a clear business benefit?
   NO → Don't refactor
   YES → Continue

2. Can we do it incrementally?
   NO → Reconsider approach
   YES → Continue

3. Do we have test coverage?
   NO → Add tests first
   YES → Continue

4. Do we have monitoring?
   NO → Add monitoring first
   YES → Continue

5. Can we rollback quickly?
   NO → Create rollback plan
   YES → Proceed with refactoring
```

---

## Anti-Patterns to Avoid

| Anti-Pattern | Why It's Bad | Better Approach |
|--------------|--------------|-----------------|
| Big Bang Rewrite | High risk, long time | Strangler fig, incremental |
| Refactor Everything | Scope creep | Focus on pain points |
| No Tests | Can't verify correctness | Test before refactor |
| No Monitoring | Can't detect issues | Instrument first |
| No Rollback Plan | Stuck if things break | Always have escape route |

---

## Checklist Before Refactoring

- [ ] Clear problem statement and success criteria
- [ ] Test coverage for affected code
- [ ] Monitoring and alerting in place
- [ ] Rollback procedure documented
- [ ] Team aligned on approach
- [ ] Incremental migration plan
- [ ] Feature flags ready
- [ ] Communication plan for stakeholders

---

*Refactoring is a journey, not a destination. Take it one step at a time.*
