# Proxy and Reverse Proxy - Deep Dive Notes

## 1. Core Concept

- **Forward Proxy** → Represents the **Client**
- **Reverse Proxy** → Represents the **Server**

---

## 2. Forward Proxy

### Definition
A forward proxy sits between client and internet.

### Flow
Client → Proxy → Server → Proxy → Client

### Key Features
- Hides client IP
- Content filtering
- Caching
- Access control

### Example
Corporate network blocking websites like YouTube.

---

## 3. Reverse Proxy

### Definition
A reverse proxy sits in front of servers.

### Flow
Client → Reverse Proxy → Backend Servers → Reverse Proxy → Client

### Key Features
- Load balancing
- SSL termination
- Caching
- Security
- Routing

---

## 4. Reverse Proxy Internal Working

1. DNS resolves domain → proxy IP
2. Request reaches proxy
3. Proxy:
   - Routes request
   - Selects backend server
   - Applies security rules
4. Backend processes request
5. Proxy returns response

---

## 5. Load Balancing Algorithms

- Round Robin
- Least Connections
- IP Hash

---

## 6. Real World Architecture

```
        Internet
            ↓
    [ Reverse Proxy ]
            ↓
   ┌────────┬────────┬────────┐
 Server A  Server B  Server C
```

---

## 7. Reverse Proxy Responsibilities

### Routing
- /api → service A
- /images → service B

### SSL Termination
- Handles HTTPS at proxy level

### Caching
- Stores frequent responses

### Compression
- gzip / brotli

---

## 8. Headers

- X-Forwarded-For → client IP
- X-Forwarded-Proto → protocol

---

## 9. Common Problems

### Wrong IP
Use X-Forwarded-For

### Infinite Redirect Loop
Fix SSL config

### Sticky Sessions
Use Redis or IP hashing

---

## 10. Reverse Proxy vs Load Balancer

| Feature | Reverse Proxy | Load Balancer |
|--------|-------------|--------------|
| Scope | Broad | Narrow |
| Role | Routing + Security | Traffic Distribution |

---

## 11. Real World Tools

- Nginx
- HAProxy
- Apache
- AWS ELB

---

## 12. System Design Use Cases

- API Gateway
- CDN
- Microservices routing
- Edge computing

---

## 13. Final Intuition

- Forward Proxy → hides client
- Reverse Proxy → hides server
