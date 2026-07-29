# ticket master sytem design
![good-final](image-5.png)

![ticket_master-first-design](image.png)

# Problem with the Basic Reservation Flow

Our current booking flow works like this:

```text
User selects seat
        │
        ▼
Booking Service
        │
        ▼
Seat Status = RESERVED
        │
        ▼
User proceeds to payment
        │
        ▼
Payment Service sends confirmation
        │
        ▼
Booking Service updates status = BOOKED
```

This approach looks correct initially, but it has one major problem.

---

# The Problem

Ticket booking applications usually have a **multi-step booking process**.

For example:

1. User searches for an event.
2. User selects one or more seats.
3. The selected seats are marked as **RESERVED** so that no other user can book them.
4. User proceeds to the payment page.
5. After successful payment, the Payment Service notifies the Booking Service.
6. Booking Service changes the seat status from **RESERVED** to **BOOKED**.

Everything works fine if the user completes the payment.

---

# What if the user never completes the payment?

Consider the following scenario:

- User selects Seat A1.
- Booking Service marks Seat A1 as **RESERVED**.
- User reaches the payment page.
- Instead of paying, the user:
  - closes the browser,
  - loses internet connection,
  - switches off the phone,
  - or simply decides not to continue.

Since no payment confirmation is received, the Booking Service never updates the seat status to **BOOKED**.

The seat remains in the **RESERVED** state indefinitely.

---

# Why is this a Problem?

If the seat remains reserved forever:

- Other users cannot book that seat.
- The seat is not actually sold.
- The event may appear to have fewer available seats than it really does.
- The system starts losing potential bookings and revenue.

In other words, the seat becomes **blocked permanently**, even though nobody purchased it.

---

# Summary

The issue with this design is that **a reservation is created immediately when the user selects a seat**, but **there is no mechanism to handle users who abandon the booking before completing the payment**.

As a result, seats can remain in the **RESERVED** state forever, making them unavailable for other users even though they were never actually booked.

![cron-job](image-1.png)

- kuch issue ho sakta hai na cron-job ke sath ki job run nhi hua to thodi der ke liye we will see less tickets as some will be still there in Reservered state right.

![redis](image-2.png)

![final-design](image-3.png)

                Event CRUD
                     |
                     v
               PostgreSQL (Source of Truth)
                     |
          CDC / Kafka / Event Bus
                     |
             Search Indexer Service
                     |
              Elasticsearch Cluster
             (Shards + Replicas)
                     |
              Search Service API
                     |
          +------------------------+
          |                        |
     Redis Hot Cache         Elasticsearch
          |                        |
          +-----------+------------+
                      |
                   API Gateway
                      |
                     CDN
                      |
                    Client

 - talk about personal recommendation system also that if it comes into picture then cashing will not work efficiently right.

 ![waiting-queue](image-4.png)

 # Ticket Booking System Design Notes

## 1. Problem: Stale Ticket Availability

### Scenario

-   Search results show seats as **Available**.
-   User A reserves Seat A1.
-   Users B and C are still looking at an old response and also click
    A1.
-   Redis prevents double booking, but users still experience "Seat
    unavailable".

### Key Insight

Redis distributed locks solve **consistency**, not **stale UI**.

### Production Solutions

1.  Elasticsearch is used only for **event discovery**.
2.  Seat availability comes from the **Booking Service (Redis +
    PostgreSQL)**.
3.  Push updates using **WebSockets/SSE** whenever a seat changes.
4.  Use a **short reservation TTL (2--5 minutes)**.
5.  Periodically refresh the seat map or refresh on reconnect.
6.  For mega events, place users in a **virtual waiting room**.

------------------------------------------------------------------------

## 2. How Does the Waiting Queue Work?

### Common Misconception

Users **do not select seats before entering the queue**.

### Actual Flow

    Users
       |
    Waiting Room
       |
    Admitted (10,000 users)
       |
    Load Seat Map
       |
    Select Seat
       |
    Reserve (Redis)
       |
    Payment
       |
    Booked

