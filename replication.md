# Replication in High-Level Design (HLD)

## 🚀 What is Replication?

Replication is the process of maintaining multiple copies of data across
different nodes to improve availability, fault tolerance, and
scalability.

------------------------------------------------------------------------

## 🎯 Goals of Replication

-   High Availability
-   Fault Tolerance
-   Read Scalability
-   Disaster Recovery

------------------------------------------------------------------------

## 🏗️ Basic Architecture

              ┌────────────┐
              │  Primary   │
              └─────┬──────┘
                    │
            ┌───────┴────────┐
            │                │
      ┌────────────┐   ┌────────────┐
      │ Replica 1  │   │ Replica 2  │
      └────────────┘   └────────────┘

------------------------------------------------------------------------

## ⚙️ Types of Replication

### 1. Synchronous Replication

-   Strong consistency
-   Higher latency

### 2. Asynchronous Replication

-   Fast writes
-   Possible data loss

### 3. Semi-Synchronous

-   Balance of both

------------------------------------------------------------------------

## 🔄 Replication Models

### Single Leader

-   One primary, multiple replicas

### Multi-Leader

-   Multiple write nodes
-   Conflict resolution required

### Leaderless

-   Any node can accept writes
-   Uses quorum (N, R, W)

------------------------------------------------------------------------

## ⚖️ Trade-offs (CAP Theorem)

-   Consistency vs Availability

------------------------------------------------------------------------

## ⏱️ Replication Lag

Delay between primary and replica sync.

------------------------------------------------------------------------

## 🧯 Failover

-   Detect failure
-   Promote replica
-   Redirect traffic

------------------------------------------------------------------------

## 🔥 Real-World Example

E-commerce: - Writes → Primary - Reads → Replicas

------------------------------------------------------------------------

## ⚠️ Common Pitfalls

-   Read-after-write inconsistency
-   Replication lag
-   Split brain

------------------------------------------------------------------------

## 🧠 Advanced Concepts

-   WAL (Write Ahead Log)
-   Snapshot replication
-   Conflict resolution

------------------------------------------------------------------------

## 🎯 Interview Tips

### One-liner:

Replication is maintaining multiple copies of data across nodes to
improve availability and scalability.

### When to use:

-   High read traffic
-   Fault tolerance requirement
-   Global systems

------------------------------------------------------------------------

## 🚀 Key Takeaway

Replication is about balancing consistency, availability, and
performance.
