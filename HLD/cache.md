# Caching

This is exactly how I teach caching in system design interviews.

One mistake almost everyone makes is trying to memorize **20 cache concepts**.

Instead, if you really understand **7-8 core concepts**, you can answer almost every caching question in an interview.

These are the topics I would spend **80% of my preparation time on**.

---

# The 8 Most Important Cache Concepts

```
Caching

├── 1. Why Cache?
├── 2. What Should We Cache?
├── 3. Cache Read Patterns
├── 4. Cache Write Patterns
├── 5. Cache Invalidation
├── 6. TTL
├── 7. Cache Eviction
└── 8. Cache Failures
```

Master these, and you can design almost any scalable system.

---

# 1. WHY DO WE NEED CACHE?

Let's not start with Redis.

Let's start with a story.

---

## Scenario 1

Imagine you own a restaurant.

Every customer orders tea.

Normally the chef does this.

```
Customer

↓

Kitchen

↓

Prepare Tea (5 mins)

↓

Customer
```

Now imagine 100 customers.

Kitchen becomes busy.

Customers wait.

---

Now imagine another approach.

Chef prepares 20 cups in advance.

```
Customer

↓

Counter

↓

Already Prepared Tea

↓

Done
```

Almost zero waiting.

That counter is the cache.

The kitchen is the database.

---

### Mapping to Software

```
Restaurant Counter
        ↓
      Redis

Kitchen
      ↓
Database
```

Instead of cooking every time...

Store the result.

---

## Example

Instagram profile.

Without cache

```
User

↓

Backend

↓

Database

↓

Return Profile
```

Every request touches the database.

Now imagine

10 million users.

Database dies.

---

With cache

```
User

↓

Redis

↓

Profile

↓

Done
```

Database isn't even touched.

---

### Lesson

Cache is not about making things fast.

It is about protecting expensive resources.

**This sentence alone is worth remembering.**

A database is expensive.

A recommendation engine is expensive.

A machine learning prediction is expensive.

A report generation is expensive.

Cache protects expensive computations.

---

# 2. WHAT SHOULD WE CACHE?

One of the biggest mistakes beginners make is:

> "Let's cache everything."

No.

Cache is also memory.

Memory is expensive.

---

Imagine your wardrobe.

Would you keep

* Winter jacket
* Raincoat
* Festival clothes
* Marriage suit

on your chair every day?

No.

You keep only things you use daily.

Cache is exactly the same.

---

Good candidates

```
Instagram Profile

Product Details

Trending Videos

Homepage

Popular Searches

Country List

Currency List
```

Bad candidates

```
OTP

Bank Balance

Stock Price

Payment Status

Live Cricket Score
```

Why?

Because they change too often.

---

### Golden Rule

Cache

* Frequently Read
* Rarely Updated

Don't cache

* Frequently Updated
* Rarely Read

---

# 3. CACHE READ PATTERN

This is asked in almost every interview.

There are many patterns.

But honestly...

Only one dominates the industry.

---

## Cache Aside (Lazy Loading)

Let's use Netflix.

You open a movie.

```
Movie

↓

Redis

↓

Not Found
```

Redis says

"I don't have it."

Now

```
Redis

↓

Database

↓

Movie

↓

Redis

↓

User
```

Second request

```
Redis

↓

Movie

↓

Done
```

Database is skipped.

---

### Why is it called Lazy?

Because we don't populate the cache until someone asks.

Imagine opening a new grocery store.

Do you buy every item in India?

No.

You buy when customers start asking.

That's Lazy Loading.

---

### Real Life Example

Suppose

```
Movie A

Nobody watches it.
```

No cache.

Fine.

Someone watches.

Now cache it.

Perfect.

---

This is why almost every company uses Cache Aside.

---

# 4. CACHE WRITE PATTERNS

This confuses many engineers.

Let's understand through banking.

---

Suppose your account has

```
₹5000
```

You deposit

```
₹1000
```

There are three places.

