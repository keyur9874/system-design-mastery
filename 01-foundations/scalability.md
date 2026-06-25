# Topic 1 — Scalability

> **Phase:** Foundations | **Depth:** Deep | **Azure Practical:** Yes

---

## Table of Contents

1. [What is Scalability?](#1-what-is-scalability)
2. [Scalability vs Performance vs Availability](#2-scalability-vs-performance-vs-availability)
3. [Vertical Scaling (Scale Up)](#3-vertical-scaling-scale-up)
4. [Horizontal Scaling (Scale Out)](#4-horizontal-scaling-scale-out)
5. [Stateless vs Stateful — The Core Problem](#5-stateless-vs-stateful--the-core-problem)
6. [Auto-Scaling](#6-auto-scaling)
7. [Database Scalability](#7-database-scalability)
8. [Identifying Bottlenecks](#8-identifying-bottlenecks)
9. [Real-World Architecture Evolution](#9-real-world-architecture-evolution)
10. [Azure Practical](#10-azure-practical)
11. [Interview Questions](#11-interview-questions)

---

## 1. What is Scalability?

Scalability is the **ability of a system to handle a growing amount of work by adding resources**.

A system is said to be scalable if:
- Adding more resources results in **proportional performance improvement**
- The system continues to work correctly under increased load
- The cost of scaling stays predictable

### The Mental Model

Think of a restaurant:
- 10 customers → 1 chef handles it fine
- 100 customers → do you buy a faster chef (vertical) or hire 9 more chefs (horizontal)?

Both approaches work, but they have very different trade-offs, costs, and failure modes.

### Why Does Scalability Matter?

- User traffic is unpredictable — a viral moment can 100x your load overnight
- Without scalability, you either over-provision (waste money) or under-provision (system crashes)
- Modern businesses are built on the assumption that systems can grow with demand

### What Scalability is NOT

| Misconception | Reality |
|---------------|---------|
| "Scalable = fast" | A slow system can scale. Scalability is about growth capacity, not speed. |
| "Just throw hardware at it" | Without architectural decisions (stateless, partitioning), more hardware does nothing |
| "Scalability = availability" | They overlap but are different. A system can be highly available but not scalable. |

---

## 2. Scalability vs Performance vs Availability

These three are often confused in interviews. Know the distinction cold.

### Performance
- How fast a **single request** is processed
- Measured in **latency** (ms per request) and **throughput** (requests per second)
- Optimization: caching, faster code, better algorithms, indexes

### Scalability
- How the system behaves as **load increases**
- A perfectly performant system at 100 users might collapse at 10,000
- Optimization: horizontal scaling, partitioning, load balancing

### Availability
- What percentage of time the system is **operational and accessible**
- Measured as uptime: 99.9% = 8.7 hours downtime/year, 99.99% = 52 minutes/year
- Optimization: redundancy, failover, health checks

### The relationship

```
High Performance + No Scalability  →  Fast but breaks under load
High Scalability + Low Performance →  Handles load but every request is slow
High Availability + No Scalability →  Always up but degrades under load
```

The goal is **all three**. In practice, trade-offs exist and you design for your specific requirements.

---

## 3. Vertical Scaling (Scale Up)

### What It Is

Upgrading the **existing machine** with more powerful hardware:
- More CPU cores
- More RAM
- Faster SSD
- Better network card

```
Before:  [Server: 4 CPU, 16GB RAM]
After:   [Server: 32 CPU, 128GB RAM]
```

### How It Works Internally

When you vertically scale:
1. The OS can now schedule work across more CPU cores
2. More data fits in RAM → fewer disk reads → lower latency
3. More parallel threads can run → higher throughput
4. The application code changes **nothing** — the OS handles it

This is why vertical scaling is the **first instinct** for engineers — no code changes, instant results.

### Advantages

| Advantage | Explanation |
|-----------|-------------|
| **Zero code changes** | Your monolith works immediately on bigger hardware |
| **No distributed complexity** | No network calls between servers, no consistency issues |
| **Lower latency** | No inter-server communication overhead |
| **Simpler operations** | One server to monitor, patch, and maintain |
| **Strong consistency** | Single database or app = no sync issues |

### Disadvantages

| Disadvantage | Explanation |
|--------------|-------------|
| **Hard ceiling** | The largest available machine on Azure today is ~416 vCPU, 11.4 TB RAM. You will hit the limit. |
| **Single point of failure** | One server goes down = entire system down |
| **Downtime to scale** | Resizing a VM usually requires a restart |
| **Diminishing returns** | Going from 4→8 CPUs doubles capacity. 256→512 CPUs may give 10% gain because of memory bandwidth and lock contention. |
| **Cost explosion** | Enterprise hardware (1TB RAM) costs 10x more than commodity hardware, but does not give 10x throughput |

### The Amdahl's Law Problem

Amdahl's Law explains why vertical scaling has diminishing returns:

```
Speedup = 1 / (S + (1-S)/N)

Where:
  S = fraction of work that is sequential (cannot be parallelized)
  N = number of processors
```

Example: If 30% of your code is sequential (locks, single-threaded parts):
- 2 CPUs → 1.6x speedup
- 4 CPUs → 2.1x speedup
- 8 CPUs → 2.9x speedup (not 8x!)
- ∞ CPUs → max 3.3x speedup (law of diminishing returns)

**Conclusion:** There is always a bottleneck in sequential code that limits how much vertical scaling helps.

### When to Choose Vertical Scaling

- Early-stage product with small, predictable load
- Stateful systems that are hard to distribute (some legacy databases)
- When you need a quick fix and have budget headroom
- When the traffic pattern is steady and max load is known

---

## 4. Horizontal Scaling (Scale Out)

### What It Is

Adding **more machines** that share the load:

```
Before:  [Server A: 4 CPU]

After:   [Server A: 4 CPU]
         [Server B: 4 CPU]
         [Server C: 4 CPU]
         [Load Balancer] → distributes traffic across A, B, C
```

### How It Works

1. A **load balancer** sits in front of all servers
2. Each incoming request is routed to one of the servers
3. Each server independently processes its request
4. Responses go back through the load balancer (or directly to client)

This works perfectly **only if servers are stateless** (covered in section 5).

### Advantages

| Advantage | Explanation |
|-----------|-------------|
| **Theoretically unlimited scale** | Add as many servers as you need |
| **No single point of failure** | If one server dies, others handle the load |
| **Linear cost scaling** | 10x more servers ≈ 10x more cost (commodity hardware) |
| **Scale on demand** | Add/remove servers based on real-time traffic (auto-scaling) |
| **Zero downtime scaling** | Add a server → it joins the pool, no restart needed |
| **Geographic distribution** | Put servers in multiple regions for lower global latency |

### Disadvantages

| Disadvantage | Explanation |
|--------------|-------------|
| **Complexity** | Need load balancer, health checks, service discovery |
| **Statelessness requirement** | Sessions, caches, file uploads must be externalized |
| **Network latency** | Servers communicate over the network, not shared memory |
| **Data consistency** | Multiple servers writing to databases causes race conditions |
| **Harder debugging** | A request may be processed by any server — logs are distributed |
| **More failure modes** | Network partitions, split-brain, partial failures |

### Shared-Nothing Architecture

The gold standard for horizontal scaling is **shared-nothing architecture**:

```
Each server has:
  - Its own CPU/RAM (no shared memory)
  - No direct connection to other app servers
  - Reads/writes go to a shared external store (DB, cache, queue)
```

This means: if server A fails, server B has everything it needs in the external store to continue.

### Linear vs Sub-linear Scaling

Perfect horizontal scaling is linear — double the servers = double the capacity. In practice:

```
Ideal (linear):        2 servers = 2x throughput
Good (sub-linear):     2 servers = 1.8x throughput (some coordination overhead)
Bad (super-linear):    2 servers = 1.2x throughput (too much coordination, DB contention)
```

The gap from ideal is caused by: shared database contention, lock overhead, cache miss rates, network serialization.

---

## 5. Stateless vs Stateful — The Core Problem

This is the **most important concept** for horizontal scaling. If you don't understand this, you cannot scale horizontally.

### What is State?

State is **any data that must persist between requests for a user session**.

Examples:
- User is logged in → session token
- Items in a shopping cart
- Upload progress of a file
- Rate limit counter for a user

### The Problem with Stateful Servers

```
Request 1: User logs in → Server A stores session in memory
Request 2: Load balancer routes to Server B → Server B has no session → User is logged out!
```

If state is stored **inside** a server's memory, the user must always return to the same server. This is called **sticky sessions** (or session affinity).

### Sticky Sessions — Why They're Dangerous

Sticky sessions pin a user to a specific server:
```
User 123 → always goes to Server A
User 456 → always goes to Server B
```

Problems:
- If Server A dies, all its users lose their sessions
- Server A may get 80% of the load while Server B sits idle (uneven distribution)
- You cannot freely add/remove servers without disrupting active users
- Auto-scaling becomes ineffective

### The Solution — Externalizing State

Move state **out of the server** into a shared external store:

```
┌─────────────┐     ┌─────────────┐     ┌─────────────┐
│   Server A  │     │   Server B  │     │   Server C  │
│  (no state) │     │  (no state) │     │  (no state) │
└──────┬──────┘     └──────┬──────┘     └──────┬──────┘
       │                   │                   │
       └───────────────────┼───────────────────┘
                           │
               ┌───────────┴───────────┐
               │                       │
        ┌──────┴──────┐       ┌────────┴────────┐
        │  Redis Cache │       │    Database      │
        │  (sessions)  │       │  (user data)     │
        └─────────────┘       └─────────────────┘
```

Now any server can handle any request — they all read the same state from Redis.

### What to Externalize

| State Type | Solution |
|------------|----------|
| User sessions / auth tokens | Redis / Azure Cache for Redis |
| Shopping cart | Redis or database |
| File uploads | Azure Blob Storage (upload directly, not through app server) |
| Rate limit counters | Redis with atomic increment |
| Application config | Azure App Configuration |
| Locks | Redis distributed locks (Redlock) |

### Truly Stateless Server Checklist

A server is truly stateless if:
- [ ] It holds no user data in memory between requests
- [ ] It can be killed and restarted without any user noticing
- [ ] Any instance can handle any request
- [ ] Two instances give identical responses for identical inputs + shared state

---

## 6. Auto-Scaling

Auto-scaling is the ability to **automatically add or remove servers based on load**.

### Types of Auto-Scaling

#### Reactive Scaling (most common)
Scale based on **current metrics**:
- CPU > 70% for 5 minutes → add 2 servers
- CPU < 30% for 10 minutes → remove 1 server

Lag problem: By the time you detect the spike and a new server boots (2–5 minutes), the damage is done.

#### Predictive Scaling
Scale based on **historical patterns**:
- "Every Monday at 9 AM, traffic spikes 300% → pre-scale at 8:45 AM"
- Azure and AWS both offer ML-based predictive scaling

#### Scheduled Scaling
Scale based on **known events**:
- Black Friday → pre-scale to 10x capacity at midnight
- End of month → scale up for batch jobs

### Scaling Metrics

| Metric | When to Use |
|--------|-------------|
| **CPU utilization** | CPU-bound workloads (computation, encryption) |
| **Memory usage** | Memory-bound workloads (in-memory caches, JVM apps) |
| **Request queue depth** | Message processing systems |
| **Requests per second** | HTTP-heavy APIs |
| **Custom metrics** | Business KPIs (orders/min, active users) |

### Scale-In vs Scale-Out Asymmetry

Scale out aggressively, scale in conservatively:
- Scale out: CPU > 60% for **2 minutes** → add servers (fast response to traffic)
- Scale in: CPU < 30% for **15 minutes** → remove servers (avoid thrashing)

This asymmetry prevents **scale thrashing** — where you keep adding and removing servers repeatedly.

### Cooldown Period

After a scale event, wait before triggering another. Without cooldown:
```
T=0:  CPU spikes → add 3 servers
T=30s: New servers not ready yet → CPU still high → add 3 more servers (unnecessary)
T=2m: All 6 new servers come online → massive over-provisioning
```

With a 3-minute cooldown, the system waits for new servers to absorb load before deciding to add more.

---

## 7. Database Scalability

The hardest part of scaling is almost always the **database**. App servers are easy to scale horizontally (stateless). Databases are not.

### Read Replicas (Scale Read Traffic)

```
                    ┌─────────────────┐
                    │  Primary DB      │  ← All writes go here
                    │  (Read + Write)  │
                    └────────┬────────┘
                             │ Replication (async)
              ┌──────────────┼──────────────┐
              │              │              │
     ┌────────┴────┐ ┌───────┴─────┐ ┌─────┴───────┐
     │  Replica 1  │ │  Replica 2  │ │  Replica 3  │
     │  (Read only)│ │  (Read only)│ │  (Read only)│
     └────────────┘ └─────────────┘ └─────────────┘
```

- Writes go to primary only
- Reads are distributed across replicas
- Replication lag: replicas may be 10–500ms behind primary
- Works well when reads >> writes (typical: 80% reads, 20% writes)

### Connection Pooling

Each DB connection is expensive (memory, thread, file descriptor). A connection pool:
- Maintains a fixed set of open connections (e.g., 100)
- App servers borrow a connection, use it, return it
- Prevents 1000 app servers × 100 connections each = 100,000 connections from crashing the DB

**Azure: Azure SQL has built-in connection pooling. For more control, use PgBouncer or ProxySQL.**

### Sharding (Horizontal DB Partitioning)

Split data across multiple databases by a **shard key**:

```
Shard Key: user_id

user_id 1–1M    → Database Shard 1
user_id 1M–2M   → Database Shard 2
user_id 2M–3M   → Database Shard 3
```

Each shard is a fully independent database with its own storage.

**Challenges:**
- Cross-shard queries (JOIN across shards) are very expensive or impossible
- Hotspot shards (one user has 10M records, others have 100)
- Resharding is painful when shards get too large
- No global transactions across shards

---

## 8. Identifying Bottlenecks

A system is only as fast as its slowest component. This is the **Theory of Constraints**.

### Common Bottleneck Locations

```
Browser → DNS → CDN → Load Balancer → App Server → Cache → Database → Disk
```

Check in this order:
1. **Network bandwidth** — is the link saturated?
2. **Load balancer** — single instance? CPU?
3. **App server CPU/Memory** — is it maxed out?
4. **Cache hit rate** — if it drops, all misses hit the DB
5. **Database connections** — pool exhausted?
6. **Database query time** — slow queries, missing indexes
7. **Disk I/O** — write bottleneck on spinning disks

### The USE Method (Utilization, Saturation, Errors)

For each resource, check:
- **Utilization:** How busy is it? (CPU: 85% busy)
- **Saturation:** Is work queuing up? (CPU run queue: 50 threads waiting)
- **Errors:** Is it failing? (connection timeout rate: 0.1%)

### Load Testing to Find Bottlenecks

Tools:
- **k6** — scripted load testing, outputs percentile latency
- **Apache JMeter** — UI-based load testing
- **Locust** — Python-based distributed load testing
- **Azure Load Testing** — managed load testing on Azure

A proper load test:
1. Start at 10 requests/second, ramp up by 10 every 30 seconds
2. Watch CPU, memory, DB connections, response time
3. Find the point where P99 latency spikes or errors appear
4. That is your current capacity ceiling
5. Fix the bottleneck, repeat

---

## 9. Real-World Architecture Evolution

How a real product scales from 1 user to 10 million:

### Stage 1 — Single Server (0–1,000 users)
```
[User] → [Single Server: App + DB + Cache]
```
- Everything on one box
- Simple to deploy, no ops overhead
- This is fine. Don't over-engineer early.

### Stage 2 — Separate DB (1K–10K users)
```
[User] → [App Server] → [Separate Database Server]
```
- DB on its own machine
- App server and DB can now scale independently
- DB gets more RAM for buffer pool (bigger than app server needs)

### Stage 3 — Load Balanced App Tier (10K–100K users)
```
[User] → [Load Balancer] → [App Server 1]
                         → [App Server 2]
                         → [App Server 3]
                              ↓
                         [Database]
                         [Redis Cache]
```
- App servers are now stateless
- Sessions stored in Redis
- DB still single primary (bottleneck approaching)

### Stage 4 — Read Replicas (100K–1M users)
```
[User] → [Load Balancer] → [App Servers]
                              ↓         ↓
                         [Primary DB]  [Redis]
                              ↓
                     [Replica 1] [Replica 2]
```
- Read-heavy traffic goes to replicas
- Writes still go to primary
- CDN added for static assets

### Stage 5 — Global Scale (1M–10M users)
```
Region: East US                    Region: West Europe
[CDN] → [Front Door] ──────────── [Front Door] → [CDN]
             ↓                          ↓
     [App Server Fleet]         [App Server Fleet]
             ↓                          ↓
     [DB Primary]       ←replication→  [DB Primary]
     [Redis Cluster]                   [Redis Cluster]
```
- Multiple regions (active-active or active-passive)
- Azure Front Door routes users to nearest region
- Data replication across regions (with consistency trade-offs)
- Microservices likely introduced to scale specific components independently

---

## 10. Azure Practical

### Practical 1: Vertical Scaling — Resize an Azure VM

**Goal:** Experience VM resizing with zero code changes.

**Steps:**
```bash
# Create a VM (B2s = 2 vCPU, 4GB RAM)
az vm create \
  --resource-group rg-systemdesign \
  --name vm-scaletest \
  --image UbuntuLTS \
  --size Standard_B2s \
  --admin-username azureuser \
  --generate-ssh-keys

# Check current size
az vm show \
  --resource-group rg-systemdesign \
  --name vm-scaletest \
  --query hardwareProfile.vmSize

# Resize to a larger VM (B4ms = 4 vCPU, 16GB RAM)
az vm resize \
  --resource-group rg-systemdesign \
  --name vm-scaletest \
  --size Standard_B4ms
# Note: This restarts the VM — downtime!

# List available VM sizes in your region
az vm list-vm-resize-options \
  --resource-group rg-systemdesign \
  --name vm-scaletest \
  --output table
```

**What to observe:**
- The VM restarts during resize (downtime = real problem in production)
- After resize: `nproc` shows more CPUs, `free -h` shows more RAM
- No code changes needed — OS picks up new resources automatically

---

### Practical 2: Horizontal Scaling — Azure VM Scale Sets (VMSS)

**Goal:** Deploy a stateless web app that auto-scales based on CPU load.

**Architecture:**
```
[Internet] → [Azure Load Balancer] → [VMSS: 2–10 VMs]
                                         ↓
                                    [Azure Redis]
```

**Step 1: Create a Resource Group**
```bash
az group create --name rg-systemdesign --location eastus
```

**Step 2: Create the VM Scale Set**
```bash
az vmss create \
  --resource-group rg-systemdesign \
  --name vmss-webapi \
  --image UbuntuLTS \
  --vm-sku Standard_B2s \
  --instance-count 2 \
  --admin-username azureuser \
  --generate-ssh-keys \
  --upgrade-policy-mode automatic \
  --load-balancer vmss-webapi-lb \
  --backend-pool-name vmss-backend
```

**Step 3: Configure Auto-Scale Rules**
```bash
# Create autoscale profile
az monitor autoscale create \
  --resource-group rg-systemdesign \
  --resource vmss-webapi \
  --resource-type Microsoft.Compute/virtualMachineScaleSets \
  --name autoscale-webapi \
  --min-count 2 \
  --max-count 10 \
  --count 2

# Scale OUT rule: CPU > 70% for 5 min → add 2 instances
az monitor autoscale rule create \
  --resource-group rg-systemdesign \
  --autoscale-name autoscale-webapi \
  --condition "Percentage CPU > 70 avg 5m" \
  --scale out 2 \
  --cooldown 3

# Scale IN rule: CPU < 30% for 10 min → remove 1 instance
az monitor autoscale rule create \
  --resource-group rg-systemdesign \
  --autoscale-name autoscale-webapi \
  --condition "Percentage CPU < 30 avg 10m" \
  --scale in 1 \
  --cooldown 5
```

**Step 4: Deploy a Simple Stateless API on Each Instance**

Create a startup script (`startup.sh`):
```bash
#!/bin/bash
apt-get update -y
apt-get install -y python3 python3-pip
pip3 install flask redis

cat > /app/app.py << 'PYEOF'
from flask import Flask, jsonify
import os
import socket
import redis

app = Flask(__name__)
r = redis.Redis(host=os.environ.get('REDIS_HOST', 'localhost'), port=6379)

@app.route('/health')
def health():
    return jsonify({"status": "ok", "server": socket.gethostname()})

@app.route('/counter')
def counter():
    # State stored in Redis, not in-server memory
    count = r.incr('global_counter')
    return jsonify({
        "count": count,
        "served_by": socket.gethostname()  # Shows different servers handling requests
    })

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=8080)
PYEOF

python3 /app/app.py &
```

**Step 5: Test Stateless Horizontal Scaling**
```bash
# Hit the /counter endpoint 10 times — watch "served_by" change
for i in {1..10}; do
  curl http://<load-balancer-ip>/counter
done

# Expected output (different servers, same counter because Redis holds state):
# {"count": 1, "served_by": "vmss-webapi-0001"}
# {"count": 2, "served_by": "vmss-webapi-0003"}  ← different server!
# {"count": 3, "served_by": "vmss-webapi-0001"}
# Counter is consistent because state is in Redis, not in-server memory
```

**Step 6: Simulate Load and Watch Auto-Scale**
```bash
# Install stress tool on one instance
az vmss run-command invoke \
  --resource-group rg-systemdesign \
  --name vmss-webapi \
  --command-id RunShellScript \
  --instance-id 0 \
  --scripts "apt-get install -y stress && stress --cpu 4 --timeout 300"

# Watch instance count change in portal or CLI
watch -n 10 'az vmss show \
  --resource-group rg-systemdesign \
  --name vmss-webapi \
  --query sku.capacity'
```

**What to observe:**
- Instance count goes from 2 → 4 → 6 as CPU rises
- Counter endpoint keeps working during scale-out (no downtime)
- Counter is consistent because Redis holds the state

---

### Practical 3: Azure Cache for Redis (Externalizing State)

```bash
# Create Redis instance
az redis create \
  --resource-group rg-systemdesign \
  --name redis-systemdesign \
  --location eastus \
  --sku Basic \
  --vm-size c0

# Get connection string
az redis list-keys \
  --resource-group rg-systemdesign \
  --name redis-systemdesign
```

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
3. Why can't you just make every server stateful and use sticky sessions?
4. What is Amdahl's Law and why does it matter for system design?
5. A company's single MySQL server is running at 95% CPU. What are your options?
6. How does auto-scaling work and what are the risks?
7. What does "stateless" mean and how do you make a service stateless?
8. Your e-commerce site gets 10x traffic on Black Friday. Walk me through your scaling plan.

---

## Key Takeaways

| Concept | Rule of Thumb |
|---------|---------------|
| Vertical vs Horizontal | Start vertical, go horizontal when you hit the ceiling or need HA |
| Stateless | Any state that lives in server memory is a horizontal scaling blocker |
| Auto-scale triggers | Scale out on CPU/queue, scale in slowly (cooldown), never scale in below minimum |
| Database bottleneck | Add read replicas first, shard only when absolutely necessary |
| Bottleneck hunting | USE method: Utilization → Saturation → Errors, for every resource |
| Scaling stages | Single server → Separate DB → Load-balanced app → Read replicas → Global |
