# Live Comments System Design

## 1. Problem Statement

We need to design a **real-time live comments system** for a video platform.

Example:

```text
Video: video123

Users watching:
    User A
    User B
    User C
    ...
    User 10,000
```

When any user posts a new comment:

```text
User A → "Amazing video!"
```

all users currently watching that video should receive the comment in near real time.

The system must support:

* Creating comments
* Persisting comments
* Fetching historical comments
* Real-time comment delivery
* Large numbers of concurrent viewers
* Millions/billions of comments
* Horizontal scaling
* Failure handling
* Resuming/reconnecting clients

---

# 2. High-Level Architecture

```text
                         ┌──────────────────┐
                         │      Client      │
                         └────────┬─────────┘
                                  │
                         HTTP + SSE│
                                  ▼
                         ┌──────────────────┐
                         │   API Gateway    │
                         └───────┬──────────┘
                                 │
                ┌────────────────┴─────────────────┐
                │                                  │
                ▼                                  ▼
       ┌──────────────────┐              ┌────────────────────┐
       │  Comment Service │              │ Realtime Comment   │
       │                  │              │      Service       │
       └────────┬─────────┘              └─────────┬──────────┘
                │                                  │
        ┌───────┴────────┐                         │
        │                │                         │
        ▼                ▼                         │
     Cache            Cassandra                    │
        │                │                         │
        └────────────────┘                         │
                                                  │
                                                  │
                                          ┌───────▼───────┐
                                          │    Pub/Sub    │
                                          └───────▲───────┘
                                                  │
                                                  │
                                      New Comment Events
```

---

# 3. Main Components

## 3.1 Client

The client is responsible for:

* Displaying existing comments
* Opening an SSE connection
* Receiving new comments
* Rendering new comments
* Sending new comments
* Reconnecting if SSE connection breaks
* Keeping track of the last received comment/sequence number

---

# 4. API Gateway

The API Gateway is the entry point for clients.

It handles:

* Authentication
* Authorization
* Rate limiting
* Routing
* Request validation
* SSE connection routing

Typical APIs:

```text
GET  /videos/{videoId}/comments
POST /videos/{videoId}/comments
GET  /videos/{videoId}/comments/stream
```

---

# 5. Comment Service

The Comment Service is responsible for the **durable comment operations**.

Responsibilities:

```text
1. Create comment
2. Validate comment
3. Persist comment
4. Fetch historical comments
5. Read/write cache
6. Publish NewComment event
```

It should NOT directly manage thousands/millions of SSE connections.

This separation is important.

The Comment Service deals with:

```text
Business logic + persistence
```

while the Realtime Comment Service deals with:

```text
Real-time connection management + fanout
```

---

# 6. Cassandra

Cassandra is used for storing comments because the system is:

* Write-heavy
* Highly scalable
* Distributed
* Append-heavy
* Often queried by video + time

A conceptual table:

```sql
comments

comment_id
video_id
author_id
content
created_at
sequence_number
```

The important access pattern is:

```text
Get comments for a video ordered by time
```

Therefore the Cassandra partitioning strategy should be designed around the query pattern.

For example:

```text
PRIMARY KEY ((video_id), created_at, comment_id)
```

However, extremely popular videos can create very large/hot partitions.

Therefore, at very large scale we may introduce buckets/shards:

```text
PRIMARY KEY ((video_id, bucket), created_at, comment_id)
```

where:

```text
bucket = time bucket
```

Example:

```text
video123 + 10:00
video123 + 10:05
video123 + 10:10
```

This prevents one enormous Cassandra partition.

---

# 7. Cache

The latest comments can be cached.

Example:

```text
video123 → latest 100 comments
```

The cache is useful because when thousands of users join the same popular video, we don't want every request to hit Cassandra.

Flow:

```text
Client
   ↓
Comment Service
   ↓
Cache
   │
   ├── HIT → return comments
   │
   └── MISS
          ↓
       Cassandra
          ↓
        Cache
          ↓
        Client
```

---

# 8. Pub/Sub

Pub/Sub decouples the Comment Service from the Realtime Comment Service.

When a comment is successfully persisted:

```text
Comment Service
       │
       ├──→ Cassandra
       │
       └──→ Pub/Sub
```

Event:

```json
{
  "eventType": "NEW_COMMENT",
  "commentId": "c999",
  "videoId": "video123",
  "authorId": "userA",
  "content": "Amazing video!",
  "createdAt": "2026-08-14T12:30:00Z",
  "sequenceNumber": 1001
}
```

The realtime layer consumes this event and broadcasts it to interested clients.

---

# 9. Why Pub/Sub?

Without Pub/Sub:

```text
Comment Service
      │
      ├── User A SSE
      ├── User B SSE
      ├── User C SSE
      ├── User D SSE
      ├── ...
      └── User 10,000 SSE
```

The Comment Service would need to maintain all real-time connections.

This creates tight coupling and makes scaling difficult.

With Pub/Sub:

```text
Comment Service
       │
       ▼
    Pub/Sub
       │
       ▼
Realtime Comment Service
       │
       ├── User A
       ├── User B
       ├── User C
       └── ...
```

Now each service has a clear responsibility.

---

# 10. Realtime Comment Service

The Realtime Comment Service manages long-lived SSE connections.

Responsibilities:

```text
1. Accept SSE connections
2. Maintain active connections
3. Track which video each connection is watching
4. Consume comment events
5. Fan out events to relevant connections
6. Handle disconnects
7. Handle reconnects
```

---

# 11. SSE

SSE = Server-Sent Events.

It provides a long-lived HTTP connection:

```text
Client
   │
   │ HTTP request
   ▼
Server
   │
   │ connection remains OPEN
   │
   ├──── comment ────→ Client
   ├──── comment ────→ Client
   ├──── comment ────→ Client
   └──── comment ────→ Client
```

Unlike normal HTTP:

```text
Request → Response → Connection closed
```

SSE is:

```text
Request → Connection stays open → Server pushes events
```

---

# 12. Why SSE Instead of WebSocket?

For this particular use case, communication is primarily:

```text
Server → Client
```

The server continuously sends new comments.

SSE is therefore a good fit.

### SSE advantages

* Simple HTTP-based protocol
* Works naturally with browsers
* Automatic reconnect support
* One-way server-to-client communication
* Easy to integrate with existing HTTP infrastructure

### WebSocket would be better when

The application requires:

```text
Client ↔ Server
```

real-time bidirectional communication.

For example:

* Chat
* Multiplayer games
* Collaborative editing
* Real-time trading interaction

For live comments:

```text
POST comment → normal HTTP
Receive comments → SSE
```

is a clean architecture.

---

# 13. Complete User Journey

This is the most important part.

Assume:

```text
User A opens video123
```

---

## Step 1: User opens video

Client loads:

```text
video123
```

The client needs:

```text
A. Existing comments
B. New comments
```

It can initiate both flows.

---

# 14. Step 2: Fetch Existing Comments

Client:

```http
GET /videos/video123/comments?limit=50
```

Flow:

```text
Client
   ↓
API Gateway
   ↓
Comment Service
   ↓
Cache
```

If cache hit:

```text
Cache
   ↓
Latest 50 comments
   ↓
Client
```

If cache miss:

```text
Cache
   ↓
Cassandra
   ↓
Comment Service
   ↓
Cache
   ↓
Client
```

The UI renders these comments.

---

# 15. Step 3: Open SSE Connection

The client also opens:

```http
GET /videos/video123/comments/stream
Accept: text/event-stream
```

Flow:

```text
Client
   ↓
API Gateway
   ↓
Realtime Comment Service
```

The connection remains open.

```text
Client ←──────────────→ Realtime Service
          SSE
```

---

# 16. Step 4: Register the Connection

The Realtime Service records:

```text
video123 → connection1
```

If more users join:

```text
video123 → [connection1, connection2, connection3]
```

Example:

```json
{
  "video123": [
    "conn1",
    "conn4",
    "conn7"
  ],

  "video456": [
    "conn2",
    "conn3"
  ]
}
```

This means:

```text
video123
   ↓
Users currently watching video123
```

---

# 17. Step 5: Another User Posts a Comment

User B posts:

```text
"This video is awesome!"
```

Client sends:

```http
POST /videos/video123/comments
```

Flow:

```text
Client
   ↓
API Gateway
   ↓
Comment Service
```

---

# 18. Step 6: Comment is Persisted

Comment Service stores:

```text
commentId = c999
videoId = video123
authorId = userB
content = "This video is awesome!"
```

in Cassandra.

```text
Comment Service
       │
       ▼
   Cassandra
```

