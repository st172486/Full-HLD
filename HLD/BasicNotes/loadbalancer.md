# Loadbalancer

Excellent topic. A **Load Balancer** is one of the most important components in System Design. Almost every large-scale system (Google, Netflix, Amazon, Instagram, Uber) has one or more load balancers.

Don't think of a load balancer as just **"distributing traffic."**

Think of it as the **traffic police + security guard + health inspector + router + sometimes cache** for your servers.

We'll go very deep, starting from the basics and gradually moving toward advanced concepts.

---

# Chapter 1: Why do we even need a Load Balancer?

Imagine you're building a website.

Initially you have one server.

```
Users
   |
   |
Internet
   |
+------------+
| Web Server |
+------------+
```

Everything works.

Now suppose your application becomes famous.

Instead of

```
100 users/day
```

you start getting

```
100,000 users/day
```

Your server starts dying.

Why?

Because every request consumes resources.

Example

```
Request 1
    CPU

Request 2
    Memory

Request 3
    Database connection

Request 4
    Thread

Request 5
    Network Socket
```

Eventually

```
CPU = 100%
Memory = Full
Threads = Exhausted

Server Crashes
```

---

So you buy another server.

```
Server 1

Server 2
```

Now comes the question...

How will users know which server to call?

```
www.amazon.com

Should it go to

Server 1 ?

or

Server 2 ?
```

That's where a Load Balancer comes in.

---

# Chapter 2: Load Balancer acts like Receptionist

Imagine a hospital.

```
Patients
     |
Receptionist
     |
------------------------
|        |            |
Doctor1 Doctor2 Doctor3
```

Patient never decides which doctor.

Receptionist decides.

Load Balancer is exactly that receptionist.

```
Users
      |
      |
+----------------+
| Load Balancer  |
+----------------+
   |      |
   |      |
Server1 Server2
```

Users never directly call servers.

---

# Chapter 3: Real Network Flow

Let's understand an actual network call.

Suppose user types

```
www.netflix.com
```

What actually happens?

---

## Step 1

Browser asks DNS

```
Where is netflix.com?
```

DNS replies

```
52.16.120.14
```

That IP is usually NOT the application server.

It is usually the Load Balancer.

```
Browser
     |
DNS Lookup
     |
Load Balancer IP
```

---

## Step 2

Browser opens TCP connection

```
Browser

    SYN
--------->

Load Balancer

    SYN ACK
<----------

Browser

ACK
---------->
```

TCP connection established.

---

Then Browser sends

```
GET /movies
```

to

Load Balancer.

---

Notice

Browser doesn't know anything about

```
Application Server 1

Application Server 2

Application Server 3
```

It only knows

```
Load Balancer
```

---

# Chapter 4: What happens inside Load Balancer?

This is where things become interesting.

Suppose

```
Server1

Server2

Server3
```

Load Balancer has a routing table.

```
Healthy Servers

Server1 ✔

Server2 ✔

Server3 ✔
```

When request arrives

```
GET /movies
```

Load Balancer decides

```
Send to Server2
```

Request becomes

```
Client

      |
      |
Load Balancer

      |
      |
Server2
```

---

Server2 processes

```
SELECT movies

Prepare JSON

Return Response
```

Server2 returns response

```
200 OK
```

to Load Balancer.

Load Balancer forwards response to browser.

Browser thinks

```
I talked to one server.
```

Actually

```
Browser

↓

Load Balancer

↓

Server2

↓

Load Balancer

↓

Browser
```

---

# Chapter 5: Does Load Balancer modify packets?

This is a very common interview question.

Answer:

Depends.

There are two major types.

---

## Layer 4 Load Balancer

Works at

```
TCP
UDP
```

It doesn't care about

```
GET /movies

POST /login

Authorization

Cookies
```

It only sees

```
IP Address

Port Number
```

Think of it like

```
Forward packet to server.
```

Nothing else.

---

## Layer 7 Load Balancer

Much smarter.

It understands

```
HTTP

HTTPS

Headers

Cookies

Path

Method

JWT

Host
```

It can say

```
If URL starts with

/api

Go to Backend

Else

Go to Frontend
```

Example

```
GET /images

↓

Image Server
```

```
GET /payment

↓

Payment Server
```

```
GET /search

↓

Search Server
```

All from same domain.

---

# Chapter 6: Multiple Servers

Suppose we have

```
             LB
              |
--------------------------------
|              |              |
S1             S2             S3
```

100 requests arrive.

Load Balancer distributes them.

Example

```
Request1 → S1

Request2 → S2

Request3 → S3

Request4 → S1

Request5 → S2
```

