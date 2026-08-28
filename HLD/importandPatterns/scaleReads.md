# Scaling Reads

## 1. Problem — Why Do We Need Read Scaling?

* Read traffic usually grows much faster than write traffic.
* Typical read/write ratio:

  * `10:1` is common.
  * Can reach `100:1+` for content-heavy systems.
* Example:

  * Instagram feed → many DB reads for one feed request.
  * User may post once/day → very few writes.
* As read traffic increases:

  * CPU becomes a bottleneck.
  * Memory becomes a bottleneck.
  * Disk I/O becomes a bottleneck.
  * Database becomes the primary bottleneck.
* **Goal of read scaling:**

  * Reduce load on the primary DB.
  * Increase read throughput.
  * Reduce read latency.
  * Maintain acceptable consistency.

---

# 2. Read Scaling — Overall Progression

Use this progression in interviews:

```text
Optimize Database
       ↓
Horizontal Database Scaling
       ↓
Caching
       ↓
CDN / Edge Caching
```

### Step 1 — Optimize the existing DB

* Indexing
* Better queries
* Better data modeling
* Denormalization
* Materialized views
* Better hardware

### Step 2 — Scale DB horizontally

* Read replicas
* Sharding

### Step 3 — Add caching

* Application-level cache
* CDN / Edge cache

---

# 3. Database Optimization

## 3.1 Indexing

* Index = data structure that helps DB find rows quickly.
* Without index:

  * DB may perform a **full table scan**.
  * Complexity roughly `O(n)`.
* With appropriate index:

  * DB can locate relevant records much faster.
  * B-tree lookup is roughly `O(log n)`.

### Example

```sql
SELECT *
FROM users
WHERE email = 'abc@example.com';
```

Add index:

```sql
CREATE INDEX idx_users_email
ON users(email);
```

### When to add indexes

* Frequently queried columns.
* Columns used in:

  * `WHERE`
  * `JOIN`
  * `ORDER BY`
  * Frequently used filters.

### Common index types

* **B-tree**

  * General-purpose index.
  * Good for range queries and sorting.
* **Hash**

  * Good for exact-match lookups.
* Specialized indexes:

  * Full-text search.
  * Geospatial queries.

### Interview point

> Before adding infrastructure, first check whether the database queries are properly indexed.

---

# 4. Vertical Scaling

* Increase resources of the existing DB server.
* Increase:

  * CPU
  * RAM
  * Disk
  * Faster SSDs

### Benefits

* Very simple.
* No major architectural changes.
* Quick way to handle additional load.

### Limitations

* Hardware has a maximum capacity.
* Expensive at larger scale.
* Still leaves you with a single DB bottleneck.

### Interview point

* Mention vertical scaling as the **first/simple option**.
* But don't stop there for a large-scale system.

---

# 5. Denormalization

## Problem with Normalization

Normalized DB:

```text
Users
Orders
OrderItems
Products
```

To fetch an order:

```text
Orders
   ↓
OrderItems
   ↓
Products

Users
   ↓
Orders
```

* Multiple joins.
* More DB work.
* Can become expensive for extremely high read traffic.

## Solution — Denormalization

* Store frequently accessed data together.
* Duplicate some data intentionally.

Example:

```text
Order Summary

order_id
user_name
order_date
product_name
price
```

Now:

```sql
SELECT user_name,
       order_date,
       product_name,
       price
FROM order_summary
WHERE order_id = 12345;
```

### Trade-off

```text
Less joins
    ↓
Faster reads
    ↓
More duplicated data
    ↓
More complex writes
```

### When to use

* Read-heavy systems.
* Data changes relatively infrequently.
* Same information is repeatedly required together.

### Important interview point

> Denormalization trades write complexity and storage for faster reads.

---

# 6. Materialized Views

* Useful when queries involve expensive calculations.
* Instead of calculating every time:

  * Calculate once.
  * Store the result.
  * Read the precomputed result.

### Example

Instead of calculating:

```sql
SELECT product_id,
       AVG(rating)
FROM reviews
GROUP BY product_id;
```

Precompute:

```text
Product
   ↓
Background Job
   ↓
Average Rating
   ↓
Materialized View
```

Then reads become very cheap.

### Good for

* Aggregations.
* Analytics.
* Expensive joins.
* Frequently requested computed data.

### Trade-off

* Data may not be completely up-to-date.
* Refresh/update mechanism is required.

---

# 7. Read Replicas

When one DB cannot handle the read load:

```text
             ┌── Read Replica 1
             │
Primary DB ──┼── Read Replica 2
             │
             └── Read Replica 3
```

### Request flow

```text
                ┌── Replica 1 ── Reads
                │
Application ────┼── Replica 2 ── Reads
                │
                └── Replica 3 ── Reads

Application ─────── Primary ─── Writes
```

### How it works

* Primary handles writes.
* Primary replicates data to replicas.
* Replicas handle read traffic.
* Read traffic gets distributed across replicas.

### Replication models

#### Synchronous

```text
Write
  ↓
Primary
  ↓
Replica
  ↓
ACK
```

* Stronger consistency.
* Higher write latency.

#### Asynchronous

```text
Write
  ↓
Primary ──ACK
  ↓
Replica updated later
```

* Lower write latency.
* Possible replication lag.

---

# 8. Replication Lag

### Problem

```text
T0:
Primary → user creates post

T1:
Primary has post
Replica does NOT have post yet

T2:
Replica receives post
```

If the user immediately reads from the replica:

```text
Write → Primary
Read  → Replica

Result → Old/Stale data
```

### Common solution

For operations requiring read-after-write consistency:

```text
Write → Primary
Immediate Read → Primary
Normal Read → Replica
```

### Interview question

**"What happens if a user writes and immediately reads?"**

Answer:

* Async replicas may return stale data.
* Use primary for critical read-after-write operations.
* Or use session/sticky routing.
* Or use synchronous replication if strong consistency is required.

---

# 9. Database Sharding

* Split data across multiple databases.
* Each DB stores only part of the overall dataset.

```text
             Application
                 |
        ┌────────┼────────┐
        ↓        ↓        ↓
       DB1      DB2      DB3
```

### Why?

Read replicas:

* Duplicate the entire dataset.
* Increase read throughput.

Sharding:

* Splits the dataset.
* Reduces the amount of data each DB handles.
* Can distribute reads across multiple DBs.

---

## 9.1 Functional Sharding

Split by business domain.

```text
DB1 → Users
DB2 → Posts
DB3 → Likes
```

Example:

```text
Get User Profile
      ↓
User DB

Get Post
      ↓
Post DB

Get Likes
      ↓
Likes DB
```

### Benefit

* Each DB handles a specific workload.
* Smaller datasets.
* Independent scaling.

---

## 9.2 Geographic Sharding

Split data based on user geography.

```text
US Users       → US DB
European Users → Europe DB
Asian Users    → Asia DB
```

### Benefits

* Lower network latency.
* Data is closer to users.
* Load distributed geographically.

### Trade-offs

* More operational complexity.
* Cross-shard queries become difficult.
* Data movement/rebalancing becomes harder.

### Interview point

> Sharding can help read scaling, but it is primarily a database/data-distribution strategy and adds significant complexity.

---

# 10. Application-Level Caching

One of the most important read-scaling techniques.

```text
Client
  ↓
Application
  ↓
Check Cache
  │
  ├── HIT ─────→ Return data
  │
  └── MISS
        ↓
     Database
        ↓
     Update Cache
        ↓
     Return data
```

Typical technologies:

* Redis
* Memcached

### Cache Hit

```text
Application → Cache → Data
```

* Very fast.
* DB is not touched.

### Cache Miss

```text
Application → Cache
                  ↓ MISS
                Database
                  ↓
                Cache
```

* Fetch from DB.
* Store result in cache.
* Future requests become cache hits.

---

# 11. Why Caching Works

Most applications have **skewed access patterns**.

Example:

```text
100 million users
       ↓
1 million popular products/posts
       ↓
Same popular data requested repeatedly
```

Instead of:

```text
1000 requests
     ↓
1000 DB queries
```

Cache allows:

```text
1000 requests
     ↓
Cache
     ↓
Maybe 1 DB query
```

### Main benefit

```text
Database load ↓↓↓
Latency ↓↓↓
Throughput ↑↑↑
```

---

# 12. Cache Invalidation

The biggest challenge with caching:

> **How do we make sure the cache doesn't return stale data?**

