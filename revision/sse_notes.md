# Server-Sent Events (SSE) - Notes

## 🔹 What is SSE?

Server-Sent Events (SSE) is a technique where a server can push real-time updates to the client over a single long-lived HTTP/HTTPS connection.

- Communication is **one-way**: Server → Client
- Uses standard HTTP protocol
- Ideal for real-time updates

---

## 🔹 How SSE Works

1. Client sends an HTTP request to the server
2. Server responds with `Content-Type: text/event-stream`
3. Server **keeps the connection open**
4. Server continuously sends data in small chunks
5. Client listens and processes incoming events

---

## 🔹 Key Concept

SSE is a **streaming HTTP response** that is never closed.

---

## 🔹 Why Connection Stays Open?

- HTTP runs over TCP (which supports persistent connections)
- Server does NOT call `res.end()`
- Data is sent using **chunked transfer encoding**
- Connection remains active as long as:
  - Server keeps sending data
  - Or connection is not explicitly closed

---

## 🔹 Important Headers

Content-Type: text/event-stream  
Cache-Control: no-cache  
Connection: keep-alive  

---

## 🔹 Preventing Timeout

To avoid connection timeout:
- Server sends periodic "heartbeat" messages

Example:
: keep-alive

(This is a comment line, ignored by client but keeps connection alive)

---

## 🔹 Client-Side Example

```javascript
const eventSource = new EventSource('/events');

eventSource.onmessage = function(event) {
  console.log('Received:', event.data);
};
```

---

## 🔹 Server-Side Example (Node.js)

```javascript
app.get('/events', (req, res) => {
  res.setHeader('Content-Type', 'text/event-stream');
  res.setHeader('Cache-Control', 'no-cache');
  res.setHeader('Connection', 'keep-alive');

  setInterval(() => {
    res.write(`data: ${JSON.stringify({ time: new Date() })}\n\n`);
  }, 2000);
});
```

---

## 🔹 Closing the Connection

### From Client
```javascript
eventSource.close();
```

### From Server
```javascript
res.end();
```

---

## 🔹 What Happens If Not Closed?

- Connection remains open indefinitely
- Server resources stay occupied
- Browser may auto-reconnect if connection drops

---

## 🔹 Auto-Reconnection Feature

- SSE automatically reconnects if connection breaks
- Retry interval can be controlled:

retry: 5000

---

## 🔹 Use Cases

- Live notifications
- Stock price updates
- Dashboards
- News feeds
- Logs streaming

---

## 🔹 SSE vs Polling vs WebSockets

| Feature        | SSE                | Polling           | WebSockets        |
|----------------|------------------|------------------|------------------|
| Communication  | One-way          | Request/Response | Two-way          |
| Connection     | Long-lived HTTP  | Multiple requests| Persistent       |
| Real-time      | Yes              | Delayed          | Yes              |
| Complexity     | Easy             | Simple           | Complex          |

---

## 🔹 Advantages

- Simple to implement
- Works over HTTP/HTTPS
- Automatic reconnection
- Lightweight

---

## 🔹 Limitations

- Only server → client communication
- Not suitable for bidirectional apps
- Limited browser connections per domain

---

## 🔹 Interview Insight 🔥

SSE is essentially **HTTP streaming over a persistent TCP connection**

---

## 🔹 Summary

- SSE keeps connection open by not closing the response
- Uses chunked streaming over HTTP
- Requires explicit closing when done
- Supports automatic reconnection
- Best for simple real-time updates
