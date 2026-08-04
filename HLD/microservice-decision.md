# Microservice Architecture: How to Decide When to Create a Separate Microservice and Database

---

# Table of Contents

1. Introduction
2. Why Microservices Exist
3. Monolith First, Microservices Later
4. What Defines a Microservice?
5. How to Decide Whether Something Should Be a Separate Microservice
6. When NOT to Create a Microservice
7. Database per Microservice Pattern
8. Database Ownership vs Database Separation
9. Communication Between Microservices
10. Real World Examples
11. Case Study - Contest & Leaderboard
12. Checklist for Creating a New Microservice
13. Common Mistakes
14. Interview Tips
15. Key Takeaways

---

# 1. Introduction

One of the most misunderstood concepts in software architecture is deciding **when to create a new microservice**.

Many developers think:

> Every feature deserves its own microservice.

Others think:

> Every microservice must have its own physical database.

Neither statement is correct.

The decision should be based on:

- Business capabilities
- Team ownership
- Scalability
- Deployment independence
- Technology requirements
- Data ownership

**NOT** on the number of APIs or database tables.

---

# 2. Why Microservices Exist

Imagine we are building YouTube.

Initially our application contains everything.

```
YouTube Application

├── Users
├── Videos
├── Upload
├── Comments
├── Likes
├── Search
├── Notifications
├── Analytics
├── Recommendation
```

Everything is inside one codebase.

Everything uses one PostgreSQL database.

```
                 +--------------------+
                 |     Application    |
                 +--------------------+
                          |
                     PostgreSQL
```

This architecture is completely fine.

There is absolutely nothing wrong with starting like this.

Many successful companies started exactly this way.

Examples:

- Facebook
- Amazon
- Netflix
- Instagram

None of them started with hundreds of microservices.

---

# 3. Why Companies Split into Microservices

As the company grows, different parts of the application evolve differently.

For example:

```
Search

Needs Elasticsearch

---------------------

Upload

Needs S3

---------------------

Notifications

Processes millions of events

---------------------

Recommendation

Uses Machine Learning
```

Now different modules have different requirements.

Keeping everything together becomes difficult.

This is when microservices become valuable.

---

# 4. What Defines a Microservice?

A microservice is **not** simply a small project.

It is an independently deployable unit that owns a specific business capability.

A microservice should:

- Own one business domain
- Own its own data
- Be independently deployable
- Scale independently
- Be developed by one team
- Have well-defined APIs

Think of a microservice as a small company.

Example:

```
Notification Company

Responsible for

• Email
• SMS
• Push Notifications

Nothing else.
```

---

# 5. How to Decide Whether Something Should Be a Separate Microservice

There is no universal formula.

Instead, evaluate several factors.

---

## Rule 1 — Business Capability

Ask:

> Does this represent an independent business function?

Example:

```
Authentication

- Login
- Signup
- Password Reset
- MFA

```

Everything is related.

This naturally becomes:

```
Authentication Service
```

Now consider:

```
Notifications

- Email
- SMS
- Push
```

Notifications are completely unrelated to authentication.

Therefore:

```
Authentication Service

Notification Service
```

---

## Rule 2 — Independent Scaling

Different parts of the system experience different traffic.

Example:

```
100 Million Users

Daily

Login Requests

10 Million

----------------------

Notifications

500 Million

----------------------

Payments

200 Thousand
```

Should Login Service scale because notifications spike?

No.

Instead:

```
Authentication

4 Instances

Notification

30 Instances

Payment

2 Instances
```

Each service scales independently.

---

## Rule 3 — Different Technologies

Suppose everything currently uses PostgreSQL.

Now Search requires Elasticsearch.

Architecture:

```
Search Service

↓

Elasticsearch

-------------------------

User Service

↓

PostgreSQL
```

Keeping Search inside User Service creates unnecessary complexity.

Better to isolate it.

---

## Rule 4 — Different Release Frequency

Suppose:

Recommendation Algorithm

