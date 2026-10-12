Absolutely. Here are **structured, interview-ready Markdown notes** for Optimistic Locking, including the deeper concepts we discussed.

````md
# Optimistic Concurrency Control (Optimistic Locking)

## 1. What is Optimistic Locking?

Optimistic locking is a concurrency-control technique where we **assume that conflicts are relatively rare**.

Instead of blocking other transactions with a lock, we:

1. Read the data.
2. Remember a version/concurrency token.
3. Perform our business logic.
4. At write time, verify that the data has not changed.
5. If it changed, detect a conflict and retry or reject the operation.

### Core idea

> "I will let everyone proceed, but when you write, prove that the data you originally read is still current."

---

# 2. The Problem Optimistic Locking Solves

Consider:

```text
concert
--------------------------------
concert_id       weeknd_tour
available_seats  1
````

Alice and Bob both read the row.

```text
Alice:
available_seats = 1

Bob:
available_seats = 1
```

Both conclude:

```text
"I can buy the last seat."
```

This creates a race between:

```text
READ
  ↓
make decision
  ↓
WRITE
```

The data can change during this gap.

This can lead to:

* Lost updates
* Incorrect business decisions
* Overselling
* Stale writes

---

# 3. Pessimistic vs Optimistic Concurrency

## Pessimistic Concurrency

Pessimistic concurrency assumes:

> "Someone will probably interfere, so I'll lock the resource first."

```text
READ + LOCK
     ↓
Business logic
     ↓
WRITE
     ↓
COMMIT
     ↓
UNLOCK
```

Other transactions may have to wait.

### Example

```sql
SELECT *
FROM concerts
WHERE concert_id = 'weeknd_tour'
FOR UPDATE;
```

---

## Optimistic Concurrency

Optimistic concurrency assumes:

> "Most of the time nobody will interfere, so I won't block anyone."

```text
READ
 ↓
Remember version
 ↓
Business logic
 ↓
Conditional WRITE
 ↓
Check affected rows
```

If another transaction changed the row:

```text
UPDATE → 0 rows
```

This means:

```text
CONFLICT
```

---

# 4. Core Difference

```text
Pessimistic:
Prevent the conflict.

Optimistic:
Detect the conflict.
```

Or:

```text
Pessimistic = "Wait first."

Optimistic  = "Try first."
```

---

# 5. Version Column

The most common implementation of optimistic locking uses a dedicated version column.

Example:

```text
concert
------------------------------------------------
concert_id       available_seats       version
------------------------------------------------
weeknd_tour      1                      42
```

When we read the row:

```sql
SELECT concert_id, available_seats, version
FROM concerts
WHERE concert_id = 'weeknd_tour';
```

We get:

```text
available_seats = 1
version = 42
```

Our application remembers:

```text
"I read version 42."
```

---

# 6. Conditional UPDATE

When we want to modify the row:

```sql
UPDATE concerts
SET available_seats = available_seats - 1,
    version = version + 1
WHERE concert_id = 'weeknd_tour'
  AND version = 42;
```

The important part is:

```sql
AND version = 42
```

This means:

> "Only update this row if it is still version 42."

---

# 7. Successful Update

Suppose nobody changed the row.

Before:

```text
available_seats = 1
version = 42
```

Execute:

```sql
UPDATE concerts
SET available_seats = available_seats - 1,
    version = version + 1
WHERE concert_id = 'weeknd_tour'
AND version = 42;
```

Result:

```text
Rows affected = 1
```

New state:

```text
available_seats = 0
version = 43
```

The operation succeeded.

---

# 8. Conflicting Update

Suppose Alice updated the row first.

Database now contains:

```text
available_seats = 0
version = 43
```

Bob still has an old snapshot:

```text
available_seats = 1
version = 42
```

Bob executes:

```sql
UPDATE concerts
SET available_seats = available_seats - 1,
    version = version + 1
WHERE concert_id = 'weeknd_tour'
AND version = 42;
```

But the database currently has:

```text
version = 43
```

Therefore:

```text
Rows affected = 0
```

Bob has detected a concurrency conflict.

---

# 9. The Affected Rows Check is Critical

Optimistic locking commonly does NOT result in a database exception.

Instead:

```text
UPDATE
   ↓
0 rows affected
```

Your application must interpret:

```text
0 rows = concurrency conflict
1 row  = success
```

Conceptually:

```csharp
var affectedRows = Update(...);