Persistence should happen before treating the comment as successfully created.

---

# 19. Step 7: Publish Event

After successful persistence:

```text
Comment Service
       │
       ▼
    Pub/Sub
```

Event:

```json
{
  "eventType": "NEW_COMMENT",
  "commentId": "c999",
  "videoId": "video123",
  "content": "This video is awesome!",
  "sequenceNumber": 1001
}
```

---

# 20. Step 8: Realtime Service Consumes Event

Realtime Service receives:

```text
NEW_COMMENT(video123)
```

It determines:

```text
Which users are watching video123?
```

Suppose:

```text
video123
   ↓
conn1
conn4
conn7
conn20
...
```

It broadcasts the event to those connections.

---

# 21. Step 9: SSE Fanout

```text
                    NEW COMMENT
                         │
                         ▼
                Realtime Service
                         │
             ┌───────────┼───────────┐
             ▼           ▼           ▼
          conn1        conn4       conn7
             │           │           │
             ▼           ▼           ▼
          User A       User C      User D
```

Each connected client receives:

```text
event: new-comment

data:
{
    "commentId": "c999",
    "content": "This video is awesome!"
}
```

The UI immediately displays it.

---

# 22. End-to-End Flow

The entire process:

```text
                    USER JOINS VIDEO
                           │
                           ▼
                    ┌─────────────┐
                    │    Client   │
                    └──────┬──────┘
                           │
              ┌────────────┴────────────┐
              │                         │
              ▼                         ▼
       Fetch old comments          Open SSE
              │                         │
              ▼                         ▼
       API Gateway               API Gateway
              │                         │
              ▼                         ▼
       Comment Service         Realtime Service
              │                         │
         Cache/Cassandra               │
              │                         │
              ▼                         ▼
       Existing comments       Connection registered
              │                         │
              ▼                         │
            Client                     │
                                        │
                                        │
                 SOMEONE POSTS COMMENT  │
                           │            │
                           ▼            │
                    Comment Service     │
                           │            │
                    ┌──────┴──────┐     │
                    ▼             ▼     │
                Cassandra       Pub/Sub │
                                  │     │
                                  ▼     │
                         Realtime Service
                                  │
                                  ▼
                         Find video123
                         connections
                                  │
                     ┌────────────┼────────────┐
                     ▼            ▼            ▼
                   User A       User B       User C
```

---

# 23. Why `hash(videoId) % N`?

Suppose we have:

```text
N = 1000 realtime instances
```

We calculate:

```text
instance = hash(videoId) % 1000
```

Example:

```text
hash(video123) % 1000 = 347
```

Therefore:

```text
video123 → Realtime Instance 347
```

All SSE connections for `video123` are routed to instance 347.

---

# 24. Why Do This?

Imagine:

```text
10,000 realtime instances
```

and:

```text
1 million active videos
```

We don't want every realtime server to know about every connection.

Instead:

```text
video123 → server347
video456 → server782
video789 → server129
```

This allows us to distribute videos across realtime servers.

---

# 25. Fanout

Suppose:

```text
video123
```

has:

```text
1,000,000 viewers
```

A single comment arrives.

The realtime service needs to perform:

```text
1 comment
    ↓
1,000,000 SSE writes
```

This is called:

```text
Fanout
```

Fanout is one of the biggest scalability challenges in live-comment systems.

---

# 26. Important Scalability Problem

A simple:

```text
hash(videoId) % N
```

approach can create a hotspot.

Suppose:

```text
video123 = 10 million viewers
```

and:

```text
hash(video123) % 1000 = server347
```

Then server347 may have:

```text
10 million SSE connections
```

while other servers are mostly idle.

Therefore, pure video-level hashing isn't always sufficient for extremely popular videos.

---

# 27. Possible Solution: Shard a Video

Instead of:

```text
video123 → server347
```

we can use:

```text
video123 + shard0 → server347
video123 + shard1 → server348
video123 + shard2 → server349
...
```

For example:

```text
hash(videoId + shardId) % N
```

Then:

```text
video123
   │
   ├── shard0 → Server 347
   ├── shard1 → Server 348
   ├── shard2 → Server 349
   └── shard3 → Server 350
```

Now 10 million viewers can be distributed.

---

# 28. Connection Mapping

Each realtime server maintains an in-memory mapping.

Example:

```text
video123:
    conn1
    conn2
    conn3

video456:
    conn4
    conn5

video789:
    conn6
```

This data does not necessarily need to be persisted.

Why?

Because:

```text
SSE connections are ephemeral.
```

If a realtime server dies:

```text
connections die
```

Clients reconnect.

---

# 29. Should Connection Mapping Be Stored in Redis?

Usually, the active SSE connection itself should remain local to the realtime server.

For example:

```text
Server 347 memory:

video123 → [conn1, conn2, conn3]
```

There is no need to put the actual socket/HTTP connection in Redis.

Redis could potentially store metadata such as:

```text
video123 → realtime-server-347
```

but not the actual network connection.

---

# 30. What Happens When a User Disconnects?

Suppose:

```text
User A
   ↓
conn1
```

disconnects.

The realtime server receives the disconnect.

It removes:

```text
conn1
```

from:

```text
video123 → [conn1, conn4, conn7]
```

Result:

```text
video123 → [conn4, conn7]
```

This prevents memory leaks.

---

# 31. What Happens When the User Changes Video?

Suppose:

```text
video123 → video456
```

Client:

```text
1. Close SSE(video123)
2. Open SSE(video456)
```

Realtime Service:

```text
Remove:
video123 → conn1

Add:
video456 → conn1
```

---

# 32. Critical Problem: Initial Comments + SSE Race Condition

This is an important interview discussion.

Suppose the client does:

```text
10:00:00 → GET comments
10:00:01 → New comment C100 arrives
10:00:02 → SSE connection established
```

The client might miss C100.

Another race:

```text
10:00:00 → SSE established
10:00:01 → C100 arrives through SSE
10:00:02 → Historical comments API returns C100
```

Now the client could display C100 twice.

---

# 33. Solution: Sequence Number / Cursor

Every comment should have a monotonically increasing ordering mechanism.

Example:

```text
comment C98 → sequence 98
comment C99 → sequence 99
comment C100 → sequence 100
```

Client remembers:

```text
lastSeenSequence = 100
```

When reconnecting:

```http
GET /comments?videoId=video123&afterSequence=100
```

The server returns:

```text
101
102
103
...
```

This allows the client to recover missed comments.

---

# 34. Better Join Flow

A robust client flow can be:

```text
1. Open SSE connection
2. Establish a cursor/last-seen position
3. Fetch historical comments
4. Deduplicate using commentId/sequenceNumber
5. Process live events
```

Or use a server-defined snapshot cursor.

Example:

```text
GET /comments?videoId=video123
```

Response:

```json
{
  "comments": [...],
  "nextCursor": 1000
}
```

Then establish/continue live delivery from:

```text
cursor = 1000
```

The exact protocol depends on the implementation, but the key requirement is:

> **The client must have a way to recover comments that arrive around connection establishment.**

---

# 35. SSE Reconnection

SSE clients can reconnect when the connection breaks.

Example:

```text
Client
   │
   │ SSE
   ▼
Realtime Server
   X
Connection lost
   │
   ▼
Client reconnects
```

The client should tell the server what it last received.

For example:

```text
Last-Event-ID: 1000
```

The server can then send:

```text
1001
1002
1003
...
```

This prevents missed events.

---

# 36. Failure Scenario: Realtime Server Crashes

Suppose:

```text
video123 → Server347
```

and Server347 crashes.

All SSE connections are lost.

Clients reconnect.

The load balancer routes them to another healthy realtime server.

For example:

```text
Server347 ❌

Client A ──┐
Client B ──┼──→ Server500
Client C ──┘
```

The client provides its last-seen sequence.

The new server can recover missed events from the durable/event system.

---

# 37. Failure Scenario: Comment Service Crashes

Suppose:

```text
Client
   ↓
Comment Service ❌
```

The API request fails.

The client can retry.

No real-time event should be published unless the comment has successfully been persisted.

---

# 38. Failure Scenario: Cassandra is Down

If Cassandra is unavailable:

```text
Comment Service
      ↓
   Cassandra ❌
```

The comment should not be considered successfully created.

Depending on requirements, we can:

* Retry
* Queue the write
* Return an error
* Use a durable messaging layer

The important principle is:

> Do not broadcast a comment that has not been durably accepted unless the product explicitly allows optimistic delivery.

---

# 39. Failure Scenario: Pub/Sub is Down

This is an important distinction.

Suppose:

```text
Comment Service
      │
      ├── Cassandra → SUCCESS
      │
      └── Pub/Sub → FAILURE
```

Now:

```text
Comment exists in Cassandra
BUT
Realtime users haven't received it
```

This creates an inconsistency between:

```text
Durable state
```

and:

```text
Real-time state
```

A common solution is to use an **outbox pattern**.

---

# 40. Outbox Pattern

Instead of:

```text
Write Cassandra
Publish Pub/Sub
```

as two independent operations, maintain a durable event/outbox record.

Conceptually:

```text
Comment
   │
   ├── comment data
   │
   └── event/outbox
```

A background publisher reads pending events:

```text
Outbox
   ↓
Pub/Sub
```

If Pub/Sub fails:

```text
event remains pending
```

and can be retried.

This prevents permanently losing the real-time event.

---

# 41. Duplicate Events

Pub/Sub systems can potentially deliver duplicate events.

Example:

```text
C100
C100
C101
```

Realtime Service should therefore be idempotent where necessary.

The client can also deduplicate using:

```text
commentId
```

or:

```text
sequenceNumber
```

For example:

```text
if sequenceNumber <= lastSeenSequence:
    ignore
```

---

# 42. Ordering

Comments need to appear in a sensible order.

For example:

```text
C100
C101
C102
```

should not become:

```text
C101
C100
C102
```

Ordering can become difficult when:

* Multiple producers exist
* Events are processed in parallel
* Multiple partitions are used
* A video is sharded

A sequence number or timestamp-based ordering strategy helps.

---

# 43. Hot Videos

The biggest scalability challenge is a viral video.

Example:

```text
Normal video:

10,000 viewers

Viral video:

20,000,000 viewers
```

If all viewers are connected to one realtime server:

```text
20 million SSE connections
```

that server becomes a bottleneck.

Therefore, popular videos may need:

```text
Video sharding
```

or:

```text
Hierarchical fanout
```

---

# 44. Hierarchical Fanout

Instead of:

```text
Pub/Sub
   ↓
One server
   ↓
10 million clients
```

use:

```text
                 Pub/Sub
                    │
          ┌─────────┼─────────┐
          ▼         ▼         ▼
       Server A  Server B  Server C
          │         │         │
       100K       100K       100K
      clients    clients    clients
```

This allows fanout work to be distributed.

---

# 45. Complete Write Path

```text
User
  │
  │ POST comment
  ▼
API Gateway
  │
  ▼
Comment Service
  │
  ├── Validate
  │
  ├── Authenticate
  │
  ├── Persist
  ▼
Cassandra
  │
  ▼
Outbox/Event
  │
  ▼
Pub/Sub
  │
  ▼
Realtime Comment Service
  │
  ▼
Find connections for video
  │
  ▼
SSE
  │
  ├── User A
  ├── User B
  ├── User C
  └── ...
```

---

# 46. Complete Read/Join Path

```text
User opens video
       │
       ├──────────────────────────┐
       │                          │
       ▼                          ▼
GET historical comments       Open SSE
       │                          │
       ▼                          ▼
API Gateway                 API Gateway
       │                          │
       ▼                          ▼
Comment Service            Realtime Service
       │                          │
       ▼                          ▼
Cache/Cassandra             Register connection
       │                          │
       ▼                          │
Initial comments                   │
       │                          │
       └────────────┬─────────────┘
                    ▼
               User sees comments
                    │
                    ▼
             Wait for new events
                    │
                    ▼
                SSE event
                    │
                    ▼
              Update UI
```

---

# 47. Responsibilities Summary

| Component        | Responsibility                               |
| ---------------- | -------------------------------------------- |
| Client           | UI, SSE connection, reconnect, deduplication |
| API Gateway      | Authentication, routing, rate limiting       |
| Comment Service  | Comment CRUD/business logic                  |
| Cache            | Fast access to recent comments               |
| Cassandra        | Durable comment storage                      |
| Pub/Sub          | Asynchronous event distribution              |
| Realtime Service | SSE connections and fanout                   |
| Load Balancer    | Distribute realtime connections              |
| Outbox           | Reliable event publishing                    |

---

# 48. Important Design Principles

### Principle 1: Separate persistence from real-time delivery

```text
Comment Service
      ↓
Persistence

Realtime Service
      ↓
Fanout
```

---

### Principle 2: Don't store SSE connections in Cassandra

