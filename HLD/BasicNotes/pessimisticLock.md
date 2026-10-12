# Pessimistic Locking — Detailed Notes

## 1. What is Pessimistic Locking?

**Pessimistic locking** is a database concurrency-control technique where a transaction **acquires a lock on data before working with it**, assuming that another transaction may try to modify the same data.

The basic idea is:

> **"I expect a conflict, so I'll protect the data before the conflict happens."**

The common SQL pattern is:

```sql
BEGIN TRANSACTION;

SELECT *
FROM seats
WHERE seat_number = 'A15'
FOR UPDATE;

-- application makes its decision

UPDATE seats
SET status = 'sold'
WHERE seat_number = 'A15';

COMMIT;
```

The important part is:

```sql
FOR UPDATE
```

It tells the database that the selected rows are being read as part of a transaction that may update them, so other transactions cannot perform **conflicting operations** on those rows while the lock is held.

---

# 2. The Problem Pessimistic Locking Solves

Suppose we have:

```text
Seat A15
status = available
```

Two users try to book it simultaneously.

Without proper concurrency control:

```text
Transaction 1                 Transaction 2

READ A15 = available          READ A15 = available
       |                              |
       |                              |
   decide to book                 decide to book
       |                              |
       |                              |
   UPDATE A15                  UPDATE A15
       |                              |
       +------------ conflict --------+
```

Both transactions may make decisions based on the same old state.

This is a classic **read-modify-write race**.

Pessimistic locking changes the sequence:

```text
Transaction 1                 Transaction 2

LOCK A15
   |
READ A15
   |
DECIDE
   |
UPDATE
   |
COMMIT
   |
UNLOCK
                              |
                              LOCK A15
                              |
                              READ latest state
                              |
                              DECIDE
```

Transaction 2 has to wait for the conflicting lock to be released.

---

# 3. The Simplest Mental Model

Think about a bathroom with one lock.

```text
Bathroom → LOCKED 🔒
```

Suraj enters and locks it.

Rahul arrives:

```text
Rahul → Can I use it?
Bathroom → No, someone is already using it.
Rahul → waits...
```

Suraj finishes:

```text
Bathroom → UNLOCKED 🔓
```

Rahul can now use it.

Pessimistic locking works similarly:

```text
LOCK
  ↓
WORK WITH DATA
  ↓
UPDATE
  ↓
COMMIT
  ↓
UNLOCK
```

---

# 4. What Does `FOR UPDATE` Actually Mean?

Consider:

```sql
SELECT *
FROM seats
WHERE seat_number = 'A15'
FOR UPDATE;
```

The important point is:

> `FOR UPDATE` does **not** simply mean "nobody can read this row."

Instead, it means that the transaction is acquiring a lock that prevents **conflicting operations**, such as another transaction trying to update/delete the row or acquire a conflicting locking read.

So think:

```text
Normal SELECT
    ↓
"I just want to look at it."

SELECT ... FOR UPDATE
    ↓
"I am looking at it because I may modify it,
and I want protection while I do that."
```

---

# 5. Can Another Transaction Read a Locked Row?

This is one of the most important concepts.

Suppose Transaction 1 executes:

```sql
BEGIN;

SELECT *
FROM seats
WHERE seat_number = 'A15'
FOR UPDATE;
```

Now A15 is locked for update by Transaction 1.

Transaction 2 executes a normal read:

```sql
SELECT *
FROM seats
WHERE seat_number = 'A15';
```

A normal read will generally **not have to wait merely because Transaction 1 holds a row lock**.

Modern databases such as PostgreSQL and InnoDB in MySQL use MVCC (Multi-Version Concurrency Control), which allows many ordinary reads to proceed without blocking behind a write lock.

However, if Transaction 2 attempts a conflicting operation:

```sql
UPDATE seats
SET status = 'sold'
WHERE seat_number = 'A15';
```

or:

```sql
SELECT *
FROM seats
WHERE seat_number = 'A15'
FOR UPDATE;
```

then Transaction 2 will generally have to wait for Transaction 1's conflicting lock to be released.

### Important distinction

| Operation by Transaction 2 | Typical behavior |
|---|---|
| Normal `SELECT` | Can generally read |
| `SELECT ... FOR UPDATE` | Waits for conflicting lock |
| `UPDATE` | Waits for conflicting lock |
| `DELETE` | Waits for conflicting lock |