if (affectedRows == 0)
{
    // Concurrency conflict
}
else
{
    // Success
}
```

If you ignore the affected-row count, optimistic locking can fail completely.

---

# 10. Why the INSERT Must Be in the Same Transaction

Consider a ticket purchase.

You need:

```text
1. Reserve/decrement seat
2. Create ticket
```

These operations should be atomic.

Correct structure:

```text
BEGIN TRANSACTION

    UPDATE concert
    WHERE version = old_version

    Check affected rows

    If 0:
        ROLLBACK

    If 1:
        INSERT ticket

COMMIT
```

Incorrect:

```text
UPDATE concert
↓
ignore result
↓
INSERT ticket
```

If the UPDATE affected zero rows but the INSERT still executes:

```text
concert:
available_seats = 0

tickets:
Bob has a ticket
```

Now the system contains an invalid business state.

---

# 11. Complete Example

Initial state:

```text
concert
--------------------------------
concert_id       weeknd_tour
available_seats  1
version          42
```

Alice reads:

```text
available_seats = 1
version = 42
```

Bob reads:

```text
available_seats = 1
version = 42
```

---

## Alice Writes First

```sql
BEGIN;

UPDATE concerts
SET available_seats = available_seats - 1,
    version = version + 1
WHERE concert_id = 'weeknd_tour'
AND version = 42;
```

Result:

```text
1 row affected
```

Database:

```text
available_seats = 0
version = 43
```

Alice inserts the ticket:

```sql
INSERT INTO tickets (...);
```

Then:

```sql
COMMIT;
```

---

## Bob Writes

Bob still uses:

```sql
AND version = 42
```

But current version is:

```text
43
```

Therefore:

```text
0 rows affected
```

Bob must:

```text
ROLLBACK
```

and then either:

1. Retry using fresh data.
2. Tell the user that the seat is no longer available.

---

# 12. Version Number as a Row Generation

The version number should not be thought of as a timestamp.

Instead, think:

```text
version = generation of the row
```

Example:

```text
version 42
    ↓
write
    ↓
version 43
    ↓
write
    ↓
version 44
    ↓
write
    ↓
version 45
```

If you read version 42 and the database is now at version 44:

```text
Your snapshot is stale.
```

---

# 13. Why Increment the Version?

Every successful write should advance the version.

Example:

```text
Before:
version = 42

After successful write:
version = 43
```

This gives us a simple way to detect:

```text
"Has this row changed since I read it?"
```

If:

```text
current_version == my_version
```

then the row hasn't changed since your read.

If:

```text
current_version != my_version
```

then someone modified it.

---

# 14. Why a Dedicated Integer Version is Usually Best

A dedicated integer version has several advantages.

## 1. Simple

```text
42 → 43 → 44
```

## 2. No clock dependency

No timestamp precision issues.

## 3. No timezone problems

It is not a date/time.

## 4. Explicit semantics

It clearly represents concurrency state.

## 5. Monotonically increasing

It only moves forward.

## 6. Avoids ABA problems

A version can distinguish:

```text
A → B → A
```

because the version continues increasing.

---

# 15. Timestamp as a Concurrency Token

Instead of:

```text
version = 42
```

you could use:

```text
updated_at = 2026-08-30 14:10:23
```

Then:

```sql
UPDATE products
SET price = 500,
    updated_at = CURRENT_TIMESTAMP
WHERE id = 10
AND updated_at = '2026-08-30 14:10:23';
```

This can work, but timestamp precision matters.

If the database only stores timestamps to the nearest second:

```text
14:10:00.100
14:10:00.900
```

could both become:

```text
14:10:00
```

Now the timestamp may fail to distinguish two writes.

A dedicated version counter is generally safer.

---

# 16. Business Values as Concurrency Tokens

Sometimes an existing business value can act as a concurrency token.

Example:

```text
highest_bid = ₹10,000
```

A bidder can perform:

```sql
UPDATE auctions
SET highest_bid = 11000
WHERE id = 1
AND highest_bid = 10000;
```

If someone already bid:

```text
₹12,000
```

the UPDATE affects zero rows.

This works particularly well when the value moves monotonically:

```text
10000
 ↓
10500
 ↓
12000
 ↓
