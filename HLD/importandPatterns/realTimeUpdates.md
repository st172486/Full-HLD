# Real-Time Update Patterns — Quick Revision

## 1. Core Idea

- Real-time system = server needs to **push updates** to clients quickly.
- Avoid frequent polling when updates must arrive within milliseconds.
- Think in **2 hops**:
  1. **Source → Server:** how does the server know an update happened?
  2. **Server → Client:** how does the client receive the update?

## 2. Networking Basics

- **L3 — IP:** routing/addressing; packets can be lost, duplicated, or reordered.
- **L4 — TCP:** connection-oriented, reliable + ordered delivery.
- **L4 — UDP:** connectionless, no delivery/order guarantee.
- **L7:** application protocols such as HTTP, WebSocket, WebRTC.

### TCP Interview Points

- TCP connection requires handshake: **SYN → SYN-ACK → ACK**.
- Persistent connections maintain **state** on client/server.
- Extra round trips add latency.

## 3. Load Balancers

### L4 Load Balancer

- Works at TCP/UDP level.
- Minimal packet inspection → fast.
- Persistent TCP connection stays associated with the same backend.
- Good fit for **WebSockets**.

### L7 Load Balancer

- Understands HTTP/application data.
- Can route using URL, headers, cookies, etc.
- More flexible but more processing.
- Common fit for **HTTP-based solutions / long polling**.
- Some L7 LBs support WebSockets too.

## 4. Simple Polling

**Flow:** Client → request → Server → response → wait → repeat.

### Pros
- Simplest approach.
- Stateless.
- No special infrastructure.
- Works with standard HTTP.

### Cons
- Latency depends on polling interval.
- Wastes requests when nothing changes.
- More bandwidth/server overhead at scale.

### Use When
- True real-time is **not required**.
- Updates every few seconds are acceptable.
- Short-lived/simple update requirements.

> Interview: Start with polling if the product requirements allow it; don't over-engineer.

## 5. Long Polling

**Flow:**
1. Client sends HTTP request.
2. Server **holds request open** until update is available.
3. Server responds.
4. Client immediately sends another request.
5. Repeat.

### Pros
- Near real-time.
- Uses standard HTTP.
- Easy to implement.
- No special infrastructure.
- Server can remain stateless.

### Cons
- More latency than persistent push connections.
- HTTP request/response overhead.
- Long-lived requests consume resources.
- Monitoring becomes harder.
- Browser connection limits can matter.
- Not ideal for **high-frequency updates**.

### Use When
- Near real-time is needed.
- Updates are relatively infrequent.
- Want simplicity and HTTP compatibility.
- Good for waiting on async operations, e.g. payment status.

## 6. Quick Comparison

| Pattern | Latency | Complexity | Best For |
|---|---|---|---|
| Polling | High/interval-based | Low | Non-real-time updates |
| Long Polling | Low–medium | Low | Near real-time, infrequent updates |
| WebSocket | Very low | Medium/high | Bidirectional real-time |
| SSE | Very low | Medium | Server → client streaming |

## 7. Interview Decision Rule

- **Few-second delay okay?** → Polling
- **Near real-time + simple HTTP?** → Long Polling
- **Continuous server push / bidirectional communication?** → WebSocket
- **Server → client only?** → SSE


# Server-Sent Events (SSE)

## 1. Core Idea

* SSE = **server → client streaming** over HTTP.
* Server keeps one HTTP connection open and continuously sends updates as chunks.
* Think of it as an upgrade over **Long Polling**:

  * Long Polling → request → response → new request
  * SSE → **one long-lived request → continuous stream**

## 2. How SSE Works

1. Client establishes SSE connection.
2. Server keeps connection open.
3. When an update occurs, server sends an event/chunk.
4. Client receives it immediately.
5. Connection remains open for more events.