Connections are ephemeral.

Keep them in:

```text
Realtime server memory
```

---

### Principle 3: Don't make Comment Service manage SSE connections

Otherwise:

```text
Comment Service
      ↓
millions of connections
```

creates tight coupling and scaling problems.

---

### Principle 4: Use Pub/Sub as the bridge

```text
Comment Service
      ↓
Pub/Sub
      ↓
Realtime Service
```

---

### Principle 5: Have a recovery mechanism

Real-time systems must assume:

```text
Connections break
Servers crash
Events duplicate
Events can be delayed
```

Therefore use:

```text
sequence number / cursor / Last-Event-ID
```

---

# 49. Interview Explanation

A concise explanation you can give to the interviewer:

> "When a user opens a video, we have two flows. First, the client fetches historical comments from the Comment Service, which reads from cache and falls back to Cassandra. Second, the client establishes a long-lived SSE connection through the API Gateway to the Realtime Comment Service.
>
> When a user creates a comment, the Comment Service validates and persists it in Cassandra and publishes a NewComment event through Pub/Sub. The Realtime Comment Service consumes that event and identifies all SSE connections currently watching that video. It then fans out the event to those connections.
>
> We keep the SSE connection state in memory on the realtime servers because those connections are ephemeral. We can route a video to a realtime instance using video hashing, although extremely popular videos may need to be sharded across multiple realtime instances.
>
> Finally, to handle connection failures and the race condition between historical comments and live events, every comment should have a sequence number or cursor. The client can reconnect using the last-seen sequence and recover any missed comments."

---

# 50. Key Interview Questions to Be Ready For

### Q1. Why SSE?

Because comments primarily require:

```text
Server → Client
```

real-time communication.

---

### Q2. Why not WebSocket?

WebSocket is useful for bidirectional communication. For one-way server push, SSE is simpler.

---

### Q3. Why Pub/Sub?

To decouple:

```text
Comment persistence
```

from:

```text
Real-time fanout
```

---

### Q4. Where do you store active connections?

Locally in memory on the Realtime Service.

---

### Q5. What happens if realtime server crashes?

Clients reconnect and use their last-seen sequence/cursor to recover missed events.

---

### Q6. What if Pub/Sub fails after Cassandra succeeds?

Use an Outbox Pattern so the event is durably recorded and retried.

---

### Q7. What if the same comment event arrives twice?

Use:

```text
commentId
```

or:

```text
sequenceNumber
```

for idempotency/deduplication.

---

### Q8. What if one video has 10 million viewers?

Don't route the entire video to one realtime server.

Use:

```text
video sharding
```

and distribute connections/fanout across multiple realtime servers.

---

### Q9. How do you prevent missing comments during reconnect?

Use:

```text
sequenceNumber
cursor
Last-Event-ID
```

and replay missed comments.

---

### Q10. How do you prevent duplicate comments?

Client/server deduplication using:

```text
commentId
```

or:

```text
sequenceNumber
```

---

# 51. Final Mental Model

Remember the system as **three separate paths**:

## Historical Comments

```text
Client
  ↓
API Gateway
  ↓
Comment Service
  ↓
Cache
  ↓
Cassandra
  ↓
Client
```

## Create Comment

```text
Client
  ↓
API Gateway
  ↓
Comment Service
  ↓
Cassandra
  ↓
Pub/Sub
```

## Real-Time Delivery

```text
Pub/Sub
   ↓
Realtime Comment Service
   ↓
Video → Connections
   ↓
SSE
   ↓
All users watching that video
```

The most important architectural separation is:

```text
                  ┌─────────────────────┐
                  │   Comment Service   │
                  │                     │
                  │ Persistence         │
                  │ Business Logic      │
                  │ Historical Reads    │
                  └──────────┬──────────┘
                             │
                             │ Event
                             ▼
                       ┌────────────┐
                       │  Pub/Sub   │
                       └─────┬──────┘
                             │
                             ▼
                  ┌─────────────────────┐
                  │ Realtime Service    │
                  │                     │
                  │ SSE Connections     │
                  │ Connection Mapping  │
                  │ Fanout               │
                  └──────────┬──────────┘
                             │
                             ▼
                         Clients
```

**Comment Service owns the comment.
Pub/Sub distributes the event.
Realtime Service owns the live connections.
SSE delivers the comment to users.**
