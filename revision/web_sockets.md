# WebSockets, SSE, and Polling -- Detailed Notes

## 1. What is WebSocket?

WebSocket is a protocol that enables full-duplex, persistent
communication between a client and server over a single TCP connection.

------------------------------------------------------------------------

## 2. Why WebSockets?

Used for real-time applications like: - Chat apps - Live notifications -
Stock updates - Multiplayer games

------------------------------------------------------------------------

## 3. How WebSocket Works

### Step 1: HTTP Handshake

Client sends an HTTP request with Upgrade header.

### Step 2: Protocol Upgrade

Server responds with 101 Switching Protocols.

### Step 3: Persistent Connection

Connection remains open.

### Step 4: Full-Duplex Communication

Both client and server can send messages anytime.

### Step 5: Close Connection

Either side can close connection.

------------------------------------------------------------------------

## 4. WebSocket Features

-   Persistent connection
-   Full-duplex communication
-   Low latency
-   Efficient (less overhead)

------------------------------------------------------------------------

## 5. WebSocket vs HTTP

  Feature         HTTP               WebSocket
  --------------- ------------------ -----------
  Type            Stateless          Stateful
  Communication   Request-Response   Two-way
  Overhead        High               Low
  Use Case        CRUD               Real-time

------------------------------------------------------------------------

## 6. Why NOT use WebSockets for CRUD?

### 1. Stateful Nature

Each connection consumes memory.

### 2. Scaling Issues

Requires sticky sessions and additional tools like Redis.

### 3. Overkill for CRUD

CRUD is simple request-response.

### 4. No Built-in Semantics

Need to manually handle routing, errors, etc.

### 5. No Caching

HTTP supports caching, WebSocket does not.

### 6. Debugging Complexity

Harder than HTTP.

### 7. Connection Management

Need ping/pong, reconnection handling.

------------------------------------------------------------------------

## 7. When to Use WebSockets

-   Real-time systems
-   Continuous data flow
-   Notifications

------------------------------------------------------------------------

## 8. When NOT to Use WebSockets

-   CRUD APIs
-   Form submissions
-   Simple data fetching

------------------------------------------------------------------------

## 9. Best Practice

Use both: - HTTP for CRUD - WebSocket for real-time updates

------------------------------------------------------------------------

## 10. Mental Model

-   HTTP = Email
-   WebSocket = Phone Call

------------------------------------------------------------------------

## 11. Interview Summary

WebSockets are not used for CRUD because they are stateful, harder to
scale, and lack built-in request-response semantics.


# WebSocket Communication Notes (How Data is Sent)

## 1. Key Difference from HTTP

### HTTP Communication

-   Uses predefined methods:
    -   GET → Fetch data
    -   POST → Create data
    -   PUT → Update data
    -   DELETE → Remove data
-   Based on URL + method

------------------------------------------------------------------------

### WebSocket Communication

-   No methods like GET/POST
-   Only **messages are exchanged**
-   Communication is **event/message-based**

------------------------------------------------------------------------

## 2. Core Idea

After connection is established:

Client ⇄ Server exchange messages continuously

------------------------------------------------------------------------

## 3. How Communication Happens

We define our own protocol using JSON messages.

### Example Request (Client → Server)

``` json
{
  "type": "CREATE_USER",
  "payload": {
    "name": "Suraj",
    "email": "suraj@test.com"
  }
}
```

------------------------------------------------------------------------

### Server Handling Logic

``` javascript
ws.on("message", (msg) => {
  const data = JSON.parse(msg);

  switch (data.type) {
    case "CREATE_USER":
      // perform create operation
      break;
  }
});
```

------------------------------------------------------------------------

### Example Response (Server → Client)

``` json
{
  "type": "USER_CREATED",
  "payload": {
    "id": 1,
    "name": "Suraj"
  }
}
```

------------------------------------------------------------------------

## 4. Mapping HTTP to WebSocket

  HTTP API          WebSocket Message
  ----------------- ---------------------------
  GET /users        { "type": "GET_USERS" }
  POST /users       { "type": "CREATE_USER" }
  PUT /users/1      { "type": "UPDATE_USER" }
  DELETE /users/1   { "type": "DELETE_USER" }

------------------------------------------------------------------------

## 5. Standard Message Structure

Most production systems follow:

``` json
{
  "type": "ACTION_NAME",
  "payload": {},
  "requestId": "unique-id"
}
```

------------------------------------------------------------------------

## 6. Why requestId?

-   WebSockets are asynchronous
-   Multiple requests can be active simultaneously
-   requestId helps match response with request

