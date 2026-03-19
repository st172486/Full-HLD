# 🚀 Latency in System Design - Detailed Notes

## 📌 What is Latency?

**Latency = the time delay between sending a request and receiving a response.**

In simple terms:
> How long it takes from when you ask for something to when you start getting a response.

Example:
- Clicking a button → API call → response
- If it takes 200 ms → latency = 200 ms

---

## 🧠 Why Does Latency Exist?

Latency exists because no operation in a system is instantaneous. Every step introduces delay.

---

## 🔍 Components of Latency

### 1. Network Latency
- Time taken for data to travel between client and server
- Depends on:
  - Distance
  - Network medium (WiFi, fiber, mobile)

---

### 2. DNS Resolution
- Converts domain name to IP address
- Example:
  google.com → 142.250.x.x

---

### 3. TCP Handshake
- Connection establishment (3-way handshake)
  1. SYN
  2. SYN-ACK
  3. ACK

---

### 4. TLS/SSL Handshake
- Secure connection setup
- Certificate validation adds delay

---

### 5. Server Processing Time
- Business logic execution
- API processing

---

### 6. Database Latency
- Query execution
- Disk I/O
- Locks/contention

---

### 7. Payload Transfer Time
- Larger response → higher latency

---

### 8. Queueing Delay
- Requests wait when system is overloaded

---

## 🔄 End-to-End Latency Flow

User Click  
↓  
DNS Lookup  
↓  
TCP Handshake  
↓  
TLS Handshake  
↓  
Request Travel  
↓  
Server Processing  
↓  
DB Query  
↓  
Response Travel  

---

## ⚠️ Why Latency Matters

### User Experience
- Slow apps → poor engagement

### Business Impact
- Small delays → revenue loss

### Scalability
- High latency reduces system throughput

---

## 🛠️ Techniques to Reduce Latency

### 1. Caching
- Use Redis, browser cache, CDN
- Avoid repeated DB calls

---

### 2. CDN (Content Delivery Network)
- Serve data from nearest location

---

### 3. Reduce Network Calls
- Combine APIs
- Use batching

---

### 4. Keep-Alive Connections
- Reuse TCP connections

---

### 5. Optimize TLS
- Session reuse
- Use HTTP/2 or HTTP/3

---

### 6. Backend Optimization
- Efficient algorithms
- Async processing

---

### 7. Database Optimization
- Indexing
- Query tuning
- Read replicas

---

### 8. Reduce Payload Size
- Compression (gzip, brotli)
- Send only required fields

---

### 9. Load Balancing
- Distribute traffic across servers

---

### 10. Asynchronous Processing
- Background jobs for non-critical tasks

---

## 💡 Real-World Example

Search feature:
- Without optimization → slow response
- With optimization:
  - Caching
  - Pagination
  - Debouncing
  - Skeleton loaders

---

## 🧠 Key Insight

Latency is:
> The sum of delays across network, protocol, server, database, and data transfer layers.

---

## 🎯 Summary Table

| Component | Cause |
|----------|------|
| Network | Distance, bandwidth |
| DNS | Lookup delay |
| TCP/TLS | Connection setup |
| Server | Processing |
| Database | Query performance |
| Payload | Data size |
| Load | Queueing |

---

## 🔥 Interview Insight

> Latency is unavoidable, but can be minimized using caching, distribution, and efficient system design.

---

## 🚀 Advanced Topics (Next Level)

- P50, P95, P99 latency
- Tail latency problem
- CDN internals
- HTTP/2 vs HTTP/3
- Latency vs throughput trade-offs
