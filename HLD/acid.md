# ACID Properties in Databases

> **"ACID properties ensure that every database transaction is reliable, consistent, and safe even in the presence of failures or concurrent users."**

---

# Table of Contents

1. Why Do We Need ACID?
2. What is a Transaction?
3. ACID Overview
4. Atomicity
5. Consistency
6. Isolation
7. Isolation Levels
8. Durability
9. Internal Working of a Transaction
10. Real World Examples
11. ACID in System Design
12. Interview Questions
13. Key Takeaways

---

# Why Do We Need ACID?

Imagine you're building a banking application.

A customer transfers **₹5,000** to another user.

Behind the scenes, two database operations occur:

```text
1. Deduct ₹5,000 from Sender

2. Add ₹5,000 to Receiver
```

Suppose the system crashes after deducting money but before adding it to the receiver.

Database becomes:

```text
Sender  : ₹5,000
Receiver: ₹2,000
```

The money has disappeared.

This is unacceptable.

Instead, we need:

```text
Either

✓ Both operations succeed

OR

✓ Neither operation succeeds
```

This is exactly what a **Database Transaction** guarantees.

---

# What is a Transaction?

A **Transaction** is a group of one or more database operations treated as a single unit of work.

Without Transaction:

```text
Update A

Update B

Update C
```

If Update B fails,

Update A is already saved.

Database becomes inconsistent.

---

With Transaction:

```sql
BEGIN;

Update A;

Update B;

Update C;

COMMIT;
```

If any operation fails:

```sql
ROLLBACK;
```

Everything is restored.

---

# ACID Overview

ACID stands for:

| Property | Meaning |
|----------|----------|
| A | Atomicity |
| C | Consistency |
| I | Isolation |
| D | Durability |

Each transaction follows these four guarantees.

---

# 1. Atomicity

## Definition

Atomicity means:

> **A transaction is an indivisible unit.**

Either:

```text
Everything happens
```

OR

```text
Nothing happens
```

Never partial execution.

---

## Banking Example

Initial State

```text
Rahul = 10000

Amit = 2000
```

Transaction

```sql
BEGIN;

Rahul = Rahul - 5000;

Amit = Amit + 5000;

COMMIT;
```

Result

```text
Rahul = 5000

Amit = 7000
```

---

Now suppose server crashes after deducting money.

Without Atomicity

```text
Rahul = 5000

Amit = 2000
```

₹5000 disappeared.

---

With Atomicity

Database performs

```sql
ROLLBACK;
```

Final State

```text
Rahul = 10000

Amit = 2000
```

Exactly as before.

---

## How Atomicity Works

Databases maintain an **Undo Log**.

Example:

Before update

```text
Rahul = 10000
```

Undo Log stores

```text
Rahul -> 10000
```

If transaction fails

Database restores

```text
Rahul = 10000
```

---

## Key Point

Atomicity answers:

> **Did every operation complete successfully?**

---

# 2. Consistency

## Definition

Consistency means:

> **The database always moves from one valid state to another valid state.**

It ensures all business rules remain valid.

---

## Important

Database Consistency is **NOT**

```text
Data being same everywhere.
```

That is **Distributed System Consistency**.

Database Consistency means:

```text
Constraints remain valid.
```

---

## Example

Rule

```text
Balance cannot be negative.
```

Current

```text
Balance = 2000
```

Withdraw

```text
5000
```

Result

```text
Balance = -3000
```

Invalid.

Database rejects transaction.

---

## Other Examples

### Primary Key

Cannot duplicate IDs.

### Foreign Key

Cannot reference non-existing records.

### Check Constraint

```sql
Age > 0
```

Cannot insert

```text
Age = -10
```

### Unique Constraint

Email must remain unique.

---

## Key Point

Consistency answers:

> **Is the resulting data valid?**

---

# Atomicity vs Consistency

These are often confused.