Exact behavior depends on the database engine, isolation level, lock mode, indexes, and query plan.

The safest mental model is:

> **A row lock prevents conflicting operations; it does not necessarily prevent every read.**

---

# 6. Why Do We Need a Transaction?

Pessimistic locking is normally used inside a transaction:

```sql
BEGIN;

SELECT ... FOR UPDATE;

UPDATE ...;

COMMIT;
```

Why?

Because the database needs to know:

> "How long should I hold this lock?"

The answer is normally until the transaction ends:

```text
BEGIN
  ↓
Acquire lock
  ↓
Read
  ↓
Decide
  ↓
Update
  ↓
COMMIT / ROLLBACK
  ↓
Lock released
```

So the transaction defines the protected period.

---

# 7. Seat Booking Example

Suppose a concert has:

```text
A15 → available
A16 → available
A17 → available
A18 → available
A19 → available
```

A customer wants **four consecutive seats**.

The application needs to:

1. Read the seats.
2. Find four adjacent available seats.
3. Claim them.

The difficult part is step 2.

The application may need to inspect multiple rows and make a decision.

A simplified implementation:

```sql
BEGIN;

SELECT seat_number, status
FROM seats
WHERE concert_id = 'weeknd_tour'
  AND section = 'floor'
  AND status = 'available'
FOR UPDATE;

-- Application examines the returned rows.
-- It finds A15, A16, A17, A18.

UPDATE seats
SET status = 'sold',
    user_id = 'user123'
WHERE concert_id = 'weeknd_tour'
  AND seat_number IN ('A15', 'A16', 'A17', 'A18');

COMMIT;
```

The conceptual sequence is:

```text
Transaction 1

LOCK available seats
       ↓
Read seats
       ↓
Find A15-A18
       ↓
Update A15-A18
       ↓
COMMIT
       ↓
Release locks
```

A second transaction trying to perform conflicting operations on those rows must wait.

---

# 8. Why Is the Seat Example More Difficult Than a Simple Counter?

Consider an inventory counter:

```text
quantity = 5
```

We want to sell one item.

The requirement can often be expressed as:

```sql
UPDATE inventory
SET quantity = quantity - 1
WHERE product_id = 123
  AND quantity > 0;
```

The database can perform the check and update atomically:

```text
Is quantity > 0?
       ↓
      YES
       ↓
quantity = quantity - 1
```

There is no need for the application to separately:

```text
READ
 ↓
THINK
 ↓
WRITE
```

But four consecutive seats require more application-level reasoning:

```text
Read seats
    ↓
Analyze arrangement
    ↓
Find four adjacent seats
    ↓
Claim them
```

This creates a larger read-decide-write problem.

Pessimistic locking can protect the rows during that decision.

---

# 9. Pessimistic Locking vs Conditional UPDATE

This distinction is extremely important.

## Conditional UPDATE

Use this when the condition can be expressed directly in SQL.

Example:

```sql
UPDATE inventory
SET quantity = quantity - 1
WHERE product_id = 123
  AND quantity > 0;
```

Mental model:

> **"Database, check the condition and update atomically."**

```text
CHECK + UPDATE
      ↓
   Atomic
```

---

## Pessimistic Locking

Use this when you need to inspect data, make a more complicated decision, and then update it.

```sql
BEGIN;

SELECT *
FROM seats
WHERE ...
FOR UPDATE;

-- Application decides what to do.

UPDATE seats
SET ...
WHERE ...;

COMMIT;
```

Mental model:

> **"Lock the relevant data first, then let me safely inspect and decide."**

```text
LOCK
 ↓
READ
 ↓
DECIDE
 ↓
WRITE
 ↓
UNLOCK
```

---

# 10. The Decision Rule

A useful decision tree is:

```text
Do I have concurrent access?
        |
       YES
        |
Is there a race condition?
        |
   +----+----+
   |         |
  NO        YES
   |         |
No lock     Can the database perform
            the operation atomically?
                 |
            +----+----+
            |         |
           YES       NO
            |         |
 Conditional        Consider
 UPDATE             pessimistic
                    locking
```

If pessimistic locking is being considered, ask one more question:

```text
Are conflicts frequent?
       |
   +---+---+
   |       |
  YES     NO
   |       |
Pessimistic  Consider
locking      optimistic
may fit      concurrency
             control
```

---