### Example Response with requestId

``` json
{
  "type": "RESPONSE",
  "requestId": "123",
  "data": {}
}
```

------------------------------------------------------------------------

## 7. Communication Patterns

### 1. Action-Based Pattern

``` json
{
  "type": "CREATE_ORDER",
  "payload": {}
}
```

------------------------------------------------------------------------

### 2. Event-Based Pattern

``` json
{
  "event": "userJoined",
  "data": { "userId": 1 }
}
```

------------------------------------------------------------------------

## 8. Libraries That Simplify This

-   Socket.IO
-   SignalR (.NET)

They provide: - Event-based communication - Auto reconnection -
Rooms/channels - Cleaner APIs

### Example (Socket.IO)

``` javascript
socket.emit("createUser", { name: "Suraj" });

socket.on("userCreated", (data) => {
  console.log(data);
});
```

------------------------------------------------------------------------

## 9. Important Concept

-   WebSocket = Transport layer
-   You define the **application protocol**

------------------------------------------------------------------------

## 10. Mental Model

-   HTTP → Predefined structured communication
-   WebSocket → Flexible messaging system (you design structure)

------------------------------------------------------------------------

## 11. Interview Summary

WebSocket communication works by sending structured messages (usually
JSON) where action types replace HTTP methods, and the application
defines its own protocol for handling requests and responses.

# WebSocket Scaling (Billions of Users) -- Detailed Notes

## 1. Problem Statement

Can a single server handle billions of WebSocket connections?

👉 Answer: NO (Not feasible)

------------------------------------------------------------------------

## 2. Why Single Server Fails

### 1. Connection Limits

-   Each user = 1 TCP connection
-   1 billion users = 1 billion connections
-   OS file descriptor limits prevent this

------------------------------------------------------------------------

### 2. Memory Constraints

-   Each connection consumes memory (buffers, metadata)
-   Even 10KB per connection:

1B users × 10KB = 10 TB RAM ❌

------------------------------------------------------------------------

### 3. CPU & Network Bottleneck

-   Handling messages
-   Managing heartbeats (ping/pong)
-   TLS encryption (wss)

👉 Single server will crash

------------------------------------------------------------------------

## 3. Real-World Solution: Distributed Architecture

Instead of 1 server → use multiple servers

------------------------------------------------------------------------

## 4. High-Level Architecture

### 1. Load Balancer

-   Distributes incoming connections across servers

------------------------------------------------------------------------

### 2. WebSocket Gateway Servers

-   Maintain persistent connections
-   Handle send/receive messages
-   Each server handles \~100K--1M connections

------------------------------------------------------------------------

### 3. Message Broker

Used for communication between servers:

-   Kafka
-   Redis (Pub/Sub)
-   RabbitMQ

------------------------------------------------------------------------

## 5. Message Flow Example

User A → User B

1.  A sends message to Server 1
2.  Server 1 pushes message to broker
3.  Broker identifies B's server (Server 5)
4.  Broker forwards message to Server 5
5.  Server 5 sends message to B

------------------------------------------------------------------------

## 6. Visual Flow

User A → Server 1 → Broker → Server 5 → User B

------------------------------------------------------------------------

## 7. Key Concepts

### 1. Horizontal Scaling

-   Add more servers instead of scaling vertically

------------------------------------------------------------------------

### 2. Sticky Sessions

-   User stays connected to the same server

------------------------------------------------------------------------

### 3. User-to-Server Mapping

-   Mapping: userId → serverId
-   Stored in Redis for fast lookup

------------------------------------------------------------------------

### 4. Heartbeats (Ping/Pong)

-   Detect dead connections
-   Clean up inactive users

------------------------------------------------------------------------

### 5. Sharding

-   Divide users by:
    -   Region (India, US, etc.)
    -   Hash-based distribution

------------------------------------------------------------------------

## 8. Typical Tech Stack

-   Load Balancer → Nginx / AWS ELB
-   WebSocket Servers → Node.js / Go / Erlang
-   Broker → Kafka / Redis
-   Database → MySQL / Cassandra

------------------------------------------------------------------------

## 9. Offline Handling

-   Messages stored in DB
-   Delivered when user reconnects

------------------------------------------------------------------------

## 10. Key Insight

👉 WebSockets scale horizontally, NOT vertically

------------------------------------------------------------------------

## 11. Interview Summary

A single server cannot handle billions of WebSocket connections due to
limits in memory, CPU, and OS constraints. Real-world systems use
horizontally scaled WebSocket servers, load balancers, and message
brokers to efficiently distribute and route messages.
