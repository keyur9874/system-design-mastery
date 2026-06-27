# Topic 1 — Scalability: Scale from Zero to Millions of Users

> **Phase:** Foundations | **Source:** Alex Xu Vol 1 Ch.1 + Ch.2 | **Azure Practical:** Yes

---

## Table of Contents

1. [What is Scalability?](#1-what-is-scalability)
2. [Scalability vs Performance vs Availability](#2-scalability-vs-performance-vs-availability)
3. [The Evolution: Single Server to Millions of Users](#3-the-evolution-single-server-to-millions-of-users)
   - [Stage 1: Single Server](#stage-1-single-server)
   - [Stage 2: Separate Database](#stage-2-separate-database)
   - [Stage 3: Load Balancer](#stage-3-load-balancer)
   - [Stage 4: Database Replication](#stage-4-database-replication)
   - [Stage 5: Cache Layer](#stage-5-cache-layer)
   - [Stage 6: CDN](#stage-6-cdn)
   - [Stage 7: Stateless Web Tier](#stage-7-stateless-web-tier)
   - [Stage 8: Multiple Data Centers](#stage-8-multiple-data-centers)
   - [Stage 9: Message Queue](#stage-9-message-queue)
   - [Stage 10: Logging, Metrics, Automation](#stage-10-logging-metrics-automation)
   - [Stage 11: Database Sharding](#stage-11-database-sharding)
4. [Vertical vs Horizontal Scaling — Deep Dive](#4-vertical-vs-horizontal-scaling--deep-dive)
5. [Stateless vs Stateful — The Core Problem](#5-stateless-vs-stateful--the-core-problem)
6. [Auto-Scaling](#6-auto-scaling)
7. [Back-of-the-Envelope Estimation](#7-back-of-the-envelope-estimation)
8. [Identifying Bottlenecks](#8-identifying-bottlenecks)
9. [Key Principles — Alex Xu Summary](#9-key-principles--alex-xu-summary)
10. [Azure Practical](#10-azure-practical)
11. [Interview Questions](#11-interview-questions)

---

## 1. What is Scalability?

Scalability is the **ability of a system to handle a growing amount of work by adding resources**.

A system is scalable if:
- Adding more resources results in **proportional performance improvement**
- The system continues to work **correctly** under increased load
- The cost of scaling remains **predictable**

### The Mental Model

Think of a restaurant:
- 10 customers → 1 chef handles it fine
- 100 customers → buy a faster chef (vertical) or hire 9 more chefs (horizontal)?

Both work, but with very different trade-offs, costs, and failure modes.

### What Scalability is NOT

| Misconception | Reality |
|---------------|---------|
| "Scalable = fast" | A slow system can scale. Scalability is about growth capacity, not speed |
| "Just throw hardware at it" | Without statelessness and partitioning, more hardware does nothing |
| "Scalability = availability" | They overlap but are different. A system can be highly available but not scalable |

---

## 2. Scalability vs Performance vs Availability

### Performance
- How fast a **single request** is processed
- Measured in **latency** (ms per request) and **throughput** (requests/second)

### Scalability
- How the system behaves as **load increases**
- A perfectly performant system at 100 users may collapse at 10,000

### Availability
- What percentage of time the system is **operational and accessible**
- Measured in "nines" — from Alex Xu Ch.2:

| Availability | Downtime per year | Downtime per month | Downtime per day |
|-------------|-------------------|--------------------|------------------|
| 99%         | 3.65 days         | 7.2 hours          | 14.4 minutes     |
| 99.9%       | 8.77 hours        | 43.8 minutes       | 1.44 minutes     |
| 99.99%      | 52.60 minutes     | 4.38 minutes       | 8.66 seconds     |
| 99.999%     | 5.26 minutes      | 26.28 seconds      | 0.87 seconds     |

> Azure, AWS, and Google Cloud all set their SLAs at **99.9% or above** for compute.

A Service Level Agreement (SLA) is a formal contract between the provider and customer defining the uptime guarantee. The more nines, the more expensive and complex to achieve.

---

## 3. The Evolution: Single Server to Millions of Users

Alex Xu's approach: build incrementally. Start simple, identify the bottleneck at each stage, solve it, and repeat.

---

### Stage 1: Single Server

Everything runs on one machine — web app, database, cache.

```
[User] ──DNS lookup──> [DNS Server] ──returns IP──> [User]
[User] ──HTTP request──> [Single Server: App + DB + Cache]
```

**Request flow:**
1. User accesses `api.mysite.com` → DNS resolves to server IP
2. Browser sends HTTP request to that IP
3. Server returns HTML or JSON

**Traffic sources:**
- **Web app:** Server-side languages (Java, Python) handle business logic; HTML/JS for frontend
- **Mobile app:** Communicates via HTTP; JSON is the standard API response format (`GET /users/12`)

**When it breaks:** Even small traffic spikes cause slowness or downtime. Everything shares the same CPU/RAM/Disk.

---

### Stage 2: Separate Database

Split the single server into two: one for the web/app tier, one for the database. This allows them to **scale independently**.

```
[Users] → [Web Server] → [Database Server]
```

**Which database to use?**

**Relational databases (SQL):** MySQL, PostgreSQL, Oracle
- Store data in tables and rows
- Support JOIN operations across tables
- 40+ years of production hardening
- Best default choice for most applications

**Non-relational databases (NoSQL):** CouchDB, Cassandra, HBase, DynamoDB, MongoDB
- Grouped into: key-value stores, graph stores, column stores, document stores
- JOIN operations generally not supported

**Choose NoSQL when:**
- You need super-low latency
- Data is unstructured or has no relational shape
- You need to serialize/deserialize data (JSON, XML, YAML)
- You need to store a **massive** amount of data
- You need horizontal partitioning from day one

---

### Stage 3: Load Balancer

A single web server has no failover. A load balancer solves this.

```
[Users] → [Load Balancer: Public IP]
                    │
         ┌──────────┴──────────┐
         │                     │
  [Web Server 1]        [Web Server 2]
  (private IP)          (private IP)
         │                     │
         └──────────┬──────────┘
                    │
             [Database Server]
```

**How it works (Alex Xu):**
- Users connect to the **public IP of the load balancer**, not directly to servers
- Servers use **private IPs** — unreachable directly from the internet (security benefit)
- Load balancer communicates with servers over private IPs

**Failover behavior:**
- If Server 1 goes offline → all traffic routes to Server 2 automatically
- When traffic grows → add Server 3, load balancer automatically distributes to it
- No code changes, no user impact

**What this does NOT solve:** The database is still a single point of failure. Database replication is next.

---

### Stage 4: Database Replication

> "Database replication can be used in many database management systems, usually with a master/slave relationship between the original (master) and the copies (slaves)." — Wikipedia [Alex Xu source]

```
                    ┌──────────────────┐
                    │  Master Database  │  ← Writes (INSERT, UPDATE, DELETE)
                    └────────┬─────────┘
                             │ Replication
              ┌──────────────┼──────────────┐
              ▼              ▼              ▼
       [Slave DB 1]   [Slave DB 2]   [Slave DB 3]
       (Read only)    (Read only)    (Read only)
```

**Key rules:**
- Master: handles **all write operations** only
- Slaves: receive copies from master, serve **read operations** only
- Most applications have far more reads than writes → more slaves than masters is typical

**Advantages of replication:**

| Advantage | Detail |
|-----------|--------|
| **Better performance** | Reads distributed across slaves → more queries processed in parallel |
| **Reliability** | If a server is destroyed (disaster), data is preserved across other slaves |
| **High availability** | Website stays up even if one DB goes offline |

**Failover scenarios:**

- **Slave goes offline:** Read operations temporarily route to master. New slave is provisioned and added.
- **Master goes offline:** A slave is **promoted** to master. All writes redirect to the new master. A new slave replaces the old one for replication. ⚠️ In production, promoted slave may have stale data — data recovery scripts must run to fill gaps.

**Full architecture after this stage:**
```
User → DNS → Load Balancer → Web Server 1 or 2
                                    │               │
                             [Read: Slave]   [Write: Master]
```

---

### Stage 5: Cache Layer

> "A cache is a temporary storage area that stores the result of expensive responses or frequently accessed data in memory so that subsequent requests are served more quickly." — Alex Xu

**The problem:** Every page load makes one or more database calls. Repeated DB calls for the same data waste time and resources.

**Cache tier:**
```
[Web Server] → check cache first
                    │
         Hit ───────┘       Miss ──→ [Database] ──→ store in cache ──→ return
         │
      return data
```

**Read-through cache strategy:**
1. Web server checks if response is in cache
2. If yes (hit): return cached data immediately
3. If no (miss): query database → store result in cache → return to client

**Considerations for using cache (Alex Xu):**

**1. When to use cache:**
- Use when data is **read frequently but modified infrequently**
- Cached data is in volatile memory — if cache server restarts, **all data is lost**
- Therefore: never use cache as the **sole storage** for critical data

**2. Expiration policy:**
- Set a TTL (Time-To-Live) on every cached item
- Too short TTL → frequent database reloads
- Too long TTL → stale data served to users
- Rule of thumb: TTL should match how frequently the source data changes

**3. Consistency:**
- Keeping the database and cache in sync is hard
- Data-modifying operations on DB and cache are **not in a single transaction**
- When scaling across multiple regions, consistency across the cache layer becomes a major challenge
- Reference: Facebook's paper "Scaling Memcache at Facebook" (NSDI '13) — the definitive resource

**4. Mitigating failures (SPOF):**
- A single cache server is a **Single Point of Failure (SPOF)**
- Recommendation: **multiple cache servers across different data centers**
- Also: **overprovision memory** by a certain percentage as a buffer for unexpected growth

**5. Eviction policy (what happens when cache is full):**

| Policy | Description | Best For |
|--------|-------------|----------|
| **LRU** (Least Recently Used) | Evict the item not accessed for the longest time | Most general use cases |
| **LFU** (Least Frequently Used) | Evict the item accessed fewest times overall | When access frequency matters |
| **FIFO** (First In First Out) | Evict the oldest item regardless of access | Simple, predictable use cases |

LRU is the **most popular** and the default for Redis and Memcached.

---

### Stage 6: CDN

> "A CDN is a network of geographically dispersed servers used to deliver static content." — Alex Xu

**What CDN caches:** images, videos, CSS, JavaScript files — static assets.

**How CDN works (step by step):**
1. User requests `image.png` via a CDN URL (e.g., `https://mysite.cloudfront.net/logo.jpg`)
2. CDN server nearest to user checks its cache
3. If **cache miss**: CDN fetches from origin server (or Azure Blob Storage), stores it with TTL
4. If **cache hit**: returns image directly from CDN edge node
5. Next user requesting same image gets it from CDN cache — origin server not hit

**Performance impact:** Users in London hitting an Azure CDN edge in Frankfurt get the file in ~5ms. Without CDN, they'd hit your origin server in East US = 80–150ms.

**CDN considerations (Alex Xu):**

**1. Cost:**
- CDN is run by third-party providers (Azure CDN, Cloudflare, Akamai)
- Charged for **data transfers out of CDN**
- Don't cache infrequently used assets — it costs money with no benefit
- Only cache assets with high access frequency

**2. Cache expiry:**
- Set appropriate TTL per content type
- Too long → stale content served (user sees old logo)
- Too short → frequent origin fetches, CDN provides less value
- Rule: static assets with versioning (e.g., `logo.v3.jpg`) can have very long TTLs

**3. CDN fallback:**
- What if the CDN itself goes down?
- Clients should detect CDN failure and fall back to **origin server**
- Implement health checks in your frontend to detect CDN errors and switch origin

**4. Invalidating files:**
Two strategies to force CDN to fetch fresh content before TTL expires:
- **API-based invalidation:** Call the CDN vendor's API to purge specific objects (Azure CDN supports this)
- **Object versioning:** Rename the file with a version parameter → `image.png?v=2` or `image.v2.png`. Old URL serves old cached content; new URL forces a fresh fetch.

**Architecture after CDN + Cache:**
```
Static assets (JS/CSS/images) → served directly from CDN (not web server)
Dynamic data → web server → check Redis cache → database if miss
```

---

### Stage 7: Stateless Web Tier

To scale web servers horizontally, you **must** remove state from the web tier.

**Stateful architecture (problem):**
```
User A → must always go to Server 1 (session stored there)
User B → must always go to Server 2
User C → must always go to Server 3
```
If Server 1 dies → User A is logged out. Load cannot be rebalanced freely.

**Stateless architecture (solution):**
```
HTTP requests from users → any web server (doesn't matter which)
Web servers fetch state from shared data store
State data is stored externally — NOT inside the web server
```

```
[User] → [Load Balancer] → [Web Server 1]  ←→  [Shared State Store]
                        → [Web Server 2]  ←→  [Redis / NoSQL]
                        → [Web Server 3]  ←→
```

**Alex Xu's quote:** _"A stateless system is simpler, more robust, and scalable."_

**What to externalize:**

| State | Where to Store |
|-------|---------------|
| User sessions / auth | Redis / Azure Cache for Redis |
| Shopping cart | Redis or database |
| File uploads | Azure Blob Storage (direct upload, bypass app server) |
| Rate limit counters | Redis atomic increment |
| App feature flags | Azure App Configuration |
| Distributed locks | Redis Redlock |

**Result:** Once stateless, **auto-scaling works correctly** — add/remove servers freely, users never notice.

---

### Stage 8: Multiple Data Centers

> "To improve availability and provide a better user experience across wider geographical areas, supporting multiple data centers is crucial." — Alex Xu

**Normal operation:**
- Users are **geoDNS-routed** (geo-routed) to the closest data center
- Example: 70% US-East, 30% US-West split based on user location
- **geoDNS:** A DNS service that resolves domain names to IP addresses based on the **user's geographic location**

**Failover:**
- If Data Center 2 (US-West) goes offline → 100% traffic redirects to Data Center 1 (US-East)
- DNS TTL determines how fast this switchover propagates

**Technical challenges for multi-data-center setup:**

**1. Traffic redirection:**
- GeoDNS routes users to nearest data center
- Azure Front Door handles this automatically with health probes

**2. Data synchronization:**
- Users in different regions may hit different local databases/caches
- In failover, traffic goes to a data center where some data may be unavailable
- Solution: **asynchronous multi-data-center replication** (Netflix's approach)
- Trade-off: replication lag means data may be slightly stale in secondary region

**3. Test and deployment:**
- Must test the application from multiple geographic locations
- Automated deployment tools must keep all data centers consistent
- Feature flags should roll out regionally, not globally

```
Region: East US                          Region: West Europe
[Azure Front Door]                       [Azure Front Door]
       ↓                                        ↓
[App Server Fleet]                       [App Server Fleet]
       ↓                                        ↓
[Azure SQL Primary] ←── async repl ───→ [Azure SQL Primary]
[Azure Redis]                            [Azure Redis]
[Azure Blob CDN]                         [Azure Blob CDN]
```

---

### Stage 9: Message Queue

> "A message queue is a durable component, stored in memory, that supports asynchronous communication. It serves as a buffer and distributes asynchronous requests." — Alex Xu

**Why it enables scalability:** By decoupling producers and consumers, each can scale independently.

**Basic model:**
```
[Producer / Publisher] → [Message Queue] → [Consumer / Subscriber]
```

**Key property:** The producer can post messages even when the consumer is unavailable. The consumer can read messages even when the producer is down.

**Real example (Alex Xu):**
Your app supports photo processing (crop, sharpen, blur). These are slow operations.

**Without queue:**
```
User uploads photo → App server processes synchronously → User waits 30 seconds
```

**With queue:**
```
User uploads photo → App server publishes job to queue (instant) → Returns "processing..."
Photo workers → pick jobs from queue → process asynchronously
User polls or gets notified when done
```

**Scaling behavior:**
- Queue grows large → add more workers (scale out consumer independently)
- Queue is mostly empty → reduce workers (scale in, save cost)
- Photo upload traffic spikes → producers publish faster, queue absorbs the spike, workers drain at their own pace

**Azure services:** Azure Service Bus (enterprise messaging), Azure Queue Storage (simple FIFO), Azure Event Grid (event-driven), Azure Event Hubs (high-throughput streaming).

---

### Stage 10: Logging, Metrics, Automation

> "When working with a small website that runs on a few servers, logging, metrics, and automation support are good practices but not a necessity. However, now that your site has grown to serve a large business, investing in those tools is essential." — Alex Xu

**Logging:**
- Monitor error logs to identify errors and problems early
- Options: per-server logs, or aggregate to centralized service (Azure Log Analytics, Elasticsearch)
- Centralized logging enables cross-server search and correlation

**Metrics — three categories:**

| Category | Examples |
|----------|----------|
| **Host level** | CPU utilization, memory usage, disk I/O, network bandwidth |
| **Aggregated level** | Database tier performance, cache hit rate, load balancer request distribution |
| **Key business metrics** | Daily active users (DAU), retention rate, revenue per hour, orders per minute |

**Automation:**
- **CI/CD:** Every code commit triggers automated tests, build, and deployment
- Enables early detection of regressions
- Automate: build, test, deploy, infrastructure provisioning
- Azure DevOps / GitHub Actions for pipeline automation

**Azure observability stack:**
- **Azure Monitor:** infrastructure metrics and alerts
- **Application Insights:** application-level metrics, distributed tracing, dependency tracking
- **Log Analytics Workspace:** centralized log aggregation with KQL queries
- **Azure Dashboards / Grafana:** visualization

---

### Stage 11: Database Sharding

When the database itself becomes the bottleneck, sharding is the answer.

**What is sharding:**
> "Sharding separates large databases into smaller, more easily managed parts called shards. Each shard shares the same schema, though the actual data on each shard is unique to the shard." — Alex Xu

```
[App] → [Sharding Logic: user_id % 4]
              │
    ┌─────────┼────────────┐
    │         │            │
[Shard 0]  [Shard 1]  [Shard 2]  [Shard 3]
user_id%4=0 user_id%4=1 user_id%4=2 user_id%4=3
```

**Sharding key (partition key):**
- One or more columns that determine which shard a row goes to
- Most important criteria: **choose a key that distributes data evenly**
- Common choices: `user_id`, `tenant_id`, geographic region

**Sharding challenges (Alex Xu — all four):**

**1. Resharding data:**
Required when:
- A shard can no longer hold more data due to rapid growth
- Uneven distribution causes some shards to fill up ("shard exhaustion") faster than others
- Solution: **Consistent hashing** (Chapter 5) minimizes data movement when resharding

**2. Celebrity problem (hotspot key problem):**
- Excessive access to a specific shard causes server overload
- Example: Justin Bieber, Katy Perry, and Lady Gaga all land on Shard 2 in a social app
- Shard 2 gets overwhelmed with millions of read requests
- Solution: Allocate a **dedicated shard per celebrity**. That shard might need further partitioning.

**3. Join and de-normalization:**
- Cross-shard SQL JOIN operations are **extremely expensive or impossible**
- Solution: **De-normalize** the database — store redundant data so queries can be answered from a single table/shard

**4. Operational complexity:**
- Schema changes must be applied to all shards
- Cross-shard transactions require distributed coordination
- Debugging becomes harder (which shard has the data?)

**Real-world example (Alex Xu):**
- StackOverflow in 2013: **10 million+ monthly unique visitors** with only **1 master database** — vertical scaling worked long enough. Sharding was deferred until truly necessary.

---

## 4. Vertical vs Horizontal Scaling — Deep Dive

### Vertical Scaling (Scale Up)

Upgrade the **existing machine** with more powerful hardware.

```
Before:  [Server: 4 vCPU, 16GB RAM]
After:   [Server: 32 vCPU, 128GB RAM]
```

**The Amdahl's Law Problem:**

Amdahl's Law explains why vertical scaling has diminishing returns:

```
Max Speedup = 1 / (S + (1-S)/N)

Where:
  S = fraction of code that is sequential (cannot be parallelized)
  N = number of processors
```

If 25% of your code is sequential (DB locks, global mutexes, single-threaded parts):
- 2 CPUs → 1.6x speedup
- 4 CPUs → 2.1x speedup
- 8 CPUs → 2.9x speedup (not 8x!)
- ∞ CPUs → **max 4x speedup** — wall hit!

**Hard limits (from Alex Xu):**
- AWS high-memory instances go up to 24 TB RAM on RDS
- Azure largest VMs go to ~416 vCPU, 11.4 TB RAM
- These are the absolute ceilings — once hit, only horizontal scaling remains

### Horizontal Scaling (Scale Out)

Add more machines. Each machine handles a fraction of the load.

**Shared-nothing architecture (the gold standard):**
```
Each server:
  ✓ Own CPU/RAM — no shared memory between servers
  ✓ No direct connection to other app servers
  ✓ Reads/writes go to a shared EXTERNAL store (DB, cache, queue)
```

If Server A dies → Server B has everything it needs in the external store to continue.

**Linear vs Sub-linear Scaling:**

```
Ideal (linear):    2 servers = 2.0x throughput
Good:              2 servers = 1.8x throughput (some coordination overhead)
Bad:               2 servers = 1.2x throughput (DB contention, too much locking)
```

### Decision Framework

| Factor | Vertical | Horizontal |
|--------|----------|------------|
| Traffic predictable, bounded | ✓ | |
| Need HA / no SPOF | | ✓ |
| Stateful legacy system, hard to distribute | ✓ | |
| Traffic unpredictable, needs auto-scale | | ✓ |
| Early-stage product | ✓ | |
| Large scale, cost efficiency | | ✓ |
| Downtime acceptable for upgrades | ✓ | |

---

## 5. Stateless vs Stateful — The Core Problem

### What is State?
Any data that must persist between requests for a user session:
- User login session
- Shopping cart contents
- Upload progress
- Rate limit counter

### Stateful Server (Problem)
```
User A → Request 1 → Server 1 stores session in memory
User A → Request 2 → Load balancer sends to Server 2 → Server 2: "Who is this?" → Error
```
User must always return to Server 1. This is **sticky sessions**.

### Sticky Sessions — Why They're Dangerous
- If Server 1 dies → all its users lose sessions
- Server 1 may be at 90% load while Server 2 is at 10% — cannot rebalance
- Auto-scaling adds new servers, but existing users stay pinned to old ones
- Rolling deployments take down servers → session loss

### Stateless Server (Solution)
```
User A → Request 1 → Server 1 → fetches session from Redis
User A → Request 2 → Server 3 → fetches SAME session from Redis → works!
```
Any server, any request. State lives in Redis, not in server memory.

### Truly Stateless Server Checklist
- [ ] No user data stored in server memory between requests
- [ ] Server can be killed and restarted without any user noticing
- [ ] Any instance produces identical responses for identical inputs
- [ ] Two instances return same session data (because Redis holds it)

---

## 6. Auto-Scaling

**Three types:**

| Type | How | Best For |
|------|-----|----------|
| **Reactive** | Scale when metric crosses threshold | General purpose — most common |
| **Predictive** | ML models predict load from history | Recurring traffic patterns |
| **Scheduled** | Pre-scale at a specific time | Known events (Black Friday, end-of-month batch) |

**Scaling metrics:**

| Metric | When to Use |
|--------|-------------|
| CPU utilization | CPU-bound workloads (computation, encryption) |
| Memory usage | Memory-bound (in-memory caches, JVM heaps) |
| Request queue depth | Message processing systems |
| Requests per second | HTTP-heavy APIs |
| Custom metrics | Business KPIs (orders/min, active connections) |

**Asymmetric cooldowns (critical pattern):**
- Scale out fast: CPU > 70% for **2 minutes** → add servers
- Scale in slow: CPU < 30% for **15 minutes** → remove servers

This prevents **scale thrashing** — constant add/remove cycles that waste money and cause instability.

**Boot time lag:** New instances take 2–5 minutes to boot, configure, and warm up. During a sudden spike, the system is under-provisioned during this window. Mitigation: pre-warm instances, use predictive scaling, set minimum instance count high before known events.

**Connection draining:** When removing an instance, don't kill it immediately. Give it 30 seconds to finish serving current requests before terminating. Azure VMSS and App Service both support graceful shutdown/drain.

---

## 7. Back-of-the-Envelope Estimation

### What is it?

A **quick rough calculation** done on a whiteboard to decide if a design will work **before you build it**.

Like a civil engineer estimating "this bridge needs roughly 500 tonnes of steel" before doing exact math. Not precise — but good enough to make the right architectural decision.

Interviewers use this to check: can you think in numbers? Do you know your system's limits?

---

### The 3-Tier Mental Model (Memorize This, Not the Full Table)

Don't try to memorize every row. Just remember these 4 numbers:

```
RAM (memory read)       →   0.1 ms    → instant, like thinking of a word
Same-datacenter network →   0.5 ms    → like saying the word out loud
Disk seek               →   10 ms     → like walking to another room to check a book
Cross-continent network →   150 ms    → like calling someone overseas
```

**One rule to remember everything:**
> RAM is 100x faster than network. Disk is 100x slower than network.

**Full reference table (Dr. Jeff Dean, Google):**

| Operation | Latency | Human analogy |
|-----------|---------|---------------|
| L1 cache reference | 0.5 ns | Reflex |
| L2 cache reference | 7 ns | Blink |
| Main memory (RAM) reference | 100 ns | Thinking of a word |
| Compress 1 KB (Snappy) | 10 µs | Saying the word |
| Send 2 KB over 1 Gbps network | 20 µs | Texting someone |
| Read 1 MB from memory | 250 µs | Reading a sentence |
| Same-datacenter round trip | 0.5 ms | Calling a colleague |
| Disk seek | **10 ms** | Walking to another room |
| Read 1 MB from disk | 30 ms | Reading a page |
| Cross-continent packet (CA→Europe→CA) | **150 ms** | Calling someone overseas |

---

### When Do You Use This? — Real Interview Situations

#### Situation 1: "Should we add a cache?"
> Interviewer: "Your product page hits the database every time. Is caching worth it?"

```
Without cache: DB read from disk  → ~10 ms per request
With cache:    Redis read from RAM → ~0.1 ms per request
Improvement:   100x faster
```
**Decision:** Yes, absolutely cache it. The numbers justify it.

---

#### Situation 2: "How many servers do we need?"
> Interviewer: "Design Twitter. How many servers for the API tier?"

```
DAU                    = 150 million users
Requests per user/day  = 10 (timeline, tweet, like, search...)
Total requests/day     = 150M × 10 = 1.5 billion
QPS                    = 1.5B ÷ 86,400 seconds ≈ 17,000 req/sec

1 server handles       ≈ 1,000 req/sec (standard web server)
Servers needed         = 17,000 ÷ 1,000 = ~17 servers minimum
```
**Decision:** You need at least 17 servers + load balancer. Single server is out.

---

#### Situation 3: "How much storage do we need?"
> Interviewer: "Design YouTube. How much storage for 5 years of videos?"

```
Upload rate:    500 hours of video per minute (real YouTube number)
1 min HD video: ~50 MB
500 hours       = 30,000 minutes
Storage/minute  = 30,000 × 50 MB = 1.5 TB per minute
Per day         = 1.5 TB × 60 × 24 ≈ 2 PB per day
```
**Decision:** You need object storage (Azure Blob Storage), not a database. A database cannot handle petabytes.

---

#### Situation 4: "Should we do this synchronously or async?"
> Interviewer: "User uploads a photo. We resize it to 5 sizes. Do it in the API request?"

```
Resize 1 image = disk read (10ms) + CPU (50ms) + disk write (10ms) = ~70ms
5 sizes        = 350ms
```
**Decision:** 350ms just for resizing = bad UX. Use a **message queue** and do it async.

---

### Twitter QPS + Storage Estimation (Alex Xu Example)

**Step 1 — Write your assumptions first (always do this in interviews):**
- 300 million monthly active users
- 50% use it daily → **150 million DAU**
- Each user posts 2 tweets/day
- 10% of tweets have media (avg 1 MB per media)
- Store data for 5 years

**Step 2 — QPS:**
```
Tweet QPS  = 150M × 2 ÷ 86,400  ≈  3,500 tweets/sec
Peak QPS   = 2 × 3,500           =  7,000 tweets/sec  (assume 2x for spikes)
```

**Step 3 — Storage:**
```
Per tweet:   tweet_id (64B) + text (140B) + media (1MB for 10%)

Daily media  = 150M users × 2 tweets × 10% × 1MB  =  30 TB/day
5-year total = 30 TB × 365 × 5                     ≈  55 PB
```
**Decision:** At 55 PB over 5 years, you need distributed object storage + CDN, not a single database.

---

### Data Volume Reference

| Unit | Value | Real-world feel |
|------|-------|-----------------|
| 1 KB | 1,024 Bytes | A short text message |
| 1 MB | 1,024 KB | A photo thumbnail |
| 1 GB | 1,024 MB | A movie (compressed) |
| 1 TB | 1,024 GB | 1,000 movies |
| 1 PB | 1,024 TB | 1 million movies |

---

### Interview Tips

| Tip | Example |
|-----|---------|
| Round aggressively | `99,987 / 9.1` → use `100,000 / 10` |
| Always write assumptions | "I'm assuming 300M MAU, 50% DAU..." |
| Label every unit | Write "30 TB", never just "30" |
| State what the number means | "At 7,000 QPS, a single server won't cut it" |

The interviewer doesn't want the exact answer. They want to see your **thinking process** — assumptions → math → decision.

---

## 8. Identifying Bottlenecks

### What is it?

Finding **which single component is slowing down the entire system**. No matter how fast everything else is, the slowest part sets the limit.

> "A chain is only as strong as its weakest link" — this is called the **Theory of Constraints**.

**Why it matters in interviews:** When an interviewer says "your system is slow, what do you do?" — they expect you to methodically find the bottleneck, not guess randomly.

---

### The USE Method — How to Diagnose Any Resource

For **every resource** (CPU, memory, disk, network, DB), ask these 3 questions:

| Letter | Question | Example |
|--------|----------|---------|
| **U** — Utilization | How busy is it? | CPU at 85% |
| **S** — Saturation | Is work piling up waiting? | 50 threads waiting for CPU |
| **E** — Errors | Is it failing silently? | 0.1% DB connection timeouts |

**How to use it in practice:**

Start with CPU → if U is low, S is low, E is low → CPU is NOT the bottleneck, move on.
Find the resource where U is high OR S is growing → that is your bottleneck.

**Real example:**
```
CPU:     U=20%,  S=0 threads waiting  → not the problem
Memory:  U=40%,  S=0                  → not the problem
DB:      U=95%,  S=200 queries queued → BOTTLENECK HERE
Network: U=10%,  S=0                  → not the problem
```
**Decision:** Fix the database — add an index, add a read replica, or add caching.

---

### Where Bottlenecks Hide — Check in This Order

Every request travels this path:

```
User → Browser → DNS → CDN → Load Balancer → App Server → Cache → Database → Disk
```

Always check from left to right — the leftmost bottleneck affects everything downstream.

| # | Where | What to look for | Most common cause |
|---|-------|-----------------|-------------------|
| 1 | **Network** | Bandwidth saturated? Packet loss? | Under-provisioned link |
| 2 | **Load Balancer** | CPU maxed? Single instance? | No redundancy, needs scaling |
| 3 | **App Server** | CPU/Memory maxed? Thread pool full? | Code inefficiency or too few servers |
| 4 | **Cache** | Hit rate dropped below 80%? | Cache too small, bad TTL, cold start |
| 5 | **DB Connections** | Connection pool exhausted? | Too many app servers hitting one DB |
| 6 | **DB Query Time** | Slow queries? Full table scans? | Missing index — fix this first, always |
| 7 | **Disk I/O** | Write queue backed up? IOPS maxed? | Spinning disk — switch to SSD |

**Interview tip:** Always say "I'd check for missing indexes first" — it's the most common fix and requires no infrastructure changes.

---

### Understanding P50 / P95 / P99 Latency

You'll see these in every load test and monitoring dashboard. Here's what they mean:

Imagine 100 requests. Sort them by response time slowest to fastest:
```
P50 (median)  = the 50th request → half are faster, half are slower
P95           = the 95th request → 95% of users get this or better
P99           = the 99th request → the worst 1% of experiences
```

**Why P99 matters more than average:**
```
Average response time: 50ms ← looks fine
P99 response time:     3000ms ← 1 in 100 users waits 3 seconds
```
At 10,000 req/sec → 100 users/second have a terrible experience. Average hides this.

**Rule:** Always optimize for P99, not average. Interviewers notice if you only mention "average response time."

---

### Load Testing — Which Tool, When

| Tool | Best For | Key Feature |
|------|----------|-------------|
| **k6** | Scripted, code-first teams | JavaScript scripts, outputs P50/P95/P99 |
| **Apache JMeter** | Teams that prefer UI | GUI-based, great for complex scenarios |
| **Locust** | Python teams | Write test scenarios in Python |
| **Azure Load Testing** | Azure workloads | Managed service, built-in Azure Monitor integration |

**Load test protocol (use this in interviews when asked "how would you test capacity"):**
```
Step 1: Start at 10 req/sec
Step 2: Ramp up by +10 req/sec every 30 seconds
Step 3: Watch: CPU, memory, DB connections, P99 latency, error rate
Step 4: Find the breaking point — when P99 spikes or errors appear
Step 5: That is your current capacity ceiling
Step 6: Fix the bottleneck → run again → new ceiling
Repeat until you meet your target capacity
```

**What "fixing the bottleneck" looks like:**
- DB query slow → add index → re-test
- App server CPU maxed → add servers → re-test
- Cache hit rate low → increase cache size or fix TTL → re-test

---

## 9. Key Principles — Alex Xu Summary

These are Alex Xu's 8 principles to scale from zero to millions of users. For each one, I've added **what goes wrong if you skip it** — because that's how you remember rules.

---

### 1. Keep the Web Tier Stateless

**Rule:** Never store user data (sessions, cart, uploads) in app server memory.

**What goes wrong if you skip it:**
- You're forced into sticky sessions
- If a server dies → those users lose their session (logged out, cart emptied)
- Auto-scaling adds new servers, but old users are stuck on old ones
- You cannot freely add/remove servers → horizontal scaling is broken

**Fix:** Store all state in Redis. Any server can then handle any request.

---

### 2. Build Redundancy at Every Tier

**Rule:** No single server at any layer should be the only one. Every tier needs at least 2.

**What goes wrong if you skip it:**
- 1 load balancer → it dies → entire site goes down
- 1 database → it dies → entire site goes down
- 1 cache server → it dies → all traffic slams the database

**Fix:** Minimum 2 instances at every tier. Use Azure Availability Sets or Zones.

---

### 3. Cache Data as Much as You Can

**Rule:** If a piece of data is read more than it is written, it should be cached.

**What goes wrong if you skip it:**
- Every page load hits the database
- Database CPU climbs to 95% under moderate traffic
- You scale the database unnecessarily, which is expensive and hard

**Fix:** Add Redis between app and DB. Even a 70% cache hit rate removes 70% of DB load instantly.

---

### 4. Support Multiple Data Centers

**Rule:** Run your system in at least 2 geographic regions with automatic failover.

**What goes wrong if you skip it:**
- One region goes down (fire, power outage, Azure incident) → entire site goes down globally
- Users in Europe hitting servers in US East → 150ms added latency every request

**Fix:** Azure Front Door routes users to the nearest healthy region. GeoDNS + active-passive or active-active setup.

---

### 5. Host Static Assets in CDN

**Rule:** Images, CSS, JS, videos should never be served by your app servers.

**What goes wrong if you skip it:**
- App servers waste CPU/bandwidth serving static files
- Users in Asia downloading a 2MB image from a US server = 300ms delay
- Static file traffic spikes eat capacity that should handle API requests

**Fix:** Upload static files to Azure Blob Storage → serve via Azure CDN. App servers only handle dynamic requests.

---

### 6. Scale the Data Tier by Sharding

**Rule:** When the database can no longer scale vertically, partition data horizontally across multiple databases.

**What goes wrong if you skip it:**
- Single database becomes the ceiling for your entire system
- No matter how many app servers you add, they all bottleneck at one DB
- Database CPU/disk maxes out → entire system slows down

**Fix:** Shard by a natural partition key (user_id, tenant_id). Each shard handles a fraction of the data.
**Warning:** Shard only when you must — it adds major operational complexity.

---

### 7. Split Tiers into Individual Services

**Rule:** As the system grows, separate distinct functions (auth, payments, notifications, search) into independent services.

**What goes wrong if you skip it:**
- A spike in search traffic also slows down checkout (shared resources)
- Deploying a change to notifications requires redeploying the entire monolith
- You cannot scale the notification system without scaling everything

**Fix:** Move to microservices or at least logical service boundaries. Each service scales independently based on its own load profile.

---

### 8. Monitor Your System and Use Automation

**Rule:** You cannot fix what you cannot see. Automate everything you do more than once.

**What goes wrong if you skip it:**
- DB runs out of disk at 3 AM — no alert fires — site goes down for 6 hours
- Deployment is manual — developer makes a mistake — wrong config goes to production
- A slow query degrades P99 latency for days before anyone notices

**Fix:** Azure Monitor + Application Insights for observability. GitHub Actions / Azure DevOps for CI/CD. Alert on P99 latency, error rate, disk usage, cache hit rate.

---

### Quick Recall Table

| # | Principle | Skip it and… |
|---|-----------|-------------|
| 1 | Stateless web tier | Sticky sessions, auto-scaling breaks |
| 2 | Redundancy everywhere | One crash = full outage |
| 3 | Cache aggressively | DB gets hammered, costs explode |
| 4 | Multiple data centers | Regional outage = global outage |
| 5 | Static assets on CDN | App servers waste capacity on files |
| 6 | Shard the database | DB becomes the single ceiling |
| 7 | Split into services | One spike affects everything |
| 8 | Monitor + automate | Flying blind, slow to respond |

---

## 10. Azure Practical

> **Goal of these practicals:** Don't just run commands — understand *why* each step exists and what it proves about the theory.

---

### Practical 1: Vertical Scaling — Resize an Azure VM

**What you'll prove:** Vertical scaling gives more resources instantly with zero code changes, but causes downtime and has a hard ceiling.

```bash
# Step 1: Create Resource Group
az group create --name rg-systemdesign --location eastus

# Step 2: Create a small VM (B2s = 2 vCPU, 4GB RAM)
az vm create \
  --resource-group rg-systemdesign \
  --name vm-scaletest \
  --image Ubuntu2204 \
  --size Standard_B2s \
  --admin-username azureuser \
  --generate-ssh-keys

# Step 3: Check current size (before resize)
az vm show \
  --resource-group rg-systemdesign \
  --name vm-scaletest \
  --query hardwareProfile.vmSize \
  --output tsv
# Output: Standard_B2s

# Step 4: See all available sizes you can resize to
az vm list-vm-resize-options \
  --resource-group rg-systemdesign \
  --name vm-scaletest \
  --output table
# This shows the hard ceiling — there are a finite number of options

# Step 5: Resize to a bigger VM (4 vCPU, 16GB RAM)
az vm resize \
  --resource-group rg-systemdesign \
  --name vm-scaletest \
  --size Standard_B4ms
```

**What to observe and what it teaches you:**

| Observation | What it proves |
|-------------|---------------|
| VM restarts automatically during resize | Vertical scaling = downtime in production |
| After resize: `nproc` shows 4 CPUs | OS picks up new CPUs without any code change |
| After resize: `free -h` shows 16 GB | More RAM = more data in memory = fewer disk reads |
| The resize options list is finite | There IS a hard ceiling — you will eventually hit it |

**Key insight:** This is why vertical scaling is a short-term fix. At some point, you run out of bigger VMs.

---

### Practical 2: Horizontal Scaling — Azure VM Scale Sets + Redis

**What you'll prove:** Horizontal scaling with stateless servers means any server can handle any request. State lives in Redis — kill any server and users don't notice.

**Step 1: Create Azure Cache for Redis (external state)**
```bash
az redis create \
  --resource-group rg-systemdesign \
  --name redis-scalelab \
  --location eastus \
  --sku Basic \
  --vm-size c0

# Get the connection key
az redis list-keys \
  --resource-group rg-systemdesign \
  --name redis-scalelab \
  --query primaryKey \
  --output tsv
```

**Step 2: Create VM Scale Set**
```bash
az vmss create \
  --resource-group rg-systemdesign \
  --name vmss-webapi \
  --image Ubuntu2204 \
  --vm-sku Standard_B2s \
  --instance-count 2 \
  --admin-username azureuser \
  --generate-ssh-keys \
  --upgrade-policy-mode automatic \
  --load-balancer vmss-lb \
  --backend-pool-name vmss-backend
```

**Step 3: Configure Auto-Scale Rules**
```bash
# Create autoscale profile (min 2, max 10, default 2)
az monitor autoscale create \
  --resource-group rg-systemdesign \
  --resource vmss-webapi \
  --resource-type Microsoft.Compute/virtualMachineScaleSets \
  --name autoscale-webapi \
  --min-count 2 \
  --max-count 10 \
  --count 2

# Scale OUT: CPU > 70% for 5 min → add 2 instances
az monitor autoscale rule create \
  --resource-group rg-systemdesign \
  --autoscale-name autoscale-webapi \
  --condition "Percentage CPU > 70 avg 5m" \
  --scale out 2 \
  --cooldown 3

# Scale IN: CPU < 30% for 10 min → remove 1 instance (asymmetric cooldown)
az monitor autoscale rule create \
  --resource-group rg-systemdesign \
  --autoscale-name autoscale-webapi \
  --condition "Percentage CPU < 30 avg 10m" \
  --scale in 1 \
  --cooldown 5
```

**Step 4: Stateless API — state in Redis, not in server**

`app.py`:
```python
from flask import Flask, jsonify
import redis, os, socket

app = Flask(__name__)
r = redis.Redis(
    host=os.environ['REDIS_HOST'],
    port=6380,
    password=os.environ['REDIS_KEY'],
    ssl=True
)

@app.route('/health')
def health():
    return jsonify({"status": "ok", "server": socket.gethostname()})

@app.route('/counter')
def counter():
    # State in Redis — ANY server can serve this correctly
    count = r.incr('global_counter')
    return jsonify({
        "count": int(count),
        "served_by": socket.gethostname()  # Watch different servers appear
    })

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=8080)
```

**Step 5: Test stateless horizontal scaling**
```bash
# Hit /counter 10 times — watch: different servers respond, but counter is always consistent
for i in {1..10}; do curl http://<lb-ip>/counter; done
```

Expected output:
```json
{"count": 1, "served_by": "vmss-webapi-0001"}
{"count": 2, "served_by": "vmss-webapi-0003"}   ← different server!
{"count": 3, "served_by": "vmss-webapi-0001"}
{"count": 4, "served_by": "vmss-webapi-0002"}   ← third server!
```

**What this proves:**
- Three different servers handled these requests (load balancer distributed them)
- Counter is consistent across all servers (because state is in Redis, not in-server memory)
- If vmss-webapi-0001 crashes right now, requests 5–10 still work perfectly
- This is the whole point of stateless architecture

**Step 6: Trigger auto-scale and watch it happen**
```bash
# Artificially spike CPU on one instance to trigger scale-out
az vmss run-command invoke \
  --resource-group rg-systemdesign \
  --name vmss-webapi \
  --command-id RunShellScript \
  --instance-id 0 \
  --scripts "apt-get install -y stress && stress --cpu 4 --timeout 300"

# Watch the instance count increase every 10 seconds
watch -n 10 'az vmss show \
  --resource-group rg-systemdesign \
  --name vmss-webapi \
  --query sku.capacity'
```

**What to observe:**
```
T=0 min:   capacity = 2  (starting point)
T=5 min:   capacity = 4  (CPU threshold crossed, 2 new servers added)
T=10 min:  capacity = 6  (still high CPU, 2 more added)
T=15 min:  stress ends, CPU drops
T=25 min:  capacity = 5  (scale-in is slower — asymmetric cooldown working)
T=35 min:  capacity = 2  (back to minimum)
```

**What this teaches you:** Scale-out is fast (2 min cooldown), scale-in is slow (5 min cooldown). This is the asymmetric cooldown pattern preventing thrashing.

---

### Practical 3: CDN — Serve Static Assets from Azure CDN

**What you'll prove:** Static files served from CDN edge nodes are dramatically faster than serving from your origin server, and it removes load from your app servers entirely.

```bash
# Create storage account for static assets
az storage account create \
  --name stscalelab$RANDOM \
  --resource-group rg-systemdesign \
  --location eastus \
  --sku Standard_LRS \
  --kind StorageV2

# Enable static website hosting
az storage blob service-properties update \
  --account-name <storage-account-name> \
  --static-website \
  --index-document index.html

# Create CDN profile and endpoint
az cdn profile create \
  --resource-group rg-systemdesign \
  --name cdn-scalelab \
  --sku Standard_Microsoft

az cdn endpoint create \
  --resource-group rg-systemdesign \
  --profile-name cdn-scalelab \
  --name endpoint-scalelab \
  --origin <storage-account>.z13.web.core.windows.net \
  --origin-host-header <storage-account>.z13.web.core.windows.net
```

**After setup — test the difference:**
```bash
# Measure response time from origin (your storage account directly)
curl -o /dev/null -s -w "Origin time: %{time_total}s\n" \
  https://<storage-account>.z13.web.core.windows.net/logo.png

# Measure response time from CDN edge node
curl -o /dev/null -s -w "CDN time: %{time_total}s\n" \
  https://endpoint-scalelab.azureedge.net/logo.png
```

**What to observe:**
- First CDN request: slower (cache miss — CDN fetches from origin)
- Second CDN request: much faster (cache hit — served from edge node near you)
- Check response headers: `X-Cache: HIT` confirms it was served from CDN cache
- Your origin server received only 1 request, not 2 — CDN absorbed the second

**What this teaches you:** After the first request warms the cache, all subsequent users get the file from the CDN edge — your origin server is never hit again until TTL expires.

---

### Cleanup
```bash
az group delete --name rg-systemdesign --yes --no-wait
```

---

## 11. Interview Questions

See [`../interview-questions/01-scalability.md`](../interview-questions/01-scalability.md) for full Q&A with model answers.

**Questions covered:**
1. What is the difference between scalability and performance?
2. When would you choose vertical scaling over horizontal?
3. Why are sticky sessions dangerous for horizontal scaling?
4. What is Amdahl's Law and why does it matter?
5. A company's MySQL is at 95% CPU. Walk me through your options.
6. How does auto-scaling work and what are the failure modes?
7. How do you make a legacy stateful service stateless?
8. Your e-commerce site gets 10x traffic on Black Friday. Walk me through your scaling plan.
9. What are the challenges of database sharding?
10. Estimate: how many servers do you need to handle 1 million requests per day?

---

## Key Takeaways (Alex Xu's 8 Principles + Deep Additions)

| Concept | Rule of Thumb |
|---------|---------------|
| Start simple | Single server is fine early. Don't over-engineer. |
| Statelessness | Any server-side state is a horizontal scaling blocker. Externalize to Redis/DB. |
| DB replication | Master for writes, slaves for reads. Slave promotion is non-trivial in production. |
| Cache | Use LRU eviction, set TTL, avoid SPOF with multiple nodes, never sole storage for critical data |
| CDN | Version your static assets, set long TTLs on versioned files, always plan for CDN fallback |
| Sharding | Last resort. Choose shard key for even distribution. Beware: resharding, hotspot, cross-shard joins |
| Multi-DC | GeoDNS for routing, async replication for data sync, automate deployment across regions |
| Bottleneck | USE method: Utilization → Saturation → Errors. Load test to find ceiling before scaling. |
| Estimation | Memorize latency numbers. Memory ~100ns, Disk ~10ms. Practice Twitter-style QPS/storage math. |