| Atomicity | Consistency |
|------------|-------------|
| All operations execute | Data remains valid |
| Deals with execution | Deals with correctness |

Example

Transaction completes:

```text
Rahul = -500

Amit = 9000
```

Atomicity

✅ Yes

Consistency

❌ No

Negative balance violates rules.

---

# 3. Isolation

## Definition

Isolation means:

> Multiple transactions should not interfere with each other.

Even if transactions execute simultaneously,

they should behave as if executed one after another.

---

## Example

Balance

```text
10000
```

Transaction A

Withdraw

```text
7000
```

Transaction B

Withdraw

```text
5000
```

Both read

```text
10000
```

Transaction A writes

```text
3000
```

Transaction B writes

```text
5000
```

Final Balance

```text
5000
```

One withdrawal disappeared.

---

Isolation prevents this.

---

# Problems Without Isolation

## Dirty Read

Transaction reads uncommitted data.

Example

Transaction A

```text
Balance = 5000
```

Not committed.

Transaction B reads

```text
5000
```

Later Transaction A rolls back.

Actual balance

```text
10000
```

Transaction B read invalid data.

---

## Non-Repeatable Read

Same row gives different values.

Example

Read Balance

```text
10000
```

Another transaction updates it.

Read again

```text
7000
```

Different value.

---

## Phantom Read

Query

```sql
SELECT * FROM Orders
WHERE Amount > 1000;
```

Returns

```text
10 rows
```

Another transaction inserts one matching row.

Running the same query again returns

```text
11 rows
```

New rows "appear".

---

# Isolation Levels

---

## 1. Read Uncommitted

Weakest level.

Allows

- Dirty Read
- Non-repeatable Read
- Phantom Read

Fastest.

Rarely used.

---

## 2. Read Committed

Prevents

- Dirty Read

Allows

- Non-repeatable Read
- Phantom Read

Default in many databases.

---

## 3. Repeatable Read

Prevents

- Dirty Read
- Non-repeatable Read

May allow Phantom Reads (depends on implementation).

Default in MySQL InnoDB.

---

## 4. Serializable

Strongest isolation.

Prevents

- Dirty Read
- Non-repeatable Read
- Phantom Read

Safest

But slowest.

---

# Isolation Level Summary

| Isolation Level | Dirty Read | Non-repeatable Read | Phantom Read |
|-----------------|------------|----------------------|---------------|
| Read Uncommitted | ✅ | ✅ | ✅ |
| Read Committed | ❌ | ✅ | ✅ |
| Repeatable Read | ❌ | ❌ | Depends on DB |
| Serializable | ❌ | ❌ | ❌ |

---

# 4. Durability

## Definition

Durability means:

> Once a transaction commits, it is permanently saved.

Even if:

- Server crashes
- Power failure
- Database restarts

Committed data must survive.

---

## Example

```sql
BEGIN;

Update Balance;

COMMIT;
```

Immediately after commit

Server crashes.

After restart

Balance remains updated.

---

## How Durability Works

Databases first write changes to a **Write Ahead Log (WAL)** or **Redo Log**.

Flow

```text
Transaction

↓

Write Log

↓

Commit

↓

Update Database
```

If crash occurs

Database replays logs.

Data is recovered.

---

# Internal Working of a Transaction

```text
BEGIN

↓

Acquire Locks

↓

Create Undo Log

↓

Execute Queries

↓

Write Redo/WAL Log

↓

Validate Constraints

↓

COMMIT

↓

Release Locks
```

If failure occurs

```text
ROLLBACK

↓

Undo Log restores data

↓

Release Locks
```

---

# Real World Example

## E-Commerce Checkout

Customer buys a laptop.

Database operations

```text
Decrease Inventory

Create Order

Reserve Payment

Generate Invoice

Update Loyalty Points

Send Confirmation
```

Without transaction

Inventory may reduce

Payment may fail

Order partially created.

---

With ACID

Either

