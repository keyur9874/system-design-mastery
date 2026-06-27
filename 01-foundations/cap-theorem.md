# Topic 2 — Latency, Throughput & CAP Theorem

> **Phase:** Foundations | **Source:** Alex Xu Vol 1 Ch.2 + Ch.6 | **Azure Practical:** Yes

---

## Table of Contents

1. [Latency](#1-latency)
2. [Throughput](#2-throughput)
3. [The Latency vs Throughput Trade-off](#3-the-latency-vs-throughput-trade-off)
4. [Availability](#4-availability)
5. [CAP Theorem](#5-cap-theorem)
6. [Consistency Models](#6-consistency-models)
7. [Quorum — Tuning Consistency vs Speed](#7-quorum--tuning-consistency-vs-speed)
8. [PACELC — CAP's Realistic Extension](#8-pacelc--caps-realistic-extension)
9. [Real-World Database Classification](#9-real-world-database-classification)
10. [Azure Practical](#10-azure-practical)
11. [Interview Questions](#11-interview-questions)

---

## 1. Latency

### What is it?

Latency is the **time it takes to complete one operation** — from the moment a request is sent to the moment a response is received.

**Simple analogy:** You order food at a restaurant. Latency is the time from when you place the order to when the food arrives at your table. You're just waiting — doing nothing.

### Types of Latency

There are 3 sources of latency in any system. Every slow request is caused by one or more of these:

#### 1. Network Latency
Time spent traveling across the network — from client to server and back.

```
Client sends request → travels over wires/fibre → reaches server
Server sends response → travels back → reaches client
```

| Distance | Approximate Latency |
|----------|-------------------|
| Same machine (localhost) | < 1 ms |
| Same data center | 0.5 ms |
| Same region (e.g., UK to Germany) | 10–20 ms |
| Across continents (US to Europe) | 80–150 ms |
| Across the world (US to Australia) | 200–300 ms |

**Real example:** If your API server is in East US and your user is in India, every API call has ~180ms of network latency before your code even runs. This is why CDN and regional deployments matter.

#### 2. Processing Latency
Time your code actually spends computing the response.

```
Time to run database query     → 20ms
Time to run business logic     → 5ms
Time to serialize JSON         → 2ms
Total processing time          → 27ms
```

This is what your code optimizations (indexes, caching, better algorithms) reduce.

#### 3. Disk I/O Latency
Time spent reading from or writing to disk.

```
RAM read:  0.0001 ms  → almost instant
SSD read:  0.1 ms     → very fast
HDD read:  10 ms      → 100x slower than SSD
```

This is why databases are tuned to keep hot data in RAM (buffer pool) and why SSDs matter for databases.

---

### How Latency is Measured

In production, you don't look at average latency — you look at **percentiles**:

```
P50 = 20ms   → half your users get this or better
P95 = 120ms  → 95% of users get this or better  
P99 = 800ms  → 99% of users get this or better (worst 1%)
```

**Why not just use average?**

Imagine 100 requests with these response times:
```
98 requests → 20ms each
2 requests  → 5,000ms each (5 seconds — something went wrong)

Average = (98 × 20 + 2 × 5000) / 100 = 119ms  ← looks acceptable
P99     = 5,000ms                               ← tells the real story
```

The average looks fine but 2% of your users waited 5 seconds. P99 catches this. **Always design to meet your P99 target, not your average.**

---

### How to Reduce Latency

| Problem | Solution |
|---------|----------|
| High network latency (user far from server) | CDN for static assets, deploy to multiple regions |
| Slow database queries | Add indexes, use read replicas, add caching |
| Slow disk reads | Keep hot data in RAM, use SSDs, use Redis |
| Sequential operations that could run in parallel | Use async processing, parallel requests |
| Large response payloads | Compress responses (gzip), paginate, return only needed fields |

---

## 2. Throughput

### What is it?

Throughput is the **number of operations your system can handle per unit of time**.

**Simple analogy:** A highway has lanes. Each lane can move cars. Throughput is how many cars per minute pass through the entire highway.

- Adding lanes (more servers) → higher throughput
- Faster cars (better code) → not necessarily higher throughput (the bottleneck may be elsewhere)

**Measured as:**
- Requests per second (RPS) — for web APIs
- Transactions per second (TPS) — for databases
- Messages per second — for message queues
- MB/s or GB/s — for data pipelines

---

### What Limits Throughput?

A system's throughput is always limited by its **bottleneck** — the slowest component.

**Example:**
```
App servers can handle:  10,000 req/sec
Database can handle:     1,000 req/sec  ← bottleneck
Cache can handle:        100,000 req/sec

System throughput = 1,000 req/sec (limited by DB, not app servers)
```

Adding more app servers does nothing here. You need to fix the database — add read replicas, add caching, or optimize queries.

### How to Increase Throughput

| Approach | How it works |
|----------|-------------|
| **Horizontal scaling** | More servers = more parallel processing capacity |
| **Caching** | Serve repeated requests from memory, skip the DB entirely |
| **Async processing** | Use message queues — don't make users wait for slow operations |
| **Database read replicas** | Distribute reads across multiple DB servers |
| **Connection pooling** | Reuse DB connections instead of opening new ones per request |
| **Batching** | Group multiple small operations into one larger efficient one |

---

## 3. The Latency vs Throughput Trade-off

This is one of the most important and most misunderstood relationships in system design.

**Common misconception:** "High throughput means low latency."

**Reality:** They often pull in opposite directions.

### The Batching Example

Imagine a database that processes write operations:

**Option A — Process each write immediately (low latency, low throughput):**
```
Write 1 → flush to disk immediately → 10ms
Write 2 → flush to disk immediately → 10ms
Write 3 → flush to disk immediately → 10ms
Each write: 10ms latency | 100 writes/sec throughput
```

**Option B — Batch writes every 50ms (higher latency, much higher throughput):**
```
Collect 500 writes in 50ms buffer → flush all 500 to disk at once → 50ms
Each write: 50ms latency | 10,000 writes/sec throughput (100x more!)
Throughput went up 100x by accepting 5x higher latency
```

This is exactly how Kafka, Cassandra, and most high-throughput databases work internally.

### The Queue Example

A message queue adds latency (message sits in queue before processing) but massively increases throughput (producers don't wait for consumers).

```
Without queue:  API handles request synchronously → 500ms/request → 2 req/sec per server
With queue:     API publishes to queue → 5ms/request  → 200 req/sec per server
                (consumers process asynchronously at their own pace)
```

**The rule:** When you need maximum throughput, accept higher latency. When you need minimum latency, accept lower throughput. Design for the requirement of your specific workload.

---

## 4. Availability

### What is it?

Availability is the **percentage of time a system is operational and responding correctly**.

**Simple analogy:** A coffee shop is "available" when it's open and serving customers. If it closes for repairs for 2 hours a week, its availability is ~98.8%.

### The Nines

Availability is measured in "nines." More nines = less downtime = higher engineering cost.

| Availability | Downtime per Year | Downtime per Month | Downtime per Day | What it means |
|-------------|-------------------|--------------------|--------------------|---------------|
| 99% (2 nines) | 3.65 days | 7.2 hours | 14.4 minutes | Acceptable for internal tools |
| 99.9% (3 nines) | 8.77 hours | 43.8 minutes | 1.44 minutes | Standard SLA for most services |
| 99.99% (4 nines) | 52.6 minutes | 4.38 minutes | 8.66 seconds | High availability — needs redundancy |
| 99.999% (5 nines) | 5.26 minutes | 26.28 seconds | 0.87 seconds | Carrier-grade — very expensive |

**Real world context:**
- Azure, AWS, and Google Cloud SLAs are 99.9%+ for compute
- Azure Cosmos DB guarantees 99.999% for multi-region deployments
- Banks and telecom aim for 99.999% (5 nines)
- Most startups target 99.9% (3 nines)

### How to Achieve High Availability

Every layer must be redundant. If any layer has a single point of failure, that layer defines your availability ceiling.

```
Load Balancer: 99.99%  →  has redundant instances
App Servers:   99.99%  →  3+ servers behind load balancer
Cache:         99.99%  →  Redis cluster (3 nodes minimum)
Database:      99.99%  →  Primary + replica + auto-failover
────────────────────────────────────────────────────
Combined:      99.96%  (weakest layer sets the ceiling)
```

**Key insight:** Availability is multiplicative. If each component has 99.9% availability and you have 3 in series (all must work), combined availability = 99.9% × 99.9% × 99.9% = 99.7%. This is why redundancy at every layer matters — the more layers, the more critical each one's reliability.

---

## 5. CAP Theorem

### What is it?

CAP theorem states that **a distributed system can guarantee only 2 of these 3 properties simultaneously**:

- **C — Consistency:** Every node returns the most recent write. All clients see the same data at the same time.
- **A — Availability:** Every request gets a response (even if some nodes are down). No timeouts or errors.
- **P — Partition Tolerance:** The system keeps working even when network communication between nodes fails.

> "CAP theorem states it is impossible for a distributed system to simultaneously provide more than two of these three guarantees." — Alex Xu

---

### Understanding Each Property with a Real Example

Imagine you have 3 database nodes: **Node 1 (London)**, **Node 2 (New York)**, **Node 3 (Tokyo)**.

A user in London updates their profile name from "John" to "John Smith."

#### Consistency (C)
Until ALL 3 nodes have "John Smith", no node should show ANY user the profile.

```
User in Tokyo reads profile → gets "John Smith" ✓  (consistent)
User in New York reads profile → gets "John Smith" ✓  (consistent)
```

If Tokyo hasn't received the update yet → Tokyo must either wait or return an error. It cannot return "John."

#### Availability (A)
Every request gets a response, no matter what.

```
Tokyo node hasn't synced yet → still returns "John" (stale data)
But the user gets a response → system is "available"
```

#### Partition Tolerance (P)
A network cable between London and Tokyo gets cut. The two sides can't talk.

```
London writes "John Smith" → Tokyo can't receive this update
Tokyo still needs to serve requests → what does it do?
```

---

### Why You Can Never Sacrifice P (Partition Tolerance)

Network failures are **unavoidable** in any distributed system.

- Cables get cut
- Data center routers fail
- Cloud provider has a network incident
- A VM's network card fails

**Alex Xu:** _"Since network failure is unavoidable, a distributed system must tolerate network partition. Thus, a CA system cannot exist in real-world applications."_

**This means in practice, the real choice is always: CP or AP.**

You don't choose between C, A, and P.
You choose: **when a partition happens, do I sacrifice Consistency or Availability?**

---

### CP vs AP — The Real Trade-off

#### CP System (Consistency + Partition Tolerance)

**"When a partition happens, the system stops accepting requests until it's consistent again."**

```
Partition occurs between Node 1 and Node 3
                ↓
CP system: Node 3 stops accepting writes
           Users connecting to Node 3 get an ERROR
           System is unavailable but never serves stale data
```

**When to use CP:**
- **Banking and payments** — You cannot show a customer the wrong balance. It's better to return an error than show "$1,000" when the real balance is "$0".
- **Inventory management** — Can't oversell an item. Better to return "can't process now" than sell the same item twice.
- **Any system where wrong data causes financial or legal harm**

**Real example:** You try to transfer $500 but the bank's servers are in a network partition. The bank returns: *"Service temporarily unavailable. Please try again."* — That's a CP system working correctly.

---

#### AP System (Availability + Partition Tolerance)

**"When a partition happens, the system keeps serving requests — even if some data is stale."**

```
Partition occurs between Node 1 and Node 3
                ↓
AP system: Node 3 keeps accepting reads and writes
           Users may see old data (stale) but system is always up
           When partition heals → nodes sync (eventual consistency)
```

**When to use AP:**
- **Social media feeds** — If your Twitter timeline shows a tweet from 2 minutes ago instead of 10 seconds ago, nobody cares.
- **Product recommendations** — Slightly stale "You might like..." is fine.
- **DNS** — DNS updates take minutes to propagate worldwide. Old cached results are served. Nobody expects instant global consistency from DNS.
- **Shopping cart (sometimes)** — Amazon's shopping cart uses AP. It's more important that you can always add items than that every device is perfectly in sync.

**Real example:** You post a photo on Instagram. Your friend in Tokyo can see it in 2 seconds. Your friend in London sees it in 4 seconds. Stale for 2 seconds — AP system. The post is never lost, just slightly delayed.

---

### The Diagram

```
        Consistency (C)
              /\
             /  \
            /    \
           /  CA  \
          / (only  \
         /  single  \
        /   server)  \
       /──────────────\
      /  CP  \  /  AP  \
     /        \/        \
    ────────────────────
   Partition Tolerance (P)
```

| System Type | Guarantees | Sacrifices | Real Databases |
|-------------|-----------|------------|----------------|
| **CP** | Consistency + Partition Tolerance | Availability (errors during partition) | HBase, Zookeeper, MongoDB (default), Redis Cluster |
| **AP** | Availability + Partition Tolerance | Consistency (stale data during partition) | Cassandra, DynamoDB, CouchDB, DNS |
| **CA** | Consistency + Availability | Partition Tolerance | Single-node MySQL/PostgreSQL (only works on one server) |

---

## 6. Consistency Models

CAP theorem is binary — consistent or not. In practice, consistency is a **spectrum**. These are the models from strongest to weakest:

### Strong Consistency

**"Every read returns the most recent write. Always. No exceptions."**

```
Write: Update balance to $500 → acknowledged
Read (immediately after, from any node): returns $500 ✓
```

**How it's achieved:** Every write must be confirmed by ALL replicas before it's committed. Reads wait until all nodes agree.

**Trade-off:** High latency (must wait for all replicas), low availability during partitions.

**When to use:** Banking, stock trading, anything where stale data causes real harm.

---

### Eventual Consistency

**"Given enough time, all replicas will converge to the same value. But right after a write, different nodes may return different values."**

```
Write to Node 1: name = "John Smith"  → immediately acknowledged
Read from Node 2 (500ms later):        → might return "John" (not synced yet)
Read from Node 2 (5 seconds later):    → returns "John Smith" ✓ (synced now)
```

**How it's achieved:** Write to one node, replicate asynchronously to others. No waiting.

**Trade-off:** Very fast writes, very high availability. But users may briefly see stale data.

**When to use:** Social media, DNS, email (delivered "eventually"), shopping carts, product catalogs.

**Real example from Alex Xu:** Amazon DynamoDB and Apache Cassandra both use eventual consistency by default. From Cassandra's docs: *"eventual consistency means that if no new updates are made to a given data item, eventually all accesses to that item will return the last updated value."*

---

### Weak Consistency

**"After a write, reads may or may not return the new value. No guarantees about when."**

This is even looser than eventual consistency — there's no guarantee the data will ever propagate.

**When to use:** Real-time systems where freshness doesn't matter — multiplayer game positions, live video streams, VoIP calls (dropped packets are just gone, not retried).

---

### Consistency Model Summary

| Model | Speed | Staleness risk | Use for |
|-------|-------|---------------|---------|
| **Strong** | Slowest | None | Banking, payments, inventory |
| **Eventual** | Fast | Brief (ms to seconds) | Social feeds, DNS, carts |
| **Weak** | Fastest | High (no guarantee) | Live video, games, VoIP |

---

## 7. Quorum — Tuning Consistency vs Speed

From Alex Xu Chapter 6 — this is how distributed databases like Cassandra and DynamoDB let you **tune** the trade-off between consistency and latency.

### The Variables

```
N = total number of replicas
W = minimum replicas that must confirm a WRITE before it's considered successful
R = minimum replicas that must respond to a READ before it's considered successful
```

**Simple example:** You have 3 database replicas (N=3). A user writes their profile.

- If **W=1**: Write is confirmed the moment 1 replica acknowledges it. Fast, but other 2 replicas might not have it yet.
- If **W=3**: Write is confirmed only when all 3 acknowledge it. Slow, but guaranteed all replicas have the data.

---

### The Key Formula

```
If W + R > N → Strong Consistency guaranteed
If W + R ≤ N → Eventual Consistency (reads may return stale data)
```

**Why does W + R > N guarantee consistency?**

With N=3, W=2, R=2: W+R = 4 > 3 (N)

When you write, at least 2 nodes have the latest data.
When you read, you ask at least 2 nodes.
The 2 nodes you read from MUST overlap with the 2 nodes that received the write.
So you're guaranteed to get at least one node that has the latest data.

---

### Common Configurations

| W | R | N | Effect |
|---|---|---|--------|
| 1 | 1 | 3 | Fastest reads and writes, eventual consistency |
| 1 | 3 | 3 | Fast writes, strong consistency reads (read all replicas) |
| 3 | 1 | 3 | Slow writes (all must confirm), fast reads |
| 2 | 2 | 3 | Balanced — strong consistency with moderate speed ✓ Most common |

**Real interview question:** "How would you configure Cassandra for a payment system?"

Answer: N=3, W=2, R=2 → W+R=4 > N=3 → strong consistency. You sacrifice some write speed but guarantee no stale reads.

**Real interview question:** "How would you configure Cassandra for a user activity feed?"

Answer: N=3, W=1, R=1 → W+R=2 ≤ N=3 → eventual consistency. Ultra-fast reads and writes. A 1-second stale feed is acceptable.

---

## 8. PACELC — CAP's Realistic Extension

CAP only describes what happens **during a network partition**. But what about when everything is working fine (no partition)?

**PACELC** extends CAP to cover normal operation too:

```
If Partition (P):  choose between Availability (A) or Consistency (C)
Else (E):          choose between Latency (L) or Consistency (C)
```

Even when there's no partition, you still face a trade-off: do you want **lower latency** (return the local replica's data immediately) or **stronger consistency** (wait for all replicas to agree)?

### PACELC Classification of Real Databases

| Database | During Partition | During Normal Operation | Classification |
|----------|-----------------|------------------------|----------------|
| DynamoDB | Availability | Latency | PA/EL |
| Cassandra | Availability | Latency | PA/EL |
| MongoDB | Consistency | Consistency | PC/EC |
| MySQL | Consistency | Consistency | PC/EC |
| HBase | Consistency | Consistency | PC/EC |
| Azure Cosmos DB | Configurable | Configurable | Tunable |

**Azure Cosmos DB is unique** — it lets you choose from 5 consistency levels per operation:
1. **Strong** — always consistent, highest latency
2. **Bounded Staleness** — at most X versions or T seconds behind
3. **Session** — consistent within a session (most used default)
4. **Consistent Prefix** — reads never see out-of-order writes
5. **Eventual** — highest availability, lowest latency

This is why Cosmos DB is powerful for global applications — you tune consistency per use case.

---

## 9. Real-World Database Classification

Now you can look at any database and understand exactly why it was designed that way:

### Relational Databases (MySQL, PostgreSQL, Azure SQL)
- **Single node:** CA — consistent and available, no partition tolerance
- **With replication:** CP by default (primary stops writes if can't reach quorum)
- **Use for:** Transactions, financial data, anything requiring ACID guarantees

### Cassandra
- **Type:** AP + Eventual Consistency (configurable toward CP with quorum settings)
- **Why AP?** Designed for always-on global scale. Netflix uses Cassandra — a streaming service cannot afford downtime even for a partition.
- **Use for:** Time-series data, event logs, user activity, IoT sensor data

### MongoDB
- **Type:** CP by default
- **Primary election:** If primary goes down, secondaries elect a new primary. During election (~10-30 seconds), writes are rejected (consistency preserved).
- **Use for:** General-purpose document storage, content management, product catalogs

### Redis
- **Type:** CP (Redis Cluster)
- **Why CP?** In Redis Cluster, if a shard loses its primary and can't elect a new one, that shard becomes unavailable. Data is never stale.
- **Use for:** Sessions, caching, rate limiting, distributed locks

### Azure Cosmos DB
- **Type:** Fully tunable — PA/EL to PC/EC depending on consistency level chosen
- **Unique:** The only major database that lets you choose your CAP trade-off at the API level, per request
- **Use for:** Global applications with varying consistency needs

### DynamoDB
- **Type:** AP + Eventual Consistency (configurable to strong consistency for extra cost)
- **Use for:** User preferences, shopping carts, leaderboards, session storage

---

## 10. Azure Practical

### Practical 1: See Eventual Consistency in Action with Azure Cosmos DB

**What you'll prove:** Different consistency levels produce different behaviour — eventual consistency is faster but may return stale data; strong consistency is slower but always correct.

**Step 1: Create a Cosmos DB account**
```bash
az group create --name rg-cap-lab --location eastus

az cosmosdb create \
  --name cosmos-cap-lab \
  --resource-group rg-cap-lab \
  --default-consistency-level Eventual \
  --locations regionName=eastus failoverPriority=0 \
  --locations regionName=westus failoverPriority=1
```

> We create 2 regions (East US + West US). This gives us a real distributed system to observe consistency behaviour on.

**Step 2: Create a database and container**
```bash
az cosmosdb sql database create \
  --account-name cosmos-cap-lab \
  --resource-group rg-cap-lab \
  --name CapLabDB

az cosmosdb sql container create \
  --account-name cosmos-cap-lab \
  --resource-group rg-cap-lab \
  --database-name CapLabDB \
  --name Users \
  --partition-key-path "/userId"
```

**Step 3: Write data and observe consistency**

`consistency_demo.py`:
```python
from azure.cosmos import CosmosClient, PartitionKey
import os, time

# Connect to Cosmos DB
client = CosmosClient(
    url=os.environ['COSMOS_ENDPOINT'],
    credential=os.environ['COSMOS_KEY']
)
container = client.get_database_client('CapLabDB').get_container_client('Users')

# Write a user record
user = {"id": "user-001", "userId": "user-001", "name": "Keyur", "balance": 1000}
container.upsert_item(user)
print(f"Written: {user}")

# Read immediately — with Eventual consistency, might get stale data from another region
time.sleep(0.1)  # 100ms — not enough for replication across regions
result = container.read_item(item="user-001", partition_key="user-001")
print(f"Read immediately: {result['name']}, balance: {result['balance']}")

# Update balance
container.upsert_item({**user, "balance": 500})
print("Updated balance to 500")

# Read again immediately from a different session
result2 = container.read_item(item="user-001", partition_key="user-001")
print(f"Read after update: balance = {result2['balance']}")
# With Eventual consistency → might still show 1000 for a brief moment
# With Strong consistency  → always shows 500
```

**Step 4: Switch to Strong Consistency and observe the difference**
```bash
# Switch to Strong Consistency
az cosmosdb update \
  --name cosmos-cap-lab \
  --resource-group rg-cap-lab \
  --default-consistency-level Strong
```

Run the same script → reads now always return the most up-to-date value, but with higher latency.

**What to observe:**

| Consistency Level | Read Latency | Staleness Risk | Right for |
|-------------------|-------------|----------------|-----------|
| Eventual | ~5ms | Yes (briefly) | Feeds, activity logs |
| Session | ~10ms | Only across sessions | Most applications |
| Strong | ~50ms+ | Never | Payments, balances |

---

### Practical 2: Observe Throughput vs Latency with Azure Cache for Redis

**What you'll prove:** High throughput (pipeline/batch) sacrifices latency per operation but dramatically increases operations per second.

```bash
# Create Redis
az redis create \
  --name redis-cap-lab \
  --resource-group rg-cap-lab \
  --location eastus \
  --sku Standard \
  --vm-size c1
```

`throughput_vs_latency.py`:
```python
import redis
import time

r = redis.Redis(host='<redis-host>', port=6380, password='<key>', ssl=True)

# Method 1: Individual commands — low throughput, measures per-command latency
print("=== Individual Commands (Low Throughput) ===")
start = time.time()
for i in range(1000):
    r.set(f"key:{i}", f"value:{i}")
elapsed = time.time() - start
print(f"1,000 individual writes: {elapsed:.2f}s")
print(f"Throughput: {1000/elapsed:.0f} writes/sec")
print(f"Avg latency per write: {elapsed/1000*1000:.1f}ms")

# Method 2: Pipeline — batch all commands, send at once → high throughput, higher latency per batch
print("\n=== Pipelined Commands (High Throughput) ===")
start = time.time()
pipe = r.pipeline()
for i in range(1000):
    pipe.set(f"key:{i}", f"value:{i}")
pipe.execute()  # Send all 1,000 writes in ONE network round trip
elapsed = time.time() - start
print(f"1,000 pipelined writes: {elapsed:.2f}s")
print(f"Throughput: {1000/elapsed:.0f} writes/sec")
print(f"Total latency for batch: {elapsed*1000:.1f}ms")
```

**Expected output:**
```
=== Individual Commands (Low Throughput) ===
1,000 individual writes: 5.20s
Throughput:  192 writes/sec
Avg latency per write: 5.2ms   ← each command waits for ACK

=== Pipelined Commands (High Throughput) ===
1,000 pipelined writes: 0.08s
Throughput: 12,500 writes/sec  ← 65x more throughput!
Total latency for batch: 80ms  ← higher batch latency, but 65x throughput
```

**What this teaches you:** Batching (like pipelining in Redis, or batching in Kafka) dramatically increases throughput by reducing network round trips — at the cost of higher per-batch latency. This is the latency/throughput trade-off made real.

---

### Cleanup
```bash
az group delete --name rg-cap-lab --yes --no-wait
```

---

## 11. Interview Questions

See [`../interview-questions/02-cap-theorem.md`](../interview-questions/02-cap-theorem.md) for full Q&A with model answers.

**Questions covered:**
1. What is latency and how is it different from throughput?
2. Why do we use P99 instead of average latency?
3. Explain CAP theorem. Can you give a real-world example of each trade-off?
4. Why can't a CA system exist in a real distributed environment?
5. What is the difference between eventual consistency and strong consistency?
6. When would you choose AP over CP? Give a concrete example.
7. What is a quorum? How does W + R > N guarantee strong consistency?
8. Your payment API is slow. You're considering adding eventual consistency to speed it up. Is this a good idea?
9. What consistency model does Cosmos DB use by default and why?
10. Explain the PACELC theorem and how it extends CAP.

---

## Key Takeaways

| Concept | Rule of Thumb |
|---------|---------------|
| **Latency** | Time for one request. Measure P99, not average. |
| **Throughput** | Requests per second. Limited by the slowest component. |
| **Latency vs Throughput** | Batching increases throughput but raises per-batch latency — a deliberate trade-off |
| **Availability** | More nines = less downtime but higher cost. Redundancy at every layer. |
| **CAP** | P is non-negotiable. Real choice is CP vs AP. |
| **CP** | Errors during partition. Use for banking, payments, inventory. |
| **AP** | Stale data during partition. Use for social feeds, DNS, carts. |
| **Strong consistency** | Always correct, slowest. For financial data. |
| **Eventual consistency** | Fast, may be briefly stale. For social, recommendations, DNS. |
| **Quorum (W+R>N)** | The formula that turns eventual consistency into strong consistency. |
| **Cosmos DB** | The only major DB that lets you tune consistency per request (5 levels). |