15000
```

However, a dedicated version number is usually easier to reason about.

---

# 17. The ABA Problem

The ABA problem occurs when a value changes:

```text
A → B → A
```

and your concurrency check only sees the final value.

Example:

Initial:

```text
balance = 100
```

You read:

```text
balance = 100
```

Another transaction changes:

```text
100 → 50
```

Then another transaction changes:

```text
50 → 100
```

Now your check:

```sql
WHERE balance = 100
```

passes.

But the balance did change.

Your application cannot distinguish:

```text
100
```

from:

```text
100 → 50 → 100
```

---

# 18. How Version Numbers Solve ABA

Initial:

```text
balance = 100
version = 42
```

First change:

```text
balance = 50
version = 43
```

Second change:

```text
balance = 100
version = 44
```

Your old snapshot:

```text
balance = 100
version = 42
```

Your update checks:

```sql
WHERE version = 42
```

Current version:

```text
44
```

Therefore:

```text
0 rows affected
```

Even though the balance returned to 100, the version reveals that the row changed.

---

# 19. Optimistic Locking and Transactions

Optimistic locking does NOT replace transactions.

They solve different problems.

### Transaction

Provides:

* Atomicity
* Consistency
* Isolation behavior
* Durability

### Optimistic locking

Detects:

```text
"Someone changed the data I based my decision on."
```

They are often used together.

Example:

```text
BEGIN TRANSACTION

    UPDATE entity
    WHERE id = X
    AND version = 42

    Check affected rows

    INSERT related entity

COMMIT
```

---

# 20. Optimistic Locking Does Not Mean "No Database Locks"

This is a subtle but important distinction.

Optimistic locking means the application does not hold a lock during the entire:

```text
READ
 ↓
business logic
 ↓
WRITE
```

period.

The database will still use its normal internal locking mechanisms while executing the UPDATE.

Therefore:

```text
Optimistic locking
≠
No locks anywhere
```

Instead:

> We avoid holding an application-visible lock across the read/processing period.

---

# 21. Why Optimistic Locking Can Be Faster

Consider 1,000 users editing different products.

Most operations don't conflict.

With pessimistic locking:

```text
Lock
 ↓
Process
 ↓
Unlock
```

There is coordination overhead.

With optimistic concurrency:

```text
Read
 ↓
Process
 ↓
Conditional write
```

Most operations simply succeed.

Only conflicting operations pay the retry cost.

Therefore:

```text
Low contention
      ↓
Optimistic concurrency is attractive
```

---

# 22. The Retry Pattern

When a conflict occurs:

```text
UPDATE
  ↓
0 rows
  ↓
Conflict
  ↓
Read fresh data
  ↓
Recalculate
  ↓
Retry
```

Example:

```text
Attempt 1
---------
Read version 42
Write WHERE version = 42
Conflict

Attempt 2
---------
Read version 43
Recalculate
Write WHERE version = 43
Success
```

---

# 23. Do NOT Blindly Retry

A retry does not necessarily mean:

```text
"Run the exact same SQL again."
```

Usually the application needs to:

```text
READ AGAIN
     ↓
UNDERSTAND NEW STATE
     ↓
RECALCULATE BUSINESS DECISION
     ↓
WRITE AGAIN
```

Example:

You read:

```text
price = ₹100
```

and decide:

```text
apply 10% discount
```

Someone changes the price to:

```text
₹200
```

Your version check fails.

After rereading, the business rule might need to be recalculated.

The retry may therefore produce:

```text
₹180
```

rather than simply replaying an old decision.

---

# 24. Optimistic Locking and Lost Updates

Suppose:

```text
balance = 100
version = 42
```

Alice reads:

```text
100 / version 42
```

Bob reads:

```text
100 / version 42
```

Alice calculates:

```text
100 + 50 = 150
```

Bob calculates:

```text
100 + 20 = 120
```

Without concurrency protection:

```text
Alice → 150
Bob   → 120
```

Final result:

```text
120
```

Alice's update was lost.

---

## With Optimistic Locking

Alice:

```sql
UPDATE accounts
SET balance = 150,
    version = version + 1
WHERE id = 1
AND version = 42;
```

Success.

Database:

```text
balance = 150
version = 43
```

Bob:

```sql
UPDATE accounts
SET balance = 120,
    version = version + 1
WHERE id = 1
AND version = 42;
```

Fails:

```text
0 rows affected
```

The stale write is prevented.

---

# 25. Optimistic Locking vs Atomic Updates

Not every concurrency problem requires a version column.

Sometimes the operation itself can be expressed atomically.

For example:

```sql
UPDATE inventory
SET stock = stock - 1
WHERE product_id = 10
AND stock > 0;
```

This says:

> "Decrease stock only if stock is greater than zero."

This is a conditional atomic operation.

You may not need:

```text
SELECT
 ↓