# 11. When Should You Use Pessimistic Locking?

## Case 1: Read → Decide → Write

This is the strongest signal.

Example:

```text
Read several rows
      ↓
Analyze them
      ↓
Make a decision
      ↓
Update rows
```

If another transaction changing those rows during the process could invalidate your decision, pessimistic locking may be appropriate.

---

## Case 2: Multiple Rows Must Be Consistent

Example:

> "Reserve Product A, Product B, and Product C as one bundle only if all three are available."

You might need:

```text
Product A → available
Product B → available
Product C → available
       ↓
All available?
       ↓
Reserve all
```

If another transaction can modify one of those resources while you are making the decision, locking may be appropriate.

---

## Case 3: Finding a Resource Based on Complex Conditions

Example:

> "Find a suitable room that satisfies several conditions, then reserve it."

The application may need to:

```text
Read candidate rooms
      ↓
Check availability
      ↓
Check other constraints
      ↓
Choose one
      ↓
Reserve it
```

A pessimistic lock can protect the selected resources while the decision is being made.

---

## Case 4: High Contention

Pessimistic locking can be useful when many transactions are expected to compete for the same scarce resource.

Examples:

- Highly popular concert tickets
- Limited inventory during a flash sale
- Reservation systems
- Scarce appointment slots
- Shared resource allocation

The exact design still depends on workload and database behavior.

---

# 12. When Should You Avoid Pessimistic Locking?

## Case 1: An Atomic Conditional UPDATE Is Enough

If you can safely write:

```sql
UPDATE inventory
SET quantity = quantity - 1
WHERE product_id = 123
AND quantity > 0;
```

don't automatically replace it with:

```sql
BEGIN;

SELECT quantity
FROM inventory
WHERE product_id = 123
FOR UPDATE;

UPDATE inventory
SET quantity = quantity - 1
WHERE product_id = 123;

COMMIT;
```

The second approach introduces unnecessary locking and a longer transaction.

General rule:

> **If one atomic database statement can safely solve the problem, prefer it over unnecessary explicit locking.**

---

# 13. Avoid Locking Too Many Rows

Bad example:

```sql
SELECT *
FROM seats
WHERE concert_id = 123
FOR UPDATE;
```

Suppose the concert has 50,000 seats.

You may have locked 50,000 rows just to reserve four.

That creates unnecessary contention:

```text
Transaction 1 → locks huge number of seats
Transaction 2 → waits
Transaction 3 → waits
Transaction 4 → waits
...
```

Try to keep the locked set as small as correctness allows.

---

# 14. Avoid Holding Locks for Too Long

Locks should generally be held for as short a time as possible.

Bad:

```text
BEGIN
 ↓
LOCK
 ↓
Call payment API
 ↓
Wait 2 seconds
 ↓
Call another service
 ↓
Wait
 ↓
Do processing
 ↓
UPDATE
 ↓
COMMIT
 ↓
UNLOCK
```

During all that time, other transactions may be waiting.

A better architecture is often:

```text
Do expensive external work
        ↓
Take lock
        ↓
Perform short critical section
        ↓
UPDATE
        ↓
COMMIT
```

However, the exact payment/reservation workflow needs careful business design because external payment and database state must also be coordinated correctly.

The key rule is:

> **Keep the database transaction and locked critical section as short as practical.**

---

# 15. Slow I/O Inside a Transaction

Be especially careful with:

- HTTP calls
- Payment APIs
- Third-party services
- File operations
- Long-running calculations
- Waiting for user input

For example:

```text
LOCK ROW
   ↓
Call payment provider
   ↓
Payment provider takes 3 seconds
   ↓
Other transactions wait
```

This can turn a small lock into a major bottleneck.

---

# 16. Deadlocks

Another important failure mode is a **deadlock**.

Suppose:

```text
Transaction 1:
Lock A
 ↓
tries to lock B
 ↓
WAIT

Transaction 2:
Lock B
 ↓
tries to lock A
 ↓
WAIT
```

Now:

```text
Transaction 1 → waiting for Transaction 2
Transaction 2 → waiting for Transaction 1
```

Neither can continue.

Conceptually:

```text
      locks A
       ↓
Transaction 1
       ↓
   wants B
       ↑
       |
   holds B
       |
Transaction 2
       ↑
   wants A
```

This forms a cycle.

---

# 17. How to Reduce Deadlocks

