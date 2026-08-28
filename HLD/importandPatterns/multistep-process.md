# Multi-Step Processes — The Problem

## 🎯 Core Problem

A **multi-step process** is a business operation that requires multiple steps across different services/systems.

The challenge is:

> **How do we reliably coordinate these steps when any step can fail, timeout, take a long time, or succeed without our application knowing — and our own application can crash at any point?**

---

## 🛒 Example: Order Fulfillment

```text
Charge Payment
      ↓
Reserve Inventory
      ↓
Create Shipping Label
      ↓
Wait for Warehouse Pickup
      ↓
Pick & Pack
      ↓
Ship
      ↓
Send Confirmation
```

This looks simple, but each step may involve a different service.

```text
Order Service
    ├── Payment Service
    ├── Inventory Service
    ├── Shipping Service
    ├── Warehouse System
    └── Email Service
```

---

## 💥 Why Is This Hard?

### 1. Steps can fail

```text
Payment          ✅
Inventory        ❌
```

Now we may need to **refund the payment**.

```text
Payment
   ↓
Inventory ❌
   ↓
Refund Payment
```

What if the refund also fails?

We now need retries and recovery.

---

### 2. Timeouts are ambiguous

Suppose we send:

```text
Charge $100
```

The payment service processes it successfully, but the response is lost.

Our service sees:

```text
TIMEOUT
```

Did the payment happen?

```text
Maybe YES
Maybe NO
```

Retrying blindly could charge the customer twice.

---

### 3. The application can crash

Suppose:

```text
Payment        ✅
Inventory      ✅
Shipping       ⏳
```

Our server crashes.

When it comes back, it needs to know:

> "Where did I stop?"

The process must therefore have **persistent state**.

---

### 4. Some steps take a long time

Example:

```text
Create Label
     ↓
Wait for Warehouse
     ↓
Worker picks item 8 hours later
```

We cannot keep an HTTP request/server process alive for 8 hours.

We need to:

```text
Save State
   ↓
Wait
   ↓
Receive Event
   ↓
Resume Process
```

---

### 5. Distributed systems cannot use one normal DB transaction

We cannot simply do:

```text
BEGIN TRANSACTION

Charge Payment
Reserve Inventory
Create Shipping Label
Wait for Warehouse
Send Email

COMMIT
```

because these operations belong to **different systems** that we don't control.

---

## 🧠 The Real Problem

The problem is **NOT**:

> "How do I execute A → B → C → D?"

The problem is:

> **How do I reliably coordinate a long-running sequence of distributed operations when operations can fail, timeout, retry, partially succeed, or require waiting — while the application itself can crash?**

---

## 🔑 Mental Model

Whenever you hear:

**Multi-Step Process / Saga / Workflow / Durable Execution**

Think:

```text
Multiple Steps
      +
Multiple Systems
      +
Failures
      +
Retries
      +
Long Waits
      +
Crash Recovery
      =
Multi-Step Process Problem
```

### The key requirement

> **The business process must survive failures and interruptions without losing its state or producing incorrect results.**

---

## Example Failure Scenario

```text
Charge Payment       ✅
Reserve Inventory    ❌
       ↓
Need Refund
       ↓
Refund Service       ❌
       ↓
Retry later
```

The system must remember:

```text
Order #123
Payment = SUCCESS
Inventory = FAILED
Refund = PENDING
```

So it can recover and continue later.

---

## 🚀 Why We Need Special Patterns

Because simple sequential code becomes difficult to maintain when we add:

* retries
* timeouts
* failures
* compensation/refunds
* persistent state
* delayed execution
* human interaction
* crash recovery
* duplicate requests
* partial success

This is why systems use patterns such as:

```text
Workflow Engines
Event-Driven Sagas
Orchestration
Choreography
Durable Execution
State Machines
```

**First understand the problem. Then learn these solutions.**

# Multi-Step Processes / Workflows — Detailed Notes

> **Core idea:**
> A multi-step process is a business operation that requires multiple distributed steps to complete. The difficult part is not executing the steps; it is making the entire process **reliable in the presence of failures, retries, timeouts, crashes, asynchronous callbacks, and long waits**.

---

# 1. What Is a Multi-Step Process?

A business operation often consists of multiple dependent steps.

For example, an e-commerce order:

```text
Charge Payment
      ↓
Reserve Inventory
      ↓
Create Shipping Label
      ↓
Wait for Warehouse Pickup
      ↓
Pick & Pack
      ↓
Ship
      ↓
Send Confirmation Email
```

Each step may involve a different service:

```text
                     ┌─────────────────┐
                     │ Payment Service │
                     └────────┬────────┘
                              │
                              │
Client → API Server ──────────┼──────────→ Inventory Service
                              │
                              ├──────────→ Shipping Service
                              │
                              └──────────→ Email Service
```

The process looks simple when everything works.

The difficulty starts when **something goes wrong**.

---

# 2. The Real Problem

The problem is NOT:

> "How do I execute A → B → C → D?"

That is easy.

The real problem is:

> **How do I reliably coordinate a long-running sequence of distributed operations when any operation can fail, timeout, retry, partially succeed, or require waiting — while the application itself can crash?**

This creates several challenges:

* Server crashes
* Service failures
* Network timeouts
* Duplicate requests
* Partial success
* Asynchronous callbacks/webhooks
* Long-running operations
* Human actions
* Retries
* State persistence
* Recovery
* Compensation

---

# 3. Start With the Simplest Solution

The simplest implementation is to let one API server execute everything sequentially.

```text
Client
  |
  | Order
  ↓
API Server
  |
  ├── Payment Service
  |
  ├── Inventory Service
  |
  ├── Shipping Service
  |
  └── Email Service
```