version
 ↓
business logic
 ↓
UPDATE
```

for such a simple operation.

---

# 26. Two Important Patterns

## Pattern 1: Version-Based OCC

```sql
SELECT stock, version
FROM inventory
WHERE product_id = 10;
```

Then:

```sql
UPDATE inventory
SET stock = 4,
    version = version + 1
WHERE product_id = 10
AND version = 42;
```

Use this when your application makes a decision based on the state it read.

---

## Pattern 2: Atomic Conditional Update

```sql
UPDATE inventory
SET stock = stock - 1
WHERE product_id = 10
AND stock > 0;
```

Use this when the business operation can be safely expressed as one atomic database operation.

---

# 27. Low Contention vs High Contention

Optimistic locking works best when conflicts are relatively rare.

### Low contention

```text
1000 operations
10 conflicts
```

Optimistic locking is attractive.

### High contention

```text
1000 operations
800 conflicts
```

Optimistic locking may become expensive.

You could end up with:

```text
Read
 ↓
Write
 ↓
Conflict
 ↓
Read
 ↓
Write
 ↓
Conflict
 ↓
Read
 ↓
Write
```

The retry overhead can become significant.

---

# 28. When to Prefer Optimistic Locking

Optimistic locking is a good choice when:

* Contention is low.
* Most transactions modify different rows.
* Conflicts are relatively rare.
* Retry is inexpensive.
* The application can safely recalculate after a conflict.
* Holding locks for a long time would be expensive.
* Users can tolerate conflict responses.
* The entity has a clear version/concurrency token.

Examples:

```text
Product editing
Profile editing
Document editing
Configuration updates
Admin dashboards
CRUD applications
Order updates with relatively low contention
```

---

# 29. When to Be Careful With Optimistic Locking

Be careful when:

* Many users compete for the same row.
* Conflicts are extremely frequent.
* Retry is expensive.
* The operation has side effects.
* The business operation is difficult to recalculate.
* Users cannot tolerate losing their work.
* The invariant spans multiple rows.

Examples:

```text
Highly contested inventory
Scarce resources
Popular event with one remaining seat
Financial operations
High-contention counters
```

The correct solution may instead be:

* Atomic updates
* Pessimistic locking
* Database constraints
* Serializable transactions
* Other coordination mechanisms

---

# 30. The Biggest Limitation: Multi-Row Invariants

Optimistic locking protects the rows whose versions you check.

But sometimes the business rule spans multiple rows.

Example:

```text
doctors
----------------
Alice → on_call = TRUE
Bob   → on_call = TRUE
```

Business rule:

> At least one doctor must remain on call.

Alice reads:

```text
Alice = TRUE
Bob = TRUE
```

Bob also reads:

```text
Alice = TRUE
Bob = TRUE
```

Alice decides:

```text
"I can turn myself off because Bob is on."
```

Bob decides:

```text
"I can turn myself off because Alice is on."
```

Alice updates:

```text
Alice → FALSE
```

Bob updates:

```text
Bob → FALSE
```

Neither update conflicts.

Final state:

```text
Alice = FALSE
Bob   = FALSE
```

Business rule is violated.

---

# 31. Write Skew

The previous example is called **write skew**.

The important point is:

```text
Alice writes Alice
Bob writes Bob
```

They never write the same row.

Yet:

```text
Alice's decision depended on Bob.
Bob's decision depended on Alice.
```

This exposes a limitation of simple row-level optimistic locking.

---

# 32. Read Set vs Write Set

A useful advanced concept is to distinguish:

### Read Set

Rows a transaction reads.

### Write Set

Rows a transaction modifies.

Alice:

```text
Read Set:
    Alice
    Bob

Write Set:
    Alice
```

Bob:

```text
Read Set:
    Alice
    Bob

Write Set:
    Bob
```

They have overlapping read sets:

```text
Alice ↔ Bob
```

but their write sets don't overlap:

```text
Alice writes Alice
Bob writes Bob
```

Therefore simple row-version checks may not detect the conflict.

---

# 33. Think About Invariants, Not Just Rows

When designing concurrency control, don't only ask:

> "Which row am I updating?"

Ask:

> "What data did my decision depend on?"

Example:

```text
Decision:
Allow doctor to go off call.

Depends on:
Whether another doctor is on call.
```

Another example:

```text
Decision:
Approve loan.

Depends on:
Credit score
Existing loans
Account balance
Risk limits
```

Another:

```text
Decision:
Allow booking.

