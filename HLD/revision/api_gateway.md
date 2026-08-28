# 🚀 API Gateway – Detailed System Design Notes

## 🧠 What is an API Gateway?

An API Gateway is a single entry point for all client requests into a backend system.  
It acts as a traffic controller, security layer, and request orchestrator.

---

## 🏗️ Why API Gateway?

### Problems Without Gateway
- Clients call multiple services directly
- Duplicate authentication logic
- Hard to manage and scale
- Security risks

### Benefits With Gateway
- Centralized authentication
- Smart routing
- Rate limiting
- Logging & monitoring
- Response transformation
- Caching

---

## 🏛️ Architecture

Client → API Gateway → Microservices → Database

---

## 🔄 Request Lifecycle

1. Client sends request
2. Gateway authenticates (JWT/OAuth)
3. Rate limiting applied
4. Request routed to service
5. Response processed & returned

---

## 🔥 Core Responsibilities

### 1. Authentication & Authorization
- JWT validation
- OAuth
- API keys

### 2. Routing
- Path-based routing
- Service discovery

### 3. Rate Limiting
- Prevent abuse & DDoS

### 4. Transformation
- REST ↔ gRPC
- Modify payloads

### 5. Aggregation (BFF)
- Combine multiple service responses

### 6. Caching
- Reduce backend load

### 7. Logging & Monitoring
- Centralized observability

---

## 🌐 API Gateway + WebSockets

### Use Cases
- Chat apps
- Live updates
- Notifications

### Flow
Client ↔ Gateway ↔ WebSocket Service

### Challenges
- Stateful connections
- Scaling difficulty
- Resource heavy

---

## 🔔 API Gateway + Webhooks

### Use Cases
- Payment notifications
- External event triggers

### Flow
External System → Gateway → Service

### Challenges
- Reliability
- Security validation
- Idempotency

---

## ⚖️ API Gateway vs Load Balancer

| Feature | API Gateway | Load Balancer |
|--------|------------|--------------|
| Layer | L7 | L4/L7 |
| Auth | Yes | No |
| Routing | Smart | Basic |
| Transformation | Yes | No |

---

## 🧱 Types of API Gateways

### Managed
- AWS API Gateway
- Azure API Management

### Self-Hosted
- Kong
- NGINX
- Envoy

---

## 🚨 Cons

- Single point of failure
- Added latency
- Complexity
- Bottleneck risk
- Vendor lock-in

---

## 🧠 Best Practices

- Keep gateway thin
- Use rate limiting
- Implement caching
- Add circuit breakers
- Enable observability
- Use BFF pattern

---

## 🔥 Real-World Example

Food Delivery App:

Mobile App → API Gateway → Services

- Auth Service
- Order Service
- Payment Service

WebSockets: Real-time tracking  
Webhooks: Payment updates

---

## 🎯 Final Summary

- API Gateway = Control layer
- WebSockets = Real-time communication
- Webhooks = Event-driven communication