```
Updated Weekly
```

Authentication

```
Updated Twice a Year
```

If both are inside one service,

Every recommendation deployment risks authentication.

Separating services eliminates unnecessary deployment risk.

---

## Rule 5 — Team Ownership

Imagine:

```
Team A

Authentication

----------------

Team B

Payments

----------------

Team C

Recommendation

----------------

Team D

Notifications
```

Each team should own its own service.

Otherwise:

- Merge conflicts increase
- Coordination becomes difficult
- Deployment slows down

---

## Rule 6 — Different Failure Characteristics

Imagine Notification Service crashes.

Should users lose the ability to login?

No.

Authentication should continue working.

Independent services isolate failures.

---

## Rule 7 — Independent Lifecycle

Some domains evolve continuously.

Example:

Recommendation

```
Every month

New ranking algorithm
New ML model
New personalization logic
```

Users

```
Rare changes
```

Recommendation deserves its own service.

---

# 6. When NOT to Create a Microservice

Sometimes splitting creates more problems.

Example:

```
Job

├── Skills
├── Salary
├── Benefits
├── Description
```

Should these become:

```
Job Service

Skill Service

Salary Service

Benefit Service
```

Absolutely not.

Every Job request needs all of them.

Instead:

```
Job Service

Owns everything related to Job.
```

---

## Another Example

E-commerce

```
Product

Price

Inventory

Description
```

All belong together.

Splitting them creates unnecessary network calls.

---

# 7. Database Per Microservice

One of the most famous principles:

> Every microservice owns its data.

Notice the wording carefully.

It says

**Owns**

NOT

**Has a separate database server**

These are different concepts.

---

# 8. Database Ownership vs Physical Database

There are three common approaches.

---

## Option 1

Separate Database Server

```
User Service

↓

Postgres Server A

-----------------------

Payment Service

↓

Postgres Server B

-----------------------

Notification Service

↓

MongoDB
```

Advantages

- Maximum isolation
- Independent scaling
- Better security

Disadvantages

- Expensive
- Operational overhead

Usually used by large companies.

---

## Option 2

Same PostgreSQL Instance

Different Databases

```
PostgreSQL

├── user_db
├── payment_db
├── notification_db
```

Advantages

- Lower cost
- Logical isolation
- Independent schemas

Very common.

---

## Option 3

Same Database

Different Schemas

```
PostgreSQL

├── user
├── payment
├── notification
```

Each service owns its schema.

Still acceptable.

Ownership matters more than physical separation.

---

# 9. Why Data Ownership Matters

Suppose Payment Service owns:

```
payments

id

amount INTEGER
```

Later it changes to

```
amount NUMERIC(18,2)
```

If Reporting Service directly queries

```
payments
```

Reporting immediately breaks.

Instead

```
Reporting

↓

Payment API

↓

Payment Database
```

Payment Service hides internal implementation.

Other services remain unaffected.

---

# 10. Never Share Tables

Bad Architecture

```
            Users Table

        /       |       \

User    Order   Notification
Service Service Service
```

Everyone modifies the same table.

Problems:

- Tight coupling
- Migration failures
- Deployment dependency
- Schema conflicts

---

Good Architecture

```
User Service

↓

Users Table

------------------

Order Service

↓

Orders Table

------------------

Notification Service

↓

Notification Table
```

Ownership is clear.

---

# 11. How Services Communicate

Since services cannot directly query each other's tables,

they communicate using:

## Synchronous

```
HTTP

gRPC
```

Example

```
Order Service

↓

GET /users/123

↓

User Service
```

---

## Asynchronous

```
Kafka

RabbitMQ

SQS
```

Example

```
Payment Completed

↓

Kafka

↓

Notification

↓

Analytics

↓

Email
```

No direct dependency.

---

# 12. Real World Example — Netflix

```
User Service

↓

User Database

-----------------------

Billing Service

↓

Billing Database

-----------------------

Recommendation Service

↓

Recommendation Database

-----------------------

Streaming Service

↓

Video Metadata Database
```