Code might look like:

```ts
async function fulfillOrder(order: Order) {
    await chargePayment(order);
    await reserveInventory(order);
    await createShippingLabel(order);
    await sendConfirmationEmail(order);

    return { success: true };
}
```

Conceptually:

```text
A → B → C → D
```

This is perfectly reasonable as a starting point.

But now introduce failures.

---

# 4. Problem #1 — Server Crash

Suppose the process is:

```text
Charge Payment       ✅
Reserve Inventory    ⏳
Create Label         ⏳
Send Email           ⏳
```

Payment succeeds.

Then:

```text
💥 API SERVER CRASHES
```

The process running inside the server disappears.

When the server restarts, it may not know:

```text
Did payment succeed?
Did inventory succeed?
Which step was running?
Should payment be retried?
Should inventory be retried?
```

The problem is that the process state existed only in the memory of that server.

---

# 5. Why Simply Retrying Everything Is Dangerous

Suppose:

```text
Charge Payment
      ↓
SUCCESS
      ↓
Server crashes
```

When the server restarts, it might restart from the beginning:

```text
Charge Payment again
```

This could result in:

```text
-$1000
-$1000
```

The customer gets charged twice.

Therefore:

> **We cannot blindly restart a multi-step process from the beginning.**

We need to know which steps have already completed.

---

# 6. The Timeout Problem

Distributed systems introduce another subtle problem.

Suppose our application calls the payment service:

```text
API Server
    |
    | Charge $100
    ↓
Payment Service
```

The payment service processes the payment successfully.

But the network response is lost.

Our API server sees:

```text
TIMEOUT
```

What does the timeout mean?

### Possibility 1

Payment never happened.

```text
Payment = NOT DONE
```

### Possibility 2

Payment happened, but the response was lost.

```text
Payment = SUCCESS
```

The API server cannot necessarily distinguish between these cases.

If we retry blindly:

```text
Charge $100 again
```

we may duplicate the payment.

This is one reason distributed operations require:

* Idempotency
* Persistent state
* Request IDs
* Deduplication
* Careful retry policies

---

# 7. Solution #1 — Persist the Process State

The obvious improvement is to store workflow state in a database.

Instead of relying on server memory:

```text
API Server
    ↓
Memory
    ↓
Current Step
```

we use:

```text
API Server
    ↓
Order State DB
```

For example:

```text
orders
------------------------------------
order_id
payment_status
inventory_status
shipping_status
email_status
```

For Order #123:

```text
order_id         = 123
payment_status   = SUCCESS
inventory_status = PENDING
shipping_status  = PENDING
email_status     = PENDING
```

Now the state survives a server crash.

---

# 8. Why Persistent State Helps

Suppose:

```text
Payment          ✅
Inventory        ⏳
```

Then:

```text
💥 Server crashes
```

Another server can read:

```text
Order #123

Payment   = SUCCESS
Inventory = PENDING
```

and continue from:

```text
Reserve Inventory
```

instead of charging the customer again.

This is a huge improvement.

---

# 9. But Now We Have Another Problem — Who Continues the Workflow?

The database remembers:

```text
Order #123
Inventory = PENDING
```

But the database does not automatically execute:

```text
reserveInventory()
```

This is a critical distinction:

> **The database provides durable memory, but not workflow execution.**

The database knows:

```text
WHAT STATE ARE WE IN?
```

It does not automatically know:

```text
WHAT SHOULD I DO NEXT?
```

---

# 10. The Poller Problem

One solution is to create a poller/worker.

```text
                Order State DB
                     ↑
                     |
                  Poller
                     |
                     ↓
              Find pending work
```

Every few seconds:

```sql
SELECT *
FROM orders
WHERE status = 'PENDING';
```

The poller discovers:

```text
Order #123
Inventory = PENDING
```

and continues the process:

```text
Poller
   ↓
Order #123
   ↓
Inventory PENDING
   ↓
Reserve Inventory
```

This solves the "who continues the workflow?" problem.

But it introduces more complexity.

---

# 11. Problem — Two Workers Can Process the Same Order

Suppose there are multiple API servers/workers:

```text
Server 1
Server 2
Server 3
```

Both Server 1 and Server 2 query the database:

```text
Order #123
Inventory = PENDING
```

Both may think:

> "I should process this."

Now:

```text
Server 1
    ↓
Reserve Inventory


Server 2
    ↓
Reserve Inventory
```

The same operation may execute twice.

Therefore we need a mechanism to **claim ownership** of the work.

For example:

```text
Order #123

status      = PROCESSING
locked_by   = Server 1
```

Server 2 sees:

```text
Already claimed
```

and doesn't process it.

This introduces:

* Locks
* Leases
* Ownership
* Lock expiration
* Worker coordination

---

# 12. Problem — What If the Worker Dies While Holding the Lock?

Suppose:

```text
Server 1
   ↓
Claims Order #123
   ↓
status = PROCESSING
```

Then:

```text
💥 Server 1 crashes
```

Now:

```text
Order #123
status = PROCESSING
locked_by = Server 1
```

But Server 1 doesn't exist anymore.

How does Server 2 know it can take over?

We may need:

```text
lock timeout
lease expiration
heartbeat
```

For example:

```text
Lock expires after 5 minutes
```

Then:

```text
Server 1 crashes
       ↓
5 minutes
       ↓
Lock expires
       ↓
Server 2 claims order
       ↓
Continue workflow
```

Again, more reliability infrastructure.

---

# 13. Asynchronous Operations and Webhooks

Some external services don't immediately return the final result.

For example:

```text
API Server
    |
    | Charge Payment
    ↓
Payment Gateway
```

The payment gateway might respond:

```text
Payment processing...
```

and later send:

```text
Webhook
```

For example:

```json
{
  "orderId": 123,
  "status": "SUCCESS"
}
```

This may happen minutes later.

The original API request might have been handled by:

```text
Server 1
```

while the webhook may arrive at:

```text
Server 7
```

because requests are distributed through a load balancer.

Therefore Server 7 needs to:

1. Identify the order
2. Load its state
3. Determine what happened
4. Continue the workflow

---

# 14. Why Pub/Sub Helps With Webhooks

Instead of doing all processing directly inside the webhook endpoint:

```text
Payment Service
      ↓
Webhook
      ↓
API Server
      ↓
Process everything
```

we can use:

```text
Payment Service
      ↓
Webhook
      ↓
Pub/Sub
      ↓
Worker
      ↓
Order State DB
      ↓
Continue Workflow
```

Example event:

```json
{
  "type": "PaymentCompleted",
  "orderId": 123
}
```

Any available worker can consume the event.

This helps with:

* Decoupling
* Scalability
* Asynchronous processing
* Failure recovery
* Load distribution

But it does **not** eliminate workflow complexity.

---

# 15. We Are Now Building a State Machine Manually

At this point, our database might contain:

```text
payment       = SUCCESS
inventory     = SUCCESS
shipping      = PENDING
email         = PENDING
```

We have effectively created a state machine.

For example:

```text
PAYMENT_PENDING
       ↓
PAYMENT_SUCCESS
       ↓
INVENTORY_PENDING
       ↓
INVENTORY_SUCCESS
       ↓
SHIPPING_PENDING
       ↓
SHIPPING_SUCCESS
       ↓
EMAIL_PENDING
       ↓
COMPLETED
```

The problem is that we now have to manually implement all of this.

We need to decide:

* What state comes next?
* What happens when a state fails?
* When should we retry?
* How many times should we retry?
* How long should we wait?
* How do we resume after a crash?
* How do we prevent duplicate execution?
* Who owns the workflow?
* What happens when a webhook arrives twice?

This is where things start becoming messy.

---

# 16. Compensation — Another Major Problem

Suppose:

```text
Charge Payment       ✅
Reserve Inventory    ❌
```

The order cannot continue.

But payment already succeeded.

We need to undo its business effect.

We can't usually do:

```text
ROLLBACK TRANSACTION
```

because payment happened in another system.

Instead, we perform a compensating action:

```text
Charge Payment
      ↓
SUCCESS
      ↓
Reserve Inventory
      ↓
FAILED
      ↓
Refund Payment
```

The refund is called a:

> **Compensating Action**

---

# 17. Compensation Example

Suppose:

```text
Payment        ✅
Inventory      ✅
Shipping       ❌
```

We may need:

```text
Shipping failed
      ↓
Release Inventory
      ↓
Refund Payment
```

Conceptually:

```text
Forward steps:

Payment
   ↓
Inventory
   ↓
Shipping


Compensation:

Shipping failed
   ↓
Release Inventory
   ↓
Refund Payment
```

Notice that compensation is not necessarily a database rollback.

It is usually another business operation.

---

# 18. Compensation Can Also Fail

Now imagine:

```text
Payment       ✅
Inventory     ❌
Refund        ❌
```

Now we need:

```text
Retry Refund
Retry Refund
Retry Refund
...
```

But what if the refund service is down for 6 hours?

The workflow must remain alive.

This means we need:

* Retry policy
* Backoff
* Persistent state
* Scheduled retry
* Failure tracking
* Idempotency

---

# 19. Long-Running Workflows

Not every workflow finishes in milliseconds.

Consider:

```text
Create Account
      ↓
KYC Verification
      ↓
Wait for External Verification
      ↓
Credit Check
      ↓
Create Bank Account
```

KYC may take:

```text
5 minutes
1 hour
1 day
```

We cannot keep the original HTTP request alive.

Instead:

```text
Create Account
      ↓
Save State
      ↓
WAIT
      ↓
External System Completes
      ↓
Webhook/Event
      ↓
Resume Workflow
```

This is another reason durable workflow state is required.

---

# 20. Human Tasks

Some workflows involve humans.

Example:

```text
Insurance Claim
      ↓
Validate Claim
      ↓
Fraud Check
      ↓
Human Review
      ↓
Approve / Reject
      ↓
Payment
```

The human review might take:

```text
2 hours
1 day
3 days
```

The system must remember:

```text
Claim #123
Current State = WAITING_FOR_HUMAN_REVIEW
```

Later:

```text
Reviewer approves
      ↓
Event
      ↓
Resume workflow
```

Again, the process must survive for days.

---

# 21. Another Example — Food Delivery

A food delivery workflow could be:

```text
Order Created
      ↓
Restaurant Accepts
      ↓
Food Prepared
      ↓
Driver Assigned
      ↓
Driver Picks Up
      ↓
Order Delivered
      ↓
Payment Settled
      ↓
Receipt Sent
```

Potential failures:

### Restaurant doesn't accept

```text
Wait 2 minutes
      ↓
No response
      ↓
Cancel order
      ↓
Refund
```

### Driver doesn't accept

```text
Driver 1
   ↓
No response
   ↓
Driver 2
   ↓
No response
   ↓
Driver 3
```

### Server crashes

```text
Driver Assigned
      ↓
💥 Server Crash
```

When the server restarts, we need to know:

```text
Order #123
Driver = 456
Status = DRIVER_ASSIGNED
```

### Webhook arrives

```text
Driver Service
      ↓
Webhook
      ↓
Different API Server
```

That server must identify the order and continue the workflow.

Same fundamental problem.

---

# 22. Another Example — Ride Sharing