### Example

-   3,000,000 users arrive.
-   Waiting room admits only 10,000 users.
-   Remaining users only see:

```{=html}
<!-- -->
```
    Queue Position: 18,543
    Estimated Wait: 15 minutes

They **cannot see or select seats** until admitted.

### Why?

If users selected seats before entering the queue:

    1,000,000 users
          |
    Everyone selects A1
          |
    Wait 30 minutes

By the time many users enter, A1 would already be sold.

------------------------------------------------------------------------

## 3. What Happens When 10,000 Users Click the Same Seat?

### Scenario

-   Waiting room admits 10,000 users.
-   Every user clicks Seat A1 at exactly the same time.

```{=html}
<!-- -->
```
    10,000 Requests
           |
     Booking Service
           |
     Redis

### Booking Service

Each request executes something like:

``` text
SET seat:A1 userId NX EX 300
```

where: - NX = Set only if key does not exist. - EX 300 = Expire after 5
minutes.

### Why Only One User Wins?

Redis is **single-threaded**.

Although 10,000 requests arrive almost simultaneously, Redis executes
commands one at a time.

Internally:

    Request 532
    Request 81
    Request 900
    ...
    Request 10000

### First Request

    SET seat:A1 user532 NX EX 300

    Result:
    OK

Seat reserved.

### Remaining Requests

    SET seat:A1 user81 NX EX 300

    Result:
    nil

Every remaining request gets the same response because the key already
exists.

### Final Result

    10,000 Requests

    ↓

    Redis

    ↓

    1 Success

    9,999 Fail

### API Responses

Winner:

    HTTP 200

    Seat Reserved

    Proceed to Payment

Others:

    HTTP 409

    Seat Already Reserved

------------------------------------------------------------------------

## 4. How Can the User Experience Be Improved?

Instead of returning:

    Seat unavailable

The Booking Service can immediately suggest nearby seats.

Example response:

``` json
{
  "status": "SeatUnavailable",
  "suggestedSeats": [
    "A2",
    "A3",
    "A5"
  ]
}
```

The frontend can instantly highlight these seats.

------------------------------------------------------------------------

## 5. Why Doesn't Redis Become the Bottleneck?

Redis is extremely fast.

A simple command like:

``` text
SET key NX EX
```

takes only microseconds.

A single Redis instance can process hundreds of thousands (and often
over a million) simple operations per second depending on hardware.

Therefore:

-   Redis is usually **not** the bottleneck.
-   Database writes, payment processing, and network latency are
    typically much slower.

Redis is chosen because it provides: - Atomic operations - High
throughput - Very low latency

------------------------------------------------------------------------

# Final Architecture

    Users
       |
    Waiting Room
       |
    10,000 Active Users
       |
    Booking Service
       |
    +----------------------+
    | Redis (Reservation)  |
    | PostgreSQL           |
    +----------------------+
       |
    Seat Reserved Event
       |
    Kafka / Redis PubSub
       |
    WebSocket Server
       |
    Update all connected clients

## Key Takeaways

-   Waiting room limits the number of active users.
-   Users choose seats **only after** entering the booking flow.
-   Redis guarantees only one reservation for a seat.
-   WebSockets keep active users' seat maps synchronized.
-   Elasticsearch is for search, not real-time seat availability.
-   Short reservation TTLs prevent inventory from being locked
    unnecessarily.
-   Returning alternative seat suggestions significantly improves user
    experience.

# Ticketmaster System Design Notes: Cron Job vs Redis for Reservation Expiration

## Problem Statement

In a ticket booking system, users first **reserve** seats before making the payment. Since payment can take several minutes, we need a mechanism to automatically release seats if the user never completes the payment.

The question is:

> Should we use a cron job that periodically scans the database for expired reservations, or should we use Redis with TTL?

The answer is that **both approaches are valid**, but they have different trade-offs depending on the scale of the system.