Every service owns its own data.

No cross-database joins.

---

# 13. Real World Example — Amazon

```
Customer

↓

Customer DB

---------------------

Orders

↓

Order DB

---------------------

Inventory

↓

Inventory DB

---------------------

Payments

↓

Payment DB
```

Everything communicates using APIs/events.

---

# 14. Case Study — Contest Platform

Suppose we are designing LeetCode.

Initially:

```
Contest

Submission

Leaderboard

Users
```

Should Leaderboard become its own service?

Let's evaluate.

---

## Business Capability

Leaderboard has its own responsibility.

✓ YES

---

## Scaling

Contest

```
100 submissions/sec
```

Leaderboard

```
200,000 reads/sec
```

Different scaling.

✓ YES

---

## Technology

Contest

```
PostgreSQL
```

Leaderboard

```
Redis Sorted Set
```

Different databases.

✓ YES

---

## Deployment

Leaderboard ranking algorithm changes frequently.

Contest logic rarely changes.

✓ YES

---

## Team

Contest Team

Leaderboard Team

Possible.

✓ YES

---

Result

```
Contest Service

↓

Postgres

↓

Publishes Event

↓

Kafka

↓

Leaderboard Service

↓

Redis

↓

Leaderboard API

↓

WebSocket

↓

Clients
```

Leaderboard deserves its own microservice.

---

# 15. Counter Example

Contest

```
Contest Rules

Prize

Start Time

End Time
```

Should these become independent services?

No.

Reasons

- Same business capability
- Same team
- Same scaling
- Same deployment
- Same lifecycle

Keep them together.

---

# 16. Practical Decision Matrix

| Question | Yes → Separate Service? |
|------------|------------------------|
| Independent business capability? | ✅ |
| Independent scaling required? | ✅ |
| Different deployment cycle? | ✅ |
| Different team ownership? | ✅ |
| Different technology stack? | ✅ |
| Independent failure tolerance? | ✅ |
| Own data independently? | ✅ |
| Requires constant synchronous communication with another service? | ❌ Prefer keeping together |

---

# 17. Common Mistakes

## Mistake 1

One table = One microservice

Wrong

Tables are implementation details.

Business capability defines services.

---

## Mistake 2

Every service needs its own database server

Wrong.

Different schemas or databases on the same PostgreSQL instance are often sufficient.

---

## Mistake 3

Sharing tables

Very dangerous.

Always access another service's data through APIs or events.

---

## Mistake 4

Creating too many tiny services

Example

```
Address Service

Phone Service

Email Service

Profile Picture Service
```

This is called a **Distributed Monolith**.

Too many network calls.

Too much complexity.

---

## Mistake 5

Ignoring team boundaries

Microservices should align with how teams work.

If one team owns everything,

microservices may not provide much value.

---

# 18. Interview Tips

When asked to design a large system,

do **NOT** immediately create 30 microservices.

Instead:

Start with:

```
User Service

Video Service

Comment Service

Notification Service

Search Service
```

Then justify each split.

Interviewers care far more about **why** you separated services than about the exact number of services.

Always explain your reasoning using:

- Business capability
- Scaling
- Technology
- Deployment
- Ownership
- Data isolation

---

# 19. Key Takeaways

- Start with a monolith whenever possible.
- Split services only when there is a clear business or technical reason.
- A microservice should own a single business capability.
- Every microservice should own its data.
- Ownership is more important than physical database separation.
- Never allow multiple services to directly modify the same tables.
- Services should communicate using APIs or asynchronous events.
- Different scaling, deployment frequency, technology, or ownership are strong indicators that a separate microservice is appropriate.
- Avoid creating overly fine-grained services that introduce unnecessary network overhead and operational complexity.
- Design microservices around **business domains**, not around database tables or CRUD operations.

---

# Golden Rule

> **A microservice should represent an independently deployable business capability with clear ownership of its data and lifecycle—not just a collection of APIs or database tables.**