```text
Client ──────── HTTP Request ────────> Server
       <────── Event 1 ───────────────
       <────── Event 2 ───────────────
       <────── Event 3 ───────────────
       <────── Event 4 ───────────────
                 ...
```

## 3. Important Headers

```text
Content-Type: text/event-stream
Cache-Control: no-cache
Connection: keep-alive
```

* Response is streamed using **chunked transfer encoding**.
* Data is sent incrementally instead of waiting for one complete response.

## 4. Client

* Browsers provide built-in `EventSource`.
* Handles connection + reconnection automatically.

```javascript
const eventSource = new EventSource('/api/updates');

eventSource.onmessage = (event) => {
    const data = JSON.parse(event.data);
    updateUI(data);
};
```

## 5. Advantages

* **One-way server → client communication**
* Built into modern browsers.
* Automatic reconnection.
* Works over HTTP.
* Less overhead than Long Polling.
* No polling interval required.
* Simple to implement.

## 6. Disadvantages

* **One-way only** → client cannot use same connection to send updates.
* Streaming support can be problematic with some **proxies/load balancers**.
* Intermediate infrastructure may **buffer** the response → breaks real-time streaming.
* Long-lived connections make monitoring harder.
* Browser connection limits per domain can matter.

## 7. When to Use

Use SSE when:

* Updates primarily flow **Server → Client**.
* Client doesn't need frequent bidirectional communication.
* Need continuous/low-latency updates.

### Common Examples

* AI chat → stream generated tokens.
* Live notifications.
* Live feeds.
* Stock/price updates.
* Live comments.

## 8. SSE Reconnection

* SSE supports a **Last Event ID**.
* Client remembers the last event received.
* If connection drops:

  1. Client reconnects.
  2. Sends/identifies the last received event ID.
  3. Server can send missed events.
* Important for handling gaps during reconnection.

## 9. SSE vs Long Polling

|                 | Long Polling       | SSE                    |
| --------------- | ------------------ | ---------------------- |
| Connection      | Repeated requests  | One persistent request |
| Server → Client | Yes                | Yes                    |
| Streaming       | No                 | Yes                    |
| Latency         | Low–medium         | Very low               |
| HTTP            | Yes                | Yes                    |
| Reconnection    | Application logic  | Built-in               |
| Best for        | Infrequent updates | Continuous server push |

### Interview One-Liner

> **"SSE is a persistent HTTP connection where the server continuously streams events to the client. It's ideal when communication is primarily server → client, such as live notifications, feeds, or AI token streaming."**


# WebSockets — Full-Duplex Communication

## 1. Core Idea

* WebSocket = **persistent, bidirectional connection**.
* Both **client ↔ server** can send messages at any time.
* Best when you have **high-frequency reads + writes**.

```text
Client <════════ WebSocket ════════> Server
        messages in both directions
```

## 2. How WebSocket Works

1. Client starts with an **HTTP handshake**.
2. Connection is upgraded to **WebSocket**.
3. TCP connection stays open.
4. Client and server can both send messages.
5. Connection closes explicitly or due to failure.

* Can reuse HTTP information such as **cookies/headers** during the upgrade.
* Messages can contain JSON, strings, Protobuf, binary data, etc.

## 3. Why WebSocket?

Unlike SSE:

```text
SSE:
Client ───────────────> Server
       <─────────────── Server updates

WebSocket:
Client ═══════════════ Server
       <── messages ──>
```

* **SSE:** server → client
* **WebSocket:** client ↔ server

## 4. Advantages

* **Full-duplex** communication.
* Very low latency.
* Efficient for frequent messages.
* Less HTTP overhead after connection establishment.
* Widely supported by browsers.

## 5. Disadvantages

* More complex than SSE/Long Polling.
* Requires WebSocket-aware infrastructure.
* **Stateful persistent connections** make scaling harder.
* Load balancing becomes more complicated.
* Must handle reconnection.
* Deployments/server restarts can terminate connections.

