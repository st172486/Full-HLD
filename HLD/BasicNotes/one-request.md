This is one of the most important topics in System Design. If you understand this end-to-end, you'll understand how **almost every production application** (Netflix, Amazon, Swiggy, Uber, LinkedIn, etc.) handles incoming requests.

Let's build a real production architecture step by step.

---

# A Simple Question

Suppose a user opens

```text
https://www.netflix.com
```

What actually happens?

Most people think:

```text
Browser
    ↓
Server
```

But in production, it looks more like this:

```text
                +----------------+
                |      DNS       |
                +----------------+
                        |
                        |
                 +--------------+
                 |     CDN      |
                 +--------------+
                        |
                        |
                 +--------------+
                 |     WAF      |
                 +--------------+
                        |
                        |
               +-------------------+
               |   API Gateway     |
               +-------------------+
                        |
                        |
                +---------------+
                | Load Balancer |
                +---------------+
                 /      |       \
                /       |        \
         App Server1 App Server2 App Server3
               |          |          |
        -------------------------------
                     |
             Redis / Cache
                     |
              Message Queue
                     |
              Microservices
                     |
              Databases
```

Notice something.

The request doesn't directly reach your application.

It passes through many layers.

Every layer has a responsibility.

---

# Let's follow ONE request

Suppose I open

```text
https://www.netflix.com/movies
```

Let's trace the request.

---

# Step 1 — DNS

Browser first asks

```text
Where is netflix.com?
```

DNS replies

```text
104.x.x.x
```

That IP usually belongs to

* CDN
* Global Load Balancer

NOT your application server.

---

# Step 2 — CDN

The request reaches CDN first.

```text
Browser
     |
     |
   CDN
```

Now CDN asks

```text
Do I already have this content?
```

Suppose user requested

```text
/logo.png
```

CDN checks

```text
Cache?
```

If available

```text
Return immediately.
```

Application server is never contacted.

Example

```text
Browser

↓

CDN

↓

Image Found

↓

Browser
```

Only few milliseconds.

---

## Why?

Because static content doesn't change often.

Examples

```text
Images

CSS

JavaScript

Fonts

Videos

PDFs
```

No need to hit backend.

---

Suppose user requests

```text
GET /api/profile
```

CDN cannot answer.

Forward request.

```text
Browser

↓

CDN

↓

Next Layer
```

---

# Step 3 — WAF

Now request reaches Web Application Firewall.

```text
Browser

↓

CDN

↓

WAF
```

WAF acts like security.

It asks

```text
Is this request dangerous?
```

Example

```text
GET /login
```

Looks normal.

Allowed.

---

Suppose hacker sends

```sql
?id=1 OR 1=1
```

Classic SQL Injection.

WAF detects

```text
Malicious Pattern
```

Blocks request.

Application never receives it.

---

Another example

```text
<script>alert(1)</script>
```

XSS attack.

Blocked.

---

Another

```text
100000 requests/sec
```

DDoS attack.

Blocked.

---

WAF protects against

```text
SQL Injection

Cross Site Scripting

Command Injection

File Upload attacks

Known attack signatures

Rate limiting
```

---

# Step 4 — API Gateway

Now request reaches API Gateway.

Many beginners confuse API Gateway with Load Balancer.

They are different.

Gateway works at API level.

Example

Client requests

```text
GET /users/10
```

Gateway decides

```text
Send to User Service
```

Another request

```text
GET /payments
```

Gateway

```text
Payment Service
```

Another

```text
GET /orders
```

Gateway

```text
Order Service
```

Gateway understands

```text
URLs

Headers

JWT

Authentication

Authorization

Versioning
```

---

Gateway responsibilities

Authentication

```text
Validate JWT
```

Authorization

```text
Can this user access this API?
```

---

Rate limiting

```text
Only

100 requests/minute
```

---

Logging

Every request.

---

Monitoring

Latency

Errors

Traffic

---

API Versioning

```text
/v1/users

/v2/users
```

---

Routing

```text
/users

↓

User Service
```

---

Request transformation

Example

Client sends

```json
{
  "name":"Suraj"
}
```

Gateway converts

```json
{
   "fullName":"Suraj"
}
```

---

Response transformation also possible.

---

# Step 5 — Load Balancer

Now request reaches LB.

Suppose

```text
10 Application Servers
```

```text
          LB
           |
-----------------------------
|    |    |    |     |
S1   S2   S3   S4    S5
```

LB decides

```text
Send to Server 4
```

---

Now Server4 handles request.

---

# Step 6 — Application Server

Application server contains business logic.

Example

```text
GET /profile
```

Code

```text
Validate user

Fetch profile

Fetch orders

Merge data

Return JSON
```

---

Suppose profile is cached.

---

Application checks

```text
Redis
```

---

# Step 7 — Redis

Redis is much faster.

Application asks

```text
Profile available?
```

If yes

Done.

Database skipped.

```text
Server

↓

Redis

↓

Response
```

Milliseconds.

---

Suppose cache miss.

Now

```text
Redis

↓

Not Found
```

Application goes to database.

---

# Step 8 — Database

Database executes

```sql
SELECT *
FROM Users
WHERE Id=10;
```