Depends on:
Capacity
Reservations
Inventory
Availability
```

If the decision depends on multiple pieces of state, simple row-level optimistic locking may not be enough.

---

# 34. Optimistic Locking in HTTP APIs

Optimistic concurrency is not limited to databases.

HTTP uses a similar concept through:

```text
ETag
```

Suppose:

```http
GET /products/123
```

Server responds:

```http
ETag: "abc123"
```

Client modifies the product and sends:

```http
PUT /products/123
If-Match: "abc123"
```

This means:

> "Apply this update only if the resource is still version abc123."

If someone already changed it, the server can return:

```text
412 Precondition Failed
```

The concepts map closely:

```text
Database OCC             HTTP

version = 42             ETag = "abc123"

WHERE version = 42       If-Match: "abc123"

0 rows affected          412 Precondition Failed
```

---

# 35. Optimistic Locking Mental Model

Think of a version as representing:

```text
"The exact generation of the data I observed."
```

Suppose:

```text
version = 42
```

You are effectively saying:

> "My decision was made using generation 42."

Then:

```sql
WHERE version = 42
```

means:

> "Don't execute my decision if the entity has moved to another generation."

Therefore:

```text
READ
 ↓
capture version
 ↓
make decision
 ↓
conditional WRITE
 ↓
check affected rows
```

---

# 36. Complete Optimistic Concurrency Flow

```text
                 ┌─────────────────┐
                 │   Read Entity   │
                 └────────┬────────┘
                          │
                          ↓
                  Capture Version
                          │
                          ↓
                 Business Processing
                          │
                          ↓
                 Conditional UPDATE
                          │
                 ┌────────┴────────┐
                 │                 │
                 ↓                 ↓
          1 row affected       0 rows
                 │                 │
                 ↓                 ↓
              SUCCESS          CONFLICT
                                   │
                            ┌──────┴──────┐
                            │             │
                            ↓             ↓
                          Retry        Reject
                            │
                            ↓
                       Read Again
                            │
                            ↓
                   Recalculate Decision
                            │
                            ↓
                         Retry
```

---

# 37. Generic SQL Pattern

The standard pattern is:

```sql
-- Step 1: Read

SELECT id, data, version
FROM table
WHERE id = ?;
```

Application receives:

```text
version = 42
```

Then:

```sql
-- Step 2: Conditional update

UPDATE table
SET data = ?,
    version = version + 1
WHERE id = ?
AND version = 42;
```

Then:

```text
if affected_rows == 1:
    SUCCESS

if affected_rows == 0:
    CONCURRENCY CONFLICT
```

---

# 38. Pseudocode

```text
entity = READ(id)

oldVersion = entity.version

newState = performBusinessLogic(entity)

affectedRows =
    UPDATE(
        id,
        newState,
        WHERE version = oldVersion
    )

if affectedRows == 0:
    return CONFLICT

return SUCCESS
```

With retry:

```text
for attempt in 1..MAX_RETRIES:

    entity = READ(id)

    newState = calculate(entity)

    affectedRows =
        UPDATE(
            id,
            newState,
            WHERE version = entity.version
        )

    if affectedRows == 1:
        return SUCCESS

return CONFLICT
```

---

# 39. Important Design Questions

When implementing optimistic locking, ask:

### 1. What is my concurrency token?

Prefer:

```text
version
```

over arbitrary business values when possible.

### 2. When does the version change?

It should change whenever relevant state changes.

### 3. What does zero affected rows mean?

Usually:

```text
Concurrency conflict
```

### 4. Should I retry?

Only if retrying is safe and useful.

### 5. What happens after a retry?

Usually:

```text
Read fresh state
→ Recalculate
→ Retry
```

### 6. Is the operation transactional?

If multiple writes must succeed together, use a transaction.

### 7. Does the invariant span multiple rows?

If yes, simple row-level OCC may not be sufficient.

---

# 40. Common Mistakes

## Mistake 1: Forgetting the version check

Bad:

```sql
UPDATE users
SET name = ?
WHERE id = ?;
```

Good:

```sql
UPDATE users
SET name = ?,
    version = version + 1
WHERE id = ?
AND version = ?;
```

---

## Mistake 2: Ignoring affected rows

Bad:

```text
UPDATE
INSERT
COMMIT
```

without checking whether UPDATE succeeded.

Good:

```text
UPDATE
 ↓
check affected rows
 ↓
0 → rollback
1 → continue
```

---

## Mistake 3: Blindly retrying

Don't blindly replay a stale decision.

Instead:

```text
Read fresh state
 ↓
