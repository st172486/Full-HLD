# Polling in System Design

## 🔁 What is Polling?
Polling is a technique where a client repeatedly sends requests to a server at regular intervals to check for updates.

Instead of the server pushing updates, the client keeps asking:
- "Any new data?"
- "Anything now?"

---

## 📦 How Polling Works

1. Client sends a request to the server  
2. Server responds with current data  
3. Client waits for a fixed interval (e.g., 5 seconds)  
4. Client sends the request again  
5. This process repeats continuously  

---

## 💻 Example (JavaScript)

```javascript
setInterval(async () => {
  const response = await fetch('/api/status');
  const data = await response.json();
  console.log(data);
}, 5000); // every 5 seconds
```

---

## 🔄 Types of Polling

### 1. Short Polling
- Client sends requests at fixed intervals
- Server responds immediately

**Pros:**
- Simple to implement

**Cons:**
- Many unnecessary requests
- Inefficient

---

### 2. Long Polling
- Client sends request
- Server holds the request until new data is available
- Once response is sent, client sends another request

**Pros:**
- More efficient than short polling

**Cons:**
- Still not fully real-time

---

## ⚖️ Polling vs Other Approaches

| Approach        | Description                          | Use Case              |
|----------------|--------------------------------------|-----------------------|
| Polling        | Client keeps asking server           | Simple apps           |
| Long Polling   | Server delays response               | Basic chat apps       |
| WebSockets     | Persistent connection                | Real-time apps        |

---

## 🧠 Real-life Analogy

- Polling → Refreshing a website repeatedly  
- Long Polling → Waiting for result before response  
- WebSockets → Instant push notification  

---

## ⚡ When to Use Polling

- When real-time updates are not critical  
- When system is simple  
- When backend does not support push mechanisms  

---

## 🚫 Problems with Polling

- High number of API calls  
- Increased server load  
- Delayed updates based on interval  

---

## 💡 Improvements

- Use Long Polling instead of short polling  
- Use WebSockets for real-time communication  
- Use event-driven systems (Kafka, Pub/Sub)  