```text
Everything succeeds
```

OR

```text
Everything rolls back
```

---

# Where ACID is Required

| Application | Need ACID? | Reason |
|-------------|------------|---------|
| Banking | ✅ | Money cannot disappear |
| Payment Gateway | ✅ | Avoid duplicate charges |
| Airline Booking | ✅ | Prevent seat overbooking |
| Inventory | ✅ | Prevent incorrect stock |
| Hospital Systems | ✅ | Patient data integrity |
| Tax Systems | ✅ | Financial correctness |

---

# Where ACID is Less Critical

| Application | Reason |
|-------------|---------|
| Social Media Likes | Temporary inconsistency acceptable |
| Analytics | Eventual correctness is acceptable |
| Logging | Missing few logs usually acceptable |
| Recommendation Systems | Approximate data is acceptable |

---

# ACID vs BASE

| ACID | BASE |
|------|------|
| Strong consistency | Eventual consistency |
| Reliable | Highly scalable |
| Relational databases | NoSQL databases |
| Banking | Social media |

---

# Common Interview Questions

### Q1. What is ACID?

Four guarantees provided by database transactions:

- Atomicity
- Consistency
- Isolation
- Durability

---

### Q2. Difference between Atomicity and Consistency?

Atomicity ensures:

> All operations complete.

Consistency ensures:

> Data remains valid after transaction.

---

### Q3. What causes Dirty Reads?

Reading data modified by another transaction before it commits.

---

### Q4. Which Isolation Level is safest?

Serializable.

---

### Q5. Why isn't Serializable always used?

Because it reduces concurrency and affects performance.

---

### Q6. What is Rollback?

Restores database to its previous state when transaction fails.

---

### Q7. What are Undo Logs?

Used for rollback.

---

### Q8. What are Redo/WAL Logs?

Used for crash recovery after commit.

---

# Key Takeaways

- ACID makes transactions reliable.
- Transactions ensure all-or-nothing execution.
- Atomicity prevents partial updates.
- Consistency ensures business rules remain valid.
- Isolation protects concurrent transactions.
- Durability guarantees committed data survives crashes.
- Banking, payment, inventory, and booking systems rely heavily on ACID.
- Understanding isolation levels is essential for system design interviews.

---

# Mental Model

```text
                Transaction
                     │
                     ▼
      ┌────────────────────────────┐
      │ Atomicity                  │
      │ All or Nothing             │
      ├────────────────────────────┤
      │ Consistency                │
      │ Valid State → Valid State  │
      ├────────────────────────────┤
      │ Isolation                  │
      │ Concurrent = Sequential    │
      ├────────────────────────────┤
      │ Durability                 │
      │ Commit = Permanent         │
      └────────────────────────────┘
```

---

# One-Line Summary

> **ACID properties guarantee that every database transaction is executed completely, preserves data correctness, remains isolated from concurrent transactions, and survives system failures.**

# ACID Properties in SQL vs NoSQL Databases

## Common Misconception

Many people believe that ACID properties exist only in SQL databases.

This is **not true**.

> ACID is a property of **transactions**, not of SQL databases.

Many modern NoSQL databases also support ACID transactions, although the level and scope of support differ from traditional relational databases.

---

# What is ACID?

ACID stands for:

- Atomicity
- Consistency
- Isolation
- Durability

These four properties ensure that transactions execute safely and reliably.

---

# SQL Databases

Examples

- MySQL
- PostgreSQL
- SQL Server
- Oracle

SQL databases were designed around transactions.

They provide strong ACID guarantees.

Example:

```sql
BEGIN TRANSACTION;

UPDATE Accounts
SET Balance = Balance - 1000
WHERE Id = 1;

UPDATE Accounts
SET Balance = Balance + 1000
WHERE Id = 2;

COMMIT;
```

Either both updates succeed or both fail.

---

# NoSQL Databases

NoSQL does **not** mean "No ACID."