```
User

↓

Application

↓

Cache

↓

Database
```

Now...

Who should update first?

---

## Write Through

```
Update Cache

↓

Update Database

↓

Success
```

Everything stays consistent.

Safe.

But slower.

---

Example

Bank account.

Consistency matters.

---

## Write Back

```
Update Cache

↓

Success

↓

Later

↓

Database
```

User gets response immediately.

Very fast.

Danger?

If Redis crashes...

Money disappears.

---

Where do we use it?

Analytics.

Likes.

Views.

Logs.

Temporary counters.

---

## Write Around

```
Database

↓

Done

↓

Cache ignored
```

Cache fills only when someone reads.

Useful when writes are frequent but reads are not.

---

### Interview Tip

Whenever someone says

"Bank"

Immediately think

```
Write Through
```

Whenever someone says

"Likes"

Think

```
Write Back
```

---

# 5. CACHE INVALIDATION

This is probably the most important topic.

People even joke

> Cache invalidation is one of the hardest problems in computer science.

Let's understand why.

---

Imagine Amazon.

Laptop price

```
₹60,000
```

Stored in Redis.

Seller changes it.

```
₹50,000
```

Database updated.

But Redis still says

```
₹60,000
```

Customer buys.

Who pays the loss?

Amazon.

---

So...

Whenever data changes...

Cache must also change.

---

Three common solutions.

---

## Solution 1

Delete cache.

```
Update Database

↓

Delete Cache
```

Next user

```
Cache Miss

↓

Database

↓

New Cache
```

Most common approach.

Simple.

Reliable.

---

## Solution 2

Update cache immediately.

```
Database

↓

Redis

↓

Both Updated
```

Useful when the item is very popular.

---

## Solution 3

TTL

Don't bother updating.

Just let it expire.

---

# Real World Example

Weather App.

Weather changes every hour.

No need to update every minute.

Simply

```
TTL

30 minutes
```

After 30 minutes

Fresh weather.

Easy.

---

# 6. TTL (Time To Live)

Think of milk.

```
Milk

Expiry

3 Days
```

After expiry

Throw away.

Same with cache.

```
User Profile

TTL

10 Minutes
```

After 10 minutes

Cache disappears.

Next request rebuilds it.

---

### Why TTL?

Suppose you forget invalidation.

TTL eventually fixes stale data.

It is the safety net.

---

### Real Examples

Weather

```
30 mins
```

Stock

```
2 seconds
```

User Profile

```
10 mins
```

Country List

```
1 day
```

---

Notice something.

More stable data

Longer TTL.

---

# 7. CACHE EVICTION

Cache is not unlimited.

Imagine your phone.

```
Storage

128 GB
```

Full.

Now what?

Delete old photos.

Same thing.

Redis memory becomes full.

Which data should go?

---

## LRU

Least Recently Used.

Imagine Netflix.

```
Movie A

Yesterday
```

Movie B

```
Last watched

2 years ago
```

Which one should stay?

Obviously

Movie A.

LRU removes

Movie B.

---

## LFU

Least Frequently Used.

Example.

```
Movie A

Viewed

100000 times
```

Movie B

Viewed

```
2 times
```

Keep Movie A.

Delete Movie B.

---

### Easy Memory Trick

LRU

Recent.

LFU

Popular.

---

# 8. CACHE FAILURES

This is what interviewers love.

---

## Cache Stampede

Imagine

IPL Final.

Everyone opens Hotstar.

At exactly

```
8 PM
```

Cache expires.

Now

```
10 Million Users

↓

Database
```

Database crashes.

---

Solution

Only one request goes to DB.

Everyone else waits.

```
User

↓

Lock

↓

Database

↓

Redis

↓

Everyone Gets Result
```

---

## Cache Penetration

Someone keeps requesting

```
User ID

999999999
```

Doesn't exist.

Redis

Miss.

Database

Not Found.

Again.

Again.

Again.

Database keeps working for nothing.

---

Solution

Cache

```
NOT FOUND
```

