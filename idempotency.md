# 📄 Idempotency in System Design (Detailed Notes)

## 🧠 What is Idempotency?

An operation is **idempotent** if performing it multiple times produces the same result as performing it once.

---

## ⚠️ Why Idempotency is Critical

Distributed systems naturally produce duplicate requests due to:
- Network retries
- API Gateway retries
- Load balancer retries
- Message queue redelivery
- User double-clicks

---

## 🔥 Real-world Example: Payments

Without idempotency:
- User clicks "Pay" twice → charged twice ❌

With idempotency:
- Duplicate request → same response returned ✅

---

## 🏗️ Where Idempotency is Used

- Payment systems
- Order creation
- Message queues (Kafka, RabbitMQ)
- Webhooks
- Background jobs

---

## 🧩 HTTP Methods & Idempotency

| Method | Idempotent? |
|--------|------------|
| GET    | Yes        |
| PUT    | Yes        |
| DELETE | Yes        |
| POST   | No (can be made idempotent) |

---

## ⚙️ Implementation Approaches

### 1. Idempotency Key (Recommended)

Client sends:
```
Idempotency-Key: abc123
```

Server:
- If key exists → return stored response
- Else → process + store result

---

### 🗄️ Storage Schema

```
idempotency_table
--------------------------
key (PK)
request_hash
response
status
created_at
expiry
```

---

### 🔐 Request Hash Validation

If same key is reused with different payload:
→ Reject request ❌

---

## ⚡ 2. Natural Idempotency

Use DB constraints:

```
order_id UNIQUE
```

Duplicate insert → prevented by DB

---

## 🔁 3. Idempotent Consumers (Event Systems)

Maintain processed event IDs:

```
if event_id exists:
   skip processing
```

---

## 🧠 Advanced Design Considerations

### TTL (Time-to-Live)
- Payments: ~24 hours
- Webhooks: few hours

---

### Storage Options

| Storage | Use Case |
|--------|--------|
| Redis | Fast + short TTL |
| DB | Strong consistency |
| Hybrid | Best approach |

---

### Race Conditions

Problem:
- Two requests with same key processed simultaneously

Solution:
- DB unique constraint
- Distributed locks

---

## 🏗️ Production Flow

```
Client
  ↓
API Gateway
  ↓
Idempotency Layer
  ↓
Check Key
  ↓
[Exists] → Return cached response
[New] → Process request
  ↓
Store result
  ↓
Return response
```

---

## 🚨 Common Pitfalls

- Not storing response
- Ignoring payload mismatch
- No TTL (memory issues)
- Partial failures
- Weak key generation

---

## ⚖️ Trade-offs

- Extra storage cost
- Increased latency
- Added complexity
- Stale response risk

---

## 🔥 Key Insights

- Idempotency enables **safe retries**
- Systems are **at-least-once**, idempotency simulates **exactly-once**
- Always assume duplicate requests

---

## 🚀 When to Use

✅ Payments  
✅ Orders  
✅ External APIs  
✅ Webhooks  
✅ Retry-heavy systems  

---

## ❌ When Not Needed

❌ Read-only operations  
❌ Non-critical internal processes  

---

## 🎯 Final Thought

> Idempotency = Protection layer against duplicate execution in distributed systems
