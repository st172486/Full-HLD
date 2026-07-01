# 🔥 CAP Theorem --- System Design Notes

## 📌 Definition

CAP theorem (Eric Brewer) states that a distributed system can guarantee
only 2 out of: - Consistency (C) - Availability (A) - Partition
Tolerance (P)

------------------------------------------------------------------------

## ⚙️ The Three Pillars

### 1. Consistency (C)

All nodes see the same data at the same time.

### 2. Availability (A)

Every request receives a response.

### 3. Partition Tolerance (P)

System continues despite network failures.

------------------------------------------------------------------------

## 🚨 Key Insight

In real systems, partitions are inevitable.

👉 So you choose: - CP (Consistency + Partition Tolerance) - AP
(Availability + Partition Tolerance)

------------------------------------------------------------------------

## 🧠 Diagram

               CAP THEOREM
            /      |      \
           C       A       P

    When Partition happens:

            /             \
       Choose C        Choose A
         (CP)           (AP)

     Reject requests   Return stale data

------------------------------------------------------------------------

## 🔍 Examples

### CP Systems

-   HBase
-   MongoDB (strong consistency)

### AP Systems

-   Cassandra
-   DynamoDB

### CA Systems

-   MySQL (single node)
-   PostgreSQL

------------------------------------------------------------------------

## ⚖️ Trade-offs

  Property Lost     Effect
  ----------------- ---------------
  No Availability   Requests fail
  No Consistency    Stale data

------------------------------------------------------------------------

## 🌍 Real-World Mapping

  Use Case               Choice
  ---------------------- --------
  Banking                CP
  E-commerce inventory   CP
  Social media feed      AP
  Messaging apps         AP

------------------------------------------------------------------------

## 🧠 Eventual Consistency

Data becomes consistent over time.

Example: - Instagram likes - Feed updates

------------------------------------------------------------------------

## 🎯 Interview Q&A

### Q1: What is CAP theorem?

A: It states that in presence of partition, system must choose between
consistency and availability.

------------------------------------------------------------------------

### Q2: Why is CA not practical?

A: Because network failures are inevitable.

------------------------------------------------------------------------

### Q3: What is eventual consistency?

A: Data syncs over time across nodes.

------------------------------------------------------------------------

### Q4: Which system would you choose for banking?

A: CP (Consistency is critical)

------------------------------------------------------------------------

### Q5: Which system for social media?

A: AP (Availability is critical)

------------------------------------------------------------------------

## 🚀 Pro Tip

Mention: 👉 "CAP applies only during network partition"

------------------------------------------------------------------------

## 📌 Summary

-   Partition tolerance is mandatory
-   Trade-off between consistency & availability
-   Real systems use hybrid approaches
