# 📈 Scaling Writes — System Design Interview Notes

> **Goal:** Understand how to scale a system when the **write throughput** becomes the bottleneck.

---

# 1. The Core Problem

As an application grows, write traffic can increase from:

```text
Hundreds of writes/sec
        ↓
Thousands
        ↓
Millions
```

Eventually, a single database/server reaches limits in:

* Disk I/O
* CPU
* Network bandwidth
* Lock/contention
* Transaction processing

The important distinction is:

```text
Scaling Reads
    ├── Read Replicas
    ├── Caching
    └── CDN

Scaling Writes
    ├── Vertical Scaling
    ├── Database Optimization / Choice
    ├── Sharding / Partitioning
    ├── Queues / Load Shedding
    └── Batching / Aggregation
```

Write-heavy workloads are especially difficult because every write may require actual persistence and can create contention.

---

# 2. The Four Major Strategies

A strong interview answer usually progresses through these strategies:

```text
                    WRITE SCALING
                         │
          ┌──────────────┼──────────────┐
          │              │              │
          ▼              ▼              ▼
   Vertical Scale    Horizontal     Handle Bursts
   + DB Optimize     Sharding       + Queues
          │              │              │
          └──────────────┴──────────────┘
                         │
                         ▼
                 Batching /
             Hierarchical Aggregation
```

The four strategies are:

1. **Vertical Scaling + Database Optimization**
2. **Sharding / Partitioning**
3. **Queues + Load Shedding**
4. **Batching + Hierarchical Aggregation**

A good interview progression is:

```text
Can one server handle it?
        │
        ├── YES → Optimize it
        │
        └── NO
             ↓
      Can we distribute writes?
             │
             └── Sharding
                    ↓
           Are writes bursty?
                    │
                    └── Queues
                          ↓
              Can we aggregate writes?
                          │
                          └── Batching
```

---

# 3. First Step: Vertical Scaling

Before introducing distributed-system complexity, make sure the existing hardware is actually exhausted.

## What can limit writes?

```text
                 Database Server
                      │
        ┌─────────────┼─────────────┐
        ▼             ▼             ▼
      CPU          Disk I/O       Network
```

For example:

* CPU = 100%
* Disk is saturated
* Network bandwidth is exhausted

If the database is otherwise healthy, upgrading the machine may be the simplest solution.

---

## Why Vertical Scaling Matters in Interviews

Don't immediately jump to:

> "Let's add 10 database servers."

First ask:

> "Can a larger machine handle this workload?"

Modern infrastructure can provide machines with significantly more CPU, memory, disk and network capacity.

So your initial answer can be:

> "I'd first determine whether we're CPU-, disk-, or network-bound and see whether vertical scaling can handle the required throughput before introducing sharding."

---

# 4. Database Choice Matters

Hardware isn't the only consideration.

The **database itself** can determine write throughput.

Different databases make different trade-offs between:

```text
Write Performance
        ↕
Read Performance
        ↕
Consistency
        ↕
Query Flexibility
```

---

# 5. Write-Optimized Databases

A classic example is **Cassandra**.

Cassandra is designed for workloads with very high write throughput.

Instead of constantly updating data in place, an append-oriented architecture can write sequentially.

Conceptually:

```text
Traditional DB

Update Row
   ↓
Find location
   ↓
Modify data
   ↓
Update indexes
   ↓
Disk I/O


Append-oriented DB

New Write
   ↓
Append
   ↓
Sequential Disk Write
```

Sequential writes can be substantially cheaper than random disk updates.

---

# 6. The Write vs Read Trade-off

A very important interview concept:

> **Optimizing writes can hurt reads.**

For example:

```text
          Write Performance
                 ▲
                 │
        Cassandra│
                 │
                 │
                 │
                 │
                 └──────────────► Read Performance
```

A write-optimized database may require more work when reading because data can be spread across multiple files/structures that need to be merged.

Therefore:

```text
Write-heavy workload
        ↓
Optimize for writes

Read-heavy workload
        ↓
Optimize for reads
```

Don't simply say:

> "Use Cassandra because it's faster."

Say:

> "This workload is write-heavy, and Cassandra's storage architecture is better suited to high-volume writes. I'm accepting some read/query trade-offs in exchange."

---

# 7. Other Write-Friendly Database Choices

Depending on the workload:

### Time-series databases

Examples:

* InfluxDB
* TimescaleDB

Useful for:

```text
timestamp + metric + value
```

Typical workloads:

* Metrics
* Monitoring
* Sensor data
* Events

---

### Log-structured databases

Example:

* LevelDB

General idea:

```text
New data
   ↓
Append
   ↓
Don't constantly update data in-place
```

---

### Column stores

Example:

* ClickHouse

Useful particularly for analytics workloads where writes can be efficiently batched and data is stored in a column-oriented format.

---

# 8. Optimizing an Existing Database

Even if you keep your current database, there are several optimizations.

### 1. Reduce expensive features

Potential sources of write overhead:

* Foreign key constraints
* Complex triggers
* Full-text search indexes
* Excessive secondary indexes

---

### 2. Tune Write-Ahead Logging

For databases such as PostgreSQL, WAL behavior can be tuned to improve write efficiency.

The general idea:

```text
Many small writes
      ↓
Batch / optimize persistence
      ↓
Fewer expensive disk flushes
```

---

### 3. Reduce Index Overhead

Every additional index can make writes more expensive.

Example:

```text
INSERT row
   │
   ├── Update primary index
   ├── Update index A
   ├── Update index B
   ├── Update index C
   └── Update index D
```

More indexes:

```text
Better reads
     BUT
More expensive writes
```

Therefore:

> Keep only the indexes that provide meaningful read/query value.

---

# 9. Horizontal Scaling: Sharding

Eventually, one server isn't enough.

Suppose:

```text
1 server = 1,000 writes/sec
```

Ideally:

```text
10 servers ≈ 10,000 writes/sec
```

So instead of making one machine bigger, distribute data across multiple machines.

These machines are called **shards**.

---

# 10. What Is a Shard?

Think of the database logically as one system:

```text
                Logical Database
                       │
        ┌──────────────┼──────────────┐
        ▼              ▼              ▼
     Shard 1        Shard 2        Shard 3
     Server A       Server B       Server C
```

Each shard owns only a subset of the data.

The application/client determines where a particular piece of data belongs.

---

# 11. Example: Hash-Based Sharding

Suppose we shard using:

```text
userId
```

Conceptually:

```text
userId
  │
  ▼
Hash Function
  │
  ▼
Partition / Slot
  │
  ▼
Responsible Server
```

Example:

```text
User 101 ──hash──> Slot 2 ──> Server B
User 202 ──hash──> Slot 0 ──> Server A
User 303 ──hash──> Slot 3 ──> Server D
```

This is similar to how Redis Cluster distributes keys using hash slots.

---

# 12. Sharding Diagram

```text
                         Client
                           │
                           │ userId
                           ▼
                     Hash Function
                           │
                           ▼
                    Partition / Slot
                           │
             ┌─────────────┼─────────────┐
             ▼             ▼             ▼
          Shard A       Shard B       Shard C
             │             │             │
          Users         Users         Users
          0-...         ...           ...
```

The important point:

> The client doesn't randomly choose a server. The partitioning algorithm determines the responsible shard.

---

# 13. Choosing the Partition Key

This is one of the **most important interview questions**.

You don't just say:

> "We'll shard."

You need to explain:

> **"What key will you shard on, and why?"**

A good partition key should distribute writes relatively evenly.

Common candidates:

* `userId`
* `postId`
* `orderId`
* `tenantId`

Depending on the workload.

---

# 14. Good Partition Key

Suppose we hash `userId`.

```text
Hash(userId)
      │
      ├── Shard A
      ├── Shard B
      ├── Shard C
      └── Shard D
```

Ideally:

```text
Shard A → 25% writes
Shard B → 25% writes
Shard C → 25% writes
Shard D → 25% writes
```

The exact percentages won't necessarily be perfect, but the goal is to minimize imbalance.

### Key principle:

> **Flat is good.**

You want the write distribution across shards to have low variance.

---

# 15. Bad Partition Key

Suppose instead we shard using:

```text
country
```

Imagine:

```text
China       → 70% of writes
India       → 15%
USA         → 10%
New Zealand → 1%
Others      → 4%
```

Now:

```text
China Shard
███████████████████████████████████

New Zealand Shard
█
```

One shard becomes overloaded while others sit mostly idle.

This creates a **hot shard**.

---

# 16. Hot Shard

A hot shard is a shard receiving disproportionately high traffic.

```text
             Incoming Writes
                    │
        ┌───────────┼───────────┐
        ▼           ▼           ▼
     Shard A     Shard B      Shard C
      20%          70%          10%
                    ▲
                    │
                HOT SHARD
```

Adding more servers does not automatically solve the problem if your partitioning algorithm keeps sending most traffic to one shard.

This is why **partition-key selection is critical**.

---

# 17. Hashing vs Consistent Hashing

Two common ways to distribute data:

### Hash → partition/slot

```text
key
 ↓
hash
 ↓
slot
 ↓
server
```

### Consistent hashing

```text
              Server A
                 ●
        ●                    ●
     Server D              Server B
        ●                    ●
              Server C

          Consistent Hash Ring
```

Consistent hashing helps when servers are added/removed because it can minimize how much data needs to move.

In interviews, understand:

* Hashing
* Consistent hashing
* Virtual nodes
* Slot assignment

---

# 18. Sharding Has a Major Trade-off

Even if writes are perfectly distributed, reads can become expensive.

Suppose:

```text
4 shards
```

and a request needs data from all users.

The request might become:

```text
                    Query
                      │
          ┌───────────┼───────────┐
          ▼           ▼           ▼
       Shard A     Shard B      Shard C
          │           │           │
          └───────────┼───────────┘
                      ▼
                   Merge
```

Instead of one database query, you now have multiple network calls and a merge operation.

---

# 19. Read Pattern Must Influence Sharding

When choosing a partition key, ask:

### Question 1

> How many shards does this request need to hit?

### Question 2

> How frequently does this request happen?

Example:

```text
Request A
→ only Shard 2
→ very efficient


Request B
→ Shard 1
→ Shard 2
→ Shard 3
→ Shard 4
→ expensive
```

So the ideal partitioning strategy should:

```text
                    Partitioning
                         │
             ┌───────────┴───────────┐
             ▼                       ▼
       Spread writes            Localize reads
       evenly                   when possible
```