## 12.1 TTL

Set expiration time.

```text
Cache Entry
   ↓
TTL = 5 minutes
   ↓
Automatically expires
```

### Pros

* Simple.
* Easy to implement.

### Cons

* Data can remain stale until TTL expires.

---

## 12.2 Write-Through / Active Invalidation

When DB data changes:

```text
Write
 ↓
DB updated
 ↓
Cache updated/deleted
```

### Pros

* Better consistency.

### Cons

* Adds complexity/latency to writes.

---

## 12.3 Write-Behind / Async Invalidation

```text
Write
 ↓
DB
 ↓
Event / Queue
 ↓
Cache invalidation
```

* Reduces write latency.
* But stale data may exist temporarily.

---

## 12.4 Tagged Invalidation

Group related cache entries using tags.

Example:

```text
user:123:profile
user:123:posts
user:123:comments
```

Tag:

```text
user:123
```

When user changes:

```text
Invalidate tag user:123
        ↓
Remove related cache entries
```

---

## 12.5 Versioned Keys

Instead of:

```text
user:123
```

Use:

```text
user:123:v1
```

After update:

```text
user:123:v2
```

Old version naturally becomes unused.

---

# 13. Choosing Cache TTL

Don't choose TTL randomly.

Base it on the **staleness requirement**.

Example:

```text
Requirement:
Search results can be 30 sec stale

→ TTL ≈ 30 sec
```

```text
Requirement:
User profile can be 5 min stale

→ TTL ≈ 5 min
```

### Interview point

> TTL should be driven by the business requirement for acceptable staleness.

---

# 14. CDN / Edge Caching

CDN moves cached content closer to users.

```text
                  ┌── Edge → User
                  │
User → CDN ───────┼── Edge → User
                  │
                  └── Edge → User
                         │
                      Origin
```

### Without CDN

```text
User in India
      ↓
US Server
      ↓
200ms
```

### With CDN

```text
User in India
      ↓
India CDN Edge
      ↓
<10ms
```

### Benefits

* Lower latency.
* Reduces origin traffic.
* Handles huge read volumes.
* Great for globally distributed users.

---

# 15. What Should We Cache in CDN?

Good candidates:

* Public posts.
* Product catalog.
* Images.
* Videos.
* Thumbnails.
* Public profiles.
* Search results where appropriate.

Avoid CDN caching for:

* Private messages.
* Personal preferences.
* Account settings.
* Highly user-specific/private data.

### Key idea

> CDN works best when the same content is requested by many users.

---

# 16. Read Scaling Decision Tree

Use this mental model in interviews:

```text
High Read Traffic
       ↓
Is DB query optimized?
       │
       ├── NO → Index / Query Optimization
       │
       └── YES
            ↓
Can we improve the DB itself?
            │
            ├── YES → Vertical Scaling
            │
            └── Still insufficient
                    ↓
              Read Replicas
                    ↓
              Still insufficient?
                    ↓
              Application Cache
                    ↓
              Global users?
                    ↓
              CDN / Edge Cache
                    ↓
              Dataset too large?
                    ↓
                 Sharding
```

---

# 17. Interview Approach

When interviewer says:

> "How will you scale reads?"

Don't immediately say **Redis**.

Follow this progression:

### Step 1 — Identify the bottleneck

* Which API is read-heavy?
* How many QPS?
* What is the read/write ratio?
* What data is being queried?

### Step 2 — Optimize DB

* Indexes.
* Query optimization.
* Better schema.
* Denormalization.
* Materialized views.

### Step 3 — Scale DB

* Vertical scaling.
* Read replicas.

### Step 4 — Add caching

* Redis/Memcached.
* Define TTL.
* Decide invalidation strategy.
* Handle cache misses.

### Step 5 — Global scale

* CDN.
* Edge caching.
* Geographic distribution.

### Step 6 — Very large datasets

* Sharding.
* Functional sharding.
* Geographic sharding.

---

# 18. Common Interview Scenarios

## URL Shortener

```text
Short URL → Long URL
```

* Extremely read-heavy.
* Cache mapping aggressively.
* Redis is a good fit.
* CDN can help with global traffic.
* DB mainly handles cache misses.

---

## Ticket Booking

* Event pages → cache aggressively.
* Venue information → cache.
* Seating charts → cache.
* **Actual seat availability → don't blindly cache.**
* Critical purchase path should use strongly consistent data.

---

## News Feed

* Read-heavy.
* Cache recent posts.
* Precompute feeds where appropriate.
* Use pagination.
* Don't load the entire feed at once.

---

## Video Platform

Cache:

* Video metadata.
* Titles.
* Descriptions.
* Thumbnails.
* Channel information.

Use CDN heavily for:

* Video files.
* Images.
* Thumbnails.

View counts can often tolerate eventual consistency.

---

# 19. When NOT to Over-Engineer Read Scaling

### Small-scale application

If:

```text
1000 users
```

Don't immediately introduce:

```text
Redis + CDN + Sharding + 10 Read Replicas
```

A properly indexed DB may be enough.

### Write-heavy systems

Example:

```text
Driver location updates
```

Focus on write scaling first.

### Strong consistency requirements

Examples:

* Financial transactions.
* Inventory.
* Seat booking.

Caching must be carefully designed because stale data can cause incorrect decisions.

### Real-time systems

Examples:

* Google Docs-like collaboration.
* Live collaborative editing.

If data changes every second/millisecond, aggressive caching may provide little benefit.

---

# 20. Key Interview Takeaways

* **First optimize the DB before adding infrastructure.**
* **Indexes are usually the first read optimization.**
* **Vertical scaling is simple but has limits.**
* **Read replicas increase read throughput.**
* **Async replication introduces replication lag.**
* **Denormalization trades write complexity for faster reads.**
* **Materialized views precompute expensive queries.**
* **Caching removes repeated reads from the DB.**
* **Cache invalidation is the major caching challenge.**
* **TTL should be based on acceptable data staleness.**
* **CDN moves frequently accessed content closer to users.**
* **Sharding splits the dataset across multiple DBs.**
* **Don't use every scaling technique blindly.**
* **The goal is not just lower latency — it's primarily reducing database load and increasing read capacity.**

# Deep Dive 1 — Queries Become Slow as Dataset Grows

## Problem

* Application starts with:

  * `10K users`
  * Queries are fast.
* Application grows to:

  * `10M users`
  * Simple queries start taking seconds or even tens of seconds.
* Database CPU reaches `100%`.
* Queries may look simple:

  * Find user by email.
  * Fetch user's orders.
  * Find records by status.

### Why does this happen?

* Without proper indexes, DB performs a **Full Table Scan**.
* DB has to examine every row to find matching records.

---

## Example — Finding User by Email

### Without Index

```sql
SELECT *
FROM users
WHERE email = 'user@example.com';
```

Database may perform:

```text
10 Million Users
       ↓
Scan Row 1
       ↓
Scan Row 2
       ↓
Scan Row 3
       ↓
...
       ↓
Scan Row 10 Million
       ↓
Find User
```

### Problem

* If each row is approximately `200 bytes`:

  * `10M × 200 bytes ≈ 2 GB`
  * DB may need to scan a huge amount of data for a single lookup.
* With hundreds of concurrent requests:

  * Disk I/O increases.
  * CPU increases.
  * Query latency increases.
  * Database becomes overloaded.

---

# Solution — Add Indexes

Create an index on frequently queried columns:

```sql
CREATE INDEX idx_users_email
ON users(email);
```

Now the DB can use the index instead of scanning every row.

### Before

```sql
EXPLAIN
SELECT *
FROM users
WHERE email = 'user@example.com';
```

```text
Seq Scan on users
```

### After

```sql
EXPLAIN
SELECT *
FROM users
WHERE email = 'user@example.com';
```

```text
Index Scan using idx_users_email
```

### Conceptually

```text
Without Index:

Query
 ↓
Full Table Scan
 ↓
10M rows
 ↓
Find matching row


With Index:

Query
 ↓
Index
 ↓
Locate matching row
 ↓
Fetch row
```

---

# Joins Make the Problem Worse

Consider:

```text
Users
  ↓
Orders
```

Query:

```sql
SELECT *
FROM users u
JOIN orders o
  ON u.id = o.user_id
WHERE u.email = 'user@example.com';
```

Without proper indexes:

* DB may scan the `users` table.
* Then scan the `orders` table.
* Matching rows need to be compared.
* Large tables can lead to extremely expensive joins.

### Important

For joins, consider indexes on the columns used for joining:

```sql
users.email
orders.user_id
```

---

# Compound / Composite Indexes

Suppose queries commonly use:

```sql
SELECT *
FROM users
WHERE status = 'ACTIVE'
AND created_at > '2026-01-01';
```

A composite index can be:

```sql
CREATE INDEX idx_users_status_created
ON users(status, created_at);
```

### Why Column Order Matters

Index:

```text
(status, created_at)
```

Can efficiently support:

```sql
WHERE status = ...
```

and:

```sql
WHERE status = ...
AND created_at = ...
```

But it generally does **not** provide the same benefit for:

```sql
WHERE created_at = ...
```

when `status` is not part of the lookup.

### Interview Rule

> The order of columns in a composite index matters.

Think of it like:

```text
(status → created_at)
```

The DB can efficiently navigate through `status` first, then `created_at`.

---

# EXPLAIN — Important Interview Tool

When a query becomes slow:

```sql
EXPLAIN
SELECT ...
```

Use `EXPLAIN` to understand how the database executes the query.

Look for things such as:

```text
Seq Scan
Index Scan
Rows examined
Estimated cost
Join strategy
```

### Example

```text
Before:

Seq Scan
cost = 0.00..412000
```

After adding index:

```text
Index Scan
cost = 0.43..8.45
```

The exact numbers depend on the database and data distribution, but the important idea is:

```text
Full Table Scan
       ↓
Very expensive as data grows

Index Scan
       ↓
Much smaller search space
```

---

# Interview Answer

If the interviewer asks:

> **"What happens when queries become slower as the dataset grows?"**

Answer in this order:

1. **Check the query execution plan using `EXPLAIN`.**
2. Look for **full/sequential table scans**.
3. Identify frequently queried/filter/join columns.
4. Add appropriate indexes.
5. For compound queries, use **composite indexes**.
6. Verify that the **column order** in the composite index matches the query patterns.
7. Check joins and ensure join columns are properly indexed.
8. Re-run `EXPLAIN` and verify that the database is using the expected index.

---

# Key Interview Takeaways

* Large datasets make inefficient queries increasingly expensive.
* **Full table scans are a common cause of read degradation.**
* Index frequently queried columns.
* Index columns used in joins.
* Use `EXPLAIN` to identify query execution problems.
* Composite index column order matters.
* Don't blindly add indexes:

  * Indexes consume storage.
  * Index maintenance adds write overhead.
* In an interview, proactively mention important indexes when designing the schema.

### One-Liner to Remember

> **When reads become slow as data grows, first inspect the query plan and eliminate unnecessary full-table scans with appropriate indexes.**

# Scaling Millions of Concurrent Reads — Hot Keys & Read Scaling

## 1. The Core Problem

Imagine a celebrity posts something on our platform.

Normally:

```text
Users
  |
  v
Application
  |
  v
Redis
  |
  v
Database
```

Redis handles:

```text
50,000 requests/sec
```

But suddenly millions of users request the **same data**:

```text
GET post:999
```

Traffic becomes:

```text
500,000 requests/sec
        |
        v
    post:999
```

This creates a **Hot Key / Hotspot Problem**.

---

# 2. Why Is One Hot Key a Problem?

Distributed caches such as Redis usually distribute traffic based on the key.

For example:

```text
hash(key) → cache node
```

Suppose:

```text
hash("post:999") → Redis Node 3
```

Then:

```text
                    post:999
                       |
                       v
                     Node 3
                   🔥 500K req/s

Node 1 → 0
Node 2 → 0
Node 4 → 0
Node 5 → 0
```

Adding more Redis nodes does **not automatically solve** this problem because all requests for the same key may still go to the same node.

This is fundamentally different from normal distributed traffic.

---

# 3. First Question: Where Is the Bottleneck?

Before choosing a solution, ask:

```text
Where is the bottleneck?
```

Possible bottlenecks:

```text
Client
  ↓
CDN
  ↓
Application
  ↓
Local Cache
  ↓
Redis
  ↓
Database
```

Different bottlenecks require different solutions.

---

# 4. Solution #1 — CDN

## Idea

If the data is public and cacheable, don't let millions of requests reach your infrastructure.

Use a CDN:

```text
             1M Users
                 |
                 v
                CDN
           /      |      \
       Edge 1   Edge 2   Edge 3
                 |
          Cache Misses Only
                 |
                 v
              App
```

For example:

```text
GET /article/999
```

If the article is cached at the CDN:

```text
User → CDN → Response
```

Your application doesn't see the request.

## Best for

* Public articles
* Images
* Videos
* Public profiles
* Product pages
* Static JSON
* Public API responses
* Mostly immutable data

## Key idea

> **Push caching as close to the user as possible.**

For massive public traffic, CDN is often better than trying to make Redis handle everything.

---

# 5. Solution #2 — Local / In-Process Cache

Instead of:

```text
User
 ↓
App
 ↓
Redis
```

keep extremely hot data in application memory:

```text
User
 ↓
App
 ↓
RAM
```

Suppose:

```text
100 application servers
500,000 requests/sec
```

Traffic might become:

```text
App1 → ~5K req/s → Local RAM
App2 → ~5K req/s → Local RAM
App3 → ~5K req/s → Local RAM
...
```

Redis receives far fewer requests.

## Example

Celebrity profile:

```text
celebrity:taylor-swift
```

If the data changes very rarely, every application server can keep a copy.

## Advantages

* Extremely fast
* No Redis network hop
* Reduces Redis load
* Excellent for extremely hot data

## Disadvantages

* Uses application memory
* Multiple copies
* Cache invalidation becomes harder
* Data can become temporarily inconsistent

Best for:

* Static metadata
* Celebrity profiles
* Product metadata
* Configuration
* Feature flags
* Rarely changing data

---

# 6. Solution #3 — Request Coalescing

## Problem

Suppose:

```text
user:999
```

is not currently in cache.

Suddenly:

```text
10,000 users
      |
      v
GET user:999
```

Without coalescing:

```text
10,000 requests
       |
       v
10,000 backend fetches
       |
       v
Database 💥
```

This is called the:

> **Thundering Herd Problem**

---

# 7. How Request Coalescing Works

The application server keeps track of currently running requests:

```text
inflight = {
    "user:999": Future
}
```

First request:

```text
User 1
  ↓
App
  ↓
Is user:999 already being fetched?
  ↓
NO
  ↓
Create Future
  ↓
Fetch Redis / DB
```

Second request:

```text
User 2
  ↓
App
  ↓
Is user:999 already being fetched?
  ↓
YES
  ↓
WAIT for existing Future
```

Third request:

```text
User 3
  ↓
App
  ↓
YES
  ↓
WAIT
```

So:

```text
              User 1
              User 2
              User 3
              User 4
                |
                v
          Application
                |
          One in-flight
             request
                |
                v
             Redis/DB
```

---

# 8. How Do Waiting Requests Get the Response?

The requests are **not rejected or cancelled**.

They wait asynchronously for the same Future/Promise.

Backend returns:

```json
{
  "id": 999,
  "name": "Suraj"
}
```

The Future is resolved:

```text
Future
  |
  +----> User 1
  +----> User 2
  +----> User 3
  +----> User 4
  ...
```

Everyone receives the same result.

Therefore:

```text
10,000 requests
      ↓
1 backend request
      ↓
result
      ↓
10,000 responses
```

---

# 9. What Does Coalescing Protect?

Primarily:

* Database
* Expensive backend services
* Expensive computation
* Cache refresh operations

Especially useful during cache misses.

Important:

> Coalescing does NOT necessarily solve a hot Redis node.

If every application server is still doing:

```text
App → Redis
```

for the same key, Redis can still become overloaded.

That's where fanout/replication/local caching help.

---

# 10. Request Coalescing Across Multiple App Servers

Suppose:

```text
           Load Balancer
                |
      ----------------------
      |         |          |
     App1      App2       App3
```

Each application server has its own `inflight` map.

If 100,000 users request the same key:

```text
App1 → 1 backend request
App2 → 1 backend request
App3 → 1 backend request
```

So the backend might receive roughly:

```text
N requests
```

where N is the number of application servers handling that key.

---

# 11. Solution #4 — Key Fanout

Now consider a different problem.

Redis itself is overloaded because one key is extremely hot:

```text
post:999
```

Suppose:

```text
Redis node capacity = 50K req/s
Traffic = 500K req/s
```

We can create multiple copies:

```text
post:999:0
post:999:1
post:999:2
post:999:3
post:999:4
post:999:5
post:999:6
post:999:7
post:999:8
post:999:9
```

Each contains the same data.

Traffic can now be distributed:

```text
500K requests
      |
      v
10 cache keys
      |
      +---- 50K → key :0
      +---- 50K → key :1
      +---- 50K → key :2
      ...
      +---- 50K → key :9
```

---

# 12. How Does the Application Know Which Fanout Key to Use?

This is extremely important.

The user still asks:

```text
GET /user/999
```

The application converts the logical key:

```text
user:999
```

into a physical cache key:

```text
user:999:N
```

For example:

```text
shard = random(0, 9)

cacheKey = "user:999:" + shard
```

Possible requests:

```text
Request 1 → user:999:3
Request 2 → user:999:7
Request 3 → user:999:1
Request 4 → user:999:9
```

The application is responsible for this mapping.

It does **not happen automatically in Redis**.

---

# 13. Logical Key vs Physical Cache Key

This distinction is useful.

### Logical key

```text
user:999
```

This is what your application thinks about.

### Physical keys

```text
user:999:0
user:999:1
user:999:2
...
user:999:9
```

These are the actual Redis keys.

Your cache abstraction handles the mapping:

```text
getUser(999)
     |
     v
Is this a hot key?
     |
    YES
     |
choose shard
     |
     v
user:999:7
     |
     v
Redis
```

---

# 14. How Does Fanout Distribute Across Redis Nodes?

Suppose Redis uses hashing:

```text
hash(cacheKey) → Redis node
```

Then:

```text
user:999:0 → Node 1
user:999:1 → Node 4
user:999:2 → Node 2
user:999:3 → Node 5
user:999:4 → Node 3
```

Instead of:

```text
user:999 → Node 3 → 500K req/s 🔥
```

we get:

```text
Node 1 → 100K
Node 2 → 100K
Node 3 → 100K
Node 4 → 100K
Node 5 → 100K
```

The exact distribution depends on hashing and traffic.

---

# 15. Fanout Trade-off

Fanout means storing multiple copies.

If:

```text
data size = 100 KB
copies = 10
```

Memory usage:

```text
100 KB × 10 = 1 MB
```

So we're trading:

```text
Memory
   +
More complicated invalidation
```

for:

```text
Much higher read capacity
```

This is often worth it for extremely hot data.

---

# 16. Fanout Is Best for Read-Heavy Data

Ideal:

```text
Reads  = 500K/sec
Writes = 1/sec
```

Examples:

* Viral article
* Celebrity profile
* Popular product metadata
* Movie metadata
* Static configuration

Less ideal:

```text
Reads  = 500K/sec
Writes = 100K/sec
```

Because every write might require updating/invalidation of many copies.

---

# 17. Solution #5 — Cache Replication

Instead of creating different keys:

```text
user:999:0
user:999:1
user:999:2
```

we can replicate:

```text
             user:999
            /    |    \
           R1    R2    R3
```

Requests are distributed across replicas.

Conceptually:

```text
Request 1 → R1
Request 2 → R2
Request 3 → R3
Request 4 → R1
```

The goal is similar to fanout:

> **Distribute reads for a hot piece of data across multiple cache servers.**

---

# 18. Fanout vs Replication

### Fanout

Different physical keys:

```text
user:999:0
user:999:1
user:999:2
```

Application chooses a key.

### Replication

Same logical key exists on multiple replicas:

```text
user:999
   |
   +--- R1
   +--- R2
   +--- R3
```

The cache/client layer chooses a replica.

Both solve the hot-node problem.

---

# 19. Solution #6 — Cache Warming

If we know a spike is coming, populate the cache beforehand.

Example:

```text
India vs Pakistan
World Cup Final
```

We know:

```text
match:IND-PAK
```

will become extremely hot.

Before the match:

```text
Database
   ↓
Cache
   ↓
match:IND-PAK
```

When millions of users arrive:

```text
Millions of requests
       ↓
Cache HIT
```

instead of:

```text
Millions of requests
       ↓
Cache MISS
       ↓
Database 🔥
```

---

# 20. Solution #7 — Stale-While-Revalidate

Instead of blocking users when cache expires:

```text
Request
  ↓
Cache expired
  ↓
Wait for DB
```

allow slightly stale data:

```text
Request
  ↓
Stale value available
  ↓
Return immediately
```

At the same time:

```text
Background refresh
        ↓
      DB
        ↓
     Cache
```

So:

```text
             Request
                |
                v
              Cache
             /     \
            /       \
     stale data    refresh
        |             |
        v             v
      User           DB
```

Great when a few seconds of stale data is acceptable.

---

# 21. Solution #8 — TTL Jitter

Suppose 1 million cache entries all have:

```text
TTL = 60 seconds
```

They might expire simultaneously:

```text
10:00:00 → inserted

10:01:00 → 1M keys expire
                  ↓
             1M cache misses
                  ↓
              DB spike
```

Instead use:

```text
TTL = 60 + random(0, 30)
```

Now expiration is spread:

```text
60 sec
63 sec
68 sec
74 sec
82 sec
...
```

This prevents synchronized cache expiration.

---

# 22. Solution #9 — Read Replicas

If the bottleneck is the database rather than Redis:

```text
             Application
                  |
        -----------------------
        |        |            |
       DB1      DB2          DB3
      Primary  Replica      Replica
```

Reads can be distributed across replicas.

But remember:

> Read replicas don't solve a hot Redis key if requests never reach the database.

---

# 23. Solution #10 — Materialized / Precomputed Data

Suppose every request requires an expensive query:

```text
User
 ↓
Complex JOINs
 ↓
Aggregation
 ↓
Sorting
 ↓
DB
```

Instead, precompute:

```text
Raw data
   ↓
Background processing
   ↓
Precomputed result
   ↓
Cache
```

Then:

```text
User
 ↓
Cache
 ↓
Ready-to-serve result
```

Useful for:

* Leaderboards
* Analytics
* Recommendation results
* Aggregated feeds
* Dashboards

---

# 24. Solution #11 — Rate Limiting / Load Shedding

Sometimes traffic is simply beyond your capacity.

Suppose:

```text
Incoming = 5M req/sec
Capacity = 1M req/sec
```

Allowing all traffic could bring down the entire system.

Instead:

```text
5M requests
     ↓
Rate limiter
     ↓
1M accepted
4M throttled/rejected
```

This protects availability.

---

# 25. Solution #12 — Graceful Degradation

Instead of completely failing:

```text
Full response:

Post
Comments
Recommendations
Likes
Related Posts
Ads
Analytics
```

under extreme load you might return:

```text
Post
Likes
```

and temporarily disable expensive secondary features.

Goal:

> Keep the core functionality alive even when the system is overloaded.

---

# 26. Solution #13 — Asynchronous Counters / Updates

Suppose a viral post gets:

```text
10M likes
```

Don't necessarily perform:

```text
every like
    ↓
DB UPDATE
    ↓
Cache UPDATE
```

Instead:

```text
Like
 ↓
Kafka / Queue
 ↓
Aggregator
 ↓
Periodic DB update
```

This is useful for:

* Likes
* Views
* Counters
* Analytics
* Engagement metrics

The read path can remain fast.

---

# 27. Putting Everything Together

A mature architecture could look like:

```text
                         Millions of Users
                                |
                                v
                              CDN
                                |
                         Cacheable content?
                           /          \
                         YES           NO
                          |             |
                       CDN hit      Application
                                       |
                                       v
                                Local Cache
                                       |
                                 Cache hit?
                                /          \
                              YES           NO
                               |             |
                             Return        Redis
                                             |
                                      Hot Key?
                                      /      \
                                    NO        YES
                                    |          |
                                Normal     Fanout /
                                Cache      Replicas
                                               |
                                               v
                                             Redis
                                               |
                                         Cache Miss?
                                               |
                                              YES
                                               |
                                        Request Coalescing
                                               |
                                               v
                                             DB
                                               |
                                        Read Replicas /
                                        Precomputed Data
```

Additional protection:

```text
Cache warming
TTL jitter
Stale-while-revalidate
Rate limiting
Graceful degradation
```

---

# 28. The Most Important Mental Model