Now no server becomes overloaded.

---

# Chapter 7: How does LB choose a server?

There are many algorithms.

---

## Round Robin

Simplest.

```
S1

S2

S3
```

Requests

```
1 → S1

2 → S2

3 → S3

4 → S1

5 → S2
```

Very easy.

Problem?

What if

```
S1

Powerful

32 CPU
```

and

```
S2

2 CPU
```

Both receive same traffic.

Bad.

---

## Weighted Round Robin

Give weight.

```
S1 weight = 5

S2 weight = 1

S3 weight = 1
```

Traffic becomes

```
S1

S1

S1

S1

S1

S2

S3
```

Better.

---

## Least Connections

Suppose

```
Server1

100 Active Users
```

Server2

```
10 Active Users
```

New request goes to

```
Server2
```

because

```
Least Active Connections
```

Very common.

---

## Least Response Time

Measure

```
S1

20 ms
```

S2

```
300 ms
```

Send traffic to

```
S1
```

---

## IP Hash

Hash

```
Client IP
```

Same user

Always

Same server.

Useful for

```
Sticky Sessions
```

We'll discuss later.

---

# Chapter 8: Health Checks

Imagine

```
S1

S2

S3
```

Suddenly

```
S2 crashes.
```

If LB still sends requests there...

```
Everything fails.
```

So Load Balancer continuously checks

```
GET /health
```

Every few seconds.

Response

```
200 OK
```

means

Healthy.

If

```
500

Timeout

Connection Refused
```

Load Balancer marks server unhealthy.

```
Healthy

S1 ✔

S2 ❌

S3 ✔
```

Now

No traffic goes to S2.

---

# Chapter 9: When Server Comes Back

Suppose

```
S2 rebooted.
```

Health check

```
200 OK
```

Load Balancer automatically adds it back.

No manual work.

---

# Chapter 10: One request journey

Let's follow one request end-to-end.

```
Browser

↓

DNS

↓

Load Balancer

↓

Select Server

↓

Forward Request

↓

Server

↓

Database

↓

Server

↓

Load Balancer

↓

Browser
```

Every request follows similar steps.

---

# Chapter 11: Can multiple Load Balancers exist?

Absolutely—and in production they almost always do.

```
                 Users
                    |
              DNS / Anycast
                    |
          +-------------------+
          |                   |
     Load Balancer A     Load Balancer B
          |                   |
     +----+----+         +----+----+
     |         |         |         |
   Server1   Server2   Server3   Server4
```

Having only one load balancer would create a single point of failure. We'll cover highly available load balancer setups, virtual IPs, failover, and global load balancing later.

---

# Key Takeaways

1. Users generally connect to the load balancer, not directly to application servers.
2. The load balancer chooses a backend server using a routing algorithm.
3. Layer 4 load balancers route based on network information (IP/port), while Layer 7 load balancers understand HTTP and can make content-aware routing decisions.
4. Health checks ensure traffic is only sent to healthy servers.
5. Different balancing algorithms (Round Robin, Least Connections, Weighted, IP Hash, etc.) suit different workloads.
6. A load balancer is much more than a traffic distributor—it improves scalability, availability, fault tolerance, and flexibility.

---

## This is only the foundation.

To truly master load balancers from a system design perspective, the next topics are where most interviewers focus:

1. **How packets actually travel through a Layer 4 load balancer (NAT, Direct Server Return, Full Proxy).**
2. **How Layer 7 proxies terminate HTTPS (TLS termination) and why this reduces server load.**
3. **Reverse Proxy vs Forward Proxy vs Load Balancer.**
4. **Session persistence (Sticky Sessions): why they're needed, how cookies/IP hashing work, and why distributed sessions are often preferred.**
5. **How load balancers maintain millions of concurrent TCP connections efficiently.**
6. **Connection pooling, keep-alive, and HTTP/2 multiplexing.**
7. **What happens when the load balancer itself fails (High Availability with active-active and active-passive setups).**
8. **Global load balancing using DNS, Anycast, and geographically distributed traffic routing.**
9. **Load balancers in cloud platforms (AWS ALB/NLB, GCP, Azure) and how they differ.**
10. **How requests flow through a CDN, WAF, API Gateway, Load Balancer, application servers, and databases in a modern production architecture.**

I recommend we cover these in order because each builds directly on the previous one, giving you a deep understanding of how companies like Netflix, Amazon, and Google route billions of requests every day.

In real life loadbalancers are the combination of multiple algorithms... like round robin + stiky sessions right if you see this read about it.