This is a critical design trade-off.

---

# 20. Horizontal vs Vertical Partitioning

Don't confuse these two.

## Horizontal Partitioning / Sharding

Splits **rows**.

```text
Original table

User 1
User 2
User 3
User 4
User 5
User 6

        ↓

Shard A       Shard B
User 1        User 4
User 2        User 5
User 3        User 6
```

---

## Vertical Partitioning

Splits **columns / types of data**.

```text
Original Table
────────────────────────────
id
user_id
content
media
likes
comments
views
analytics
created_at
────────────────────────────

             ↓

 ┌──────────────┐
 │ Post Content │
 └──────────────┘

 ┌──────────────┐
 │ Post Metrics │
 └──────────────┘

 ┌──────────────┐
 │ Post Analytics│
 └──────────────┘
```

### Remember:

```text
Horizontal → Rows
Vertical   → Columns / Data domains
```

---

# 21. Why Vertical Partitioning Helps Writes

Consider a social-media post.

A single giant table may contain:

```text
Post
├── content
├── media
├── likes
├── comments
├── shares
├── views
└── analytics
```

Different fields have very different access patterns.

For example:

```text
Post Content
→ written once
→ read many times

Post Metrics
→ updated constantly

Analytics
→ continuously appended
```

Putting everything together means these workloads interfere with one another.

---

# 22. Better Design

Split the data logically.

```text
                    POST
                     │
       ┌─────────────┼──────────────┐
       ▼             ▼              ▼
Post Content    Post Metrics    Post Analytics
       │             │              │
       ▼             ▼              ▼
 Write-once       High-frequency   Append-only
 Read-many          updates          events
```

---

# 23. Example Data Model

### Post Content

```sql
TABLE post_content (
    post_id BIGINT PRIMARY KEY,
    user_id BIGINT,
    content TEXT,
    media_urls TEXT[],
    created_at TIMESTAMP
);
```

Characteristics:

```text
Write frequency → Low
Read frequency  → High
```

---

### Post Metrics

```sql
TABLE post_metrics (
    post_id BIGINT PRIMARY KEY,
    like_count INTEGER DEFAULT 0,
    comment_count INTEGER DEFAULT 0,
    share_count INTEGER DEFAULT 0,
    view_count INTEGER DEFAULT 0,
    last_updated TIMESTAMP
);
```

Characteristics:

```text
Write frequency → Very High
```

---

### Post Analytics

```sql
TABLE post_analytics (
    post_id BIGINT,
    event_type VARCHAR(50),
    timestamp TIMESTAMP,
    user_id BIGINT,
    metadata JSONB
);
```

Characteristics:

```text
Write pattern → Append-only
Workload      → Analytics
```

---

# 24. Different Data → Different Storage

Once data is separated logically, each workload can use the storage system best suited to it.

```text
                  Application
                       │
          ┌────────────┼─────────────┐
          ▼            ▼             ▼
    Post Content   Post Metrics   Analytics
          │            │             │
          ▼            ▼             ▼
      SQL DB       In-memory /    Time-series /
      B-tree       counters       column store
```

For example:

### Content

Use a traditional relational database with B-tree indexes.

Optimized for:

```text
Read performance
```

### Metrics

Use:

```text
In-memory storage
or
specialized counters
```

Optimized for:

```text
High-frequency updates
```

### Analytics

Use:

```text
Time-series storage
or
Column-oriented storage
```

Optimized for:

```text
Append-heavy analytics
```

---

# 25. The Deeper Principle

The biggest lesson isn't simply:

> "Use sharding."

It's:

> **Understand the access pattern of your data and physically organize the system around those access patterns.**

Think:

```text
Data
 │
 ├── How often is it written?
 ├── How often is it read?
 ├── Is it updated or appended?
 ├── Is it queried together with other data?
 ├── Does it have hot keys?
 └── What consistency does it need?
```

Then choose:

```text
        Access Pattern
              │
              ▼
       Data Organization
              │
      ┌───────┼────────┐
      ▼       ▼        ▼
   DB type  Sharding  Partitioning
```

---

# 26. Interview Answer Framework

When an interviewer says:

> "Your database can't handle the write traffic anymore. What do you do?"

Use this sequence:

### Step 1 — Measure the bottleneck

```text
Is it:
CPU?
Disk?
Network?
Locks/contention?
```

### Step 2 — Vertical scaling

Ask:

> Can a larger machine handle the workload?

### Step 3 — Optimize the database

Consider:

* Remove unnecessary indexes
* Reduce expensive constraints/triggers
* Tune WAL
* Optimize schema
* Choose a write-optimized database if appropriate

### Step 4 — Shard

If one server still isn't enough:

```text
Data
 ↓
Partition Key
 ↓
Hash / Consistent Hashing
 ↓
Shard
```

### Step 5 — Validate the partition key

Ask:

> Does it distribute writes evenly?

Watch for:

```text
HOT SHARDS
```

### Step 6 — Check read implications

Ask:

> How many shards does a typical read need to contact?

### Step 7 — Separate workloads

Use vertical partitioning when different data has dramatically different access patterns.

```text
Content
Metrics
Analytics
```

can become separate logical/physical systems.

### Step 8 — Handle bursts

If traffic is not constant but arrives in spikes:

```text
Producer
   ↓
Queue
   ↓
Consumers
   ↓
Database
```

Queues and load shedding become important.

### Step 9 — Batch writes

If individual writes can be combined:

```text
1,000 individual writes
        ↓
      Batch
        ↓
1 bulk write
```

This reduces per-write overhead.

---

# 27. Quick Comparison

| Technique             | What it solves                  | Main trade-off                  |
| --------------------- | ------------------------------- | ------------------------------- |
| Vertical scaling      | Single-server capacity          | Hardware limit                  |
| DB optimization       | Unnecessary write overhead      | May reduce read capabilities    |
| Write-optimized DB    | High write throughput           | Read/query trade-offs           |
| Horizontal sharding   | Total write capacity            | Distributed complexity          |
| Good partition key    | Even load                       | Must understand access patterns |
| Consistent hashing    | Server changes / distribution   | More complexity                 |
| Vertical partitioning | Different workloads interfering | Cross-table coordination        |
| Queues                | Bursty writes                   | Added latency / async behavior  |
| Batching              | Per-write overhead              | Data may be processed later     |

---

# 28. Most Important Interview Concepts

If you have limited time, memorize these:

### ⭐ 1. Vertical scaling first

Don't introduce distributed complexity prematurely.

### ⭐ 2. Identify the actual bottleneck

```text
CPU
Disk I/O
Network
Contention
```

### ⭐ 3. Write-heavy databases make trade-offs

Example:

```text
Better writes
     ↕
Potentially worse reads
```

### ⭐ 4. Sharding requires a good partition key

Bad:

```text
country
```

Potentially good:

```text
hash(userId)
hash(postId)
```

depending on access patterns.

### ⭐ 5. Avoid hot shards

```text
Uneven distribution = bottleneck
```

### ⭐ 6. Consider reads when designing writes

A partitioning scheme that perfectly distributes writes can still make reads extremely expensive.

### ⭐ 7. Horizontal vs vertical partitioning

```text
Horizontal → split rows
Vertical   → split columns/data domains
```

### ⭐ 8. Separate workloads

```text
Content
Metrics
Analytics
```

may deserve different storage and scaling strategies.

---

# 29. One-Minute Revision

If you only have one minute before the interview:

```text
SCALING WRITES
│
├── 1. Vertical Scaling
│      └── CPU / Disk / Network
│
├── 2. Database Optimization
│      ├── Reduce indexes
│      ├── Reduce expensive features
│      ├── Tune WAL
│      └── Write-optimized DB
│
├── 3. Sharding
│      ├── Split rows across servers
│      ├── Good partition key
│      ├── Avoid hot shards
│      ├── Hashing / Consistent Hashing
│      └── Consider read fan-out
│
├── 4. Vertical Partitioning
│      ├── Split data by access pattern
│      ├── Content
│      ├── Metrics
│      └── Analytics
│
├── 5. Queues
│      └── Absorb bursts
│
└── 6. Batching
       └── Reduce per-write overhead
```

## The core mental model

> **First make one machine efficient → then distribute writes → then handle uneven/bursty traffic → then separate workloads and batch operations where possible.**

# Handling Bursts with Queues and Load Shedding

## 1. The Core Problem: Traffic > Capacity

In real-world systems, write traffic is rarely perfectly steady.

A database may be able to sustainably handle:

```text
1,000 writes/sec
```

But during a traffic spike:

```text
Incoming traffic = 4,000 writes/sec
Database capacity = 1,000 writes/sec
```

The fundamental problem is:

> **The system is receiving work faster than it can complete work.**

If this continues:

```text
Incoming:   4000/sec
Processed:  1000/sec
             ↓
Backlog:   +3000/sec
```

After one minute:

```text
Backlog = 3000 × 60
        = 180,000 writes
```

---

# 2. Three Ways to Handle Overload

When:

```text
Traffic > Capacity
```

there are three broad strategies:

### A. Increase capacity

Use:

* Partitioning
* Sharding
* Horizontal scaling
* More workers
* Better hardware

Goal:

> Increase sustainable throughput.

---

### B. Buffer the work

Use:

* Kafka
* SQS
* RabbitMQ
* Other queues

Goal:

> Accept work now and process it later.

---

### C. Drop less-important work

Use:

* Load shedding
* Sampling
* Deduplication
* Coalescing
* Rate limiting
* Graceful degradation

Goal:

> Keep the system alive by refusing work that is not worth the cost during overload.

---

# 3. Partitioning vs Queue vs Load Shedding

This distinction is extremely important.

| Technique     | Main Purpose                                         |
| ------------- | ---------------------------------------------------- |
| Partitioning  | Increase sustainable throughput                      |
| Sharding      | Spread data/work across machines                     |
| Queue         | Absorb temporary bursts                              |
| Load shedding | Prevent overload by rejecting work                   |
| Backpressure  | Slow/restrict producers when consumers can't keep up |

Think:

```text
Partitioning
    ↓
"I need more capacity."

Queue
    ↓
"I need to wait until I have capacity."

Load shedding
    ↓
"I don't need to process every request."

Backpressure
    ↓
"Slow down because I can't keep up."
```

---

# 4. Why Autoscaling Isn't a Complete Solution

A common answer is:

> "We'll autoscale."

Autoscaling is useful, but it is not instantaneous.

A typical scaling process may involve:

```text
Detect high load
      ↓
Provision resources
      ↓
Initialize resources
      ↓
Join cluster
      ↓
Rebalance / replicate
      ↓
Ready
```

The traffic spike may happen immediately:

```text
Normal:
800 writes/sec

       ↓ Black Friday

4000 writes/sec
```

But scaling might take time.

During that period:

```text
Incoming = 4000/sec
Capacity = 1000/sec
```

The system is already overloaded.

Database scaling can also involve:

* Data redistribution
* Replication
* Rebalancing
* Reduced throughput
* Operational complexity
* Sometimes downtime

Therefore:

> **Autoscaling handles capacity changes, but queues and load shedding handle immediate overload.**

---

# 5. What Is a Burst?

A burst is a temporary increase in traffic.

Example:

```text
                 BURST
                   ↓
800 ───────────── 4000 ──────────── 800
                   ↑
                2 minutes
```

The important property is:

> **The spike eventually goes away.**

This matters because queues work well for temporary bursts.

---

# 6. Queue as a Shock Absorber

A queue sits between the application and the database:

```text
Clients
   ↓
App Servers
   ↓
 Queue
   ↓
Workers
   ↓
Database
```

Without a queue:

```text
Client
  ↓
Database
  ↓
OVERLOADED
  ↓
ERROR
```

With a queue:

```text
Client
  ↓
Queue
  ↓
"Accepted"
  ↓
Worker
  ↓
Database
```

The queue absorbs the temporary difference between:

```text
Incoming rate
        and
Processing rate
```

---

# 7. Queue Example

Suppose:

```text
Incoming = 4,000 writes/sec
Database = 1,000 writes/sec
```

The queue grows by:

```text
4000 - 1000
= 3000 messages/sec
```

After 60 seconds:

```text
3000 × 60
= 180,000 messages
```

Now suppose the spike ends.

Traffic becomes:

```text
500 writes/sec
```

Database still processes:

```text
1000 writes/sec
```

Now:

```text
Incoming = 500
Processing = 1000

Backlog decreases by 500/sec
```

Eventually:

```text
Queue = 0
```

This is **burst absorption**.

---

# 8. Queue Does NOT Increase Database Capacity

This is a common misunderstanding.

A queue does not turn:

```text
Database capacity = 1000 writes/sec
```

into:

```text
Database capacity = 5000 writes/sec
```

Instead:

```text
4000 requests/sec
       ↓
     Queue
       ↓
1000 writes/sec
       ↓
Database
```

The database still processes:

```text
1000 writes/sec
```

The difference is:

> Instead of failing immediately, requests wait.

---

# 9. The Queue Equation

A useful mental model:

```text
Backlog change
    =
Incoming rate - Processing rate
```

### If:

```text
Incoming > Processing
```

then:

```text
Backlog ↑
```

### If:

```text
Incoming < Processing
```

then:

```text
Backlog ↓
```

### If:

```text
Incoming = Processing
```

then:

```text
Backlog stays stable
```

This equation explains most queue behavior.

---

# 10. Queue Works Only for Temporary Bursts

Suppose:

```text
Capacity = 1000/sec
Traffic = 4000/sec
```

If traffic stays at 4000 forever:

```text
Backlog
   ↓
100K
   ↓
500K
   ↓
1M
   ↓
10M
   ↓
...
```

The queue grows without bound.

Eventually:

* Queue storage fills
* Latency becomes huge
* Users wait longer
* Workers fall further behind
* Retries may increase traffic
* The system can fail

Therefore:

> **A queue is a temporary buffer, not a permanent capacity solution.**

Use queues when:

```text
Burst is temporary
        AND
Delayed processing is acceptable
```

---

# 11. Asynchronous Writes Change the API Contract

Without a queue:

```text
POST /order

Client
  ↓
Database
  ↓
Order created
  ↓
Response
```

The server knows:

> "The order was successfully written."

With a queue:

```text
POST /order

Client
  ↓
API
  ↓
Queue
  ↓
Response
```

The server only knows:

> "The order was successfully placed in the queue."

It does NOT necessarily know:

> "The database has successfully stored the order."

Therefore the API may return:

```text
orderId: 12345
status: PROCESSING
```

The client may later check:

```text
GET /orders/12345
```

Possible states:

```text
PROCESSING
     ↓
COMPLETED

or

PROCESSING
     ↓
FAILED
```

---

# 12. When Queues Are Appropriate

Queues are good when:

* Writes don't need immediate database confirmation
* Some delay is acceptable
* Bursts are temporary
* Processing can happen asynchronously
* Work should not be lost
* The database can eventually catch up

Examples:

* Sending emails
* Generating reports
* Video processing
* Analytics ingestion
* Notifications
* Background jobs
* Order processing in some architectures

---

# 13. When Queues Are Dangerous

Don't use a queue simply because:

> "The database can't handle our normal traffic."

Example:

```text
Normal traffic = 5000 writes/sec
Database = 1000 writes/sec
```

Putting Kafka in front doesn't solve the underlying problem.

Instead:

```text
5000/sec → Queue → 1000/sec DB
```

means the queue grows forever.

Correct solution:

> Increase steady-state processing capacity.

Use:

* Partitioning
* Sharding
* Database scaling
* Better indexing
* More workers
* Better schema/design

Then use a queue for temporary bursts.

---

# 14. Load Shedding

Load shedding means:

> **When the system is overloaded, intentionally reject or discard less-important work so the important work can continue.**

Instead of:

```text
Everything accepted
       ↓
Everything overloaded
       ↓
Everything fails
```

we do:

```text
Incoming requests
       ↓
Load Shedder
    ↙       ↘
Important   Unimportant
   ↓            ↓
Accept        Drop
```

The philosophy is:

> **Partial failure is better than total failure.**

---

# 15. Why Dropping Writes Can Be Correct

Not every write has equal business value.

Example:

### Payment

```text
CRITICAL
```

Dropping it is unacceptable.

### Order creation

```text
CRITICAL
```

Usually must be processed.

### Driver location

```text
IMPORTANT
but potentially lossy
```

One location update can sometimes be skipped because another will arrive shortly.

### Typing indicator

```text
LOW VALUE
```

Dropping some events is fine.

### Analytics impression

```text
OFTEN LOSABLE
```

Sampling may be acceptable.

---

# 16. Uber / Strava Location Example

Suppose drivers send location every 5 seconds.

```text
Driver
  ↓
Location update
  ↓
Server
  ↓
Database
```

During normal traffic:

```text
Everything accepted
```

During overload:

```text
Driver
  ↓
Location update
  ↓
Is the last update recent?
       │
    ┌──┴──┐
   YES    NO
    │      │
  DROP   ACCEPT
```

Example:

```text
10:00:00 → Bangalore
10:00:05 → Bangalore
10:00:10 → Bangalore
10:00:15 → Bangalore
```

If the system already knows the driver's recent location, processing every update may not provide enough additional value to justify the database load.

---

# 17. Why This Is Better Than Queuing Everything

Suppose:

```text
Incoming location updates = 600,000/sec
Database capacity = 200,000/sec
```

Queue approach:

```text
600K/sec
   ↓
Queue
   ↓
200K/sec
```

Backlog grows by:

```text
400K/sec
```

That's unsustainable.

But suppose the latest location is all we need.

We can discard redundant updates:

```text
600K incoming
      ↓
Load shedding / coalescing
      ↓
150K useful updates
      ↓
Database
```

Now the database can keep up.

---

# 18. The Key Question: "Can I Lose This Write?"

Whenever designing a write-heavy system, ask:

> **Can this write be safely lost?**

Classify writes:

```text
                  Can lose?
                     │
          ┌──────────┴──────────┐
          │                     │
         NO                    YES
          │                     │
     Must process          Can shed
          │                     │
       Queue                  Drop
       Retry                 Sample
       Persist              Coalesce
```

Examples:

| Write                | Can lose? |
| -------------------- | --------- |
| Payment              | ❌         |
| Order creation       | ❌         |
| Account creation     | ❌         |
| Driver location      | Sometimes |
| Typing indicator     | ✅         |
| Presence update      | Often     |
| Analytics impression | Often     |
| Debug telemetry      | Often     |

---

# 19. Load Shedding Strategies

## 19.1 Drop Low-Priority Requests

```text
Priority 1 → Accept
Priority 2 → Accept
Priority 3 → Reject
```

Critical business operations survive.

---

## 19.2 Rate Limiting

Limit how much a client can send.

Example:

```text
Maximum:
10 location updates/sec/user
```

Requests beyond the limit:

```text
HTTP 429 Too Many Requests
```

---

## 19.3 Sampling

Instead of processing every event:

```text
100% → 10%
```

Example:

```text
10 million analytics impressions
        ↓
Keep only 1 million
```

Useful when approximate analytics are acceptable.

---

## 19.4 Deduplication

If the same information arrives repeatedly:

```text
location(driver=123, Bangalore)
location(driver=123, Bangalore)
location(driver=123, Bangalore)
```

you don't necessarily need to process all of them.

Keep one.

---

## 19.5 Coalescing

If only the latest state matters:

```text
A
B
C
D
E
```

process only:

```text
E
```

Conceptually:

```text
A ─┐
B ─┤
C ─┤
D ─┤──→ Latest state = E
E ─┘
```

Perfect for:

* GPS location
* Presence
* Online/offline state
* Typing indicators
* Frequently changing metrics

---

## 19.6 Graceful Degradation

Instead of completely failing, provide a cheaper version.

For example:

```text
Normal:
Real-time recommendations

Overload:
Basic recommendations
```

Or:

```text
Normal:
Real-time analytics

Overload:
Analytics delayed by 5 minutes
```

This keeps the core product usable.

---

# 20. Load Shedding Should Happen Early

Bad:

```text
Client
  ↓
API
  ↓
Queue
  ↓
Worker
  ↓
Database
  ↓
OVERLOAD
  ↓
Reject
```

Resources were already consumed.

Better:

```text
Client
  ↓
API
  ↓
Load Shedder
  ↓
Important?
 ├── NO → DROP
 └── YES
       ↓
     Queue
       ↓
     Worker
       ↓
      DB
```

Rejecting early saves:

* CPU
* Memory
* Network
* Queue capacity
* Worker capacity
* Database connections

---

# 21. Detecting Overload

Load shedding needs signals.

Useful metrics:

```text
Database CPU
Database latency
Database error rate
Queue depth
Queue lag
Worker utilization
Request latency
Connection pool usage
```