A ride workflow:

```text
Request Ride
      ↓
Find Driver
      ↓
Driver Accepts
      ↓
Driver Travels
      ↓
Passenger Picked Up
      ↓
Trip Completed
      ↓
Payment
      ↓
Receipt
```

The entire process may take 30 minutes.

During that time:

* Servers can restart
* Network can fail
* Driver service can fail
* Payment can timeout
* Events can be duplicated

The workflow therefore needs durable state.

---

# 23. Business Logic vs Reliability Logic

This distinction is extremely important.

### Business Logic

```text
Charge Payment
Reserve Inventory
Create Shipping Label
Send Email
```

### Reliability Logic

```text
Retry
Timeout
Persist State
Recover Crash
Handle Webhook
Deduplicate
Acquire Lock
Release Lock
Compensate
Schedule Retry
Detect Stalled Workflow
Resume Workflow
```

The business process itself may be simple:

```text
A → B → C → D
```

But reliability requirements make it:

```text
                Retry
                  ↑
                  |
Persist → A → B → C → D
   ↑      ↓   ↓   ↓   ↓
   |    Failure
   |
Recovery
```

The reliability code can eventually overwhelm the actual business logic.

---

# 24. Why the Architecture Becomes "Spaghetti"

We started with:

```ts
await payment();
await inventory();
await shipping();
await email();
```

But production requirements gradually force us to add:

```text
State management
Retries
Timeouts
Locks
Pollers
Workers
Webhooks
Pub/Sub
Compensation
Deduplication
Crash recovery
Scheduling
Monitoring
```

The code becomes something like:

```text
             ┌──────── Retry ────────┐
             │                       │
             ↓                       │
Payment → Save State → Inventory → Save State
   │          │             │           │
   │          │             ↓           │
   │          │          Failure        │
   │          │             ↓           │
   │          │          Refund         │
   │          │             │           │
   └──────────┼─────────────┘           │
              │                         │
              ↓                         │
          Crash Recovery ←──────────────┘
```

The business logic is now surrounded by reliability machinery.

This is the **structural problem**.

---

# 25. Evolution of the Architecture

The progression is important.

## Level 1 — Simple API Server

```text
Client
  ↓
API Server
  ↓
Payment
  ↓
Inventory
  ↓
Shipping
  ↓
Email
```

### Advantages

* Very simple
* Easy to implement
* Easy to understand

### Problems

* Process state lives in server memory
* Server crash can lose progress
* Hard to recover
* Hard to handle asynchronous callbacks

---

# 26. Level 2 — Persistent State

Add a database:

```text
Client
  ↓
API Server
  ↓
Order State DB
  ↓
Services
```

Now we can remember:

```text
Order #123
Payment = SUCCESS
Inventory = PENDING
```

### Improvement

Another server can resume the process.

### New problems

We now need:

* Pollers
* Workers
* Locks
* Retry logic
* State transitions
* Webhook handling
* Recovery logic

---

# 27. Level 3 — Event-Driven Coordination

Introduce Pub/Sub:

```text
                 Pub/Sub
                    ↑
                    |
Payment → Webhook → Event
                    |
                    ↓
                  Worker
                    |
                    ↓
              Order State DB
```

This provides better decoupling and scalability.

But the application still needs to implement:

* State machine
* Retries
* Compensation
* Recovery
* Ownership
* Scheduling

---

# 28. Level 4 — Workflow / Saga / Durable Execution

Instead of every application team building this infrastructure manually, we can use dedicated workflow patterns/systems.

Conceptually:

```text
                Workflow Engine
                       |
          ┌────────────┼────────────┐
          ↓            ↓            ↓
       Payment      Inventory     Shipping
```

The workflow system manages durable progress:

```text
Order #123

Payment      → DONE
Inventory    → DONE
Shipping     → WAITING
Email        → PENDING
```

If a worker crashes:

```text
💥 Worker
```

the workflow state survives.

Later:

```text
Workflow Engine
      ↓
Order #123
      ↓
Shipping = WAITING
      ↓
Resume workflow
```

The exact implementation depends on the workflow architecture.

---

# 29. Database vs Workflow Engine

A very important distinction:

## Database

Provides:

> **Durable storage / memory**

It remembers:

```text
Order 123
Current Step = SHIPPING
```

But it doesn't automatically execute:

```text
createShippingLabel()
```

---

## Workflow Engine

Provides:

> **Durable execution / coordination**

Conceptually:

```text
Remember state
      +
Know what should happen next
      +
Retry
      +
Wait
      +
Resume
      +
Coordinate
```

This is why workflow systems exist.

---

# 30. The Core Challenges of Multi-Step Processes

When designing a multi-step process, always think about these categories.

### 1. State

Where is the workflow currently?

```text
PAYMENT_SUCCESS
INVENTORY_PENDING
```

---

### 2. Failure

What happens if a step fails?

```text
Inventory unavailable
```

---

### 3. Retry

Should we retry?

```text
Retry immediately?
Retry after 1 minute?
Exponential backoff?
Maximum attempts?
```

---

### 4. Idempotency

What if the same operation executes twice?

```text
ChargePayment(orderId=123)
ChargePayment(orderId=123)
```

We must avoid duplicate side effects.

---

### 5. Crash Recovery

What happens if the worker/server dies?

```text
Step 2 running
      ↓
💥 Crash
```

The workflow must resume safely.

---

### 6. Asynchronous Events

What if the next step completes later?

```text
Payment Service
      ↓
Webhook
      ↓
Resume Workflow
```

---

### 7. Long Waits

What if we need to wait:

```text
5 minutes
2 hours
2 days
```

The workflow must persist while waiting.

---

### 8. Compensation