---

# Approach 1: Database + Cron Job

## Ticket Table

```text
Ticket
------
id
eventId
seat
price
status
reservedTimestamp
userId
```

Possible ticket states:

```
AVAILABLE
RESERVED
BOOKED
```

---

# Reserve Flow

When a user reserves a seat:

```sql
UPDATE Ticket
SET
    status = 'RESERVED',
    reservedTimestamp = NOW(),
    userId = 123
WHERE id = 10;
```

The seat is now locked for the user.

---

# Confirm Flow

After successful payment:

```sql
UPDATE Ticket
SET status = 'BOOKED'
WHERE id = 10;
```

---

# Expiration Flow

A cron job runs periodically.

Example:

```sql
SELECT *
FROM Ticket
WHERE status = 'RESERVED'
AND reservedTimestamp < NOW() - INTERVAL '10 minutes';
```

For every expired reservation:

```sql
UPDATE Ticket
SET
    status = 'AVAILABLE',
    reservedTimestamp = NULL,
    userId = NULL;
```

The seat becomes available again.

---

# Architecture

```
Client
    │
Reserve Seat
    │
    ▼
Booking Service
    │
    ▼
PostgreSQL
    │
    ▼
Cron Job
    │
Release expired reservations
```

---

# Advantages of Cron Job

### 1. Very Simple

No extra infrastructure.

Only PostgreSQL is required.

---

### 2. Easy to Understand

The reservation lifecycle is entirely stored inside the database.

```
AVAILABLE
      │
      ▼
RESERVED
      │
      ▼
BOOKED
```

---

### 3. Database is the Source of Truth

Everything is stored in one place.

No synchronization problems.

---

### 4. Works Well for Small and Medium Scale

Examples:

- Movie booking
- Hotel booking
- Airline booking
- Internal reservation systems

For thousands or even hundreds of thousands of reservations per day, this approach is perfectly acceptable.

---

# Problems with Cron Jobs at Large Scale

Although this design works, it starts breaking down when the traffic becomes extremely high.

Imagine:

```
Taylor Swift Concert

100,000 seats

20 million users

Millions of reservation requests
```

Now the limitations become obvious.

---

# Problem 1: Seat Release is Delayed

Suppose reservation expires at:

```
10:00:00
```

Cron runs every minute:

```
10:01:00
```

The seat remains unavailable for one extra minute.

Another user trying to reserve at:

```
10:00:20
```

will still see:

```
Seat Reserved
```

even though the reservation should already have expired.

---

If cron runs every 5 seconds:

```
Maximum delay = 5 seconds
```

Better, but still not real-time.

---

# Problem 2: Database Scans Become Expensive

Imagine:

```
5 million RESERVED rows
```

Every cron execution performs:

```sql
SELECT *
FROM Ticket
WHERE status='RESERVED'
AND reservedTimestamp < NOW();
```

Even with indexes:

- Index lookup
- Disk I/O
- Row filtering

must happen repeatedly.

As traffic grows, this becomes expensive.

---

# Problem 3: Continuous Writes

Every reservation causes multiple database updates.

Example:

```
Reserve

↓

Reservation expires

↓

Cron updates ticket

↓

Another reservation

↓

Cron updates again
```

Millions of unnecessary writes occur every day.

---

# Problem 4: Primary Database Load

If cron runs frequently:

```
Every second

Every 2 seconds

Every 5 seconds
```

The primary database constantly processes expiration scans and updates.

Instead of serving customer traffic, it spends resources cleaning expired reservations.

---

# Redis Approach

Redis changes how expiration is handled.

Instead of asking:

> Which reservations have expired?

Redis automatically tracks expiration for every reservation.

---

# Reserve Flow

Suppose the reservation lasts for 10 minutes.

Store:

```
reservation:123
```

with

```
TTL = 600 seconds
```

Example:

```
SET reservation:123 value EX 600
```

