# 🌐 Deep Dive: How Network Calls Work (IP, DNS, TCP, HTTP, TLS, CDN, React Optimization)

---

## 🧱 1. IP Address — Identity of a Machine

### What is an IP Address?
An IP address uniquely identifies a device on a network.

Example:
```
142.250.183.14
```

### Types
- IPv4: 192.168.1.1 (32-bit)
- IPv6: 2001:db8::1 (128-bit)

### Public vs Private
- Public: Internet-visible
- Private: Internal network (e.g., WiFi)

---

## 🌍 2. DNS — Domain to IP Resolution

### Purpose
Convert human-readable domain to IP address.

Example:
```
google.com → 142.250.183.14
```

### Resolution Flow
1. Browser cache
2. OS cache
3. Router cache
4. ISP DNS
5. Root server
6. TLD server (.com)
7. Authoritative DNS

### Key Points
- Distributed system
- Uses caching
- Mostly uses UDP

---

## 🔗 3. TCP vs UDP

### TCP (Reliable)
- Connection-based
- Ordered delivery
- Retries on failure

#### 3-Way Handshake
1. SYN
2. SYN-ACK
3. ACK

### UDP (Fast)
- No connection
- No guarantees
- Low latency

### Comparison

| Feature | TCP | UDP |
|--------|-----|-----|
| Reliability | Yes | No |
| Speed | Slower | Faster |
| Use Case | APIs, HTTP | Streaming, DNS |

---

## 🚀 4. HTTP Evolution

### HTTP/1.1
- Runs on TCP
- One request per connection

### HTTPS
- HTTP + TLS encryption
- Secure communication

### HTTP/2
- Multiplexing
- Header compression
- Binary protocol

### HTTP/3
- Runs on QUIC (UDP)
- No head-of-line blocking

---

## 🔐 5. TLS / SSL Internals

### Handshake Steps
1. Client Hello
2. Server Hello + Certificate
3. Certificate verification
4. Key exchange
5. Session key generation

### Encryption Types
- Asymmetric → Key exchange
- Symmetric → Data transfer

---

## ⚖️ 6. Load Balancer

### Purpose
Distribute traffic across servers.

### Types
- L4: Based on IP/Port
- L7: Based on headers, URL

### Algorithms
- Round Robin
- Least Connections
- IP Hash

---

## 🌍 7. CDN (Content Delivery Network)

### Purpose
Reduce latency using edge servers.

### Flow
User → CDN → Origin Server (if cache miss)

### Benefits
- Faster response
- Reduced backend load

---

## ⚛️ 8. React App Optimization Techniques

### Techniques
- Caching (React Query / RTK Query)
- Debouncing
- Pagination / Infinite Scroll
- Lazy Loading
- Request Deduplication
- Optimistic Updates
- Prefetching

---

## 🧠 End-to-End Flow

```
React App
   ↓
Browser
   ↓
DNS Lookup
   ↓
TCP / QUIC Connection
   ↓
TLS Handshake
   ↓
Load Balancer
   ↓
Backend Server
   ↓
Database
   ↓
Response
   ↓
CDN (static assets)
```

---

## 🔥 Key Takeaways

- DNS resolves domain → IP
- TCP ensures reliability, UDP ensures speed
- HTTPS secures communication via TLS
- HTTP/2 and HTTP/3 improve performance
- Load balancers enable scalability
- CDN reduces latency globally
- Frontend optimizations reduce unnecessary calls

---

## 🚀 Advanced Topics (Next Steps)

- QUIC Protocol internals
- Browser rendering pipeline
- API Gateway vs Load Balancer
- Service Mesh (Istio)