for a short time.

---

## Cache Breakdown

One extremely popular key expires.

Imagine

```
Virat Kohli Profile
```

Millions request it.

Cache expires.

Everyone goes to DB.

Boom.

---

Solution

Never allow hot keys to expire unexpectedly.

Refresh them before they expire.

---

## Cache Avalanche

Imagine

```
10 Million Keys

TTL

10 PM
```

At

```
10 PM
```

Everything expires.

Database gets flooded.

---

Solution

Instead of

```
10 PM
```

Use

```
10 PM ± Random

5 mins

10 mins

20 mins
```

Different keys expire at different times.

---

# The Complete Story

Whenever you're designing a system, mentally walk through these questions:

```
User Request
      │
      ▼
1. Is this operation expensive?
      │
      ▼
Yes → Cache it
      │
      ▼
2. What should be cached?
(Frequently read, rarely updated)
      │
      ▼
3. How will it be read?
(Cache Aside)
      │
      ▼
4. How will writes work?
(Write Through / Write Back / Write Around)
      │
      ▼
5. How will stale data be handled?
(Cache Invalidation)
      │
      ▼
6. What should the TTL be?
      │
      ▼
7. What happens when memory is full?
(LRU / LFU)
      │
      ▼
8. What happens under extreme traffic?
(Stampede, Penetration, Breakdown, Avalanche)
```

## How I would prepare this for interviews

If I were preparing for companies like Amazon, Uber, Flipkart, Microsoft, Google, or Meta, I would go even deeper into these four topics because they appear repeatedly in real system design discussions:

1. **Cache Aside** – why it's the industry standard, its lifecycle, consistency issues, and when it doesn't fit.
2. **Cache Invalidation** – multiple strategies, trade-offs, race conditions, and handling stale data.
3. **TTL Design** – how to choose TTL values based on business requirements rather than arbitrary numbers.
4. **Distributed Caching with Redis** – cache key design, sharding, replication, high availability, and handling failures.

These four concepts form the backbone of most production caching architectures. Once you truly understand them with real-world scenarios, the remaining caching topics become much easier to reason about.

I would actually recommend spending an entire session on **Cache Aside alone**, because it naturally leads into cache consistency, invalidation, race conditions, and distributed systems—all of which are core system design topics.


## in Cache Stampede if we start putting locks in reads also then this would be a problem right...don't you think in normal times also this will make overall system slow?

Excellent question.

This is actually one of the questions that differentiates someone who has **memorized cache stampede** from someone who **understands distributed systems**.

The short answer is:

> **No, we don't put locks on every read.** We only lock **cache misses** for a specific key.

Let's understand why.

---

# Imagine this system

Suppose Netflix has this cache.

```
Movie: Avengers
```

10 million users request it.

Normally

```
User

↓

Redis

↓

Movie

↓

Return
```

This is just a Redis GET.

There is **no lock**.

This operation is extremely fast (microseconds).

---

## Normal Day

Suppose Redis already has the value.

```
User 1
        \
User 2 ----> Redis
        /
User 3

↓

Movie Found

↓

Return
```

Every request is independent.

No waiting.

No locking.

No problem.

---

# When does the problem happen?

Now imagine the cache expires.

At exactly 10:00 PM

Redis removes the key.

Now all users ask simultaneously.

```
User1
User2
User3
...
User100000

↓

Redis

↓

MISS
```

Now everyone says

> "I'll go to the database."

```
100000 Users

↓

Database
```

Database dies.

This is Cache Stampede.

---

# Your suggestion

You said:

> "If we lock reads also, won't everything become slow?"

Exactly.

If every read looked like this

```
Acquire Lock

↓

Read Cache

↓

Release Lock
```

then even cache hits would become serialized.

Imagine

```
1000 requests/sec

↓

One lock

↓

One by one
```

You have destroyed the entire purpose of Redis.

Redis is fast because millions of GETs can happen concurrently.

So we **never** do this.

---

# What actually happens?