Redis now knows exactly when this reservation expires.

No periodic scanning is required.

---

# Automatic Expiration

After 600 seconds:

```
reservation:123
```

is automatically removed.

Redis internally manages expiration.

No cron job scans millions of records.

---

# Using Redis Sorted Sets

Another common technique is storing reservations in a Sorted Set.

```
Seat A10 → 10:10

Seat A11 → 10:12

Seat A12 → 10:20
```

The score is simply the expiration timestamp.

Worker fetches only expired reservations:

```
ZRANGEBYSCORE
score <= currentTime
```

Instead of scanning every reservation, Redis directly returns only the expired ones.

Complexity becomes approximately:

```
O(log N)
```

instead of scanning millions of rows.

---

# Does Redis Alone Make the Seat Available Again?

No.

This is a very common misconception.

When Redis removes a key after TTL, PostgreSQL still contains:

```
status = RESERVED
```

Some process still needs to update the database.

Redis only tells us:

> This reservation has expired.

The database still needs to be updated.

---

# Recommended Architecture

Redis acts only as a **temporary lock manager**.

PostgreSQL remains the source of truth.

```
Reserve Request

↓

Acquire Redis Lock

↓

Update PostgreSQL
status = RESERVED

↓

Return Reservation
```

---

# Confirmation Flow

```
Payment Successful

↓

Update PostgreSQL

status = BOOKED

↓

Delete Redis Lock
```

---

# Expiration Flow

```
Redis TTL expires

↓

Expiration Event / Background Worker

↓

Update PostgreSQL

status = AVAILABLE

↓

Delete reservation record (optional)
```

---

# Why PostgreSQL Should Remain the Source of Truth

Redis is an in-memory system.

Although Redis supports persistence, it is primarily optimized for speed.

Permanent business data should still live in PostgreSQL.

The database contains:

- reservations
- bookings
- payments
- tickets
- audit history

Redis contains only temporary reservation locks.

---

# Comparison

| Feature | Cron Job | Redis TTL |
|----------|----------|-----------|
| Simplicity | ⭐⭐⭐⭐⭐ | ⭐⭐⭐ |
| Extra Infrastructure | None | Redis Cluster |
| Real-Time Expiration | ❌ No | ✅ Yes |
| Database Scans | Heavy | None |
| Database Writes | High | Lower |
| Scalability | Medium | Very High |
| Suitable for Millions of Users | Difficult | Excellent |

---

# Which One Should You Choose?

## Small Scale

Examples:

- Local movie booking
- Internal company booking system
- Startup ticket platform

Recommended:

```
Database + Cron Job
```

Simple.

Reliable.

Easy to maintain.

---

## Medium Scale

Examples:

- BookMyShow (regional)
- Event platforms

Either approach works.

Choice depends on expected traffic.

---

## Very Large Scale

Examples:

- Ticketmaster
- Taylor Swift Concert
- FIFA World Cup
- IPL Finals
- Olympics

Recommended:

```
Redis

+

PostgreSQL
```

Redis provides:

- extremely fast locking
- automatic expiration
- no database scans
- lower write load
- excellent scalability

while PostgreSQL stores all permanent booking information.

---

# Interview Answer

If an interviewer asks:

> "Why not simply use a cron job?"

A strong answer is:

> "A cron job is a perfectly valid solution for small and medium-scale systems because it keeps the architecture simple and the database remains the single source of truth. However, at Ticketmaster scale, continuously scanning millions of reservations introduces unnecessary database reads, delayed seat release, and additional write load. Redis provides in-memory locks with TTLs, allowing reservations to expire efficiently without scanning the database. I would use PostgreSQL as the source of truth and Redis only for temporary reservation locks and expiration management."

This answer demonstrates that you understand:
- Why the cron job approach is correct.
- Why Redis is introduced at large scale.
- The trade-offs between simplicity and scalability.
- Why PostgreSQL should remain the authoritative data store while Redis handles temporary locks.