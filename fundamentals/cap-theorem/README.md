# CAP Theorem & Consistency Models

## What is CAP Theorem?

The CAP theorem states that a distributed data system can provide at most **two of three** guarantees simultaneously:

```
                    Consistency
                        ▲
                       / \
                      /   \
                     / CP  \
                    /       \
                   /    CA   \
                  /___________\
        Availability ◄────► Partition Tolerance
```

### The Three Guarantees

| Property | Definition | Example |
|----------|------------|---------|
| **Consistency (C)** | Every read receives the most recent write | All nodes see same data at same time |
| **Availability (A)** | Every request receives a response | System always responds, even if stale |
| **Partition Tolerance (P)** | System works despite network failures | Nodes continue operating when disconnected |

---

## Why Not All Three?

In a distributed system, **network partitions are inevitable**. When a partition occurs:

```
┌─────────────┐         X         ┌─────────────┐
│   Node A    │◄───── Network ────►│   Node B    │
│  (data: 1)  │       Failure      │  (data: 1)  │
└─────────────┘                    └─────────────┘

Client writes "data: 2" to Node A...

┌─────────────┐         X         ┌─────────────┐
│   Node A    │                    │   Node B    │
│  (data: 2)  │                    │  (data: 1)  │
└─────────────┘                    └─────────────┘

Now what?
- Return stale data from B? → Sacrifice Consistency (AP)
- Reject requests to B? → Sacrifice Availability (CP)
- Wait for partition heal? → Sacrifice both during partition
```

---

## CAP in Practice

### CP Systems (Consistency + Partition Tolerance)
Sacrifice availability during partitions.

| System | Behavior |
|--------|----------|
| **MongoDB** (with majority writes) | Rejects writes if can't reach majority |
| **HBase** | Returns error if region server unavailable |
| **Zookeeper** | Stops accepting writes without quorum |
| **etcd** | Requires majority for consensus |

**Use when:** Data correctness is critical (financial transactions, inventory)

### AP Systems (Availability + Partition Tolerance)
Sacrifice consistency during partitions.

| System | Behavior |
|--------|----------|
| **Cassandra** | Eventual consistency, always writable |
| **DynamoDB** | Can return stale reads |
| **CouchDB** | Eventual consistency with conflict resolution |
| **Riak** | Available, uses vector clocks for conflicts |

**Use when:** Availability is critical (social media, caching)

### CA Systems (Consistency + Availability)
Only works without network partitions (single node).

| System | Behavior |
|--------|----------|
| **Traditional RDBMS** | Single-node, ACID transactions |
| **PostgreSQL** (single) | No partition tolerance needed |

**Use when:** System doesn't need to be distributed

---

## PACELC Theorem

CAP doesn't cover normal operation. PACELC extends it:

```
IF Partition:
    Choose Availability OR Consistency (PAC)
ELSE:
    Choose Latency OR Consistency (ELC)
```

| System | During Partition | Normal Operation |
|--------|------------------|------------------|
| Cassandra | PA | EL (low latency, eventual consistency) |
| MongoDB | PC | EC (consistency over latency) |
| DynamoDB | PA | EL |
| Spanner | PC | EC |

---

## Consistency Models

### Strong Consistency
- All nodes see the same data at the same time
- After a write completes, all reads return the new value
- Higher latency, requires coordination

```
Write(x=2) ──────────────────► Acknowledged
                    │
Read(x) ◄───────────┘ Returns 2 (guaranteed)
```

### Eventual Consistency
- Given enough time, all nodes will converge
- Reads may return stale data temporarily
- Lower latency, higher availability

```
Write(x=2) ──────────────────► Acknowledged
                    │
Read(x) ◄───────────┘ May return 1 or 2

... time passes ...

Read(x) ◄──────────────────── Returns 2 (eventually)
```

### Causal Consistency
- Preserves cause-and-effect relationships
- If A causes B, everyone sees A before B
- Doesn't guarantee order of unrelated operations

```
User A: Posts message M1
User A: Replies to M1 with M2

All users will see M1 before M2 (causal order preserved)
```

### Read-Your-Writes Consistency
- User always sees their own writes
- Others may see stale data
- Common in web applications

```
User writes profile update
User's next read shows the update
Other users may see old profile (temporarily)
```

---

## Consistency in System Design Interviews

### Questions to Ask
1. "How critical is data consistency for this feature?"
2. "Can users tolerate seeing slightly stale data?"
3. "What's the acceptable consistency window?"

### Decision Framework

```
Is immediate consistency required?
│
├─ YES → Do we need high availability?
│        │
│        ├─ YES → Impossible in distributed system
│        │        Consider: hybrid approach, sync replication
│        │
│        └─ NO → Use CP system (MongoDB, HBase)
│
└─ NO → What's acceptable staleness?
         │
         ├─ Seconds → Eventual consistency (Cassandra)
         │
         ├─ User's session → Read-your-writes (sticky sessions)
         │
         └─ Causal order only → Causal consistency
```

### Real-World Examples

| Use Case | Consistency Need | Solution |
|----------|------------------|----------|
| Bank transfer | Strong | ACID database, 2PC |
| Social media feed | Eventual | Cassandra, eventual sync |
| Shopping cart | Session | Read-your-writes |
| Message ordering | Causal | Vector clocks |
| Analytics | Eventual | Eventual, batch processing |
| Inventory count | Strong or compensate | Strong for reserve, eventual for display |

---

## Implementation Patterns

### Quorum Reads/Writes
```
N = Total replicas
W = Write acknowledgments needed
R = Read nodes queried

Strong consistency: W + R > N

Example (N=3):
- W=2, R=2 → Strong (2+2=4 > 3)
- W=1, R=1 → Eventual (1+1=2 < 3)
```

### Read Repair
```
On read:
1. Query multiple replicas
2. Return most recent version
3. Asynchronously update stale replicas
```

### Anti-Entropy
```
Background process:
1. Periodically compare replicas
2. Synchronize differences
3. Use Merkle trees for efficient comparison
```

---

## Summary

| Concept | Key Point |
|---------|-----------|
| CAP | Choose 2 of 3; P is usually mandatory |
| PACELC | Also consider latency vs consistency trade-off |
| Strong Consistency | Safe but slow |
| Eventual Consistency | Fast but temporarily stale |
| Real Systems | Often use hybrid approaches by feature |

---

## Further Reading
- [CAP Twelve Years Later](https://www.infoq.com/articles/cap-twelve-years-later-how-the-rules-have-changed/)
- [Designing Data-Intensive Applications - Chapter 9](https://dataintensive.net/)
- [Jepsen Analyses](https://jepsen.io/analyses)