Returns

```text
Profile
```

Application stores result in Redis.

Future requests become faster.

---

# Step 9 — Message Queue (optional)

Suppose user placed order.

Do we need

```text
Send Email

Send SMS

Generate Invoice

Update Analytics

Notify Warehouse
```

inside same request?

No.

Too slow.

Instead

```text
Order Created

↓

RabbitMQ

↓

Background Workers
```

Workers perform

```text
Email

SMS

Analytics

Invoice
```

independently.

User receives response immediately.

---

# Step 10 — Response

Finally

```text
Database

↓

Application

↓

Load Balancer

↓

Gateway

↓

CDN

↓

Browser
```

---

# Production Architecture

This is close to what you'll find in large-scale production systems.

```text
                                   Internet
                                       |
                                       |
                                 +-------------+
                                 |    DNS      |
                                 +-------------+
                                       |
                                       |
                         +-----------------------------+
                         |      CDN (CloudFront)       |
                         +-----------------------------+
                                       |
                                       |
                         +-----------------------------+
                         |   WAF + DDoS Protection     |
                         +-----------------------------+
                                       |
                                       |
                         +-----------------------------+
                         |      API Gateway            |
                         +-----------------------------+
                                       |
                                       |
                         +-----------------------------+
                         |     Load Balancer           |
                         +-----------------------------+
                              /        |         \
                             /         |          \
                    +-----------+ +-----------+ +-----------+
                    | App Srv 1 | | App Srv 2 | | App Srv 3 |
                    +-----------+ +-----------+ +-----------+
                         |              |              |
                         +--------------+--------------+
                                        |
                           +--------------------------+
                           |       Redis Cache        |
                           +--------------------------+
                                        |
                                        |
                         +-----------------------------+
                         |      Message Queue          |
                         | RabbitMQ / Kafka / SQS      |
                         +-----------------------------+
                           |        |        |
                           |        |        |
                  +-----------+ +-----------+ +-----------+
                  | Worker A  | | Worker B  | | Worker C  |
                  +-----------+ +-----------+ +-----------+
                           |        |        |
                           +--------+--------+
                                    |
                         +-----------------------------+
                         |      Microservices          |
                         +-----------------------------+
                           |      |        |        |
                     User Service  Order  Payment Search
                           |      |        |        |
                    ------------------------------------
                                    |
                           +----------------------+
                           | Primary Database     |
                           | (SQL / NoSQL)        |
                           +----------------------+
                                    |
                           +----------------------+
                           | Read Replica(s)      |
                           +----------------------+
                                    |
                           +----------------------+
                           | Backup / Data Lake   |
                           +----------------------+
```

---

# Why is the order important?

Every component exists to reduce work for the next one.

```text
User
   |
DNS
   |
CDN
```

👉 Remove requests for static content.

---

```text
CDN
   |
WAF
```

👉 Remove malicious requests.

---

```text
WAF
   |
API Gateway
```

👉 Authenticate, authorize, rate-limit, and route valid API traffic.

---

```text
Gateway
   |
Load Balancer
```

👉 Distribute traffic across healthy application servers.

---

```text
Application
   |
Redis
```

👉 Avoid unnecessary database access.

---

```text
Redis
   |
Database
```

👉 Fetch only data that isn't already cached.

---

```text
Application
   |
Queue
```

👉 Move slow, non-critical work to asynchronous processing.

---

# A Production-Grade Flow (E-commerce Checkout)

Let's walk through a checkout request to see everything working together.

```text
1. User clicks "Place Order"

2. DNS resolves shop.example.com

3. CDN serves the web application's JavaScript (cached)

4. API request POST /orders goes through CDN to WAF

5. WAF validates the request isn't malicious

6. API Gateway:
      - Validates JWT
      - Applies rate limiting
      - Routes to Order Service

7. Load Balancer selects a healthy Order Service instance

8. Order Service:
      - Checks inventory
      - Calculates totals
      - Calls Payment Service
      - Creates the order

9. Redis is used for frequently accessed product or pricing data

10. Database transaction commits the order

11. Events are published to Kafka/RabbitMQ:
      - Send confirmation email
      - Update analytics
      - Notify warehouse
      - Update recommendation engine

12. API returns:
      HTTP 201 Created

13. Background workers process queued events independently
```

This architecture is scalable because **each layer has a single, well-defined responsibility**, and each layer reduces the load on the next one. As traffic grows, you can scale the CDN, API Gateway, load balancers, application servers, cache, workers, and databases independently.

---

## One important correction to what many beginners assume

The pipeline is **not always**:

```text
CDN → WAF → API Gateway → Load Balancer
```

That's a very common and useful architecture, but real production systems vary.

For example:

* Some cloud providers integrate **WAF directly with the CDN or Load Balancer**.
* Some architectures **don't use an API Gateway** for traditional web applications.
* Internal service-to-service traffic often **bypasses the CDN entirely**.
* Large organizations may have **multiple layers of load balancers** (global and regional) before traffic reaches an API Gateway.

In system design interviews, however, the architecture we've discussed is an excellent mental model because it clearly illustrates the role of each component and how they work together in a modern production environment.
