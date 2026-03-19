# 🚀 Sharding in System Design (HLD Notes)

## 📌 What is Sharding?

Sharding is a database architecture pattern where a large dataset is split into smaller, independent pieces called **shards**, and distributed across multiple machines.

Instead of:
- One large database handling all requests ❌

We use:
- Multiple smaller databases (shards) handling subsets of data ✅

---

## 🧠 Why Sharding?

As applications scale:
- Data grows massively 📈
- Queries slow down 🐢
- Single DB becomes bottleneck ⚠️

Sharding enables **horizontal scaling**.

---

## 🧩 Types of Sharding

### 1. Range-Based Sharding
```
Shard 1 → user_id 1–1000
Shard 2 → user_id 1001–2000
```
Pros:
- Simple to implement

Cons:
- Hotspot problem

---

### 2. Hash-Based Sharding
```
shard = hash(user_id) % N
```
Pros:
- Even distribution

Cons:
- Difficult rebalancing

---

### 3. Directory-Based Sharding
```
User → Lookup Table → Shard
```
Pros:
- Flexible

Cons:
- Extra lookup overhead

---

## ⚙️ How Sharding Works

### Example Flow

1. Client Request:
```
GET /user?user_id=123
```

2. Routing Logic:
```
shard = hash(123) % 3 → Shard 2
```

3. Query Execution:
- Request goes to Shard 2

4. Response:
- Data returned to client

---

## 🏗️ Architecture Diagram

```
          ┌─────────────┐
          │   Client    │
          └─────┬───────┘
                │
        ┌───────▼────────┐
        │   Router/App   │
        └───────┬────────┘
   ┌────────────┼────────────┐
   ▼            ▼            ▼
Shard 1      Shard 2      Shard 3
(DB)          (DB)          (DB)
```

---

## 🔥 Benefits

### 1. Scalability
- Add more shards as data grows

### 2. Performance
- Smaller datasets → faster queries

### 3. Fault Isolation
- One shard fails → others unaffected

### 4. Cost Efficiency
- Use commodity hardware

---

## ⚠️ Challenges

### 1. Rebalancing
- Adding shards requires data migration

### 2. Cross-Shard Queries
- Queries may need to hit all shards

### 3. Complex Logic
- Application must route requests

### 4. Joins Are Hard
- Cross-shard joins are complex

### 5. Hotspots
- Uneven traffic distribution

---

## 🔁 Sharding vs Replication

| Feature | Sharding | Replication |
|--------|---------|-------------|
| Goal | Scale | Availability |
| Data | Split | Copy |
| Writes | Distributed | Same data |

---

## 🏗️ Real-World Example

### User-Based Sharding
```
Shard 1 → India Users
Shard 2 → US Users
Shard 3 → Europe Users
```

### Hash-Based
```
user_id % 3
```

---

## 🎯 When to Use Sharding

- Large datasets
- High traffic systems
- Performance bottlenecks
- Need horizontal scaling

---

## 🧠 Interview Strategy

Step-by-step scaling approach:

1. Vertical scaling
2. Read replicas
3. Caching
4. Sharding

---

## 💡 Analogy

Library system:
- One giant shelf ❌
- Multiple categorized shelves ✅

---

## 🔥 Final Takeaways

- Sharding = Horizontal scaling
- Improves performance and scalability
- Introduces complexity
- Often combined with replication

---

## 🚀 Bonus: Advanced Topics

- Consistent Hashing
- Geo-sharding
- Multi-tenant sharding
- Auto-rebalancing systems
