# 🚀 Denormalization – System Design Notes

## 📌 What is Denormalization?
Denormalization is the process of intentionally adding redundancy into a database to improve read performance.

> Trade-off: Consistency + Storage ❌ vs Speed + Scalability ✅

---

## 🧠 Why Denormalization?
- Read-heavy systems dominate modern applications
- JOIN operations are expensive at scale
- Low latency is critical

---

## 🔥 Example

### Normalized Schema
Users(user_id, name)
Orders(order_id, user_id, product)

Query:
SELECT * FROM Users u JOIN Orders o ON u.user_id = o.user_id;

### Denormalized Schema
Orders(order_id, user_id, user_name, product)

Query:
SELECT * FROM Orders;

---

## 📊 Architecture Diagram

### Normalized (JOIN required)
[Users] ---- JOIN ----> [Orders]

### Denormalized (Pre-joined)
[Orders with user data]

---

## ⚖️ Pros

- 🚀 Faster reads
- 📉 Reduced latency
- 🧠 Simpler queries
- 🌍 Better in distributed systems
- ⚡ Works well with NoSQL

---

## ❌ Cons

- 🔁 Data redundancy
- ⚠️ Inconsistency risk
- ✍️ Slower writes
- 🧩 Complex write logic
- 💾 More storage

---

## 🧠 When to Use

- Read-heavy systems (90% reads)
- Low latency requirements
- Rarely changing data
- Distributed/sharded systems

---

## 🚫 When NOT to Use

- Write-heavy systems
- Frequently changing data
- Strong consistency systems (banking)

---

## ⚔️ Normalization vs Denormalization

| Feature | Normalization | Denormalization |
|--------|-------------|----------------|
| Redundancy | No | Yes |
| Read Speed | Slow | Fast |
| Write Speed | Fast | Slow |
| Consistency | Strong | Risky |

---

## 🏗️ Real-World Examples

### 🛒 E-commerce
Product stores:
- category_name
- seller_name

### 📱 Social Media
Posts store:
- username
- profile_pic

### 📊 Analytics
Pre-aggregated tables

---

## 🎯 Interview Q&A

### Q1: What is Denormalization?
Denormalization is adding redundancy to improve read performance.

---

### Q2: Why use Denormalization?
To avoid expensive JOINs and reduce latency.

---

### Q3: Trade-offs?
- Faster reads
- Slower writes
- Risk of inconsistency

---

### Q4: Where is it used?
- Social media feeds
- E-commerce
- Analytics dashboards

---

### Q5: Normalization vs Denormalization?
Normalization avoids redundancy; denormalization improves performance.

---

## 🔥 Pro Tip

Real systems use a **Hybrid Approach**:
- OLTP → Normalized
- Read replicas / cache → Denormalized

---

## 🧾 Summary

Denormalization is a performance optimization technique used in scalable systems to reduce query complexity and improve speed.

---

*Generated on 2026-03-19*