Don't memorize all the techniques.

Memorize **what problem each one solves**.

| Technique              | Problem                                     |
| ---------------------- | ------------------------------------------- |
| CDN                    | Massive public traffic                      |
| Local Cache            | Too many Redis/network calls                |
| Redis                  | General read scaling                        |
| Fanout                 | One hot key overloads one cache node        |
| Replication            | Distribute reads across cache replicas      |
| Coalescing             | Prevent duplicate backend fetches           |
| Cache Warming          | Predictable traffic spike                   |
| Stale-While-Revalidate | Avoid blocking during refresh               |
| TTL Jitter             | Prevent synchronized expiry                 |
| Read Replicas          | Database read bottleneck                    |
| Materialized View      | Expensive repeated computation              |
| Rate Limiting          | Traffic exceeds capacity                    |
| Graceful Degradation   | Keep core system alive under overload       |
| Async Updates          | Reduce write pressure from massive counters |

---

# 29. Fanout vs Coalescing — The Critical Difference

This is probably the most important distinction.

### Request Coalescing

Question:

> "How do I stop 1 million identical requests from becoming 1 million backend fetches?"

```text
1M requests
     |
     v
Coalescing
     |
     v
1 backend fetch
     |
     v
1M responses
```

### Key Fanout

Question:

> "How do I stop 1 million requests for the same key from overloading one cache server?"

```text
1M requests
     |
     v
Multiple cache keys
     |
     +----> Redis 1
     +----> Redis 2
     +----> Redis 3
     +----> Redis 4
```

### Remember:

> **Coalescing combines duplicate work.**

> **Fanout distributes read traffic.**

---

# 30. Interview Decision Tree

When asked:

> "How do you handle millions of concurrent reads for the same data?"

Think in this order:

```text
                    Millions of Reads
                           |
                           v
              Is content public/cacheable?
                    /              \
                  YES               NO
                   |                 |
                  CDN           Application Cache
                                      |
                                      v
                               Extremely hot?
                                  /       \
                                NO         YES
                                |           |
                           Normal Cache   Local Cache
                                             |
                                      Still overloaded?
                                         /        \
                                       NO          YES
                                       |            |
                                      Done      Fanout /
                                                Replication
```

For cache misses:

```text
Cache Miss
    |
    v
Many requests for same key?
    |
   YES
    |
    v
Request Coalescing
```

For predictable spikes:

```text
Predictable spike
       |
       +----> Cache Warming
       |
       +----> CDN pre-cache
```

For expiration:

```text
Synchronized expiry?
       |
      YES
       |
       +----> TTL Jitter
       +----> Stale-While-Revalidate
```

For overload:

```text
Still overloaded?
       |
       +----> Rate Limiting
       |
       +----> Load Shedding
       |
       +----> Graceful Degradation
```

---

# 31. Interview Answer

If the interviewer asks:

> **"How would you handle millions of concurrent reads for the same cached data?"**

A strong answer:

> "First I'd determine whether the data is publicly cacheable. If it is, I'd use a CDN so that the majority of reads never reach our infrastructure.
>
> If the data needs to be served from the application, I'd use caching and potentially an in-process cache for extremely hot data. If one key becomes a hot key and overloads a single cache node, I'd distribute the data using cache replication or key fanout.
>
> For cache misses or cache refreshes, I'd use request coalescing so that thousands of concurrent requests don't trigger thousands of backend fetches.
>
> If the spike is predictable, I'd warm the cache beforehand. For expiring data, TTL jitter and stale-while-revalidate can prevent synchronized cache misses.
>
> Finally, if traffic exceeds the system's absolute capacity, I'd use rate limiting, load shedding, and graceful degradation to protect availability.
>
> The trade-offs I'd consider are memory usage, consistency, invalidation complexity, freshness requirements, and operational complexity."

---

# 32. One-Line Revision Cheat Sheet

```text
CDN
→ Keep public traffic away from our servers.

Local Cache
→ Keep extremely hot data in application RAM.

Redis
→ General-purpose distributed cache.

Fanout
→ Spread one hot key across multiple cache keys/nodes.

Replication
→ Keep hot data on multiple cache replicas.

Coalescing
→ Prevent duplicate backend fetches.

Cache Warming
→ Load predictable hot data before the traffic arrives.

Stale-While-Revalidate
→ Return slightly stale data while refreshing in background.

TTL Jitter
→ Prevent many keys from expiring simultaneously.

Read Replicas
→ Scale database reads.

Materialized Views
→ Precompute expensive results.

Async Processing
→ Move massive write/aggregation workloads off the synchronous path.

Rate Limiting
→ Prevent traffic from exceeding capacity.

Graceful Degradation
→ Keep core functionality alive during overload.
```

---

# 33. Final Mental Model

When you see:

> **"Millions of users want the same data."**

Don't immediately think:

```text
"Use Redis."
```

Think:

```text
                SAME DATA
                    |
        -------------------------
        |           |           |
      CDN        Local       Redis
                  Cache         |
                                |
                         Hot Key?
                           /    \
                         NO      YES
                         |        |
                       Normal   Fanout /
                                Replica
                                   |
                              Cache Miss?
                                   |
                                  YES
                                   |
                              Coalescing
                                   |
                                   v
                                  DB
```

Then ask:

```text
Can I cache it closer to the user?
Can I keep it in local memory?
Is one cache node becoming hot?
Can I distribute the key?
Can I combine duplicate backend work?
Can I prewarm it?
Can I tolerate stale data?
Can I spread expiration?
Can I precompute the result?
What happens if traffic exceeds capacity?
```

That is the **read-scaling mindset** you want to develop for HLD interviews.

# Cache Stampede / Thundering Herd

## 1. What is the Problem?

A **cache stampede** happens when a very popular cache entry expires and a large number of requests discover the cache miss at almost the same time.

### Example

Suppose:

```text
Homepage cache
TTL = 1 hour
Traffic = 100,000 requests/sec
```

For 59 minutes:

```text
100K req/sec
     ↓
  Cache HIT
     ↓
   Users
```

At exactly 60 minutes:

```text
Cache expires
     ↓
100K requests
     ↓
100K CACHE MISS
     ↓
100K DB queries 💥
```

But the database may only be designed to handle:

```text
1,000 queries/sec
```

Suddenly it receives:

```text
100,000 queries/sec
```

The database slows down or crashes.

This is called:

> **Cache Stampede** or **Thundering Herd**

---

# 2. Why Is It So Dangerous?

The problem becomes even worse when rebuilding the cache is expensive.

For example:

```text
Cache MISS
    ↓
JOIN 10 database tables
    ↓
Call external APIs
    ↓
Aggregate data
    ↓
Generate homepage
    ↓
Store result in cache
```

If 100,000 requests do this simultaneously:

```text
100,000 × expensive computation
```

The system can collapse.

So the goal is:

> **Only do the expensive rebuild once, or spread the rebuild work over time.**

---

# 3. Solution #1 — Distributed Lock

The simplest solution is to allow only **one request** to rebuild the cache.

### Flow

```text
                 100K requests
                      |
                      v
                  Cache MISS
                      |
          --------------------------
          |                        |
      Request 1              Requests 2..100K
          |                        |
     Acquire lock                 WAIT
          |
          v
      Rebuild cache
          |
          v
      Release lock
          |
          v
      Everyone gets
      cached result
```

The first request acquires a distributed lock.

For example, Redis can be used to implement:

```text
SET lock:homepage 1 NX EX 30
```

Where:

* `NX` → create only if the lock doesn't already exist
* `EX 30` → automatically expire the lock after 30 seconds

### What happens?

```text
Request 1 → gets lock → rebuilds cache

Request 2 → lock exists → waits
Request 3 → lock exists → waits
Request 4 → lock exists → waits
...
```

Only one request performs the expensive work.

---

# 4. What Happens When the Rebuild Completes?

Suppose the database returns:

```json
{
  "title": "Homepage",
  "posts": [...]
}
```

The first request stores it:

```text
DB
 ↓
New data
 ↓
Redis
 ↓
homepage
```

The waiting requests can then read:

```text
homepage → Cache HIT
```

So:

```text
100,000 requests
       ↓
1 rebuild
       ↓
100,000 cache hits
```

Instead of:

```text
100,000 requests
       ↓
100,000 DB queries
```

---

# 5. Problem With Distributed Locks

Distributed locks protect the database, but introduce another problem:

> **Everyone else is waiting for the lock holder.**

Suppose rebuilding takes 30 seconds:

```text
100K requests
     ↓
1 request → rebuild for 30 sec
     ↓
99,999 requests → WAIT
```

If the rebuild fails:

```text
Lock acquired
     ↓
Rebuild fails
     ↓
What should waiting requests do?
```

You now need to handle:

* Lock timeout
* Lock expiration
* Retry logic
* Rebuild failures
* Crashed lock holder
* Waiting request timeouts
* Fallback responses

Therefore:

> **Distributed locking is effective, but can become fragile under extreme load.**

---

# 6. Solution #2 — Probabilistic Early Refresh

A smarter approach is:

> **Don't wait until the cache expires. Refresh it gradually before expiration.**

Suppose:

```text
TTL = 60 minutes
```

Instead of:

```text
60 minutes → EXPIRE → everyone rebuilds
```

we start refreshing before expiration.

Conceptually:

```text
Cache age

0 min                                      60 min
 |--------------------------------------------|
 Fresh                                      Expire
          |             |             |
        small         higher         high
      probability   probability   probability
        refresh        refresh       refresh
```

Example:

```text
Minute 0-45
→ Almost no refreshes

Minute 50
→ 1% chance of triggering refresh

Minute 55
→ 5% chance

Minute 59
→ 20% chance
```

The exact probabilities depend on the implementation.

The important idea is:

> **The closer we get to expiration, the more aggressively we refresh.**

---

# 7. Why Does This Help?

Without early refresh:

```text
100K requests
      |
      v
Minute 60
      |
      v
100K cache misses
      |
      v
100K DB requests 💥
```

With probabilistic refresh:

```text
Minute 50
    ↓
Small number of refreshes

Minute 53
    ↓
More refreshes

Minute 56
    ↓
More refreshes

Minute 58
    ↓
More refreshes

Minute 59
    ↓
Cache gets refreshed

Minute 60
    ↓
Cache is already fresh
```

Instead of one huge spike:

```text
                100K
                 |
                 |
                 |
                 |
                 |
-----------------|------------------
                 60 min
```

we get smaller refreshes spread over time:

```text
        /\      /\      /\    /\
       /  \    /  \    /  \  /  \
------/----\--/----\--/----\/----\------
   50   53   55   57   59       60
```

---

# 8. Important: Don't Make the User Wait

The request that triggers a refresh should ideally **not wait for the refresh to finish**.

Instead:

```text
User request
     ↓
Cached value exists
     ↓
Return cached value immediately
     +
Trigger background refresh
```

So:

```text
                  Request
                     |
                     v
                   Cache
                  /     \
                 /       \
         Return value   Refresh
             |          in background
             v              |
           User             v
                           DB
                            |
                            v
                          Cache
```

This pattern is closely related to:

> **Stale-While-Revalidate**

---

# 9. Solution #3 — Background Refresh

For extremely important or extremely popular data, don't let users trigger the refresh at all.

Instead, continuously refresh the cache in the background.

Example:

```text
Homepage TTL = 1 hour
```

Instead of waiting 60 minutes:

```text
00:00 → Generate cache
00:50 → Background refresh
01:40 → Background refresh
02:30 → Background refresh
...
```

Architecture:

```text
                    Users
                      |
                      v
                    Cache
                      |
                  Cache HIT
                      |
                      v
                    Users


Background Worker
       |
       v
Generate Homepage
       |
       v
Update Cache
```

The users never have to rebuild the cache.

---

# 10. Advantages of Background Refresh

### Advantages

* No cache stampede
* Users don't wait for rebuild
* Predictable load
* Good for critical data
* Easier to reason about than probabilistic refresh

### Disadvantages

* Extra infrastructure
* Continuous work
* May refresh data nobody is requesting
* More operational complexity

---

# 11. When Should We Use Which?

## Normal Data

Regular TTL may be enough:

```text
Cache
 ↓
TTL expires
 ↓
Rebuild
```

Use when:

* Traffic isn't huge
* Rebuild is cheap
* Temporary misses are acceptable

---

## Popular + Expensive Data

Use:

```text
Distributed Lock
```

Only one request rebuilds.

Good when:

```text
Traffic = high
Rebuild = expensive
```

But be careful with waiting requests.

---

## Very Hot Data

Use:

```text
Probabilistic Early Refresh
+
Stale-While-Revalidate
```

This spreads refresh work across time.

Good when:

```text
Traffic = extremely high
Rebuild = expensive
Slight staleness = acceptable
```

---

## Extremely Critical Data

Use:

```text
Background Refresh
+
Stale-While-Revalidate
+
Fallback
```

Examples:

* Homepage
* Trending page
* Major sports match
* Breaking news
* Popular product page

The goal is:

> **Users should never be responsible for rebuilding critical cache entries.**

---

# 12. TTL Jitter — Related but Different

TTL jitter is another useful technique.

Suppose you have many cache keys:

```text
key1 → TTL 60 sec
key2 → TTL 60 sec
key3 → TTL 60 sec
key4 → TTL 60 sec
```

If they were created around the same time:

```text
60 seconds later
      ↓
Many keys expire together
      ↓
Huge cache miss spike
```

Instead:

```text
key1 → 61 sec
key2 → 67 sec
key3 → 73 sec
key4 → 81 sec
```

This spreads expiration.

---

# 13. TTL Jitter vs Probabilistic Refresh

Don't confuse these two.

### TTL Jitter

Solves:

> **Many different keys expiring at the same time.**

```text
key1 → 61 sec
key2 → 67 sec
key3 → 73 sec
key4 → 81 sec
```

---

### Probabilistic Early Refresh

Solves:

> **One extremely hot key approaching expiration.**

```text
One hot key
     ↓
Refresh probability increases
as expiration approaches
```

### Remember:

> **TTL jitter spreads expiration across keys.**

> **Probabilistic refresh spreads refresh work for a hot key across time.**

---

# 14. Cache Stampede vs Hot Key

These two problems are related but different.

## Hot Key

The key is still valid but receives enormous traffic:

```text
500K req/sec
      ↓
user:999
      ↓
Redis Node 3 🔥
```

Solutions:

* CDN
* Local cache
* Fanout
* Cache replication

---

## Cache Stampede

The key expires:

```text
Cache expires
      ↓
100K requests
      ↓
100K cache misses
      ↓
100K rebuilds
      ↓
Database 💥
```

Solutions:

* Distributed lock
* Request coalescing
* Probabilistic early refresh
* Stale-while-revalidate
* Background refresh
* Cache warming

---

# 15. Request Coalescing vs Distributed Lock

These concepts are related.

### Request Coalescing

Usually happens **inside an application server**.

```text
App1

Request 1 ─┐
Request 2 ─┤
Request 3 ─┤
Request 4 ─┤
            ↓
       One in-flight
          request
            ↓
          DB
```

It prevents duplicate work within the same application instance.

---

### Distributed Lock

Used when multiple application servers need to coordinate.

```text
              Load Balancer
                    |
       -------------------------
       |           |           |
      App1        App2        App3
       |           |           |
       +-----------+-----------+
                   |
            Distributed Lock
                   |
             One rebuilds
```

It coordinates across the entire distributed system.

---

# 16. Best Practical Strategy

For a highly popular cache entry, a robust design might be:

```text
                 Request
                    |
                    v
                  Cache
                    |
              Cache available?
                /          \
              YES           NO
               |             |
        Return cached    Fallback /
             data        controlled rebuild
               |
               v
       Is cache getting old?
               |
              YES
               |
               v
      Trigger background
          refresh
               |
               v
              DB
               |
               v
             Cache
```

With:

```text
TTL Jitter
+
Request Coalescing
+
Stale-While-Revalidate
+
Background Refresh
```

for the most critical data.

---

# 17. Interview Decision Tree

When asked:

> **"What happens when multiple requests try to rebuild an expired cache entry?"**

Think:

```text
                  Cache expires
                       |
                       v
                Many cache misses
                       |
                       v
             Cache Stampede?
                       |
                      YES
                       |
          ---------------------------
          |            |            |
       Simple       Popular       Critical
          |            |            |
      Locking      Probabilistic  Background
                    Refresh        Refresh
          |            |            |
       Others       Refresh       Worker
       wait         gradually     refreshes
```

---

# 18. Interview Answer

A strong answer:

> "This is a cache stampede or thundering herd problem. If a popular cache entry expires, many requests can see the miss simultaneously and all try to rebuild the same expensive data, overwhelming the database.
>
> A simple solution is a distributed lock where only one request rebuilds the cache and others wait. The downside is that waiting requests can timeout if the rebuild is slow or fails.
>
> For very hot data, I'd prefer stale-while-revalidate with probabilistic early refresh. Requests continue receiving the cached value while a small probability of requests triggers background refreshes as the entry approaches expiration. This spreads the rebuild work over time instead of creating one large spike.
>
> For extremely critical and predictable data, I'd use a background refresh process that proactively refreshes the cache before expiration. That prevents users from ever having to trigger the expensive rebuild.
>
> I'd also consider TTL jitter to avoid synchronized expiration across many keys."

---

# 19. Quick Revision Cheat Sheet

```text
CACHE STAMPEDE
──────────────

Problem:
Popular cache expires
        ↓
Many requests see MISS
        ↓
Many rebuilds
        ↓
Database overloaded


Distributed Lock
─────────────────
One request rebuilds
Others wait

Pros:
- Simple concept
- Prevents duplicate rebuilds

Cons:
- Waiting requests
- Timeouts
- Lock failure handling


Probabilistic Early Refresh
───────────────────────────
Refresh before expiration
Probability increases with age

Pros:
- Spreads refresh work
- Avoids huge spike
- Users can get cached data

Cons:
- More complex
- Requires tuning


Background Refresh
───────────────────
Worker refreshes before expiry

Pros:
- Users never rebuild
- Predictable load
- Great for critical data

Cons:
- Extra infrastructure
- May refresh unused data


Stale-While-Revalidate
──────────────────────
Return cached/stale value
+
Refresh in background

Goal:
Don't make users wait for cache rebuild.


TTL Jitter
──────────
Add randomness to TTL.

60 sec
→ 61 sec
→ 67 sec
→ 73 sec
→ 81 sec

Goal:
Prevent synchronized expiration.


KEY DISTINCTION
───────────────

Hot Key:
Same valid key gets huge traffic.

        ↓
CDN / Local Cache / Fanout / Replication


Cache Stampede:
Popular key expires and everyone rebuilds.

        ↓
Lock / Coalescing / Early Refresh /
SWR / Background Refresh
```

---

# 20. One Sentence to Remember

> **Cache stampede happens when expiration turns a huge number of cheap cache reads into a huge number of expensive backend rebuilds.**

The fundamental strategy is:

> **Don't let millions of users discover and rebuild an expired cache entry at the same time.**

# Cache Invalidation & Consistency --- Read Scaling Notes

## 1. The Core Problem

Caching improves read performance:

``` text
User
  ↓
Application
  ↓
Redis
  ↓ cache miss
Database
```

But after data changes, cached data may become stale.

Example:

``` text
DB:
event 123
venue = "Hyderabad Convention Center"

Redis:
event:123 → "Hyderabad Convention Center"
```

Organizer changes the venue:

``` text
DB:
venue = "Gachibowli Stadium"
```

But Redis may still contain the old value.

If stale data is unacceptable, we need a cache consistency strategy.

------------------------------------------------------------------------

# 2. Why Simple Cache Deletion Is Not Perfect

The naive approach is:

``` text
UPDATE DB
   ↓
DELETE Redis key
```

For example:

``` text
UPDATE events
SET venue = 'Gachibowli Stadium'
WHERE id = 123;

DEL event:123;
```

The next reader gets:

``` text
Redis MISS
    ↓
DB
    ↓
NEW DATA
    ↓
Redis
```

This looks correct, but concurrent readers can create race conditions.

------------------------------------------------------------------------

# 3. The Stale Cache Resurrection Race

Suppose initially:

``` text
DB:
venue = OLD

Redis:
event:123 = OLD
```

Now a writer updates the DB.

A problematic sequence can look like:

``` text
T1: Writer deletes Redis
T2: Reader sees cache miss
T3: Reader reads OLD data from DB
T4: Reader writes OLD data into Redis
T5: Writer commits NEW data
```

Final state:

``` text
DB     = NEW
Redis  = OLD   ❌
```

The stale value has been "resurrected."

This is why:

> `UPDATE DB + DELETE CACHE` is not automatically race-free.

The exact race depends on transaction timing, isolation level, replicas,
concurrent writers, and cache behavior.

------------------------------------------------------------------------

# 4. Multiple Cache Layers Make Invalidation Harder

A real system may look like:

``` text
Browser Cache
      ↓
CDN
      ↓
Application
      ↓
Redis
      ↓
Database
```

Suppose the event venue changes.

After updating the DB:

``` text
Browser → OLD
CDN     → OLD
Redis   → OLD
DB      → NEW
```

Deleting Redis does not invalidate browser or CDN copies.

Therefore:

> Cache invalidation becomes harder as the number of cache layers
> increases.

------------------------------------------------------------------------

# 5. Cache Versioning

Instead of using:

``` text
event:123
```

use:

``` text
event:123:v42
```

The version is stored with the entity in the database.

Example:

``` text
DB:

id      version     venue
123       42        Hyderabad Convention Center
```

Redis:

``` text
event:123:v42
    ↓
Hyderabad Convention Center
```

------------------------------------------------------------------------

# 6. Updating the Entity

When the event changes:

``` text
BEGIN TRANSACTION

UPDATE events
SET
    venue = 'Gachibowli Stadium',
    version = version + 1
WHERE id = 123;

COMMIT
```

Now:

``` text
DB:

id      version     venue
123       43        Gachibowli Stadium
```

The new cache key becomes:

``` text
event:123:v43
```

The old key:

``` text
event:123:v42
```

does not need to be immediately deleted.

It has simply become irrelevant.

------------------------------------------------------------------------

# 7. Why Versioning Prevents Stale Writers from Overwriting New Data

Without versioning:

``` text
event:123
```

A stale reader can potentially write old data back into the same key.

With versioning:

``` text
event:123:v42
event:123:v43
```

Suppose an old reader is still processing version 42.

It can write:

``` text
event:123:v42 = OLD
```

But the current version is:

``` text
event:123:v43 = NEW
```

The old reader cannot overwrite v43 because the keys are different.

This is the main benefit of versioning:

> **Versioning isolates different generations of cached data.**

------------------------------------------------------------------------

# 8. The Important Question: How Does the Reader Know the Current Version?

This is the subtle part.

Suppose:

``` text
DB:
version = 43
```

but Redis contains:

``` text
event:123:version = 42
```

The reader may do:

``` text
GET event:123:version
        ↓
        42
        ↓
GET event:123:v42
        ↓
OLD DATA
```

Therefore:

> Cache versioning does NOT automatically guarantee immediate
> consistency.

We have moved part of the problem to:

> "How do we reliably know the current version?"

------------------------------------------------------------------------

# 9. Separate the Two Problems

Think about versioning as two separate problems.

## Problem A --- Find the current version

``` text
"What version is current?"
```

Possible sources:

-   Database
-   Strongly consistent metadata store
-   Cached version pointer
-   Session/request state
-   Other consistency mechanisms

## Problem B --- Store the actual data

``` text
event:123:v43
```

Versioning is excellent at solving Problem B because different
generations cannot overwrite each other.

The consistency of Problem A is a separate design decision.

------------------------------------------------------------------------

# 10. Option 1 --- Read Current Version from the Database

The safest source of truth is the DB.

``` text
Reader
  ↓
DB → version = 43
  ↓
Redis → event:123:v43
```

This provides stronger correctness.

But now every read needs a DB lookup for the version.

That can reduce the benefit of caching.

So there is a tradeoff:

``` text
More consistency
      ↑
      │
      │
More DB reads
      │
      ↓
Less scalability
```

------------------------------------------------------------------------

# 11. Option 2 --- Cache the Version Pointer

We can store:

``` text
event:123:version → 43
```

Then:

``` text
Reader
  ↓
Redis: event:123:version
  ↓
43
  ↓
Redis: event:123:v43
```

This requires two cache lookups:

1.  Find the current version
2.  Fetch the actual data

This is fast, but the version pointer itself can become stale.

Example:

``` text
DB:
version = 43

Redis:
version = 42
```

A reader can still see v42.

Therefore:

> A cached version pointer provides eventual consistency unless its
> freshness is guaranteed.

------------------------------------------------------------------------

# 12. Option 3 --- Update the Version Pointer After the DB Write