What if later steps fail after earlier steps already succeeded?

```text
Payment       ✅
Inventory     ✅
Shipping      ❌

→ Release Inventory
→ Refund Payment
```

---

### 9. Ownership

Who is currently processing this workflow?

```text
Worker 1
Worker 2
Worker 3
```

Only one should normally claim the same work at a time.

---

### 10. Observability

We need to know:

```text
Which workflows are stuck?
Which step is failing?
How many retries?
How long has a workflow been waiting?
```

This becomes extremely important in production.

---

# 31. A Useful Interview Mental Model

When an interviewer gives you a problem involving:

* Multiple dependent services
* Long-running operations
* Asynchronous callbacks
* External APIs
* Retries
* Compensation
* Human approval
* Multiple failure points

Immediately think:

```text
"This is a multi-step workflow problem."
```

Then ask:

```text
1. What are the steps?
2. Which steps are synchronous?
3. Which are asynchronous?
4. Where is workflow state stored?
5. What happens if a step fails?
6. What happens if the worker crashes?
7. Are retries safe?
8. Are operations idempotent?
9. Do we need compensation?
10. How does the workflow resume?
```

---

# 32. The Most Important Example to Remember

Consider:

```text
                    Order
                      |
                      ↓
                Charge Payment
                      |
                      ↓
               Reserve Inventory
                      |
                 ┌────┴────┐
                 │         │
               Success   Failure
                 │         │
                 ↓         ↓
           Create Label   Refund
                 |
                 ↓
             Ship Order
                 |
                 ↓
          Send Confirmation
```

Now introduce reality:

```text
Payment can fail
Inventory can fail
Shipping can fail
Email can fail

Server can crash
Network can timeout
Webhook can arrive later
Webhook can arrive twice

Payment can succeed but response can be lost
Inventory can succeed but server can crash
Refund can fail
Warehouse can take 8 hours
```

The workflow must handle all of these.

That is the **Multi-Step Process problem**.

---

# 33. Final Mental Model

The simplest way to remember the entire discussion:

```text
                  BUSINESS PROCESS
                         |
                         ↓
                A → B → C → D
                         |
                         ↓
                  Reality Happens
                         |
       ┌─────────────────┼─────────────────┐
       ↓                 ↓                 ↓
     Crash            Timeout            Failure
       ↓                 ↓                 ↓
    Recovery           Retry          Compensation
       │                 │                 │
       └─────────────────┼─────────────────┘
                         ↓
                  Need Persistent State
                         ↓
                  Need Coordination
                         ↓
                  Need Safe Execution
                         ↓
               Multi-Step Workflow
```

---

# 34. One-Line Definition

> **A multi-step process is a long-running business workflow composed of multiple distributed operations that must remain correct and recoverable despite failures, retries, asynchronous events, and crashes.**

---

# 35. The Evolution to Remember

```text
Simple Sequential Code
        ↓
Single API Server
        ↓
Server Crash Problem
        ↓
Persist State in DB
        ↓
"Who resumes the workflow?"
        ↓
Pollers + Workers
        ↓
"Who owns the work?"
        ↓
Locks / Leases
        ↓
"How do we handle async callbacks?"
        ↓
Webhooks + Pub/Sub
        ↓
"How do we undo successful steps?"
        ↓
Compensation
        ↓
"Why are we building all this ourselves?"
        ↓
Workflow / Saga / Durable Execution
```

### The central lesson

> **The database can remember the workflow, but something still needs to execute, coordinate, retry, recover, and compensate the workflow.**

That gap between **"remembering state"** and **"reliably executing the process"** is the heart of the multi-step process problem.

# Saga Pattern — Quick Revision Notes

## 1. Why Saga?

A business transaction may span **multiple services/databases**:

```text
Order → Payment → Inventory → Shipping → Email
```

A normal DB transaction cannot easily cover all of them.

So instead of:

> "Everything succeeds or everything rolls back"

Saga follows:

> **"Each step commits independently; if a later step fails, compensate for the steps that already succeeded."**

---

## 2. Saga Pattern

A **Saga** is a sequence of local transactions where each important step has a **compensating action**.

### Example

```text
Charge Payment       ✅
Reserve Inventory    ❌
        ↓
Refund Payment       ↩️
```

### Common compensations

| Forward Action    | Compensating Action |
| ----------------- | ------------------- |
| Charge payment    | Refund payment      |
| Reserve inventory | Release inventory   |
| Create shipment   | Cancel shipment     |
| Book hotel        | Cancel booking      |
| Reserve seat      | Release seat        |

> Compensation is **not a database rollback**. It is a new business operation that semantically undoes the previous action.

---

## 3. Saga Does NOT Give Atomicity

There can be a temporary inconsistent state:

```text
Payment charged
      ↓
Inventory fails
      ↓
Refund payment
```

For a short period, the customer is charged even though the order isn't complete.

So Saga provides:

```text
Temporary inconsistency
        ↓
Compensation
        ↓
Eventually consistent state
```

---

# 4. Two Ways to Coordinate a Saga

## A. Choreography

**No central coordinator.**

Services listen to events and react.

```text
OrderPlaced
    ↓
Payment Worker
    ↓
PaymentCharged
    ↓
Inventory Worker
    ↓
InventoryReserved
    ↓
Shipping Worker
    ↓
ShipmentCreated
```

Each service follows:

```text
"When I receive X → do Y → emit Z"
```

### Advantages

* Decoupled services
* Natural event-driven architecture
* Easy to add independent reactions
* Scales well

### Disadvantages

* Overall workflow is implicit
* Difficult to understand as complexity grows
* Debugging becomes harder
* Changing the workflow may require changing event contracts

### Best for