Example:

```text
DB CPU < 70%
    ↓
Normal

DB CPU 70–90%
    ↓
Warning

DB CPU > 90%
    ↓
Start shedding low-priority traffic
```

Or:

```text
Queue lag > 30 sec
       ↓
Enable aggressive load shedding
```

---

# 22. Queue Depth Is a Critical Signal

Example:

```text
Queue depth = 1,000
```

Healthy.

Then:

```text
Queue depth = 10,000
```

Warning.

Then:

```text
Queue depth = 100,000
```

System is falling behind.

Then:

```text
Queue depth = 1,000,000
```

Serious overload.

Possible policy:

```text
Queue < 10K
    ↓
Normal

10K–100K
    ↓
Warning

>100K
    ↓
Shed low-priority traffic

>500K
    ↓
Aggressive shedding
```

---

# 23. Partitioning + Queue + Load Shedding

These techniques solve different problems.

Suppose:

```text
Database capacity = 10,000 writes/sec
```

Partitioning gives:

```text
10 partitions × 1000 writes/sec
= 10,000 writes/sec
```

Now Black Friday:

```text
Traffic = 40,000 writes/sec
```

Even after partitioning:

```text
Capacity = 10K
Traffic = 40K
```

Still overloaded.

So:

```text
Partitioning
     ↓
Increase capacity

Queue
     ↓
Absorb temporary bursts

Load shedding
     ↓
Reject unnecessary work
```

---

# 24. Backpressure

Backpressure means:

> **When downstream cannot keep up, communicate that pressure upstream so the producer slows down or stops sending work.**

Example:

```text
Producer
5000/sec
   ↓
 Queue
   ↓
Consumer
1000/sec
```

The consumer cannot keep up.

The queue gets full.

Eventually:

```text
Queue
████████████████████
        ↓
     "Slow down!"
        ↓
Producer
```

Possible responses:

* Slow down producers
* Reject requests
* Reduce event frequency
* Apply rate limits
* Shed load
* Reduce batch size
* Pause consumers/producers

---

# 25. Load Shedding vs Backpressure

These concepts are related but different.

### Backpressure

> "You are sending too much. Slow down."

### Load shedding

> "I cannot process everything. I will intentionally reject some work."

Example:

```text
Producer
   ↓
Backpressure
   ↓
Reduce traffic
```

versus:

```text
Producer
   ↓
Load Shedder
   ↓
Drop selected requests
```

Backpressure attempts to **control the source**.

Load shedding decides **which work to sacrifice**.

---

# 26. Retry Storms

Overload can create a dangerous feedback loop.

Consider:

```text
Client
  ↓
Database overloaded
  ↓
Request times out
  ↓
Client retries
  ↓
More requests
  ↓
Database becomes even more overloaded
  ↓
More timeouts
  ↓
More retries
```

This creates:

> **Retry storm**

Diagram:

```text
Overload
   ↓
Timeouts
   ↓
Retries
   ↓
More traffic
   ↓
More overload
   ↓
More timeouts
```

This can cause cascading failure.

Therefore retries should be controlled using:

* Exponential backoff
* Jitter
* Maximum retry count
* Timeouts
* Circuit breakers
* Idempotency

---

# 27. Combining Everything

A robust write architecture might look like:

```text
                         Incoming Writes
                               │
                               ↓
                       ┌───────────────┐
                       │ Rate Limiter  │
                       └───────┬───────┘
                               ↓
                       ┌───────────────┐
                       │Load Shedding  │
                       └───────┬───────┘
                               │
                 ┌─────────────┴─────────────┐
                 │                           │
             Important                  Unimportant
                 │                           │
                 ↓                           ↓
               Queue                        DROP
                 │
                 ↓
              Workers
                 │
                 ↓
          Partition / Shard
            /     |     \
           ↓      ↓      ↓
         DB-1   DB-2   DB-3
```

Each component has a different responsibility:

```text
Rate Limiting
    ↓
Prevent abuse / excessive traffic

Load Shedding
    ↓
Discard low-value work

Queue
    ↓
Absorb temporary bursts

Workers
    ↓
Control processing rate

Partitioning / Sharding
    ↓
Increase sustainable capacity

Retries
    ↓
Recover from transient failures

Backpressure
    ↓
Prevent downstream overload

Graceful Degradation
    ↓
Keep core functionality alive
```

---

# 28. The Most Important Decision Tree

When the interviewer says:

> "Traffic suddenly becomes 4×."

Think:

```text
                  Traffic > Capacity
                         │
              ┌──────────┴──────────┐
              │                     │
        Is it temporary?       Is it sustained?
              │                     │
             YES                    YES
              │                     │
            Queue             Increase capacity
              │
              ↓
       Can backlog drain?
              │
         ┌────┴────┐
        YES        NO
         │          │
      Queue works   │
                    ↓
              Load shedding
                    │
              ┌─────┴─────┐
              │           │
          Critical     Non-critical
              │           │
           Process      Drop /
                        Sample /
                        Coalesce
```

This is an excellent mental model for interviews.

---

# 29. Interview Answer Template

If asked:

> "How would you handle a sudden 4× increase in writes?"

A strong answer:

> "First I'd determine whether the spike is temporary or sustained. For a temporary burst, I'd put an asynchronous queue between the application and the database so that the queue can absorb the spike while workers process writes at the database's sustainable rate. I'd monitor queue depth and lag to ensure the backlog can eventually drain.
>
> If the queue continues growing because traffic exceeds capacity for a sustained period, a queue alone won't solve the problem. I'd increase the underlying write capacity through partitioning, sharding, or additional workers.
>
> I'd also classify writes by business importance. If the system is still approaching saturation, I'd shed low-value writes such as redundant location updates or analytics events while protecting critical operations such as payments and order creation.
>
> Finally, I'd use backpressure, rate limiting, controlled retries, exponential backoff, and graceful degradation to prevent overload from turning into a cascading failure."

---

# 30. The One Mental Model to Remember

Whenever:

```text
Traffic > Capacity
```

ask these questions:

### Question 1

> Can I increase capacity?

If yes:

```text
Partition
Shard
Scale
Optimize
```

---

### Question 2

> Is this only a temporary spike?

If yes:

```text
Queue
```

---

### Question 3

> Can every write be delayed?

If yes:

```text
Queue
```

---

### Question 4

> Can every write be lost?

If yes:

```text
Load shedding
Sampling
Deduplication
Coalescing
```

---

### Question 5

> Which writes are critical?

Protect them.

```text
Payment
Order
Account
```

---

### Question 6

> Which writes are replaceable or low-value?

Shed them.

```text
Location
Presence
Typing
Analytics
```

---

### Question 7

> What happens if the queue keeps growing?

Then:

```text
Queue is NOT the solution.
```

You need:

```text
More capacity
       +
Backpressure
       +
Load shedding
```

---

# 31. Final Summary

The complete picture:

```text
                    WRITE TRAFFIC
                         │
                         ↓
                  Traffic > Capacity?
                         │
                         ↓
                ┌──────────────────┐
                │ Can scale?       │
                └────────┬─────────┘
                         │
                       YES
                         ↓
                 Partition / Shard
                         │
                         ↓
                 Increase capacity
                         │
                         ↓
                  Still bursty?
                         │
                        YES
                         ↓
                       Queue
                         │
                         ↓
               Can backlog drain?
                    /         \
                  YES          NO
                   │            │
                   ↓            ↓
               Queue works   Load shedding
                                │
                         ┌──────┴──────┐
                         │             │
                      Critical     Non-critical
                         │             │
                      Process       Drop /
                                    Sample /
                                    Coalesce
```

### The three sentences worth memorizing

> **Partitioning/sharding increases the amount of work the system can sustainably process.**

> **Queues absorb temporary bursts by allowing work to be processed later.**

> **Load shedding protects the system during overload by intentionally dropping work that is less valuable than keeping the entire system alive.**

And the deepest principle:

> **A good distributed system does not try to successfully process every possible request at all costs. It protects the most important work and gracefully sacrifices less-important work when capacity is exhausted.**

This is what turns a system from:

```text
"Works when everything is normal"
```

into:

```text
"Survives when production gets ugly."
```

# Batching & Hierarchical Aggregation — Quick Revision

## 1. Core Idea

When writes become too expensive, don't always try to make the database faster.

Ask:

> **Can I reduce the number of writes or make them cheaper to process?**

```text
Too many writes
      ↓
Batch / Aggregate
      ↓
Fewer DB operations
      ↓
Lower load
```

---

# 2. Batching

Instead of:

```text
Write A → DB
Write B → DB
Write C → DB
Write D → DB
```

do:

```text
Write A
Write B
Write C
Write D
    ↓
  Batch
    ↓
   DB
```

### Why batching helps

Individual writes have overhead:

* Network round trips
* Transaction setup
* Query processing
* Index updates
* WAL / disk operations

A batch amortizes this overhead across multiple writes.

---

# 3. Where Can We Batch?

### Application Layer

```text
Kafka
  ↓
Application
  ↓
Batch
  ↓
Database
```

Good when Kafka/event log is already the durable source of truth.

If the application crashes, events can be replayed.

---

### Intermediate Layer

```text
Events
  ↓
Batcher / Aggregator
  ↓
Database
```

Example:

```text
Like Post A
Like Post A
Like Post A
Like Post B
```

becomes:

```text
Post A → +3
Post B → +1
```

This can dramatically reduce writes.

---

### Database Layer

Databases can sometimes buffer writes before flushing them to disk.

```text
Application
    ↓
Database buffer
    ↓
Disk
```

Useful, but generally a **big-hammer solution**. Understand durability implications before changing flush settings.

---

# 4. Batch Size vs Latency

Usually use:

```text
Flush when:

batch size >= N
        OR
time >= T
```

Example:

```text
1000 events
OR
100 ms
```

### Tradeoff

```text
Larger batch
    ↓
Better throughput
    ↓
Higher latency

Smaller batch
    ↓
Lower latency
    ↓
More overhead
```

---

# 5. Batching ≠ Aggregation

### Batching

Same information, processed together:

```text
A B C D
  ↓
[A B C D]
```

4 logical writes → 1 batch operation.

### Aggregation

Multiple events become one summary:

```text
Like
Like
Like
Like
  ↓
+4 likes
```

Aggregation can reduce the actual amount of information that must be stored/processed.

---

# 6. Batching Isn't Always Useful

If traffic is:

```text
1 like/hour/post
```

and you batch every:

```text
1 minute
```

most batches contain:

```text
0–1 events
```

So batching gives almost no benefit.

Always ask:

> **Are events actually arriving close enough together for batching to help?**

---

# 7. Hierarchical Aggregation

Used when event volume becomes extremely large.

Core idea:

> **Aggregate data in multiple stages instead of sending every raw event to one central system.**

```text
Millions of Events
        ↓
Write Processors
        ↓
Local Aggregation
        ↓
Root Processor
        ↓
Broadcast Nodes
        ↓
Millions of Users
```

---

# 8. Fan-In Problem

Millions of users send events:

```text
User A ─┐
User B ─┤
User C ─┤
User D ─┤
...     ├──→ Root
User N ─┘
```

Root becomes overloaded.

Solution:

```text
Users
 ↓
Write Processor 1
Write Processor 2
Write Processor 3
 ↓
Aggregate
 ↓
Root
```

Partition events by something like:

```text
hash(commentId)
```

So all events for the same comment go to the same processor.

---

# 9. Fan-Out Problem

After processing an event, millions of users may need to receive it.

Naive:

```text
Root
 ↓ ↓ ↓ ↓ ↓ ↓
1M users
```

Solution:

```text
              Root
             /    \
            ↓      ↓
      Broadcast  Broadcast
        Node 1     Node 2
       / |  \      / |  \
      ↓  ↓   ↓    ↓  ↓   ↓
    Users      Users
```

Broadcast nodes own subsets of users/connections.

Consistent hashing can map:

```text
hash(userId) → Broadcast Node
```

---

# 10. Example: Live Comments / Likes

Suppose:

```text
100 likes arrive for Comment X
```

Instead of:

```text
100 events
 ↓
100 DB writes
```

a write processor can maintain:

```text
Comment X → +100 likes
```

and send one aggregated update to the root.

So:

```text
100 events
    ↓
1 aggregated update
```

---

# 11. Why Hierarchical?

Because reduction happens at multiple levels:

```text
1,000,000 events
       ↓
10,000 local aggregates
       ↓
100 higher-level aggregates
       ↓
1 global state
```

Each layer reduces the amount of work the next layer needs to handle.

---

# 12. Hot-Key Problem

If one comment becomes extremely popular:

```text
Comment X
   ↓
1 million likes
```

all events may go to one processor.

That processor becomes a hot partition.

Possible solution:

```text
Comment X
   ↓
 ┌───┬───┬───┬───┐
 P1  P2  P3  P4
 ↓   ↓   ↓   ↓
+2K +3K +1K +4K
 └───┴───┴───┴───┘
         ↓
      Merge
         ↓
      +10K
```

This is another level of hierarchical aggregation.

---

# 13. Key Tradeoff

Hierarchical aggregation gives:

```text
Fewer writes
    +
Lower infrastructure cost
    +
Better scalability
```

But introduces:

```text
Additional processing
       +
Additional latency
       +
More complexity
```

Therefore it works best when:

> **Eventually consistent / slightly delayed data is acceptable.**

Good examples:

* Likes
* View counts
* Analytics
* Metrics
* Live reactions
* Presence
* Some location updates

Poor candidates:

* Payments
* Bank balances
* Inventory reservations
* Critical transactional state

---

# 14. The Most Important Question

When you see a huge write workload, ask:

> **Do I really need every individual event, or do I only need the resulting state?**

If you only need the resulting state:

```text
Raw Events
    ↓
Aggregation
    ↓
Final State
```

Example:

```text
1,000,000 likes
       ↓
Aggregated
       ↓
Post X → 1,000,000 likes
```

---

# 15. Interview Mental Model

When writes are too high:

```text
             Too many writes
                    │
        ┌───────────┴───────────┐
        ↓                       ↓
   Need more capacity?      Can reduce writes?
        │                       │
   Partition / Shard       Batch / Aggregate
                                │
                                ↓
                     Extreme scale?
                                │
                               YES
                                ↓
                    Hierarchical Aggregation
                                │
                    ┌───────────┴───────────┐
                    ↓                       ↓
               Reduce fan-in           Reduce fan-out
                    ↓                       ↓
             Write Processors        Broadcast Nodes
```

---

# 16. One-Line Definitions

**Batching**

> Process multiple writes together to amortize per-write overhead.

**Aggregation**

> Combine multiple events into a smaller representation of their result.

**Hierarchical Aggregation**

> Aggregate data progressively across multiple distributed layers to reduce both processing and communication volume.

**Fan-in**

> Many clients sending data toward a smaller number of processors.

**Fan-out**

> One event/state update being distributed to many clients.

**Broadcast Node**

> A server responsible for distributing updates to a subset of connected users.

**Coalescing**

> Keep only the latest/relevant state instead of processing every intermediate update.

---

# 17. Final Mental Model

Remember:

```text
              TOO MANY WRITES
                    ↓
        ┌──────────────────────┐
        │ Can I process faster?│
        └──────────┬───────────┘
                   ↓
             Partition / Shard
                   │
                   ↓
        Can I process together?
                   │
                   ↓
                Batching
                   │
                   ↓
        Can I reduce the data?
                   │
                   ↓
              Aggregation
                   │
                   ↓
       Is the scale extremely high?
                   │
                   ↓
        Hierarchical Aggregation
                   │
          ┌────────┴────────┐
          ↓                 ↓
      Fan-in ↓           Fan-out ↓
  Write Processors    Broadcast Nodes
```

### The key principle

> **Don't always scale the database. First ask whether you can make the database do less work.**

# 30. When to Use Write Scaling in Interviews

Write scaling should **not** be something you wait for the interviewer to explicitly ask about.

A strong candidate proactively:

```text
Identify potential bottleneck
        ↓
Estimate / validate the workload
        ↓
Decide whether it is significant
        ↓
Propose the appropriate scaling strategy
        ↓
Discuss trade-offs
```

---

## 30.1 Proactively Identify Write Bottlenecks

Suppose you're designing a social-media system with millions of users.

Don't wait for:

> "How would you scale writes?"

Instead, say something like:

> "With millions of users posting content, we could quickly hit a write bottleneck. Let me estimate the expected write throughput first."

Then do your back-of-the-envelope calculation.

If the number is significant:

> "This is substantial enough that I'll come back to write scaling as a deep dive."

This shows that you are **driving the design instead of waiting for prompts**.

---

# 31. Example: Proactively Proposing Sharding

Suppose users are constantly creating posts.

You could say:

```text
Millions of users
       ↓
Large number of post writes
       ↓
Single DB becomes bottleneck
       ↓
Partition by userId
       ↓
Writes distributed across shards
```

Example interview response:

> "For posting writes, I think it's sensible to partition the database by user ID. This should spread the load evenly across shards."

But don't stop there.

A strong candidate immediately considers the edge case:

```text
Normal users
     ↓
Writes distributed well

        BUT

One extremely active user
     ↓
Many writes
     ↓
Potential hot partition
```

Then you can introduce:

```text
Queue + Rate Limiting
```

to protect the system.

---

# 32. Important Interview Principle

### Don't just propose a solution.

Propose:

```text
Problem
   ↓
Evidence / estimation
   ↓
Solution
   ↓
Edge cases
   ↓
Trade-offs
```

For example:

```text
Write bottleneck
      ↓
Partition by userId
      ↓
Potential hot user
      ↓
Queue / rate limit
      ↓
Discuss consistency + latency
```

This is much stronger than simply saying:

> "We'll use sharding."

---

# 33. Common Interview Scenarios

Write scaling appears frequently in high-scale system design problems.

## 33.1 Instagram / Social Media

Useful write-scaling concepts:

### Sharding by user ID

```text
User
 ↓
Hash(userId)
 ↓
Shard
 ↓
Store posts
```

This distributes posting activity across the database.

### Vertical partitioning

Different types of data have different workloads:

```text
Social Media
     │
     ├── User Profiles
     ├── Posts
     └── Analytics
```

These can be separated because their access patterns differ.

### Hierarchical storage

Older posts may not need the same storage characteristics as recent posts.

Conceptually:

```text
Recent Posts
    ↓
Fast / frequently accessed storage

Older Posts
    ↓
Cheaper / slower storage
```

---

# 34. News Feeds

News feeds create an interesting **write vs read scaling problem**.

Consider a celebrity with:

```text
10 million followers
```

If their post needs to be written into every follower's feed:

```text
Celebrity posts
      ↓
10 million followers
      ↓
10 million feed writes
```

This creates a massive write spike.

```text
                  Celebrity Post
                        │
          ┌─────────────┼─────────────┐
          ▼             ▼             ▼
       Feed 1         Feed 2       Feed 3
          │             │             │
          ▼             ▼             ▼
       millions of feed updates
```

At the same time, those millions of users eventually **read** their feeds.

Therefore news feeds require careful consideration of:

```text
Write volume
     ↕
Read volume
```

This is a classic place to discuss different feed-generation strategies and their trade-offs.

---

# 35. Search Applications

Search systems can be extremely write-heavy because user data/content often needs to be **preprocessed and indexed** before it can be searched efficiently.

Conceptually:

```text
Raw Data
   ↓
Processing
   ↓
Indexing
   ↓
Search Index
   ↓
Fast Queries
```

The important write-scaling techniques include:

### Partitioning

Distribute indexing/storage work across multiple nodes.

### Batching

Instead of processing every individual update independently:

```text
Update 1 ─┐
Update 2 ─┤
Update 3 ─┼──→ Batch
Update 4 ─┤
Update 5 ─┘
             ↓
        Bulk processing
```

This reduces per-operation overhead.

---

# 36. Live Comments

Live comments are an interesting example because the number of participants can become enormous.

Imagine:

```text
Millions of viewers
        +
Millions of activities
        ↓
Potential all-to-all communication
```

An all-to-all architecture becomes impractical.

Instead, hierarchical aggregation can be used.

Conceptually:

```text
                 Millions of Users
                        │
             ┌──────────┴──────────┐
             ▼                     ▼
          Region A              Region B
             │                     │
          Aggregate              Aggregate
             │                     │
             └──────────┬──────────┘
                        ▼
                 Shared View
```

The idea is to aggregate information at intermediate levels instead of making every participant communicate with every other participant.

---

# 37. When NOT to Use Write Scaling

This is **equally important** in interviews.

Don't see:

> "This could potentially become a bottleneck"

and immediately introduce:

* Sharding
* Kafka
* Queues
* Multiple databases
* Complex aggregation

