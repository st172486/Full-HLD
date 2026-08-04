# Saga Pattern in Microservices

> A comprehensive guide to understanding the Saga Pattern, why it exists, how it works, and when to use it in distributed systems.

---

# Table of Contents

1. Introduction
2. Why Do We Need Saga?
3. Problems with Distributed Transactions
4. What is a Saga?
5. Local Transactions vs Distributed Transactions
6. Compensating Transactions
7. Saga Execution Flow
8. Amazon Order Example
9. Flight Booking Example
10. Banking Example
11. Uber Example
12. Choreography Saga
13. Orchestration Saga
14. Choreography vs Orchestration
15. Designing Compensating Transactions
16. Handling Failures
17. Idempotency
18. Saga State Management
19. Retry Strategy
20. Timeouts
21. Outbox Pattern
22. Common Mistakes
23. Where Saga Should Be Used
24. Where Saga Should NOT Be Used
25. Interview Questions
26. Key Takeaways

---

# Chapter 1: Introduction

In a monolithic application, all business operations usually happen inside a single database transaction.

Example:

```
Create Order
Reserve Inventory
Charge Payment
```

All these operations are wrapped inside:

```
BEGIN TRANSACTION

...

COMMIT
```

If any step fails:

```
ROLLBACK
```

Everything goes back to the previous state.

This guarantees:

- Atomicity
- Consistency
- Isolation
- Durability (ACID)

---

In microservices, things become very different.

Every service owns its own database.

```
                  Client
                     |
     ------------------------------------
     |          |          |            |
 Order      Inventory   Payment    Shipping
 Service      Service    Service     Service
     |          |          |            |
 Order DB   Inventory DB Payment DB Shipping DB
```

Each database transaction is independent.

One service cannot rollback another service's database.

This creates a consistency problem.

---

# Chapter 2: Why Do We Need Saga?

Imagine an e-commerce application.

Workflow:

```
Customer places order

↓

Create Order

↓

Reserve Inventory

↓

Charge Payment

↓

Create Shipment

↓

Send Email
```

Suppose:

```
Create Order          ✔

Reserve Inventory     ✔

Charge Payment        ❌
```

Now the system becomes inconsistent.

Current state:

```
Order Exists

Inventory Reserved

Customer Didn't Pay
```

Inventory remains blocked.

No shipment will happen.

Order is incomplete.

Without Saga, manual cleanup would be required.

---

# Chapter 3: Problems with Distributed Transactions

Many developers ask:

"Why don't we use one transaction across all services?"

Example:

```
Order DB

Inventory DB

Payment DB
```

Unfortunately:

A single SQL transaction cannot span multiple independent databases.

Distributed transactions exist.

Example:

```
Two Phase Commit (2PC)
```

But large companies rarely use it because it introduces several problems.

---

## Problems with 2PC

### 1. Slow

Every participant must agree before commit.

```
Coordinator

↓

Order

↓

Inventory

↓

Payment

↓

Shipping
```

Every service waits.

Latency increases.

---

### 2. Locks are Held Longer

Rows remain locked until every participant responds.

This reduces throughput.

---

### 3. Poor Availability

If one participant crashes,

everyone waits.

Entire workflow becomes blocked.

---

### 4. Scalability Issues

Large distributed systems prefer availability.

2PC hurts scalability.

---

Hence,

Modern systems use:

```
Eventual Consistency

+

Saga Pattern
```

---

# Chapter 4: What is a Saga?

A Saga is:

> A sequence of local transactions coordinated together.

Each service:

- Executes its own transaction
- Commits immediately
- Publishes an event or returns success

If something fails later,

previous work is undone using another transaction called a Compensating Transaction.

Notice:

Not rollback.

Instead:

Undo.

---

Traditional Transaction

```
BEGIN

A

B

C

COMMIT
```

Failure:

```
ROLLBACK
```

Saga:

```
A

↓

B

↓

C fails

↓

Undo B

↓

Undo A
```

---

# Chapter 5: Local Transactions

Each service performs only local database work.

Example:

Order Service

```
INSERT INTO Orders
```

Commit.

Inventory Service

```
UPDATE Inventory
```

Commit.

Payment Service

```
INSERT INTO Payments
```

Commit.

Every service is independent.

---

# Chapter 6: Compensating Transactions

Instead of rollback,

Saga executes another business operation.

Example:

Forward Transaction

```
Reserve Inventory
```

Compensation

```
Release Inventory
```

Forward Transaction

```
Charge Card
```

Compensation

```
Refund Payment
```

Forward Transaction

```
Book Hotel
```

Compensation

```
Cancel Hotel Booking
```

Forward Transaction

```
Create Order
```

Compensation

```
Cancel Order
```