A common strategy is:

> **Acquire locks in a consistent order.**

For example, if you need rows 10 and 20:

```text
Always lock:
10 → 20
```

Never sometimes:

```text
10 → 20
```

and other times:

```text
20 → 10
```

For multiple rows, use a deterministic ordering such as sorted IDs.

Even with good ordering, deadlocks can still occur in complex systems, so applications should generally be prepared to handle database deadlock errors appropriately, often by retrying the transaction where safe.

---

# 18. Pessimistic Locking Is Not "Reserve Forever"

A lock normally exists only for the duration of the transaction.

Example:

```sql
BEGIN;

SELECT *
FROM seats
WHERE seat_number = 'A15'
FOR UPDATE;

UPDATE seats
SET status = 'sold'
WHERE seat_number = 'A15';

COMMIT;
```

After `COMMIT`:

```text
Lock → released
```

If you want a seat to remain reserved for 10 minutes, that is a **business-level reservation**, not simply a database row lock.

For that, you would typically store something like:

```text
status = reserved
reserved_until = timestamp
```

A database lock and a business reservation are two different concepts.

---

# 19. Lock vs Transaction

These concepts are related but different.

### Transaction

A transaction defines a unit of database work:

```text
BEGIN
 ↓
operations
 ↓
COMMIT / ROLLBACK
```

### Lock

A lock controls concurrency around specific database resources.

```text
Row → 🔒
```

Pessimistic locking usually means:

```text
Transaction
    +
Explicit locking
```

---

# 20. Pessimistic Locking vs Optimistic Locking

These two approaches have opposite philosophies.

## Pessimistic

> **"A conflict may happen, so I'll prevent it before it happens."**

```text
LOCK
 ↓
WORK
 ↓
COMMIT
```

---

## Optimistic

> **"Conflicts are probably rare, so I'll let transactions proceed and detect conflicts if they happen."**

A common optimistic pattern uses a version number:

```text
id = 123
quantity = 10
version = 5
```

Application reads:

```text
quantity = 10
version = 5
```

Then updates:

```sql
UPDATE inventory
SET quantity = 9,
    version = 6
WHERE id = 123
  AND version = 5;
```

If another transaction already changed the row:

```text
version = 6
```

the update affects zero rows.

The application detects:

```text
Conflict!
```

---

# 21. Pessimistic vs Optimistic vs Conditional UPDATE

| Approach | Mental Model | Best Fit |
|---|---|---|
| Conditional UPDATE | "Check and update atomically" | Simple conditions |
| Pessimistic locking | "Lock it before I work with it" | Complex read-decide-write with contention |
| Optimistic locking | "Assume conflict is rare; detect it" | Low-contention concurrent updates |

Examples:

### Simple inventory decrement

```sql
UPDATE inventory
SET quantity = quantity - 1
WHERE id = 123
AND quantity > 0;
```

**Conditional UPDATE**

### Four adjacent seats

```sql
SELECT ...
FOR UPDATE;
```

Then application decides which four seats to reserve.

**Pessimistic locking may be appropriate**

### Updating an entity with a version number

```sql
UPDATE products
SET price = 100,
    version = version + 1
WHERE id = 123
AND version = 7;
```

**Optimistic locking**

---

# 22. The Core Read-Modify-Write Problem

A lot of concurrency problems have this structure:

```text
READ
 ↓
APPLICATION THINKS / DECIDES
 ↓
WRITE
```

The dangerous part is the gap:

```text
READ
 ↓
   ← another transaction can change data here
 ↓
WRITE
```

Pessimistic locking protects this critical section:

```text
LOCK
 ↓
READ
 ↓
DECIDE
 ↓
WRITE
 ↓
UNLOCK
```

Conditional updates remove the gap by moving the decision into the database:

```text
CHECK + WRITE
     ↓
  ATOMIC
```

Optimistic locking allows the gap but detects whether somebody changed the data:

```text
READ
 ↓
DECIDE
 ↓
WRITE IF VERSION IS STILL THE SAME
```

This three-way comparison is one of the most useful ways to understand database concurrency.

---

# 23. Practical Examples

## Example A — Simple Inventory

Requirement:

> Sell one item only if stock exists.

Preferred approach:

```sql
UPDATE inventory
SET quantity = quantity - 1
WHERE product_id = 123
AND quantity > 0;
```

Why?