Different NoSQL databases provide different transaction guarantees.

## MongoDB

Earlier versions

- Atomic only at a single-document level.

Modern versions (4.0+)

- Multi-document ACID transactions supported.

Example

```
Start Transaction

Update Account A

Update Account B

Insert Audit Log

Commit
```

---

## Cassandra

Designed for

- High Availability
- Horizontal Scaling
- Distributed Systems

Uses **Eventual Consistency**.

Example

Node A receives update.

Node B temporarily misses it.

After a few seconds all nodes become consistent.

Suitable for:

- Messaging
- Analytics
- Time-series data

Not suitable for bank ledgers.

---

## Redis

Supports

- Atomic commands
- Basic transactions (MULTI / EXEC)

Useful for

- Counters
- Caching
- Sessions

---

## DynamoDB

Supports

- ACID transactions through Transaction APIs.

Useful for cloud-native applications.

---

# Why was NoSQL Created?

SQL databases guarantee strong consistency.

This requires

- Row locking
- Transaction logs
- Constraints
- Synchronization

These operations reduce scalability.

NoSQL often sacrifices strict consistency to achieve

- Higher availability
- Better scalability
- Faster writes

---

# CAP Theorem

Distributed databases cannot guarantee all three simultaneously.

- Consistency
- Availability
- Partition Tolerance

Most NoSQL databases choose

Availability + Partition Tolerance

instead of strict consistency.

---

# Can Banks Use NoSQL?

Yes.

But usually **not** for the core ledger.

A typical banking system uses multiple databases.

| Component | Database |
|-----------|----------|
| Account Balance | SQL |
| Money Transfer | SQL |
| Ledger | SQL |
| Notifications | NoSQL |
| Analytics | NoSQL |
| Search | Elasticsearch |
| Cache | Redis |

Only the financial operations require strict ACID guarantees.

---

# Key Takeaways

- ACID is a property of transactions, not SQL.
- SQL databases provide strong ACID guarantees by default.
- Modern NoSQL databases also support ACID to varying degrees.
- Most real-world systems use both SQL and NoSQL.
- Use SQL where correctness is critical.
- Use NoSQL where scalability and availability are more important.

> 2. ACID Transactions vs Non-ACID Transactions

# ACID Transactions vs Non-ACID Transactions

## What is a Transaction?

A transaction is

> A group of database operations treated as a single unit of work.

Either

- Everything succeeds

or

- Everything fails

There is no partial success.

---

# Example

Transfer ₹1000

```
Step 1
Deduct ₹1000 from Suraj

Step 2
Add ₹1000 to Rahul
```

These two operations together form one transaction.

---

# ACID Transaction

```
BEGIN TRANSACTION

Deduct ₹1000

Add ₹1000

COMMIT
```

Result

```
Suraj = ₹9000

Rahul = ₹6000
```

Everything is correct.

---

Suppose power fails after deducting money.

```
BEGIN

Deduct ₹1000

Power Failure
```

The database performs

```
ROLLBACK
```

Final state

```
Suraj = ₹10000

Rahul = ₹5000
```

No data corruption.

---

# Non-ACID Transaction

Without transaction

```
Deduct ₹1000

Save

Add ₹1000

Save
```

Suppose the server crashes after the first save.

Final state

```
Suraj = ₹9000

Rahul = ₹5000
```

₹1000 disappears.

This is an inconsistent state.

---

# Real World Example

## Banking

Operations

- Debit Account
- Credit Account
- Update Ledger

All must succeed together.

Requires ACID.

---

## Instagram

Operations

- Store Like
- Update Analytics
- Notify User
- Update Recommendation

Only storing the Like is critical.

Everything else can happen later.

No strict ACID transaction required.

---

## Food Delivery

Operations

- Create Order
- Reserve Inventory
- Charge Payment
- Send Email

Email can fail.

The order should still exist.

Therefore