## 6. Load Balancer Considerations

### L4 LB

* Works at TCP level.
* Naturally supports persistent WebSocket connections.
* Same TCP connection remains associated with the backend.

### L7 LB

* Some support WebSockets, but support/configuration can vary.
* Because WebSockets depend on a persistent connection, infrastructure must explicitly support the upgrade.

> **Interview:** L4 is a natural fit for WebSocket because it operates directly on the TCP connection.

## 7. Scaling Challenges

### Stateful Connections

* Each WebSocket connection is tied to a particular server.
* Can't freely move an existing connection between servers.
* Uneven connections can create **hotspots**.

### Common Solutions

* Use **least-connections** load balancing.
* Keep WebSocket server lightweight.
* Offload heavy processing to other services.
* Use a dedicated **WebSocket service** for connection management.

```text
                 ┌── Service A
Client → L4 LB → ├── WebSocket Service
                 └── Service B
                       │
                       ↓
                Backend Services
```

### Why Dedicated WebSocket Service?

* Handles connection lifecycle.
* Manages scaling/reconnections.
* Keeps the rest of the architecture **stateless**.
* Changes/deployments are less frequent → fewer connection disruptions.

## 8. Deployment Problem

WebSocket connections are long-lived.

During deployment:

```text
Old Server
   ↓
Existing WebSocket connections
   ↓
Deployment / Restart
   ↓
Connections dropped
   ↓
Clients reconnect
```

* Simpler strategy: **terminate old connections and let clients reconnect**.
* Must handle:

  * Reconnection
  * Missed updates
  * Connection recovery

## 9. When to Use

Use WebSockets when you need:

* **High-frequency**
* **Bidirectional**
* **Low-latency**

Examples:

* Chat
* Multiplayer games
* Collaborative editing
* Real-time interactive applications

### Don't Use WebSocket Automatically

Prefer simpler solutions when possible:

* Updates only → **SSE**
* Occasional updates → **Polling / Long Polling**
* Frequent server → client + writes are occasional → **SSE + HTTP POST/PUT**

## 10. Interview Decision Rule

```text
Need real-time?
      │
      ├── No → Polling
      │
      └── Yes
           │
           ├── Server → Client only → SSE
           │
           └── Client ↔ Server
                    │
                    ├── Low frequency → SSE + HTTP writes
                    │
                    └── High frequency → WebSocket
```

### Interview One-Liner

> **"WebSockets provide a persistent full-duplex connection, making them ideal for high-frequency bidirectional communication, but they introduce stateful connection management, scaling, load-balancing, and reconnection complexity."**

# WebRTC — Peer-to-Peer Communication

## 1. Core Idea

* WebRTC enables **direct peer-to-peer communication** between clients.
* Best suited for:

  * Audio/video calls
  * Screen sharing
  * P2P data sharing
  * Some collaborative applications
* Can reduce server bandwidth because **data flows directly between peers**.

```text
              Signaling Server
              /             \
          Client A         Client B
             \               /
              ═══ P2P ═════
```

## 2. How WebRTC Works

1. Peers discover each other through a **Signaling Server**.
2. Exchange connection information / **ICE candidates**.
3. Try to establish a direct P2P connection.
4. Use **STUN** for NAT traversal.
5. Use **TURN** as a relay if direct connection fails.
6. Peers exchange audio/video/data directly.

## 3. Signaling Server

* Helps peers **discover each other**.
* Exchanges:

  * Connection information
  * Offers/answers
  * ICE candidates
* Usually lightweight because it doesn't carry the main audio/video/data traffic.
* Signaling itself needs a real-time mechanism such as:

  * WebSocket
  * SSE
  * Long Polling

> **Important:** WebRTC does NOT define the signaling mechanism; you need to design/provide one.

## 4. NAT Problem

Most clients are behind **NAT/firewalls**, so they aren't directly reachable from the internet.

WebRTC uses:

### STUN

* Helps client discover its **public IP/port**.
* Enables direct P2P connection when possible.
* Uses techniques such as NAT hole punching.

### TURN

* Acts as a **relay server**.
* Used when direct P2P connection cannot be established.
* Traffic flows through TURN instead of directly between peers.

```text
Direct:
Client A ═══════════════ Client B

Fallback:
Client A ───> TURN Server ───> Client B
```

## 5. Advantages

* Direct peer-to-peer communication.
* Very low latency.
* Reduces server bandwidth/load.
* Native support for audio/video.
* Good for real-time media.

## 6. Disadvantages

* More complex than WebSockets.
* Requires signaling infrastructure.
* NAT/firewall traversal complexity.
* TURN servers may be required.
* Connection establishment takes time.
* P2P synchronization can become complicated.

## 7. When to Use

Use WebRTC when:

* **Audio/video communication** is required.
* Screen sharing is required.
* Clients need frequent P2P communication.
* Server bandwidth needs to be reduced at large scale.

### Examples

* Video conferencing
* Voice calls
* Screen sharing
* Multiplayer/gaming scenarios
* Some collaborative editors

## 8. WebRTC vs WebSocket

|               | WebSocket                   | WebRTC           |
| ------------- | --------------------------- | ---------------- |
| Communication | Client ↔ Server             | Peer ↔ Peer      |
| Main Server   | Carries traffic             | Mostly signaling |
| Audio/Video   | Not specialized             | **Native**       |
| NAT Traversal | Not main concern            | STUN/TURN        |
| Complexity    | High                        | Very high        |
| Best For      | Real-time app communication | P2P media/data   |

## 9. Interview Decision Rule

```text
Need real-time updates?
        │
        ├── Not latency sensitive
        │       → Polling
        │
        └── Latency sensitive
                │
                ├── Server → Client
                │       → SSE
                │
                └── Bidirectional
                        │
                        ├── Client ↔ Server
                        │       → WebSocket
                        │
                        └── Peer ↔ Peer
                                │
                                ├── Audio / Video
                                │       → WebRTC
                                │
                                └── P2P collaboration
                                        → Consider WebRTC
```

### Final Cheat Sheet

| Requirement                       | Choice                 |
| --------------------------------- | ---------------------- |
| Simple / low-frequency updates    | **Polling**            |
| Near real-time, server → client   | **Long Polling / SSE** |
| Continuous server → client stream | **SSE**                |
| High-frequency client ↔ server    | **WebSocket**          |
| P2P audio/video                   | **WebRTC**             |
| P2P data collaboration            | **WebRTC**             |

### Interview One-Liner

> **"WebRTC is a peer-to-peer communication technology, primarily used for real-time audio/video and sometimes P2P data sharing. A signaling server helps peers discover and connect, while STUN handles NAT traversal and TURN provides a relay fallback."**

# Server-Side Push / Pull — Quick Revision

## 1. Core Idea

Once we know **how server → client works**, we need to answer:

> **How does the server know that an update happened?**

There are 3 common patterns:

1. **Pulling → Polling**
2. **Pushing → Consistent Hashing**
3. **Pushing → Pub/Sub**

---

## 2. Pulling with Polling

* Server/client periodically checks the **DB** for new updates.
* Example:

  * `GET messages WHERE timestamp > lastReceivedTimestamp`
* DB acts as the source of truth.

### Pros

* Very simple.
* Stateless application servers.
* No special infrastructure.

### Cons

* Higher latency.
* Unnecessary DB reads when there are no updates.
* Can create huge read load at scale.

> **Example:** 1M clients polling every 10 sec = **100K reads/sec**.

### Use When

* Real-time is not critical.
* Polling delay is acceptable.

---

# 3. Pushing via Consistent Hashing

### Problem

With SSE/WebSocket, a client has a persistent connection to **one server**.

If User C is connected to Server 2:

```text
Update → ??? → Server 2 → User C
```

The system needs to know:

> **Which server owns User C's connection?**

### Basic Idea

* Hash user/entity ID → determine responsible server.
* That server maintains the user's connection.

```text
hash(userId) → Server
```

### Simple Modulo Hashing

```text
server = hash(userId) % N
```

* `N` = number of servers.
* A coordination service such as **ZooKeeper/etcd** can maintain server information.

### Message Flow

```text
Update Server
      ↓
hash(userId)
      ↓
Responsible Server
      ↓
User's WebSocket/SSE connection
      ↓
Client
```

### Problem with Modulo Hashing

If `N` changes:

```text
hash(userId) % N
```

many users get a **different server**.

→ connections need to move/reconnect
→ huge connection churn

---

## 4. Consistent Hashing

* Put both **servers and users on a hash ring**.
* User belongs to the next server clockwise on the ring.
* Adding/removing a server moves only a **small portion of users**.

### Advantages

* Predictable server assignment.
* Minimal connection movement during scaling.
* Good for stateful connections.
* Easier dynamic scaling.

### Disadvantages

* More complex.
* Requires coordination/routing metadata.
* Servers need routing information.
* Server failure can lose connection state.

### Use When

Use consistent hashing when:

* Connections are **persistent/stateful**.
* Server maintains significant state for that connection.
* Scaling servers should cause minimal connection movement.

> **Example:** Google Docs-style collaboration where document state is expensive to rebuild.

---

## 5. Consistent Hashing vs Pub/Sub

### Use Consistent Hashing when:

```text
Connection has expensive state
            ↓
Keep user/document on
a predictable server
```

### Use Pub/Sub when:

```text
Connection server is lightweight
            ↓
State lives in Pub/Sub
            ↓
Servers can be interchangeable
```

> **Key interview insight:** If endpoint servers only need to forward small messages, **Pub/Sub is generally simpler** than maintaining connection ownership through consistent hashing.

---

## 6. Scaling Consistent Hashing

During scaling:

1. Record old + new server assignments.
2. Gradually disconnect clients from old servers.
3. Clients reconnect to new servers.
4. Update coordination metadata.
5. During transition, messages may need to reach **both old and new servers**.

### Interview Focus

Be ready to explain:

* How servers discover each other.
* How user → server mapping works.
* What happens when servers are added/removed.
* How existing connections migrate.
* What happens when a server fails.

# Pushing via Pub/Sub — Quick Revision

## 1. Core Idea

* **Pub/Sub = decouple update producers from connected clients.**
* A central Pub/Sub system collects/forwards updates to interested endpoint servers.
* Common choices: **Redis Pub/Sub, Kafka**.
* Endpoint servers maintain client connections and simply **forward messages**.

```text id="8n7v2k"
Update Source
     ↓
  Pub/Sub
     ↓
Endpoint Servers
     ↓
WebSocket / SSE
     ↓
  Clients
```

---

## 2. Connection Flow

When a client connects:

1. Client connects to **any endpoint server**.
2. Endpoint server registers/subscribes the client to the relevant topic.
3. Endpoint server maintains:

   ```text id="r0g3q8"
   topic → client connection(s)
   ```

### Important

Unlike consistent hashing:

* Client does **NOT** need to connect to a specific server.
* Any endpoint server can handle the connection.
* This makes load balancing much easier.

---

## 3. Sending an Update

```text id="p3x7yk"
Update Server
      ↓
Publish to topic
      ↓
    Pub/Sub
      ↓
Subscribed Endpoint Servers
      ↓
Existing WebSocket/SSE connection
      ↓
    Client(s)
```

Example:

```text id="2i5z7a"
User C → Topic: user:C

New message for User C
        ↓
Publish → user:C
        ↓
Pub/Sub
        ↓
Endpoint Server(s)
        ↓
User C connection
```

---

