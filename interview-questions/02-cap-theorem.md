# Interview Questions — Latency, Throughput & CAP Theorem

> Topic 2 | Phase 1: Foundations

---

## Q1. What is latency and how is it different from throughput? (Easy)

**Answer:**

**Latency** is the time taken to complete **one single operation** — from request sent to response received.
Example: "This API call takes 120ms."

**Throughput** is how many operations the system handles **per second**.
Example: "This API handles 5,000 requests per second."

They are related but different:
- A single-lane road can be fast (low latency — each car moves quickly) but have low throughput (only a few cars pass per minute)
- A multi-lane highway has high throughput (many cars per minute) but individual cars may be slower due to congestion

In system design: you can have high throughput with high latency (batching — process 10,000 records at once but each batch takes 2 seconds) or low throughput with low latency (process each record in 1ms, but only one at a time).

---

## Q2. Why do we use P99 instead of average latency? (Easy)

**Answer:**

Average latency hides outliers. Consider 100 requests:
```
98 requests → 20ms
2 requests  → 5,000ms (something went wrong — timeout, retry, slow query)

Average = (98×20 + 2×5000) / 100 = 119ms  ← looks fine
P99     = 5,000ms                           ← 1 in 100 users waits 5 seconds
```

At scale: if your system handles 10,000 req/sec and P99 is 5 seconds, that's 100 users per second having a terrible experience. Average masks this completely.

**Rule:** Design to meet P99 SLA targets. SLAs like "99% of requests must complete within 200ms" are P99 SLAs. Optimizing average only helps the easy cases.

---

## Q3. Explain CAP theorem. Give a real-world example of CP and AP. (Medium)

**Answer:**

CAP theorem states a distributed system can only guarantee 2 of:
- **C** — Consistency: all nodes return the same (most recent) data
- **A** — Availability: every request gets a response (no errors)
- **P** — Partition Tolerance: system works even when network between nodes fails

Since network failures are unavoidable, **P is always required**. The real choice is **CP vs AP**.

**CP Example — Online Banking:**
You transfer £500. The bank's East US and West US nodes lose network connectivity (partition). The bank stops accepting transfers on the West US node and returns an error: "Service temporarily unavailable."

That's CP — the bank chose consistency (never show wrong balance) over availability (some users get errors).

**AP Example — Instagram Feed:**
Your friend posts a photo. Due to a network partition, the Tokyo datacenter doesn't receive it immediately. A user in Tokyo sees a feed that's 10 seconds behind.

That's AP — Instagram chose availability (always show a feed, never error) over consistency (slightly stale feed for a moment).

---

## Q4. Why can't a CA system exist in a real distributed environment? (Medium)

**Answer:**

A CA system sacrifices Partition Tolerance — meaning it assumes the network never fails. But in any distributed system (multiple servers connected by a network), network failures DO happen:
- Network cables get cut
- Cloud provider has a regional incident
- A server's network card fails
- A firewall rule blocks traffic

If you design a system that cannot tolerate network partitions, the moment a partition occurs, your system has no defined behaviour — it either breaks or behaves unpredictably.

**The only true CA system is a single-node system** — a single database server has no network partition between nodes because there's only one node. The moment you add a second server, you have a distributed system and P becomes mandatory.

This is why MySQL or PostgreSQL on a single server is technically "CA" — but the moment you add replication, you need to choose CP or AP.

---

## Q5. What is the difference between eventual consistency and strong consistency? (Medium)

**Answer:**

**Strong Consistency:**
After a write completes, every subsequent read from ANY node returns the new value. No exceptions.

```
Write: balance = $500 (confirmed)
Read from any node, 1ms later: $500 ✓
```

Cost: Higher latency (must wait for all replicas to acknowledge), lower availability during partitions.
Use for: Banking, payments, stock trading — anywhere wrong data causes real harm.

**Eventual Consistency:**
After a write, replicas eventually converge to the same value. Reads may temporarily return old data.

```
Write to Node 1: balance = $500
Read from Node 2 (50ms later): might return $1,000 (stale)
Read from Node 2 (2 seconds later): returns $500 ✓ (synced)
```

Cost: Briefly stale data after writes.
Benefit: Much faster writes, always available.
Use for: Social feeds, DNS, product recommendations, shopping carts — briefly stale is acceptable.

**Interview tip:** The question "would you use eventual consistency here?" is really asking "can the user tolerate seeing stale data for a brief period, and is that a better trade-off than extra latency or occasional errors?"

---

## Q6. When would you choose AP over CP? Concrete example. (Medium)

**Answer:**

Choose AP when:
1. **Stale data for a brief period is acceptable** (users won't notice or won't be harmed)
2. **System availability is more important than perfect accuracy**
3. **Write throughput is critical** and you can't afford to wait for all replicas

**Concrete example — Netflix (Cassandra):**
Netflix shows you movie recommendations. These are generated by a machine learning model that runs periodically. If a network partition means your Tokyo data center shows recommendations that are 30 seconds old instead of real-time, nobody notices or cares. A 30-second stale recommendation is infinitely better than Netflix showing an error page.

Netflix explicitly chose Cassandra (AP) for this reason. They need 99.999% availability globally. They'd rather serve slightly stale data than go down during a partition.