First ask:

> **Is scaling actually necessary?**

---

# 38. Back-of-the-Envelope Math Comes First

Suppose your database can comfortably handle:

```text
10,000 writes/sec
```

and your system requires:

```text
500 writes/sec
```

There is no reason to introduce a distributed sharded architecture.

```text
Required: 500 writes/sec
Capacity: 10,000 writes/sec

             ↓

        No bottleneck
             ↓
       Keep it simple
```

But if:

```text
Required: 100,000 writes/sec
Capacity: 10,000 writes/sec

             ↓

       Real bottleneck
             ↓
      Scale the writes
```

### Key principle:

> **Don't solve a problem that doesn't exist.**

---

# 39. Every Write-Scaling Strategy Has a Cost

Scaling isn't free.

You should explicitly discuss the trade-offs.

---

## Queues

### Benefit

Absorb bursts:

```text
Traffic spike
     ↓
   Queue
     ↓
Workers process at sustainable rate
     ↓
Database
```

### Trade-off

Queues can introduce:

```text
Latency
   +
Eventual consistency
```

The user may perform a write but not immediately see the result.

---

# 40. Partitioning

### Benefit

Distributes write load:

```text
One DB
  ↓
Multiple shards
  ↓
Higher aggregate write capacity
```

### Trade-off

The read path can become more complicated.

A query may need to contact multiple shards:

```text
                    Query
                      │
          ┌───────────┼───────────┐
          ▼           ▼           ▼
       Shard A     Shard B      Shard C
          │           │           │
          └───────────┼───────────┘
                      ▼
                    Merge
```

This creates:

* More network calls
* More coordination
* More application complexity
* Potentially higher latency

---

# 41. Batching

### Benefit

Reduces per-operation overhead.

Instead of:

```text
Write → DB
Write → DB
Write → DB
Write → DB
```

we can do:

```text
Write ─┐
Write ─┤
Write ─┼──→ Batch ──→ DB
Write ─┤
Write ─┘
```

### Trade-off

Batching introduces latency because the system may need to wait for enough operations to form a batch.

So:

```text
Batching
   ↓
Higher throughput
   +
Potentially higher latency
```

---

# 42. Interview Trade-off Table

| Strategy                 | Main Benefit                                    | Main Cost                     |
| ------------------------ | ----------------------------------------------- | ----------------------------- |
| Vertical scaling         | Simple, immediate capacity increase             | Hardware ceiling              |
| Database optimization    | Better performance without architecture changes | Limited scalability           |
| Sharding                 | Higher aggregate write capacity                 | Complex reads + operations    |
| Queues                   | Absorb traffic spikes                           | Delay + eventual consistency  |
| Rate limiting            | Protects system from overload                   | Rejects/delays requests       |
| Batching                 | Reduces per-write overhead                      | Adds latency                  |
| Vertical partitioning    | Isolates different workloads                    | More data/storage complexity  |
| Hierarchical aggregation | Avoids massive fan-out/all-to-all work          | More architectural complexity |

---

# 43. The Ideal Interview Conversation

A very strong write-scaling discussion follows this pattern:

```text
                    User Requirement
                           │
                           ▼
                  Estimate write rate
                           │
                           ▼
                Is it actually large?
                    /             \
                  NO               YES
                  │                 │
                  ▼                 ▼
            Keep it simple    Identify bottleneck
                                      │
                         ┌────────────┼────────────┐
                         ▼            ▼            ▼
                       DB          Bursts       Hot keys
                    capacity        /spikes
                         │            │            │
                         ▼            ▼            ▼
                     Optimize      Queue       Partitioning
                         │                         │
                         └────────────┬────────────┘
                                      ▼
                              Discuss trade-offs
                                      │
                                      ▼
                              Validate read path
```

---

# 44. What the Interviewer Is Really Testing

When they ask about scaling writes, they usually aren't only testing whether you know:

> "Sharding exists."

They're testing whether you can reason about:

### 1. Capacity

Can you estimate whether the system actually has a write bottleneck?

### 2. Distribution

Can you distribute writes evenly?

### 3. Hotspots

What happens if one user/key generates disproportionate traffic?

### 4. Bursts

What happens when traffic suddenly spikes?

### 5. Read implications

Does your write-scaling strategy make reads harder?

### 6. Consistency

Can you tolerate asynchronous processing?

### 7. Latency

Are queues or batching acceptable?

### 8. Complexity

Are you introducing distributed-system complexity unnecessarily?

---

# 45. Final Mental Model

Remember this sequence:

```text
             WRITE SCALING
                  │
                  ▼
          "Do I actually need it?"
                  │
                  ▼
           Back-of-envelope math
                  │
                  ▼
       ┌──────────┴──────────┐
       │                     │
       ▼                     ▼
   No bottleneck         Bottleneck
       │                     │
       ▼                     ▼
   Keep simple          Find bottleneck
                             │
              ┌──────────────┼──────────────┐
              ▼              ▼              ▼
           Capacity         Burst         Hotspot
              │              │              │
              ▼              ▼              ▼
         Vertical/DB       Queue         Sharding
         optimization                     /partition
                                             │
                                             ▼
                                      Check read path
                                             │
                                             ▼
                                      Discuss trade-offs
```

## ⭐ Golden Interview Rule

> **Don't introduce write-scaling machinery just because the system is "large." First quantify the workload, prove the bottleneck, choose the simplest strategy that solves it, and explicitly discuss the trade-offs.**

# Hot Key Too Popular for a Single Shard

## 1. Problem

In a sharded system, we normally distribute data using a partition/shard key:

```text
hash(key) → shard
```

For example:

```text
post123Likes → Shard 7
```

This works well when traffic is distributed across many different keys.

But sometimes **one logical key becomes extremely popular**.

Example:

```text
post123Likes → 100,000 writes/sec
```

Even if the overall traffic is perfectly distributed across the cluster, all requests for `post123Likes` still go to the same shard:

```text
                    post123Likes
                         |
                         v
                      Shard 7
                         |
                    100K writes/sec
                         |
                         X
                   Shard overloaded
```

This is called a **hot key**.

### Important distinction

This is different from a general bad-sharding problem.

### Bad partitioning

```text
Shard 1 → 10K
Shard 2 → 10K
Shard 3 → 10K
Shard 4 → 100K  ← bad distribution
```

The partitioning strategy may be the problem.

### Hot key

```text
Shard 1 → 10K
Shard 2 → 10K
Shard 3 → 10K
Shard 4 → 10K
Shard 5 → 100K  ← one key is extremely popular
```

The partitioning may be perfectly reasonable.

The problem is that **one key itself is too hot**.

---

# 2. Example: Viral Tweet

Imagine a tweet that suddenly goes viral.

```text
Tweet ID: post123

Likes:
100,000/sec
```

Suppose we store the counter as:

```text
post123Likes → 100,000 writes/sec
```

Our partitioning function determines:

```text
hash(post123Likes) → Shard 7
```

Therefore:

```text
100K writes/sec
       |
       v
post123Likes
       |
       v
   Shard 7
       |
       X
   Overloaded
```

Even dedicating an entire shard to this key may not be enough.

So we need to **split the logical key itself**.

---

# 3. Core Idea: Split One Key into Multiple Sub-Keys

Instead of storing:

```text
post123Likes
```

we create multiple sub-keys:

```text
post123Likes-0
post123Likes-1
post123Likes-2
...
post123Likes-99
```

Now the writes can be distributed across multiple shards.

```text
                       post123Likes
                            |
            +---------------+---------------+
            |               |               |
            v               v               v
       post123-0       post123-1       post123-2
            |               |               |
            v               v               v
         Shard 1          Shard 4          Shard 8
```

Instead of:

```text
100,000 writes/sec → one shard
```

we can have approximately:

```text
1,000 writes/sec → bucket 0
1,000 writes/sec → bucket 1
1,000 writes/sec → bucket 2
...
1,000 writes/sec → bucket 99
```

assuming reasonably uniform distribution.

This technique is commonly called:

* Key splitting
* Hot-key splitting
* Key sharding
* Key salting

---

# 4. Why This Works

The important idea is:

> We are not just distributing different keys across shards anymore. We are distributing **one logical key across multiple physical keys**.

Normally:

```text
Logical key
     |
     v
Physical key
     |
     v
One shard
```

With hot-key splitting:

```text
Logical key
     |
     +--------+--------+--------+
     |        |        |        |
     v        v        v        v
  key-0    key-1    key-2    ... key-99
     |        |        |        |
     v        v        v        v
 Shard A   Shard B   Shard C   ... Shard N
```

This allows a single logical entity to consume the capacity of multiple shards.

---

# 5. Approach 1: Split All Keys

The simplest approach is to split **every key** by a fixed factor `k`.

For example:

```text
k = 10
```

Every post counter becomes:

```text
post1Likes-0
post1Likes-1
...
post1Likes-9
```

And:

```text
post2Likes-0
post2Likes-1
...
post2Likes-9
```

Even posts that are not hot are split.

---

## 5.1 Write Path

When a like arrives:

```text
Like request
     |
     v
Choose bucket
     |
     v
post123Likes-7
     |
     v
Shard determined by hash
```

The bucket can be chosen randomly:

```text
bucket = random(0, k-1)
```

For:

```text
100K writes/sec
k = 10
```

we approximately get:

```text
Bucket 0 → 10K/sec
Bucket 1 → 10K/sec
Bucket 2 → 10K/sec
...
Bucket 9 → 10K/sec
```

Now no single physical key receives the entire 100K/sec.

---

# 6. Read Path with Fixed Key Splitting

The downside appears when we read the total count.

Originally:

```text
GET post123Likes
```

required one lookup.

Now we have:

```text
GET post123Likes-0
GET post123Likes-1
...
GET post123Likes-9
```

Then:

```text
totalLikes =
    likes-0
  + likes-1
  + likes-2
  + ...
  + likes-9
```

Conceptually:

```text
                    Read post123Likes
                           |
          +----------------+----------------+
          |                |                |
          v                v                v
       key-0            key-1            key-2 ... key-9
          |                |                |
          +----------------+----------------+
                           |
                           v
                         SUM
                           |
                           v
                       Total Likes
```

---

# 7. Trade-Offs of Splitting All Keys

## Advantage

Very simple.

The system always knows:

```text
number of buckets = k
```

There is no need to determine whether a key is currently hot.

---

## Disadvantage 1: Storage Amplification

If every key gets split into `k` keys, the number of physical keys increases.