## 4. Why Pub/Sub Helps

### Without Pub/Sub

```text id="3n8b6d"
Update Server
      ↓
Which server has User C?
      ↓
Find connection
      ↓
Send message
```

→ Requires connection ownership/routing logic.

### With Pub/Sub

```text id="c9x4qa"
Update Server
      ↓
Publish(topic)
      ↓
Pub/Sub handles distribution
      ↓
Endpoint Server
      ↓
Client
```

→ Endpoint servers become **lightweight + interchangeable**.

---

## 5. Advantages

* Easy to scale endpoint servers.
* Use **least-connections** load balancing.
* Efficiently broadcast to many clients.
* Minimizes state on endpoint servers.
* Clients can connect to **any** endpoint server.
* Decouples update producers from consumers.

---

## 6. Disadvantages

* Pub/Sub becomes a potential **bottleneck / single point of failure**.
* Adds an extra hop → some latency.
* Pub/Sub must manage many subscriptions.
* Cluster introduces **many-to-many connections** between Pub/Sub nodes and endpoint servers.
* Harder to know connection/disconnection events from the Pub/Sub layer.

---

## 7. Scaling Pub/Sub

### Redis Cluster

* Shard subscriptions/topics across multiple Redis nodes.
* Increases:

  * Subscription capacity
  * Throughput

```text id="k6y3p1"
             Redis Cluster
          ┌──────┼──────┐
          ↓      ↓      ↓
       Redis1  Redis2  Redis3
          ↑      ↑      ↑
       Endpoint Servers
```

### Challenge

* Every endpoint server may need connections to multiple/all Pub/Sub nodes.
* Creates **many-to-many connections**.
* Keep cluster size manageable / partition topics carefully.

---

## 8. Load Balancing Endpoint Servers

Use:

> **Least Connections**

Why?

* Main resource consumed by endpoint servers = persistent connections.
* Distributes clients based on current connection count.
* Avoids concentrating too many long-lived connections on one server.

---

## 9. When to Use Pub/Sub

Use Pub/Sub when:

* Many clients need updates.
* Updates need to be **broadcast** efficiently.
* Endpoint servers don't maintain much client-specific state.
* You want endpoint servers to be **stateless/lightweight**.
* You don't care heavily about connect/disconnect events.

### Great Example

**Live comments**

```text id="0v5q1a"
New Comment
    ↓
  Pub/Sub
    ↓
All subscribed realtime servers
    ↓
Connected viewers
```

---

## 10. Consistent Hashing vs Pub/Sub

|                | Consistent Hashing        | Pub/Sub                         |
| -------------- | ------------------------- | ------------------------------- |
| Connection     | Specific server           | Any server                      |
| State          | On endpoint server        | Pub/Sub                         |
| Scaling        | More complex              | Easier                          |
| Load balancing | Connection ownership      | Least connections               |
| Server failure | Connection state affected | Clients can reconnect elsewhere |
| Best for       | Stateful connections      | Lightweight forwarding          |
| Broadcast      | Less natural              | **Excellent**                   |

### Interview Decision

> **Stateful connection → Consistent Hashing**

> **Lightweight connection + many clients → Pub/Sub**

---

## 11. Complete Real-Time Architecture

Remember the **two hops**:

```text id="2d8m4s"
         HOP 1
Server ───────────────→ Client
         │
         ├── Polling
         ├── Long Polling
         ├── SSE
         ├── WebSocket
         └── WebRTC

         HOP 2
Source ───────────────→ Server
         │
         ├── Polling
         ├── Consistent Hashing
         └── Pub/Sub
```

### Interview Rule

* **Not latency-sensitive** → Polling
* **Server → Client** → SSE
* **High-frequency bidirectional** → WebSocket
* **P2P audio/video** → WebRTC
* **Stateful persistent connections** → Consistent Hashing
* **Many clients + lightweight endpoint servers** → Pub/Sub