Recalculate
 ↓
Retry
```

---

## Mistake 4: Using low-precision timestamps

Timestamp-based OCC can fail if two writes receive the same timestamp.

---

## Mistake 5: Assuming row-level OCC protects everything

It doesn't automatically protect:

```text
multi-row invariants
```

This is where write skew becomes important.

---

## Mistake 6: Using OCC under extreme contention

If almost every transaction conflicts:

```text
Optimistic
    ↓
Retry
    ↓
Retry
    ↓
Retry
```

may be worse than coordinating access upfront.

---

# 41. Optimistic Locking Decision Framework

Use this mental decision tree:

```text
Is there shared mutable state?
        │
        ↓
Can the operation be expressed
as one atomic conditional UPDATE?
        │
     ┌──┴──┐
    YES    NO
     │      │
     ↓      ↓
 Atomic   Is contention low?
 Update       │
          ┌───┴───┐
         YES      NO
          │        │
          ↓        ↓
      Optimistic  Consider
      Locking     Pessimistic
                  / Serializable /
                  other coordination
```

Then ask:

```text
Does the business invariant span
multiple rows?
        │
       YES
        ↓
Simple row-level OCC may not
be sufficient.
```

---

# 42. Key Comparison

| Property                             | Pessimistic Locking                   | Optimistic Locking         |
| ------------------------------------ | ------------------------------------- | -------------------------- |
| Basic strategy                       | Prevent conflict                      | Detect conflict            |
| Blocks other transactions            | Often yes                             | Usually no                 |
| Uses version                         | Not required                          | Usually yes                |
| Best for                             | High contention                       | Low contention             |
| Conflict handling                    | Waiting                               | Retry/reject               |
| Lock held during processing          | Potentially                           | No application-level lock  |
| Retry required                       | Usually no                            | Often                      |
| Scalability under low contention     | Can be less efficient                 | Often good                 |
| Scalability under extreme contention | Can be better                         | Can degrade due to retries |
| Multi-row invariant                  | Can help depending on locks/isolation | Simple row OCC may fail    |

---

# 43. Interview Answer

If asked:

> What is optimistic locking?

A strong answer:

> Optimistic locking is a concurrency-control technique where we assume conflicts are relatively rare and therefore don't hold a lock while reading and processing data. Instead, the row contains a concurrency token, usually an incrementing version number. The application reads the row and remembers its version, then performs a conditional UPDATE such as `WHERE id = ? AND version = ?`, incrementing the version on success. If one row is affected, the update succeeded. If zero rows are affected, another transaction modified the row and our snapshot is stale, so we can retry using fresh data or return a conflict. It works well under low contention but can become inefficient under heavy contention. Also, row-level optimistic locking doesn't automatically protect invariants spanning multiple rows, where anomalies such as write skew can occur.

---

# 44. One-Minute Revision

Remember these 10 points:

```text
1. Optimistic locking assumes conflicts are rare.

2. It doesn't hold a lock during the read → process period.

3. A version column is commonly used.

4. Read version = 42.

5. Write using:
   WHERE id = X AND version = 42

6. Increment version after successful update.

7. 1 affected row = success.

8. 0 affected rows = concurrency conflict.

9. Retry means:
   READ → RECALCULATE → WRITE
   not blindly repeat the old decision.

10. Row-level OCC does not automatically protect
    multi-row invariants / write skew.
```

---

# 45. The Core Mental Model

The entire concept can be reduced to this:

```text
              I READ VERSION 42
                     │
                     ↓
              Make my decision
                     │
                     ↓
       "Apply my decision ONLY IF
        the row is still version 42"
                     │
             ┌───────┴───────┐
             ↓               ↓
        Version 42       Version ≠ 42
             │               │
             ↓               ↓
          SUCCESS         CONFLICT
                             │
                             ↓
                     Read fresh state
                             │
                             ↓
                       Recalculate
                             │
                             ↓
                           Retry
```

The fundamental principle is:

> **Optimistic concurrency doesn't prevent someone else from changing the data. It prevents you from unknowingly overwriting or acting on data that changed after your read.**

And the next important concept after this is:

```text
Optimistic Locking
       ↓
Lost Update
       ↓
Write Skew
       ↓
Isolation Levels
       ↓
Repeatable Read
       ↓
Snapshot Isolation
       ↓
Serializable
       ↓
Predicate / Range Locking
```

That progression explains the exact "blind spot" mentioned in your original material.

```
```
