# Interview Questions — Scalability

> Topic 1 | Phase 1: Foundations

---

## Q1. What is the difference between scalability and performance? (Easy)

**Answer:**

**Performance** is how fast a system handles **a single request** — measured in latency (ms) and throughput (req/sec).

**Scalability** is how well the system maintains that performance as **load increases** — measured by how throughput and latency change when you add users or data.

A system can be:
- High performance but not scalable: a single fast server that collapses at 10,000 concurrent users
- Scalable but low performance: a distributed system that handles 1 million users but each request takes 2 seconds
- Both: the goal — fast per-request AND degrades gracefully under load

**Interview tip:** Always separate the two in your design discussions. "We'll optimize performance later" and "we'll add scalability later" are different problems with different solutions.

---

## Q2. When would you choose vertical scaling over horizontal? (Easy)

**Answer:**

Choose **vertical scaling** when:
1. **Speed of delivery matters** — no code changes, instant capacity increase
2. **The workload is inherently stateful and hard to distribute** — legacy monolithic databases (Oracle, SQL Server with complex stored procedures)
3. **The load is predictable and has a known ceiling** — an internal tool used by 500 employees will never need 10,000 servers
4. **Strong consistency is required** — a single node means no distributed consistency issues
5. **You're at early stage** — don't add horizontal complexity before you need it

Choose **horizontal scaling** when:
1. You've hit the vertical ceiling (biggest available machine)
2. You need high availability (no single point of failure)
3. Traffic is unpredictable (need to scale on demand)
4. Cost at scale matters (commodity hardware is cheaper per unit than enterprise hardware)

---

## Q3. Why can't you just use sticky sessions instead of making services stateless? (Medium)

**Answer:**

Sticky sessions pin each user to a specific server. This creates several critical problems:

**1. Single point of failure per user:**
If their pinned server dies, that user loses their session (cart emptied, logged out). The system is not truly highly available.

**2. Uneven load distribution:**
If user A is a heavy user and user B is light, and both are pinned to different servers, you cannot rebalance. Server A may be at 90% CPU while Server B is at 10%.

**3. Auto-scaling doesn't work properly:**
When you add a new server, the load balancer cannot move existing sessions to it. New users go to the new server, old users stay pinned. When you remove a server, all its pinned users' sessions are destroyed.

**4. Rolling deployments become risky:**
If you take a server down for a deployment, all pinned users lose their sessions.

**The correct solution:** Externalize session state to Redis. Then any server can handle any request, sessions survive server failures, and you can freely scale up/down.

**Stick sessions are acceptable only for:** Internal tools, prototypes, or systems where session loss is tolerable (e.g., anonymous analytics).

---

## Q4. What is Amdahl's Law and why does it matter? (Medium)

**Answer:**

Amdahl's Law states that the **maximum speedup** of a program using multiple processors is limited by the fraction of the program that **cannot be parallelized**.

```
Max Speedup = 1 / (S + (1-S)/N)

S = fraction of code that is sequential
N = number of processors
```

**Why it matters for system design:**

If 25% of your code is sequential (DB locks, single-threaded leader election, global locks):
- 2x servers → 1.6x speedup (not 2x)
- 4x servers → 2.3x speedup (not 4x)
- Infinite servers → max 4x speedup (wall hit!)

**Real-world implications:**
1. Don't over-invest in adding servers if your bottleneck is a sequential operation (DB writes to single primary)
2. Identify and reduce the sequential fraction — it has higher ROI than adding more hardware
3. When architects say "that won't scale," they often mean "the sequential fraction is too high"

**Example answer in interview:** "Before adding servers, I'd profile the system to find the sequential fraction. If 40% of the workload is serialized through a single DB write path, adding 10x servers only gives ~2x improvement. I'd first look at reducing DB write contention with write-behind caching, CQRS, or async writes."

---

## Q5. A company's single MySQL server is running at 95% CPU. Walk me through your options. (Medium)

**Answer:**

This is a classic interview question. Walk through options in order of complexity:

**Step 1 — Diagnose first:**
- Is it CPU-bound (complex queries, missing indexes) or I/O-bound (disk reads)?
- Run `SHOW PROCESSLIST` to see slow queries
- Check `EXPLAIN` on the top 5 slowest queries
- Often, adding a missing index drops CPU from 95% to 30% — no scaling needed

**Step 2 — Add caching:**
- Cache frequent reads in Redis
- If 60% of queries are for the same 10,000 product records, caching eliminates those DB calls
- Cache hit rate of 80% = only 20% of queries reach the DB

**Step 3 — Read replicas:**
- Add 2–3 MySQL read replicas
- Route SELECT queries to replicas, keep only INSERT/UPDATE/DELETE on primary
- Effective if reads >> writes (typical web app)

**Step 4 — Vertical scaling:**
- Resize the DB server (more CPU/RAM)
- More RAM = larger InnoDB buffer pool = more data fits in memory = fewer disk reads
- Fast to implement, no code changes

**Step 5 — Connection pooling:**
- Add ProxySQL or PgBouncer in front of MySQL
- Reduces connection overhead