Email should not be inside the transaction.

---

# Why Not Use ACID Everywhere?

ACID requires

- Locks
- Transaction logs
- Rollbacks
- Synchronization
- Concurrency control

These increase overhead.

Applications like Instagram prioritize speed over perfect consistency.

---

# Where ACID is Required

- Banking
- Payments
- Ticket Booking
- Inventory Management
- Hospital Records
- Financial Ledgers

---

# Where Eventual Consistency is Acceptable

- Likes
- Comments
- Analytics
- Notifications
- Search
- Cache
- Recommendations

---

# Key Takeaways

- A transaction is a group of operations executed together.
- ACID transactions guarantee correctness.
- Non-ACID operations prioritize speed and scalability.
- Not every operation requires a transaction.
- Business requirements determine whether ACID is needed.

> 3. How Do We Define a Transaction? When is Something Non-Transactional?

# How Do We Define a Transaction?

## Who Defines a Transaction?

The database does **not** automatically decide what a transaction is.

The developer defines it.

Example

```sql
BEGIN TRANSACTION;

UPDATE Accounts
SET Balance = Balance - 1000
WHERE Id = 1;

UPDATE Accounts
SET Balance = Balance + 1000
WHERE Id = 2;

COMMIT;
```

Both SQL statements become one transaction.

---

Without transaction

```sql
UPDATE Accounts
SET Balance = Balance - 1000
WHERE Id = 1;

UPDATE Accounts
SET Balance = Balance + 1000
WHERE Id = 2;
```

Each statement commits independently.

If the server crashes after the first update

```
Account A = Updated

Account B = Not Updated
```

Database becomes inconsistent.

---

# Example in .NET

```csharp
using var transaction = await dbContext.Database.BeginTransactionAsync();

try
{
    // Update Account A

    // Update Account B

    await dbContext.SaveChangesAsync();

    await transaction.CommitAsync();
}
catch
{
    await transaction.RollbackAsync();
}
```

The developer explicitly starts the transaction.

---

# What Makes Something Transactional?

Ask this question

> If one operation succeeds and another fails, will the data become incorrect?

If the answer is YES

Use one transaction.

Examples

- Debit Money + Credit Money
- Create Order + Reduce Inventory
- Reserve Seat + Confirm Booking

---

# What Makes Something Non-Transactional?

If operations can fail independently without affecting business correctness.

Examples

- Send Email
- Send SMS
- Push Notification
- Update Analytics
- Update Search Index
- Update Cache

These operations can be retried later.

---

# Online Shopping Example

Operations

- Create Order
- Reduce Inventory
- Charge Card
- Generate Invoice
- Send Email

Transaction

```
BEGIN

Create Order

Reduce Inventory

Charge Card

COMMIT
```

Outside Transaction

```
Send Email

Update Analytics

Notify Warehouse
```

Even if email fails

The order remains successful.

---

# ATM Example

Operations

- Check Balance
- Debit Account
- Update Ledger
- Dispense Cash
- Send SMS

Transaction

- Debit Account
- Update Ledger

Outside Transaction

- Send SMS

---

# Instagram Example

Operations

- Save Like
- Update Feed
- Update Analytics
- Notify User
- Update Recommendation

Only the core database write is transactional.

Everything else is asynchronous.

---

# Industry Rule

Keep transactions

- Small
- Fast
- Only for business-critical operations

Do NOT include

- Emails
- Notifications
- Analytics
- Cache Updates
- Search Index Updates

inside transactions.

---

# Golden Rule

Whenever designing a system, ask

> If the application crashes immediately after this step, will the business data become incorrect?

If YES

It belongs inside the transaction.

If NO

It should remain outside the transaction.

---

# Key Takeaways

- Transactions are defined by developers.
- A transaction groups multiple operations into one logical unit.
- Use transactions only where business consistency is required.
- Keep transactions as short as possible.
- Side effects should usually happen outside the transaction.