The condition is simple and can be evaluated atomically.

No need to introduce an explicit row lock unless there is some additional business logic requiring it.

---

## Example B — Four Consecutive Seats

Requirement:

> Find four adjacent seats and reserve them together.

Potential approach:

```sql
BEGIN;

SELECT seat_number, status
FROM seats
WHERE concert_id = 123
AND section = 'floor'
AND status = 'available'
FOR UPDATE;

-- application finds four adjacent seats

UPDATE seats
SET status = 'reserved'
WHERE seat_number IN (...);

COMMIT;
```

Why?

The application needs to reason about multiple rows and their arrangement.

---

## Example C — Bundle Reservation

Requirement:

> Reserve A + B + C only if all are available.

Potential approach:

```text
LOCK relevant rows
      ↓
READ their state
      ↓
Check that all are available
      ↓
UPDATE all
      ↓
COMMIT
```

Why?

The business decision spans multiple resources.

---

# 24. Common Mistakes

### Mistake 1: Thinking `FOR UPDATE` means nobody can read

Incorrect mental model:

> "The row is locked, so nobody can read it."

Better:

> "The row is protected against conflicting operations."

---

### Mistake 2: Locking everything

```sql
SELECT *
FROM table
FOR UPDATE;
```

This may unnecessarily serialize a large portion of your workload.

---

### Mistake 3: Holding locks while calling external APIs

Avoid long transactions around slow network calls.

---

### Mistake 4: Using locks when an atomic update is enough

If this works:

```sql
UPDATE ...
WHERE condition;
```

don't automatically introduce:

```sql
SELECT ... FOR UPDATE;
```

---

### Mistake 5: Ignoring deadlocks

If multiple rows are locked in inconsistent order, deadlocks can occur.

Use deterministic ordering and make safe transactions retryable where appropriate.

---

### Mistake 6: Confusing database locks with business reservations

A database lock is usually temporary.

A "reserved for 10 minutes" state is persistent business data.

They solve different problems.

---

# 25. The Most Important Rules to Remember

## Rule 1

> **Use pessimistic locking when you need to protect a read → decide → write operation from concurrent changes.**

## Rule 2

> **If the database can perform the condition and update atomically, prefer a conditional/atomic UPDATE.**

## Rule 3

> **Don't lock more rows than necessary.**

## Rule 4

> **Don't hold locks longer than necessary.**

## Rule 5

> **Avoid slow external work inside the locked transaction.**

## Rule 6

> **Acquire multiple locks in a consistent order to reduce deadlocks.**

## Rule 7

> **A normal read is not the same as a locking read.**

## Rule 8

> **A database lock is not the same thing as a business reservation.**

---

# 26. Interview-Friendly Answer

If an interviewer asks:

> **"When would you use pessimistic locking?"**

A strong answer is:

> "I would use pessimistic locking when I have a concurrent read-modify-write operation where another transaction changing the same data could invalidate my decision. It's particularly useful when the decision involves multiple rows or application-level logic that cannot easily be expressed as one atomic database statement. For example, finding and reserving four consecutive seats. I would avoid it when a conditional or atomic UPDATE can solve the problem because unnecessary locking reduces concurrency and can introduce contention and deadlocks."

---

# 27. The 10-Second Mental Model

Remember these three patterns:

### Conditional UPDATE

```text
CHECK + UPDATE
     ↓
  ATOMIC
```

> "Database, check my condition and update it."

### Pessimistic Locking

```text
LOCK
 ↓
READ
 ↓
DECIDE
 ↓
WRITE
 ↓
UNLOCK
```

> "I'm expecting a conflict, so protect the data while I work."

### Optimistic Locking

```text
READ
 ↓
DECIDE
 ↓
UPDATE IF VERSION MATCHES
 ↓
SUCCESS / CONFLICT
```

> "I'll assume conflicts are rare and detect them if they happen."

---

# Final Takeaway

Pessimistic locking is not simply:

> **"Lock the row so nobody can access it."**

The better mental model is:

> **"I need to safely work with this data and make a decision before updating it, so I'm going to acquire a lock that prevents conflicting operations until my transaction finishes."**

The most important question before using it is:

> **"Can I solve this safely with one atomic database operation instead?"**

If yes, prefer the simpler atomic operation.

If no, and the application needs a protected **read → decide → write** section, pessimistic locking becomes a strong candidate.