**Counter-example — don't choose AP:**
Your system tracks whether a user's email is verified (to allow login). If you use AP and a partition occurs, an unverified user might briefly be shown as verified on some nodes. This allows unauthorized access. This should be CP.

**Rule of thumb:**
- User-facing content (feeds, recommendations, search results) → AP
- Security, money, or correctness-critical data → CP

---

## Q7. What is a quorum and how does W + R > N guarantee strong consistency? (Hard)

**Answer:**

**Quorum variables:**
- N = number of replicas
- W = number of replicas that must acknowledge a WRITE for it to succeed
- R = number of replicas that must respond to a READ

**The formula: W + R > N guarantees strong consistency.**

**Why?**

With N=3, W=2, R=2: (W + R = 4 > N = 3)

When you write:
- At least 2 of 3 nodes receive and store the new data

When you read:
- You ask at least 2 of 3 nodes for the data

By pigeonhole principle: the set of 2 nodes you wrote to and the set of 2 nodes you read from MUST share at least 1 node (since 2+2 > 3). That overlapping node has the latest data, so you always get the latest write.

**Common configurations:**
```
N=3, W=1, R=1: W+R=2 ≤ 3 → eventual consistency (fast, may return stale)
N=3, W=2, R=2: W+R=4 > 3 → strong consistency (moderate speed)
N=3, W=3, R=1: W+R=4 > 3 → strong consistency (write waits for all, read is fast)
N=3, W=1, R=3: W+R=4 > 3 → strong consistency (write is fast, read waits for all)
```

**Interview answer:** "For a payment system, I'd use N=3, W=2, R=2 in Cassandra. W+R=4 > N=3 guarantees strong consistency. We sacrifice some write latency (must wait for 2 replicas) but guarantee that reads always see the latest write."

---

## Q8. Your payment API is slow. Should you switch to eventual consistency to speed it up? (Hard)

**Answer:**

**No — and here's why this is a dangerous trade-off:**

Payment systems have two critical properties that eventual consistency directly violates:

**1. Double-spend prevention:**
User has $100. They quickly tap "Pay" twice (or two requests race).

With strong consistency:
- First payment writes $0 balance
- Second payment reads $0, rejects the payment ✓

With eventual consistency:
- First payment writes to Node 1 (balance = $0)
- Second payment reads from Node 2 (not yet synced, still shows $100) → approves! ✗
- User paid twice with $100 that didn't exist

**2. Regulatory compliance:**
Financial systems must show accurate balances. Banking regulations require it. A bank that shows you £500 when you have £200 could face massive legal consequences.

**Better ways to speed up the payment API without sacrificing consistency:**
- Add caching for read-heavy data (product prices, user profile) — not the balance
- Use strong consistency only for the balance write, eventual for everything else
- Optimize the database query itself (add index, reduce joins)
- Use connection pooling to reduce DB connection overhead

**Rule:** Eventual consistency is a trade-off of accuracy for speed. Only make this trade where inaccuracy is genuinely acceptable. For money: it never is.

---

## Q9. What consistency model does Azure Cosmos DB offer and why is it special? (Medium)

**Answer:**

Cosmos DB is unique — it's the only major database that offers **5 tuneable consistency levels**, configurable per-request:

| Level | Description | Use case |
|-------|-------------|----------|
| **Strong** | Every read returns latest write. Highest latency. | Financial balances |
| **Bounded Staleness** | Data is at most X versions or T seconds behind | Real-time leaderboards |
| **Session** | Consistent within a session (you always read your own writes) | User profile pages |
| **Consistent Prefix** | Reads never see out-of-order writes | Log/event viewers |
| **Eventual** | Highest availability, lowest latency, may return stale | Activity feeds, recommendations |

**Why it's special:**
Most databases force you to pick one consistency model for the whole system. Cosmos DB lets you choose per-operation. Your balance screen uses Strong. Your activity feed uses Eventual. This is the ideal implementation of PACELC — tuning the trade-off per use case rather than globally.

**Default:** Session consistency — a good default for most applications (you always read your own writes, which prevents confusing UX where you update something and immediately see the old value).

---

## Q10. Explain the PACELC theorem. How does it extend CAP? (Hard)

**Answer:**

**CAP only describes behaviour during a network partition.** But what about when everything is working normally (no partition)?

**PACELC adds the normal-operation trade-off:**

```
If Partition (P): choose between Availability (A) or Consistency (C)
Else (E):         choose between Latency (L) or Consistency (C)
```

**The "Else" trade-off:**
Even with no partition, if you want strong consistency, you must wait for all replicas to sync before returning a response → higher latency. If you want lower latency, you return the local replica's value without waiting → potentially stale.

**Example — DynamoDB (PA/EL):**
- During partition: chooses Availability (AP) — always responds, may be stale
- Normal operation: chooses Latency — returns local replica's data immediately, doesn't wait for global sync

**Example — MongoDB (PC/EC):**
- During partition: chooses Consistency (CP) — rejects writes if can't reach quorum
- Normal operation: chooses Consistency — waits for replica acknowledgment before returning

**Why PACELC matters in interviews:**
CAP alone makes you think "AP systems are always fast." PACELC shows that even AP systems face latency vs consistency trade-offs in normal operation. A system that's AP during partitions can still choose to be slow (waiting for consistency) during normal operation. DynamoDB chooses not to — it's EL (latency over consistency) in normal operation too.