For example:

```text
1 billion logical keys
k = 10

Potentially:
10 billion physical keys
```

So storage and metadata overhead increase.

---

## Disadvantage 2: Read Amplification

Without splitting:

```text
1 logical read → 1 physical read
```

With `k = 10`:

```text
1 logical read → 10 physical reads
```

Therefore:

```text
Read amplification = k
```

For large `k`, this can become expensive.

---

## Disadvantage 3: More Complexity

The application now has to understand that:

```text
post123Likes
```

is actually:

```text
post123Likes-0
post123Likes-1
...
post123Likes-k
```

instead of one simple key.

---

# 8. Approach 2: Dynamically Split Only Hot Keys

Instead of splitting every key, we can split **only keys that become hot**.

This is generally more efficient.

Suppose we have:

```text
post1 → 10 likes/sec
post2 → 20 likes/sec
post3 → 5 likes/sec
post4 → 100 likes/sec
post5 → 100,000 likes/sec 🔥
```

There is no reason to split:

```text
post1
post2
post3
post4
```

Only:

```text
post5
```

needs special treatment.

---

# 9. Dynamic Splitting Example

Initially:

```text
post5Likes
```

Traffic:

```text
100 writes/sec
```

No problem.

Later:

```text
post5Likes
100K writes/sec
```

We detect that it is hot.

We dynamically split it:

```text
post5Likes
     |
     +---- post5Likes-0
     +---- post5Likes-1
     +---- post5Likes-2
     ...
     +---- post5Likes-99
```

Now:

```text
100K writes/sec
       |
       v
100 buckets
       |
       v
~1K writes/sec/bucket
```

---

# 10. Why Dynamic Splitting Is Better

Most keys are usually not hot.

Therefore:

```text
Normal keys:
    1 physical key

Hot keys:
    N physical keys
```

This avoids unnecessarily multiplying storage and reads for every key.

It is particularly useful when hot keys are rare and unpredictable.

---

# 11. How Do We Detect a Hot Key?

The system can track traffic statistics.

For example:

```text
post123 → 50 writes/sec
post456 → 70 writes/sec
post789 → 150K writes/sec
```

If:

```text
writes/sec > threshold
```

we consider the key hot.

Example:

```text
HOT_KEY_THRESHOLD = 20,000 writes/sec
```

Then:

```text
post123 → normal
post456 → normal
post789 → HOT
```

The system can maintain these statistics using:

* Local counters
* Metrics
* Monitoring systems
* Rate estimators
* Sliding windows

The exact implementation depends on the system.

---

# 12. The Most Important Problem: Reader/Writer Coordination

Dynamic splitting introduces a critical problem.

Suppose initially we have:

```text
post123Likes
```

Writers write:

```text
INCR post123Likes
```

Then the system detects:

```text
post123Likes is HOT
```

and starts writing to:

```text
post123Likes-0
post123Likes-1
...
post123Likes-99
```

But the reader still does:

```text
GET post123Likes
```

Now the reader doesn't know that the data has moved into sub-keys.

This can result in incorrect counts.

---

# 13. Solution 1: Readers Always Check for Sub-Keys

The simplest approach is:

> Readers always understand that a key may have sub-keys.

For example:

```text
read(post123Likes)
```

checks:

```text
post123Likes

post123Likes-0
post123Likes-1
...
post123Likes-99
```

Then it aggregates whatever exists.

Conceptually:

```text
                    READ
                     |
                     v
              post123Likes
                     |
          +----------+----------+
          |                     |
          v                     v
     Base key?             Sub-keys?
                                |
                  +-------------+-------------+
                  |             |             |
                  v             v             v
                key-0         key-1         key-2 ...
                  |             |             |
                  +-------------+-------------+
                                |
                                v
                              SUM
```

---

# 14. Why This Approach Is Attractive

The biggest benefit is **simplicity**.

Readers don't need to receive a special announcement.

They already know:

```text
Any logical key may have sub-keys.
```

So writers can dynamically change the number of buckets.

Readers simply follow the same logic.

This reduces coordination requirements.

---

# 15. Disadvantage: Read Amplification

The reader may need to check many sub-keys.

For example:

```text
post123Likes-0
post123Likes-1
...
post123Likes-99
```

That's potentially:

```text
100 physical reads
```

for one logical read.

However, these reads can often be issued in parallel:

```text
                    READ
                      |
       +--------------+--------------+
       |              |              |
       v              v              v
    key-0           key-1          key-2
       |              |              |
       +--------------+--------------+
                      |
                     SUM
```

So the system pays in:

* Network requests
* Read bandwidth
* Aggregation work

rather than necessarily paying 100× latency.

---

# 16. Solution 2: Announce the Split to Readers

A more sophisticated approach is to maintain metadata describing how a key is currently split.

For example:

```text
Key Metadata

post123Likes:
    bucketCount = 100
```

Readers first determine:

```text
bucketCount = 100
```

Then read:

```text
post123Likes-0
post123Likes-1
...
post123Likes-99
```

---

# 17. Why Split Announcement Is More Complex

Suppose the system changes:

```text
bucketCount = 1
```

to:

```text
bucketCount = 100
```

We need to make sure readers and writers agree on the transition.

Potential problems include:

### Stale readers

One server thinks:

```text
bucketCount = 1
```

while another knows:

```text
bucketCount = 100
```

### Race conditions

A writer may start writing to sub-keys before some readers know about the split.

### Partial propagation

Some application servers may receive the split announcement earlier than others.

### Failure during split

What happens if the system crashes halfway through the transition?

These problems introduce distributed coordination complexity.

---

# 18. Comparison of the Two Reader Strategies

| Strategy              | Advantages                          | Disadvantages                    |
| --------------------- | ----------------------------------- | -------------------------------- |
| Always check sub-keys | Simple, robust, little coordination | Read amplification               |
| Announce split        | More efficient reads                | More coordination and complexity |

For most interviews:

> **Prefer the "always check sub-keys" approach unless the interviewer asks for deeper optimization.**

It demonstrates good engineering judgment:

```text
Don't introduce distributed coordination
unless you actually need it.
```

---

# 19. How Do Writers Choose a Sub-Key?

Once we have:

```text
post123Likes-0
...
post123Likes-99
```

we need to decide where each write goes.

---

## Option 1: Random Bucket

```text
bucket = random(0, 99)
```

Example:

```text
Like 1 → bucket 34
Like 2 → bucket 82
Like 3 → bucket 11
Like 4 → bucket 57
```

This gives approximately even distribution.

For counters, this is often the simplest solution.

---

## Option 2: Hash a Secondary Attribute

We could use:

```text
bucket = hash(userId) % 100
```

Then:

```text
userA → bucket 12
userB → bucket 87
userC → bucket 42
```

This gives deterministic distribution.

However, if one user itself generates a disproportionate amount of traffic, that user could make one bucket hot.

Therefore, for pure counters, random distribution is often easier.

---

# 20. Important Requirement: Data Must Be Aggregatable

Hot-key splitting works particularly well for data such as:

```text
Likes
Views
Clicks
Downloads
Impressions
Counters
```

because:

```text
total = bucket0 + bucket1 + ... + bucketN
```

The operation is naturally aggregatable.

---

# 21. Why It Doesn't Work as Well for Atomic Data

Consider:

```text
user123Profile
```

with:

```text
name
email
address
subscription
```

We cannot blindly distribute updates across:

```text
user123Profile-0
user123Profile-1
...
```

because the logical entity may require atomic updates.

For example:

```text
Update email + subscription
```

might need to happen atomically.

Splitting the entity introduces consistency problems.

Therefore:

> **Hot-key splitting is best suited to data where multiple independent pieces can be combined into a final aggregate.**

---

# 22. Three Different Scaling Problems to Remember

This distinction is extremely useful in system design interviews.

## Problem 1: Poor Shard Distribution

Example:

```text
Shard 1 → 10K
Shard 2 → 10K
Shard 3 → 10K
Shard 4 → 100K
```

The partitioning strategy may be poor.

### Possible solutions

* Better partition key
* Better hashing
* Consistent hashing
* Rebalancing

---

## Problem 2: One Inherently Hot Key

Example:

```text
post123 → 100K writes/sec
```

Even perfect partitioning still sends this key to one shard.

### Solution

**Split the hot key into multiple sub-keys.**

```text
post123-0
post123-1
...
post123-99
```

---

## Problem 3: Hotness Changes Dynamically

Example:

```text
10 writes/sec
      ↓
100
      ↓
10K
      ↓
100K 🔥
```

A key can become hot unexpectedly.

### Solution

**Dynamic hot-key detection + dynamic key splitting.**

---

# 23. Fixed vs Dynamic Splitting

| Feature                | Split All Keys        | Dynamic Hot-Key Splitting |
| ---------------------- | --------------------- | ------------------------- |
| Implementation         | Simple                | More complex              |
| Storage                | Higher                | Lower                     |
| Read amplification     | Always present        | Mostly for hot keys       |
| Hot-key detection      | Not needed            | Required                  |
| Coordination           | Simple                | More complex              |
| Resource efficiency    | Lower                 | Higher                    |
| Interview friendliness | Very high             | High                      |
| Best for               | Predictable workloads | Unpredictable workloads   |

---

# 24. Choosing the Number of Buckets

Suppose:

```text
Current traffic = 100K writes/sec
```

and one shard safely supports:

```text
10K writes/sec
```

Then we need approximately:

```text
100K / 10K = 10 buckets
```

So:

```text
k ≈ required write throughput / per-shard capacity
```

In practice, we should leave headroom.

For example:

```text
100K writes/sec
10K theoretical capacity/shard

Instead of k = 10

Choose perhaps:
k = 20
```

Now each bucket averages:

```text
100K / 20 = 5K writes/sec
```

giving additional safety margin.

---

# 25. Dynamic Resplitting

A sophisticated system may not stop at:

```text
100 buckets
```

Suppose traffic grows:

```text
100K → 500K → 1M writes/sec
```

We may need:

```text
100 buckets
      ↓
500 buckets
      ↓
1000 buckets
```

This is dynamic resharding at the key level.

However, this introduces additional complexity around:

* Existing data
* New writes
* Reader interpretation
* Bucket migration
* Consistency
* Metadata

For an interview, don't jump here unless the interviewer asks.

---

# 26. Important Interview Trade-Off

Hot-key splitting is fundamentally a trade:

```text
                    HOT KEY SPLITTING
                           |
            +--------------+--------------+
            |                             |
            v                             v
      WRITE SCALABILITY              READ COST
            |                             |
            v                             v
     More shards/buckets          More reads + aggregation
```

We're effectively saying:

> "Instead of forcing one shard to handle all writes, we'll distribute the writes across many shards and accept some additional work when reading."

This is often a very good trade for systems where:

```text
Writes are extremely frequent
Reads are less frequent
```

or where the write bottleneck is much more dangerous than the read amplification.

---

# 27. Example: Viral Tweet

Suppose:

```text
100K likes/sec
```

### Without hot-key splitting

```text
post123Likes
      |
      v
   Shard 7
      |
      v
100K writes/sec
      |
      X
```

### With 100 buckets

```text
                  post123Likes
                       |
       +---------------+---------------+
       |               |               |
       v               v               v
   post123-0       post123-1       post123-2 ... post123-99
       |               |               |
       v               v               v
    Shard A          Shard B          Shard C
       |               |               |
      1K/s            1K/s            1K/s
```

Then:

```text
READ

post123-0
post123-1
...
post123-99

        ↓

      SUM

        ↓

    100,000
```

---

# 28. Interview Answer

If the interviewer asks:

> **"What happens when you have a hot key that's too popular for even a single shard?"**

A strong answer:

> "At that point, normal sharding isn't enough because all traffic for that logical key still maps to one shard. For aggregatable data such as likes, views, or counters, I'd split the hot key into multiple sub-keys, for example `post123-0` through `post123-99`, and distribute writes across those buckets. Reads would aggregate the buckets to produce the logical value. We can either split all keys by a fixed factor, which is simpler but increases storage and read amplification, or dynamically split only keys that become hot, which is more resource-efficient but introduces some complexity around detecting hot keys and making sure readers understand the split. For an interview, I'd usually choose the simpler reader strategy where readers always know to check for possible sub-keys, unless the read amplification becomes significant enough to justify split metadata."

---

# 29. Follow-Up Questions Interviewers May Ask

### Q1. How do you detect a hot key?

Track per-key write rate and compare it against a threshold.

```text
if writesPerSecond(key) > threshold:
    mark key as hot
```

---

### Q2. How do you distribute writes?

Use:

```text
random(bucket)
```

or:

```text
hash(secondaryAttribute) % bucketCount
```

---

### Q3. How do readers know about the split?

Two options:

1. Readers always check possible sub-keys.
2. Writers announce the split through metadata.

Prefer **#1 for simplicity**.

---

### Q4. Doesn't this increase read traffic?

Yes.

That's the main trade-off:

```text
More write scalability
        ↕
More read amplification
```

---

### Q5. Does this work for every type of data?

No.

It works best for aggregatable data:

```text
likes
views
clicks
counters
```

It is much harder for strongly atomic entities such as:

```text
user profiles
account state
inventory
```

---

### Q6. Why not just dedicate an entire shard to the hot key?

Because the hot key may exceed the capacity of even that entire shard.

For example:

```text
Shard capacity = 20K writes/sec
Hot key = 100K writes/sec
```

One shard still cannot handle it.

We need:

```text
100K / 20K ≈ 5 shards
```

or more with headroom.

---

### Q7. What happens if the hot key becomes even hotter?

Increase the number of sub-keys:

```text
10 buckets
     ↓
100 buckets
     ↓
1000 buckets
```

But dynamic resplitting introduces additional complexity and should only be discussed if necessary.

---

# 30. The Mental Model

Remember this:

```text
NORMAL SHARDING

Different keys
     |
     +------→ Shard A
     +------→ Shard B
     +------→ Shard C


HOT KEY

One key
     |
     ↓
Too much traffic
     |
     ↓
Can't fit on one shard
     |
     ↓
Split the key
     |
     +------→ key-0 → Shard A
     +------→ key-1 → Shard B
     +------→ key-2 → Shard C
     +------→ ...
     +------→ key-N → Shard N
     |
     ↓
Read
     |
     ↓
Aggregate
     |
     ↓
Logical result
```

---

# 31. One-Line Revision

> **When one logical key becomes too hot for a single shard, split that key into multiple sub-keys, distribute writes across shards, and aggregate the sub-keys during reads.**

---

# 32. Key Takeaways

* **Hot key** = one logical key receiving disproportionately high traffic.
* Normal sharding cannot solve a key that is inherently too hot.
* **Key splitting** distributes one logical key across multiple physical keys.
* Fixed splitting is simple but increases storage and read amplification.
* Dynamic splitting is more efficient but requires hot-key detection.
* Readers and writers must agree about the split.
* The simplest strategy is for readers to **always check for possible sub-keys**.
* Split keys work especially well for **aggregatable data**.
* Random bucket selection is a simple way to distribute writes.
* The fundamental trade-off is:

```text
Write scalability
        vs
Read amplification + storage
```

* In an interview, avoid overengineering unless the interviewer pushes into coordination, metadata, or dynamic resharding.

# 46. Conclusion — Scaling Writes

## The Four Fundamental Strategies

Write scaling fundamentally comes down to **four strategies**:

```text
                         SCALING WRITES
                              │
          ┌───────────────────┼───────────────────┐
          │                   │                   │
          ▼                   ▼                   ▼
   Vertical Scaling       Sharding            Queues
   + DB Choices          + Partitioning       + Load Shedding
          │                   │                   │
          └───────────────────┴───────────────────┘
                              │
                              ▼
                    Batching + Reducers
```

### 1. Vertical Scaling + Database Choices

Make a single component capable of handling more writes.

```text
Bigger machine
     +
Better write-optimized DB
     +
Database optimization
```

Use this before introducing unnecessary distributed complexity.

---

### 2. Sharding + Partitioning

Spread the workload across multiple components.

```text
10,000 writes/sec
        ↓
 ┌──────┼──────┬──────┐
 ▼      ▼      ▼      ▼
2.5k   2.5k   2.5k   2.5k
```

The key is choosing a partition key that distributes the workload well.

---

### 3. Queues + Load Shedding

Useful when the system can tolerate:

* Asynchronous processing
* Delayed processing
* Dropping some requests

```text
Traffic Spike
     ↓
   Queue
     ↓
Controlled processing
     ↓
 Database
```

If some work is optional, load shedding can prevent the system from becoming overloaded.

---

### 4. Batching + Multi-Step Reducers

Useful for high-volume analytics and numeric aggregation.

Instead of:

```text
Write → DB
Write → DB
Write → DB
Write → DB
...
```

combine operations:

```text
Many writes
     ↓
   Batch
     ↓
Bulk operation
     ↓
 Database
```

For aggregation-heavy workloads, hierarchical reducers can further reduce the amount of work each component performs.

---

# 47. The Most Important Principle

Everything comes down to one idea:

> ## **Reduce the throughput handled by each individual component.**

Suppose the system receives:

```text
10,000 writes/sec
```

There are multiple ways to make the load manageable.

### Sharding

```text
10,000 writes/sec
        ↓
   10 shards
        ↓
~1,000 writes/sec/shard
```

### Queues

```text
10,000 writes/sec burst
        ↓
      Queue
        ↓
Controlled processing rate
```

### Batching

```text
10,000 individual writes
        ↓
100 bulk operations
        ↓
Much less per-operation overhead
```

The implementation differs, but the underlying principle is the same:

```text
                High Write Throughput
                        │
                        ▼
              Reduce work per component
                        │
          ┌─────────────┼─────────────┐
          ▼             ▼             ▼
       Sharding       Queues       Batching
          │             │             │
          ▼             ▼             ▼
       Spread it      Smooth it     Combine it
```

---

# 48. Don't Scale Without Evidence

One of the easiest mistakes in system design is:

> **Introducing scaling mechanisms when scaling isn't actually necessary.**

Always start with:

```text
Requirements
     ↓
Back-of-the-envelope calculation
     ↓
Estimate writes/sec
     ↓
Compare with component capacity
     ↓
Is there actually a bottleneck?
```

If:

```text
Required = 500 writes/sec
Capacity = 10,000 writes/sec
```

then:

> Don't introduce sharding just because the system is "large."

Keep the design simple.

---

# 49. Sharding: The Default Deep Dive

When write scaling **is** necessary, sharding/partitioning is often a strong place to start.

Why?

```text
Simple concept
     +
Easy horizontal scaling
     +
Large capacity improvement
```

For example:

```text
1 DB
↓
1,000 writes/sec

10 DB shards
↓
~10,000 writes/sec
```

But remember:

> The numbers are idealized. Real systems have coordination, replication, network, uneven workloads, and other overhead.

---

# 50. Analytics → Batching + Aggregation

For high-volume analytics or numeric workloads:

```text
Millions of events
       ↓
    Aggregate
       ↓
   Batch updates
       ↓
  Hierarchical reducers
       ↓
Final result
```

The source highlights batching and hierarchical aggregation as potentially providing **5–10× improvements** for these workloads.

The key reason is that you're reducing the number of expensive individual operations.

---

# 51. Queues → When Async Is Acceptable

Queues are especially useful when the requirements allow:

```text
Request
  ↓
Queue
  ↓
Process later
```

instead of:

```text
Request
  ↓
Process immediately
  ↓
Response
```

Before using a queue, ask:

> **Can this operation be eventually consistent?**

If yes, queues become a strong candidate.

If the user must immediately observe the write, asynchronous processing may not be appropriate.

---

# 52. Final Interview Cheat Sheet

When you see a **write-heavy system**, think:

```text
              WRITE BOTTLENECK?
                     │
                     ▼
              Do the math first
                     │
                     ▼
          ┌──────────┴──────────┐
          │                     │
         NO                    YES
          │                     │
          ▼                     ▼
     Keep simple          What kind of problem?
                                │
              ┌─────────────────┼─────────────────┐
              │                 │                 │
              ▼                 ▼                 ▼
          Too much          Burst traffic     Too many
          total load                           individual ops
              │                 │                 │
              ▼                 ▼                 ▼
          Sharding            Queue           Batching
              │                 │                 │
              ▼                 ▼                 ▼
        Partition well     Smooth load      Reduce overhead
              │
              ▼
       Check hot shards
              │
              ▼
       Check read path
```

## 🧠 One-Line Memory Trick

> **Scale writes by making each component do less: *spread it, smooth it, or combine it.***

* **Spread it** → Sharding
* **Smooth it** → Queues
* **Combine it** → Batching
* **Make the component stronger** → Vertical scaling / database optimization

That is the core mental model you should carry into the interview.
