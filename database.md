# 🧠 Database Selection Guide (System Design Perspective)

## 🎯 Goal
This document focuses on **how to choose the right database based on system requirements**, not internal implementations.

---

## 🧠 Golden Rule

> ❗ You do NOT choose a database first.  
> ✅ You choose based on constraints.

---

## 🔑 Key Decision Factors

- Data Structure (Structured / Semi-structured / Unstructured)
- Query Patterns (CRUD / Joins / Search / Aggregations)
- Scale (Thousands → Billions)
- Consistency (Strong vs Eventual)
- Latency (ms / sub-ms / acceptable delay)
- Read vs Write heavy
- Schema flexibility
- Relationships (joins vs independent data)

---

# 🧱 MySQL (Relational DB)

## ✅ Use When:
- Structured data
- Strong ACID transactions
- Relationships (joins)
- Data integrity is critical

## 🔥 Use Cases:
- Banking systems
- Order management
- User accounts

## ⚡ Strengths:
- Strong consistency
- Mature ecosystem

## ❌ Weaknesses:
- Hard horizontal scaling
- Slow joins at scale

## 🧠 Mental Model:
Structured + relational → MySQL

---

# 🐘 PostgreSQL (Advanced RDBMS)

## ✅ Use When:
- Complex queries
- Analytics
- JSON + SQL hybrid needs
- Advanced indexing

## 🔥 Use Cases:
- Fintech
- Analytics dashboards
- GIS systems

## ⚡ Strengths:
- Powerful query engine
- JSONB support

## ❌ Weaknesses:
- Heavier than MySQL
- Scaling harder than NoSQL

## 🧠 Mental Model:
MySQL but more powerful → PostgreSQL

---

# 🍃 MongoDB (NoSQL)

## ✅ Use When:
- Flexible schema
- Rapid development
- Minimal joins
- Easy horizontal scaling

## 🔥 Use Cases:
- User profiles
- Product catalogs
- Content systems

## ⚡ Strengths:
- Schema flexibility
- Easy scaling

## ❌ Weaknesses:
- No strong joins
- Data duplication

## 🧠 Mental Model:
Dynamic schema → MongoDB

---

# 🔍 Elasticsearch (Search Engine)

## ✅ Use When:
- Full-text search
- Ranking & relevance
- Log analytics

## 🔥 Use Cases:
- Product search
- Job search
- Log monitoring

## ⚡ Strengths:
- Fast search
- Scalable

## ❌ Weaknesses:
- Not primary DB
- Weak consistency

## 🧠 Mental Model:
Search → Elasticsearch

---

# ⚡ Redis (In-Memory Store)

## ✅ Use When:
- Ultra-low latency
- Caching
- Rate limiting
- Sessions

## 🔥 Use Cases:
- API cache
- Leaderboards
- Counters

## ⚡ Strengths:
- Extremely fast

## ❌ Weaknesses:
- Memory expensive
- Persistence risk

## 🧠 Mental Model:
Speed → Redis

---

# 📊 Comparison Table

| DB | Type | Best For | Avoid When |
|----|-----|---------|-----------|
| MySQL | Relational | Transactions | Large scale joins |
| PostgreSQL | Advanced RDBMS | Complex queries | Ultra high scale |
| MongoDB | NoSQL | Flexible schema | Strong consistency needed |
| Elasticsearch | Search | Full-text search | Primary storage |
| Redis | In-memory | Caching | Large persistent data |

---

# 🧠 Real System Example: E-commerce

| Component | Database |
|----------|--------|
| Users | MySQL / PostgreSQL |
| Orders | MySQL |
| Product Catalog | MongoDB |
| Search | Elasticsearch |
| Cache | Redis |

---

# 🔥 Concept: Polyglot Persistence

> Using multiple databases in one system based on use-case.

---

# 🚀 Common Mistakes

- Using one DB for everything ❌
- Replacing DB with Elasticsearch ❌
- Ignoring trade-offs ❌

---

# 🧠 Final Insight

> Each database solves a specific problem.  
> Great systems use the right tool for each job.

---

# 🔜 Next Learning

- Instagram system design
- Payment systems
- Chat systems
- Search systems