Compensation is just another business transaction.

---

# Chapter 7: Saga Execution Flow

Example:

```
Create Order

↓

Reserve Inventory

↓

Payment

↓

Shipment

↓

Notification
```

Suppose Payment fails.

Saga becomes:

```
Create Order

✔

↓

Reserve Inventory

✔

↓

Payment

❌

↓

Release Inventory

↓

Cancel Order
```

Final state:

```
Inventory Restored

Order Cancelled

Consistent System
```

---

# Chapter 8: Amazon Order Example

Customer purchases:

```
Laptop
```

Step 1

Order Service

```
Order Status

PENDING
```

Committed.

---

Step 2

Inventory Service

```
Available = 10

↓

Reserved = 1

↓

Available = 9
```

Committed.

---

Step 3

Payment Service

```
Card Declined
```

Failure.

---

Saga Compensation

Inventory Service

```
Reserved = 0

Available = 10
```

Order Service

```
Status

CANCELLED
```

Everything is consistent again.

---

# Chapter 9: Flight Booking Example

Services:

```
Flight

Hotel

Taxi

Payment
```

Workflow:

```
Reserve Flight

↓

Reserve Hotel

↓

Reserve Taxi

↓

Payment
```

Payment fails.

Saga executes:

```
Cancel Taxi

↓

Cancel Hotel

↓

Cancel Flight
```

---

# Chapter 10: Banking Example

Transfer ₹1000

```
Withdraw

↓

Deposit
```

Deposit fails.

Compensation:

```
Deposit Back

Into Source Account
```

Notice:

Never rollback.

Execute another business transaction.

---

# Chapter 11: Uber Example

Ride Completed

↓

Payment Captured

↓

Coupon Applied

↓

Wallet Updated

Suppose wallet update fails.

Saga may perform:

```
Reverse Coupon

↓

Refund Payment

↓

Mark Ride Payment Failed
```

---

# Chapter 12: Choreography Saga

There is no central coordinator.

Everything happens using events.

Flow:

```
Order Created

↓

Inventory Service

↓

Inventory Reserved

↓

Payment Service

↓

Payment Success

↓

Shipping Service
```

Every service only knows:

"What event should I react to?"

Example:

Order Service

Publishes

```
OrderCreated
```

Inventory Service

Consumes

```
OrderCreated
```

Publishes

```
InventoryReserved
```

Payment Service

Consumes

```
InventoryReserved
```

Publishes

```
PaymentCompleted
```

Shipping Service

Consumes

```
PaymentCompleted
```

Everything is event driven.

---

If Payment fails

```
PaymentFailed
```

Inventory Service listens.

```
Release Inventory
```

Order Service listens.

```
Cancel Order
```

---

Advantages

- Loose coupling
- Easy to add new services
- Event driven
- Highly scalable

Disadvantages

- Hard debugging
- Long event chains
- Difficult to understand complete workflow
- Hidden dependencies

---

# Chapter 13: Orchestration Saga

There is one central controller.

```
             Saga Orchestrator

                    |

    ----------------------------------

    |        |        |          |

 Order   Inventory Payment Shipping
```

Workflow:

```
Create Order

↓

Reserve Inventory

↓

Charge Payment

↓

Create Shipment
```

Coordinator decides every next step.

If Payment fails,

Coordinator executes:

```
Release Inventory

↓

Cancel Order
```

Advantages

- Easier debugging
- Centralized workflow
- Better monitoring
- Easier retries

Disadvantages

- Additional service
- Can become complex

---

# Chapter 14: Choreography vs Orchestration

| Choreography | Orchestration |
|--------------|---------------|
| Event Driven | Coordinator Driven |
| No Central Service | Central Coordinator |
| Loose Coupling | Central Logic |
| Hard Debugging | Easy Debugging |
| Good for Small Flows | Good for Large Flows |
| Easier to Extend | Easier to Understand |

---

# Chapter 15: Designing Compensating Transactions

Every forward action should have an opposite action.

| Forward Transaction | Compensation |
|---------------------|--------------|
| Create Order | Cancel Order |
| Reserve Inventory | Release Inventory |
| Deduct Wallet | Refund Wallet |
| Reserve Hotel | Cancel Reservation |
| Book Seat | Release Seat |
| Assign Driver | Remove Driver |
| Generate Invoice | Void Invoice |

Some actions cannot truly be undone.

Example:

```
Send Email
```

Cannot unsend.

Instead:

```
Send Correction Email
```

---

# Chapter 16: Handling Failures

Suppose:

```
Reserve Inventory

↓

Payment Failed

↓

Release Inventory

↓

Release Inventory Failed
```

Now compensation also failed.

Saga should:

- Retry
- Retry again
- Store failure state
- Raise alert
- Manual intervention if retries exhausted

---

# Chapter 17: Idempotency

Compensation may execute multiple times.

Example:

```
Release Inventory
```

Suppose retry happens.

Without idempotency:

```
Inventory

10

↓

11

↓

12
```

Incorrect.

Correct implementation:

```
If reservation already released

Do nothing
```

Multiple executions produce the same final state.

---

# Chapter 18: Saga State Management

Saga must remember progress.

Example:

```
Saga ID

Current Step

Completed Steps

Failed Step

Retry Count

Status
```

Example table:

```
SagaInstance

------------------------------

SagaId

OrderId

CurrentState

RetryCount

Status

CreatedAt
```

This enables:

- Resume after crash
- Retry
- Monitoring
- Auditing

---

# Chapter 19: Retry Strategy

Temporary failures should not immediately trigger compensation.

Example:

Payment API timeout.

Instead:

```
Retry 1

↓

Retry 2

↓

Retry 3

↓

Still Failed

↓

Compensation
```

Usually:

Exponential Backoff

```
5 sec

10 sec

20 sec

40 sec
```

---

# Chapter 20: Timeouts

Some services never respond.

Saga should define timeout.

Example:

```
Reserve Inventory

↓

Wait Payment

↓

No response for 5 minutes

↓

Cancel Order
```

Timeouts prevent workflows from remaining incomplete forever.

---

# Chapter 21: Outbox Pattern

A common problem:

```
Update Database

↓

Publish Kafka Event
```

Suppose database commit succeeds.

Kafka publish fails.

Now:

```
Database Updated

No Event Published
```

Other services never know.

Solution:

Outbox Pattern.

Instead:

```
Database Transaction

↓

Business Update

+

Insert Outbox Record

↓

Commit
```

Background worker reads Outbox table and publishes events.

Ensures database changes and event publication stay consistent.

---

# Chapter 22: Common Mistakes

### Mistake 1

Trying to rollback another database.

Impossible.

---

### Mistake 2

Compensation not idempotent.

Retries create inconsistent data.

---

### Mistake 3

Not storing Saga state.

Recovery becomes impossible.

---

### Mistake 4

Infinite retries.

Always define retry limits.

---

### Mistake 5

No monitoring.

Stuck Saga instances remain unnoticed.

---

# Chapter 23: Where Saga Should Be Used

Suitable scenarios:

- E-commerce Orders
- Airline Booking
- Hotel Reservation
- Ride Booking
- Banking
- Insurance
- Loan Processing
- Subscription Activation
- Warehouse Management
- Food Delivery
- Ticket Booking
- Healthcare Appointment Systems

Whenever:

```
One business workflow

↓

Multiple Microservices

↓

Multiple Databases
```

Saga is a strong candidate.

---

# Chapter 24: Where Saga Should NOT Be Used

Do NOT use Saga if:

Everything happens inside one database.

Example:

```
User

Address

Orders
```

Same PostgreSQL database.

Simple transaction is enough.

---

Do not use Saga for:

- Simple CRUD APIs
- Single service applications
- Operations without compensating actions
- Strong ACID requirements across all operations

---

# Chapter 25: Interview Questions

### Q1. Why can't we use SQL transactions across microservices?

Because each microservice owns its own database.
There is no global transaction.

---

### Q2. What is a compensating transaction?

A business operation that undoes previously completed work.

Example:

Reserve Inventory

↓

Release Inventory

---

### Q3. Is Saga eventually consistent?

Yes.

Saga provides:

```
Eventual Consistency
```

Not immediate consistency.

---

### Q4. Which is better?

Depends.

Small workflows:

```
Choreography
```

Complex workflows:

```
Orchestration
```

---

### Q5. Can compensation fail?

Yes.

Handle using:

- Retry
- Idempotency
- Monitoring
- Manual intervention

---

### Q6. Is Saga synchronous?

It can be both.

- HTTP Orchestrated Saga
- Event Driven Saga
- Hybrid

---

# Chapter 26: Key Takeaways

✔ Saga replaces distributed transactions in microservices.

✔ Every service executes its own local transaction.

✔ Failures are handled through compensating transactions.

✔ Saga provides eventual consistency instead of ACID consistency.

✔ Choreography uses events to coordinate services.

✔ Orchestration uses a central coordinator to drive the workflow.

✔ Compensating transactions must be idempotent and retryable.

✔ Saga state should be persisted for recovery and monitoring.

✔ Timeouts, retries, and dead-letter handling are essential for robust implementations.

✔ Saga is ideal for long-running business workflows that span multiple services and databases.

✔ Saga is one of the foundational patterns for building resilient, scalable distributed systems.