The lock is only acquired **after a cache miss**.

Let's see.

---

## Case 1: Cache Hit

```
User

↓

Redis

↓

Found

↓

Return
```

No lock.

No waiting.

99% of requests should ideally be like this.

---

## Case 2: Cache Miss

```
User

↓

Redis

↓

MISS

↓

Acquire Lock

↓

Database

↓

Redis

↓

Release Lock

↓

Return
```

Only this request acquires the lock.

---

## Meanwhile...

Another request arrives.

```
User 2

↓

Redis

↓

MISS

↓

Try Lock
```

Lock already taken.

So User 2 does **not** hit the database.

Instead it waits briefly.

```
User 2

↓

Wait 20 ms

↓

Retry Redis

↓

Found

↓

Return
```

Only one database query happens.

---

# Timeline

Suppose 1000 users request the same key.

### Without Lock

```
10:00:00

1000 Requests

↓

1000 Cache Misses

↓

1000 DB Queries
```

Database explodes.

---

### With Lock

```
10:00:00

1000 Requests

↓

1000 Cache Misses

↓

Only Request #1 gets lock

↓

Database

↓

Redis Updated

↓

999 Requests retry Redis

↓

Done
```

Only one database query.

Huge difference.

---

# Important Observation

Notice something.

The lock exists for only about **30–100 ms** (or however long it takes to fetch from the database).

After that

```
Redis Filled

↓

Everyone Reads Normally
```

The system returns to full speed.

So the lock is **very short-lived**.

---

# Even Better Solutions

Large companies don't like making requests wait if they can avoid it.

They use smarter techniques.

---

## Solution 1: Stale While Revalidate (Very Popular)

Suppose cache expires.

Instead of deleting it immediately,

keep the old value.

```
Cache

↓

Expired

↓

Still Return Old Data

↓

Background Refresh

↓

New Cache
```

Users never wait.

Only a background worker refreshes the data.

This works well for:

* Product catalogs
* News feeds
* Weather
* Homepages

A few seconds of stale data is acceptable.

---

## Solution 2: Background Refresh

Instead of waiting for the first user,

refresh hot keys before they expire.

For example:

```
TTL = 10 minutes

At 9 minutes

↓

Background Job

↓

Refresh Cache
```

The cache never actually expires for hot data.

Netflix, YouTube, and Instagram use this approach for very popular content.

---

## Solution 3: Request Coalescing (Single Flight)

This is my favorite.

Imagine 100 requests ask for the same key at the same time.

Instead of each creating its own database request:

```
100 Requests

↓

SingleFlight

↓

1 Database Query

↓

Same Result Shared

↓

100 Responses
```

This isn't necessarily implemented with a traditional distributed lock. Instead, the application notices that a fetch for that key is already in progress and lets the other requests "join" the same in-flight operation. When the result arrives, everyone gets the same response.

---

# How FAANG Companies Think

Here's the mental model:

```
Is it a Cache Hit?

Yes
│
├── Return immediately.
│
└── No locks.

↓

Is it a Cache Miss?

↓

Is someone already rebuilding this key?

↓

Yes
│
├── Wait briefly OR
├── Use stale value OR
└── Join the in-flight request

↓

No

↓

One request rebuilds the cache.
```

Notice the philosophy:

* **Optimize the common case (cache hits)** because that's 95–99% of traffic.
* Accept a little extra work on the **rare case (cache miss)**.

This is a fundamental principle in system design: **don't slow down the fast path to optimize an uncommon path**.

---

### Interview Insight

If an interviewer asks:

> "Should we put locks on every cache read?"

A strong answer is:

> "No. Locks should only be used during cache regeneration after a miss, and even then only per key. The common path—cache hits—must remain completely lock-free. In high-scale systems, techniques like request coalescing, stale-while-revalidate, or proactive background refresh are often preferred because they reduce contention while still preventing a cache stampede."

That answer demonstrates an understanding of both performance and scalability rather than just knowing the term "cache stampede."