**Step 6 — Sharding (last resort):**
- Split the data by a shard key across multiple DB instances
- High operational complexity — only do this when all else fails

**Key interview signal:** Never jump straight to "shard the database." Interviewers want to see you exhaust simpler options first.

---

## Q6. How does auto-scaling work and what are the failure modes? (Medium)

**Answer:**

**How it works:**
1. A monitoring service continuously collects metrics (CPU, memory, queue depth)
2. When a metric crosses a threshold for a sustained period → scale-out trigger fires
3. New instances are provisioned from a base image (AMI/VMSS image)
4. New instances run a startup script, register with the load balancer, and start receiving traffic
5. Reverse happens for scale-in

**Failure modes:**

**1. Boot time lag:**
New instances take 2–5 minutes to boot, configure, and warm up. During a sudden traffic spike, the system is under-provisioned during that window. Mitigation: pre-warm instances, use predictive scaling.

**2. Scale thrashing:**
Without a cooldown period, the system rapidly oscillates — add servers → CPU drops → remove servers → CPU spikes → add servers. Mitigation: asymmetric cooldowns (fast scale-out, slow scale-in).

**3. Thundering herd on scale-in:**
When removing instances, their active connections are dropped. Users see errors. Mitigation: connection draining (graceful shutdown — wait 30 seconds for existing connections to complete before terminating).

**4. Database can't keep up:**
App servers scaled 10x but the single database did not. All 10x servers hammer the DB simultaneously. Mitigation: database connection pooling, read replicas scaled in sync.

**5. Warming up the cache:**
New instances start with cold caches. First N requests are slow until cache warms. Mitigation: pre-warm caches on startup, or use a shared external cache (Redis) instead of local per-instance cache.

---

## Q7. What does "stateless" mean and how do you make a legacy stateful service stateless? (Hard)

**Answer:**

**Stateless** means: a server holds **no user-specific data in memory** between requests. Any instance can handle any request and produce the same result, given the same inputs plus shared external state.

**Making a legacy stateful service stateless — step by step:**

**1. Audit all in-memory state:**
- Session objects stored in memory
- In-process caches (HashMap, local dictionary)
- File uploads written to local disk
- WebSocket connection state tied to a specific server

**2. Externalize sessions → Redis:**
```python
# Before (in-memory — stateful)
session['user_id'] = 123

# After (Redis — stateless)
redis.setex(f"session:{token}", 3600, json.dumps({'user_id': 123}))
```

**3. Externalize local caches → Redis or Memcached:**
- Local cache in each server = N copies of the same data, inconsistent
- Shared Redis = one source of truth, consistent across all servers

**4. Move file uploads → Azure Blob Storage:**
- Instead of writing to `/tmp/uploads/` on the server, write directly to Azure Blob
- App servers become read-only pass-through for uploads

**5. Handle WebSockets (trickiest):**
- WebSocket connections are inherently stateful (persistent connection to one server)
- Solution: Use Azure Web PubSub or Azure SignalR Service — they manage connections externally and fan-out messages to all relevant connections
- Or: store connection registry in Redis — when you need to send a message to user 123, look up which server holds their connection, then route the message

**Result:** After these changes, any server can be killed and restarted without any user noticing.

---

## Q8. Your e-commerce site gets 10x traffic on Black Friday. Walk me through your scaling plan. (Hard)

**Answer:**

This is a system design narrative question. Structure your answer clearly:

**Phase 1 — Before Black Friday (preparation):**

1. **Load test at 10x current capacity** → find the bottleneck (app tier? DB? Redis?)
2. **Pre-scale infrastructure** — don't rely on auto-scale for the initial spike (boot lag)
   - Scale app servers from 5 → 30 (pre-provisioned)
   - Scale Redis cluster up (more memory, more nodes)
   - Ensure DB read replicas are running and replicated
3. **Enable a static maintenance page on CDN** as a circuit breaker — if everything collapses, route to CDN-served static page
4. **Pre-warm caches** — load popular products, categories, and homepage data into Redis before the event
5. **Feature flags** — disable expensive features under load (product recommendations ML, real-time inventory updates)

**Phase 2 — During Black Friday:**

1. **Monitor key metrics live:** P99 latency, error rate, DB CPU, Redis memory, queue depth
2. **Rate limit non-critical endpoints** — search, browsing, wishlists are less critical than checkout
3. **Queue the checkout flow** — if checkout is overwhelmed, put users in a virtual queue (show "you're #342 in line") rather than returning errors
4. **Scale auto-scale min count up** — set minimum to 20 instances (not 2) so the floor is already high

**Phase 3 — Recovery:**

1. **Scale in slowly** — wait until traffic stabilizes before reducing instances (avoid thrashing)
2. **Run post-mortem** — what was the actual bottleneck? What did you over/under-provision?

**Key signals the interviewer looks for:**
- You prepare BEFORE the event (pre-scaling), not just rely on auto-scale
- You identify the database as the hardest constraint
- You have graceful degradation (queuing, feature flags, maintenance page)
- You mention monitoring and observability, not just compute scaling