Write flow:

``` text
BEGIN TRANSACTION

UPDATE event
SET venue = NEW,
    version = 43

COMMIT

SET event:123:version = 43
```

If Redis update succeeds:

``` text
DB      = 43
Redis   = 43
```

Good.

But what if:

``` text
DB update succeeds
Redis update fails
```

Then:

``` text
DB      = 43
Redis   = 42
```

The cache is temporarily stale.

This means Redis should remain a performance optimization, not the
source of truth.

------------------------------------------------------------------------

# 13. Option 4 --- Event-Driven Cache Updates

A common architecture is:

``` text
DB Transaction
      ↓
Outbox Event
      ↓
Message Broker
      ↓
Cache Updater
      ↓
Redis
```

Example:

``` text
DB:
version = 43
```

Outbox event:

``` text
EventUpdated {
    eventId: 123,
    version: 43
}
```

Consumer updates:

``` text
event:123:version = 43
```

This is reliable and scalable, but still asynchronous.

There can be a short window:

``` text
DB committed
    ↓
  10 ms
    ↓
Redis updated
```

During that window, Redis may still contain version 42.

Therefore:

> Event-driven invalidation usually provides eventual cache consistency,
> not strict immediate consistency.

------------------------------------------------------------------------

# 14. Option 5 --- Bypass Cache for Recently Updated Data

For strong consistency around recent writes, one practical technique is:

``` text
recently_updated:123 = true
```

with a short TTL.

Read flow:

``` text
                 Request
                    ↓
          Recently updated?
             /          \
           YES          NO
           ↓             ↓
          DB           Redis
```

Example:

``` text
Writer:
UPDATE DB
recently_updated:123 = true
```

Then readers temporarily go directly to the DB.

After the cache is guaranteed to be fresh, the flag expires.

This trades some read performance for stronger consistency during the
critical window.

------------------------------------------------------------------------

# 15. Read-After-Write Consistency

A very common requirement is not necessarily:

> "Every user in the world must immediately see the update."

Instead, it may be:

> "After I update something, I should immediately see my own update."

This is called:

# Read-after-write consistency

Example:

``` text
T1:
User updates event venue

T2:
Same user immediately opens event page
```

Requirement:

``` text
T2 MUST see the update from T1
```

------------------------------------------------------------------------

# 16. Sticky Routing for Read-After-Write

Suppose we have:

``` text
Primary DB
     ↓
Read Replicas
```

Normally:

``` text
Reads → Replica
Writes → Primary
```

But after a user writes:

``` text
User
  ↓
Write → Primary
```

For a short period, route that user's reads to:

``` text
Primary
```

instead of a replica.

This avoids the problem where:

``` text
Primary = NEW
Replica = OLD
```

because replication has not caught up.

------------------------------------------------------------------------

# 17. Read Replicas Add Another Consistency Problem

Architecture:

``` text
              Primary
                 │
          replication
                 ↓
             Replica
```

Writer:

``` text
Primary:
version = 43
```

Replica may temporarily have:

``` text
version = 42
```

If the reader hits the replica:

``` text
Reader
  ↓
Replica
  ↓
version 42
  ↓
event:123:v42
```

The user can see stale data.

Therefore:

> Cache consistency and replica consistency are separate problems.

You can solve one and still have the other.

------------------------------------------------------------------------

# 18. CDN + Versioned URLs

Versioning works especially well with CDNs.

Instead of:

``` text
/events/123
```

use:

``` text
/events/123?v=42
```

After the update:

``` text
/events/123?v=43
```

The browser and CDN see these as different cache entries.

Old:

``` text
/events/123?v=42
```

New:

``` text
/events/123?v=43
```

No need to immediately delete every old CDN copy.

Eventually TTL cleans up old versions.

This is also known as:

> Cache busting.

------------------------------------------------------------------------

# 19. Deleted Items Cache

Versioning works very well for individual entities.

But consider a social-media feed.

Cached feed:

``` text
feed:user:123

Post 101
Post 102
Post 103
Post 104
Post 105
```

Post 103 gets deleted.

The post may appear in millions of cached feeds.

Invalidating every feed could be extremely expensive.

Instead maintain a small deleted-items cache:

``` text
deleted_posts:

103 → true
204 → true
888 → true
```

When serving the feed:

``` text
Cached feed
     ↓
101
102
103 ← deleted
104
105
     ↓
Filter 103
     ↓
Return 101, 102, 104, 105
```

This lets the system serve mostly-correct cached data immediately while
larger cache structures are rebuilt asynchronously.

------------------------------------------------------------------------

# 20. When to Use Which Strategy

  -----------------------------------------------------------------------
  Situation                           Strategy
  ----------------------------------- -----------------------------------
  Staleness is acceptable             TTL

  Simple entity cache                 Cache-aside + invalidation

  Stale writers/race conditions are a Versioned cache
  concern                             

  CDN/static assets                   Versioned URLs / cache busting

  Strong consistency for recent       Bypass cache temporarily
  changes                             

  Read-after-write requirement        Sticky routing / primary reads

  Feeds/search/aggregations           Event invalidation / recomputation

  Deleted content appears in many     Deleted-items cache
  caches                              

  Large distributed cache             Event-driven invalidation

  Extremely strong correctness        DB/strongly consistent source of
                                      truth
  -----------------------------------------------------------------------

------------------------------------------------------------------------

# 21. The Most Important Mental Model

### Normal invalidation

Think:

``` text
"DELETE THE OLD THING"
```

``` text
event:123
     ↓
DELETE
```

The problem is that you must make sure every copy disappears.

------------------------------------------------------------------------

### Versioning

Think:

``` text
"DON'T DELETE THE OLD THING.
MAKE EVERYONE STOP ASKING FOR IT."
```

``` text
v42 → OLD
v43 → NEW

CURRENT = v43
```

Old v42 can safely remain until TTL.

------------------------------------------------------------------------

# 22. The Critical Limitation of Versioning

Do NOT say:

> "Cache versioning guarantees immediate consistency."

A better statement is:

> "Cache versioning prevents stale cache generations from overwriting
> newer generations, but the mechanism used to determine the current
> version still needs to satisfy the application's consistency
> requirements."

This distinction is extremely important.

------------------------------------------------------------------------

# 23. Interview Answer

If asked:

> "How would you handle cache invalidation when updates need to be
> immediately visible?"

A strong answer:

> "For entity-level data, I may use versioned cache keys. For example,
> instead of `event:123`, I use `event:123:v43`. When the entity is
> updated, I increment the version in the database transaction. This
> prevents stale readers from overwriting the newer cache generation
> because v42 and v43 are different keys."
>
> "However, versioning by itself doesn't guarantee immediate consistency
> because the reader still needs to know the current version. If I cache
> the version pointer, that pointer can itself become stale. For strict
> read-after-write requirements, I can use the database or a strongly
> consistent metadata source for the current version, or temporarily
> bypass the cache for recently updated entities."
>
> "For eventually consistent systems, I can cache the version pointer
> and update it asynchronously using an outbox or message broker. For
> feeds or aggregated data, versioning becomes less practical, so I'd
> consider event-driven invalidation, recomputation, or a deleted-items
> cache."
>
> "The key is to separate two problems: versioning protects different
> cache generations from overwriting each other, while the
> current-version lookup determines what readers are allowed to see."

------------------------------------------------------------------------

# 24. The 5 Questions to Test Yourself

Before an interview, make sure you can answer these:

### Q1

Why can:

``` text
UPDATE DB
DELETE Redis
```

still have race conditions?

### Q2

Why does:

``` text
event:123:v42
event:123:v43
```

prevent a stale writer from overwriting v43?

### Q3

What happens if:

``` text
DB version = 43
Redis version pointer = 42
```

?

### Q4

Why is versioning excellent for:

``` text
product:123
event:123
user:123
```

but awkward for:

``` text
feed:user:123
search:iphone
```

?

### Q5

What is the difference between:

``` text
cache consistency
```

and:

``` text
database replica consistency
```

?

------------------------------------------------------------------------

# 25. One-Line Revision Summary

> **Cache invalidation is not just about deleting stale data. Versioning
> isolates cache generations, but you still need a reliable way to
> determine the current version. For strong read-after-write
> consistency, use a strongly consistent source or temporarily bypass
> the cache; for eventual consistency, cached version pointers and
> asynchronous invalidation are usually sufficient.**