> **Simple to medium-complexity workflows with relatively independent services.**

---

# 5. Orchestration

A central **Saga Orchestrator** controls the workflow.

```text
                 Orchestrator
                /      |      \
               ↓       ↓       ↓
           Payment  Inventory Shipping
```

The orchestrator explicitly knows:

```text
1. Charge payment
2. Reserve inventory
3. Create shipment
4. Send email
```

If inventory fails:

```text
Charge Payment      ✅
Reserve Inventory   ❌
        ↓
Refund Payment
```

### Advantages

* Workflow is explicit
* Easier debugging
* Central visibility
* Better for complex workflows
* Easier to manage retries and compensations

### Disadvantage

* Orchestrator becomes an important component
* More centralized coordination

### Best for

> **Complex workflows with many steps, branches, retries, and compensations.**

---

# 6. Choreography vs Orchestration

|                    | Choreography | Orchestration  |
| ------------------ | ------------ | -------------- |
| Coordinator        | ❌ None       | ✅ Central      |
| Flow               | Implicit     | Explicit       |
| Communication      | Events       | Commands/calls |
| Simple workflows   | ⭐⭐⭐          | ⭐⭐⭐            |
| Complex workflows  | Difficult    | Better         |
| Debugging          | Harder       | Easier         |
| Central visibility | Low          | High           |

### Easy memory trick

```text
Choreography = Dancers coordinate themselves

Orchestration = Conductor controls everyone
```

---

# 7. The Crash Problem

Suppose an orchestrator executes:

```text
Charge Payment       ✅
Reserve Inventory    ✅
💥 Server crashes
```

After restart:

```text
Did payment happen?
Did inventory happen?
Where should I continue?
```

We need **durable workflow state**.

---

# 8. Durable Execution

Durable execution means:

> **Workflow progress survives crashes and restarts.**

Instead of losing progress:

```text
Step 1 ✅
Step 2 ✅
Crash 💥
```

The workflow engine remembers:

```text
Step 1 → completed
Step 2 → completed
```

and continues from:

```text
Step 3
```

---

# 9. Temporal Mental Model

Temporal separates:

```text
Workflow
   ↓
Decides WHAT happens next

Activity
   ↓
Actually performs the external side effect
```

Example:

```text
Workflow
   │
   ├── processPayment()     → Activity
   ├── reserveInventory()   → Activity
   ├── shipOrder()          → Activity
   └── sendEmail()          → Activity
```

The workflow contains the business flow.

Activities interact with:

* Payment service
* Database
* Inventory service
* Shipping service
* Email provider

---

# 10. Replay

Temporal records Activity results in workflow history.

Suppose:

```text
processPayment()       ✅
reserveInventory()     ✅
Server crashes         💥
```

On recovery, workflow code is replayed.

```text
processPayment()
      ↓
History already has result
      ↓
Return recorded result

reserveInventory()
      ↓
History already has result
      ↓
Return recorded result

shipOrder()
      ↓
No previous result
      ↓
Actually execute
```

So the workflow continues from where it stopped.

---

# 11. Determinism

Workflow code must be **deterministic**.

Given:

```text
Same input + Same history
```

it should produce:

```text
Same decisions
```

Avoid nondeterministic logic directly in workflow code, such as arbitrary random values or external network calls.

External side effects belong in **Activities**.

---

# 12. Idempotency

Activities must generally be **idempotent** because they may be retried.

### Dangerous

```text
ChargePayment()
```

If retried:

```text
Charge $100
Charge $100 again
```

Customer gets charged twice ❌

### Idempotent approach

Use an idempotency key:

```text
orderId = ORDER-123
```

First request:

```text
ORDER-123 → Charge $100 → SUCCESS
```

Retry:

```text
ORDER-123 already processed
→ return previous result
```

Customer is charged only once ✅

> **Retries + side effects → Idempotency is essential.**

---

# 13. Kafka / Event Store

In choreography, a durable event log such as Kafka can store events:

```text
OrderPlaced
PaymentCharged
InventoryReserved
InventoryFailed
PaymentRefunded
ShipmentCreated
```

Workers consume these events:

```text
Kafka
  │
  ├── Payment Worker
  ├── Inventory Worker
  ├── Shipping Worker
  └── Email Worker
```

Benefits:

* Fault tolerance
* Scalability
* Audit trail
* Replay
* Event-driven communication

### Important

```text
Kafka ≠ Saga
```

Kafka is infrastructure for events.

Saga is the **business transaction/workflow pattern**.

---

# 14. Saga vs Temporal

```text
Saga
 ↓
Business pattern
"How do I handle distributed transactions?"

Temporal
 ↓
Implementation/tool
"How do I reliably execute long-running workflows?"
```

Temporal can be used to implement Saga-style workflows with:

* Durable state
* Retries
* Compensation
* Crash recovery
* Workflow history

---

# 15. Complete Mental Model

```text
Distributed Business Transaction
            ↓
     Multiple Services
            ↓
Can't use one DB transaction
            ↓
          SAGA
            ↓
     ┌──────┴──────┐
     ↓             ↓
Choreography   Orchestration
     ↓             ↓
  Events       Coordinator
                   ↓
             Complex workflow
                   ↓
             Crash recovery?
                   ↓
          Durable Execution
                   ↓
        ┌──────────┴─────────┐
        ↓                    ↓
     Temporal          Step Functions
```

---

# 16. Interview Answer

If asked **"What is Saga Pattern?"**, answer:

> A Saga is a way to manage a distributed business transaction that spans multiple services when we cannot use a single distributed database transaction. We split the transaction into local steps, and each successful step has a compensating action. If a later step fails, we execute compensations for the previously completed steps. The Saga can be coordinated using choreography, where services communicate through events, or orchestration, where a central coordinator controls the workflow.

---

# 17. 30-Second Revision

Remember these **7 words/concepts**:

```text
Saga
 ↓
Local Transactions
 ↓
Compensation
 ↓
Choreography / Orchestration
 ↓
Durable Execution
 ↓
Determinism
 ↓
Idempotency
```

### One-line definitions

* **Saga** → Distributed transaction pattern
* **Local transaction** → Each service commits independently
* **Compensation** → Business operation that undoes a previous action
* **Choreography** → Services coordinate through events
* **Orchestration** → Central coordinator controls workflow
* **Durable execution** → Workflow survives crashes
* **Determinism** → Same history → same decisions
* **Idempotency** → Retry doesn't duplicate side effects


# Two-Phase Commit (2PC) & Three-Phase Commit (3PC)

## 1. Why Do We Need Distributed Transactions?

A transaction may span multiple databases/services:

```text
Order Service
     ↓
Payment Service
     ↓
Inventory Service
```

We may need:

> **Either ALL services commit, or NONE commit.**

Example:

```text
Debit $100 from Bank A
Credit $100 to Bank B
```

We don't want:

```text
Bank A → -$100 ✅
Bank B → +$0   ❌
```

This is the distributed transaction problem.

---

# 2. Two-Phase Commit (2PC)

**2PC = Two-Phase Commit**

A **Coordinator** manages a distributed transaction involving multiple **Participants**.

```text
                 Coordinator
                /            \
               ↓              ↓
           Database A      Database B
           Participant     Participant
```

The two phases are:

```text
Phase 1 → PREPARE
Phase 2 → COMMIT / ABORT
```

---

# 3. Phase 1 — Prepare

Coordinator asks every participant:

```text
"Can you commit this transaction?"
```

```text
Coordinator
    │
    ├── PREPARE → DB A
    │                ↓
    │              YES
    │
    └── PREPARE → DB B
                     ↓
                   YES
```

Each participant checks:

* Constraints
* Required resources
* Data validity
* Whether the operation can succeed

If successful:

```text
YES → PREPARED
```

### Important

**Prepared ≠ Committed**

Prepared means:

> "I am ready to commit when you tell me to."

The participant may hold locks/resources while waiting.

---

# 4. Phase 2 — Commit

If **everyone says YES**:

```text
Coordinator
    │
    ├── COMMIT → DB A
    │
    └── COMMIT → DB B
```

Final:

```text
DB A → COMMITTED ✅
DB B → COMMITTED ✅
```

---

# 5. What If Someone Says NO?

Example:

```text
DB A → YES
DB B → NO
```

Coordinator sends:

```text
ABORT → DB A
ABORT → DB B
```

Result:

```text
DB A → Rollback
DB B → Rollback
```

So:

```text
Everyone YES → COMMIT ALL
Any NO       → ABORT ALL
```

---

# 6. Complete 2PC Flow

```text
                  Coordinator
                       │
                 PREPARE REQUEST
                 /            \
                ↓              ↓
              DB A            DB B
                │              │
              YES              YES
                \              /
                 \            /
                  ↓          ↓
                 Everyone YES?
                      │
                     YES
                      ↓
                    COMMIT
                   /      \
                  ↓        ↓
                DB A      DB B
                 ✅         ✅
```

---

# 7. Example: Bank Transfer

Transfer $100:

```text
Bank A → Debit $100
Bank B → Credit $100
```

### Phase 1

```text
Coordinator → Bank A: PREPARE
Bank A → YES

Coordinator → Bank B: PREPARE
Bank B → YES
```

### Phase 2

```text
Coordinator → Bank A: COMMIT
Coordinator → Bank B: COMMIT
```

Result:

```text
Bank A: -$100
Bank B: +$100
```

Atomic distributed transaction achieved.

---

# 8. Major Problem with 2PC — Blocking

Consider:

```text
Bank A → PREPARED
Bank B → PREPARED

Coordinator 💥 CRASHES
```

Participants don't know:

```text
COMMIT?
```

or

```text
ABORT?
```

So they may need to wait.

They can also be holding locks/resources while waiting.

```text
Coordinator crashes
       ↓
Participants wait
       ↓
Locks/resources remain
       ↓
Other transactions may be blocked
```

Therefore:

> **2PC is a blocking protocol.**

---

# 9. Why Is 2PC Expensive?

Main problems:

* Coordinator dependency
* Participants may hold locks for a long time
* Slow participant can delay everyone
* Coordinator failure can leave participants uncertain
* Poor fit for long-running workflows
* External services may not support 2PC

For example:

```text
Payment Service → Stripe
Shipping Service → FedEx
Email Service → Email Provider
```

You can't simply ask all of them to participate in your 2PC transaction.

---

# 10. Three-Phase Commit (3PC)

3PC extends 2PC by adding an intermediate phase.

### 2PC

```text
PREPARE
   ↓
COMMIT
```

### 3PC

```text
CAN-COMMIT
   ↓
PRE-COMMIT
   ↓
DO-COMMIT
```

The extra phase gives participants more information about the coordinator's progress.

---

# 11. Phase 1 — CanCommit

Coordinator asks:

```text
"Can you commit?"
```

```text
Coordinator
    │
    ├── CAN-COMMIT → DB A → YES
    │
    └── CAN-COMMIT → DB B → YES
```

If someone says NO:

```text
ABORT
```

---

# 12. Phase 2 — PreCommit

If everyone says YES:

```text
Coordinator
    │
    ├── PRE-COMMIT → DB A
    └── PRE-COMMIT → DB B
```

Participants prepare to commit and acknowledge:

```text
DB A → ACK
DB B → ACK
```

Now the coordinator knows that participants reached the pre-commit stage.

---

# 13. Phase 3 — DoCommit

Coordinator sends:

```text
COMMIT → DB A
COMMIT → DB B
```

Participants finally commit.

```text
DB A → COMMITTED ✅
DB B → COMMITTED ✅
```

---

# 14. Complete 3PC Flow

```text
                 Coordinator
                      │
                CAN-COMMIT?
                 /       \
                ↓         ↓
              DB A       DB B
                │         │
              YES         YES
                \         /
                 ↓       ↓
                 PRE-COMMIT
                 /       \
                ↓         ↓
              DB A       DB B
               ACK        ACK
                \         /
                 ↓       ↓
                   COMMIT
                 /         \
                ↓           ↓
              DB A         DB B
               ✅            ✅
```

---

# 15. Why Add the Third Phase?

The goal is to reduce the uncertainty/blocking problem of 2PC.

2PC has:

```text
PREPARED
   ↓
????
```

A participant may not know whether the coordinator decided to commit or abort.

3PC introduces:

```text
CAN-COMMIT
   ↓
PRE-COMMIT
   ↓
COMMIT
```

So there is a clearer intermediate state.

> **3PC tries to make failure recovery less blocking by separating preparation from the final commit more explicitly.**

---

# 16. 2PC vs 3PC

|                  | 2PC          | 3PC          |
| ---------------- | ------------ | ------------ |
| Phases           | 2            | 3            |
| Phase 1          | Prepare      | CanCommit    |
| Phase 2          | Commit/Abort | PreCommit    |
| Phase 3          | —            | DoCommit     |
| Blocking risk    | Higher       | Reduced      |
| Complexity       | Lower        | Higher       |
| Failure handling | Simpler      | More complex |
| Practical usage  | More common  | Less common  |

### Memory Trick

```text
2PC = Prepare → Commit

3PC = CanCommit → PreCommit → Commit
```

---

# 17. 2PC vs Saga

This is the most important comparison for system-design interviews.

## 2PC

Goal:

> **Atomicity**

```text
All commit
    OR
All abort
```

Example:

```text
Prepare Payment
Prepare Inventory
       ↓
Everyone YES
       ↓
Commit Payment
Commit Inventory
```

---

## Saga

Goal:

> **Eventually reach a consistent business state through compensation.**

```text
Charge Payment
      ↓
Payment committed ✅
      ↓
Reserve Inventory
      ↓
FAILED ❌
      ↓
Refund Payment
```

The original payment transaction is **not rolled back**.

A new compensating transaction is executed.

---

# 18. Saga Example

Order workflow:

```text
Charge Payment
      ↓
Reserve Inventory
      ↓
Create Shipment
```

Suppose:

```text
Charge Payment      ✅
Reserve Inventory   ✅
Create Shipment     ❌
```

Saga compensates:

```text
Cancel/Release Inventory
      ↓
Refund Payment
```

So:

```text
Forward:

Payment → Inventory → Shipping ❌


Compensation:

Shipping ❌
    ↓
Release Inventory
    ↓
Refund Payment
```

---

# 19. Core Difference

### 2PC

```text
"Don't commit anything until everyone is ready."
```

### Saga

```text
"Commit each step independently and compensate if something later fails."
```

---

# 20. When to Use What?

### Use 2PC when:

* Strong atomicity is required
* All participants support distributed transactions
* Transactions are relatively short
* Holding locks/resources is acceptable

### Use Saga when:

* Multiple microservices are involved
* Workflow can be long-running
* External services are involved
* Participants don't support distributed transactions
* Temporary inconsistency is acceptable
* Compensation is possible

### 3PC

Mostly important as a **distributed transaction concept** and interview topic. It is more complex and less commonly used in modern application architectures.

---

# 21. Interview Mental Model

When you hear:

```text
"Multiple databases need to commit together."
```

Think:

```text
             Distributed Transaction
                       │
              ┌────────┴────────┐
              ↓                 ↓
             2PC               Saga
              │                 │
        Atomic commit       Compensation
              │                 │
       "All or nothing"    "Undo what happened"
```

Then:

```text
2PC
 ↓
Prepare
 ↓
Commit
 ↓
Possible blocking
```

```text
3PC
 ↓
CanCommit
 ↓
PreCommit
 ↓
Commit
 ↓
Reduced blocking/uncertainty
```

```text
Saga
 ↓
Local transaction
 ↓
Local transaction
 ↓
Failure
 ↓
Compensation
```

---

# 22. One-Line Interview Definitions

### 2PC

> A distributed transaction protocol where a coordinator first asks all participants to prepare and then tells them to either commit or abort together.

### 3PC

> An extension of 2PC that adds a pre-commit phase to reduce uncertainty and blocking during failures.

### Saga

> A distributed transaction pattern where each local transaction commits independently and failures are handled using compensating transactions.

---

# 23. Final Cheat Sheet

```text
2PC
━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Prepare → Commit

Goal:
Atomicity

Problem:
Blocking, locks, coordinator failure


3PC
━━━━━━━━━━━━━━━━━━━━━━━━━━━━
CanCommit → PreCommit → Commit

Goal:
Reduce 2PC uncertainty/blocking

Problem:
More complexity + assumptions about failures/timing


Saga
━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Do → Do → Failure → Compensate

Goal:
Reliable distributed business workflow

Advantages:
Loosely coupled, long-running, microservice friendly

Problem:
Temporary inconsistency + compensation complexity
```

## ⭐ Remember This

> **2PC = All-or-nothing transaction**
> **3PC = 2PC + an extra pre-commit stage**
> **Saga = Independent commits + compensation**
