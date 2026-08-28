# API(Application Programming Interface)

- API (Application Programming Interface) is simply a way for two software applications to communicate with each other.

- Real-life Example

Imagine you're sitting in a restaurant.

There are three parties:

You (Customer)
Waiter
Kitchen

You  --->  Waiter  ---> Kitchen
          <---

You don't go into the kitchen.

Instead:

You tell the waiter what you want.
The waiter carries your request.
The kitchen prepares food.
The waiter brings it back.

The waiter is the API.    

# RESTful APIs vs SOAP

> **Interview Notes for System Design & Backend Development**

---

# What is an API?

An **API (Application Programming Interface)** is a contract that allows two applications to communicate with each other.

For example:

```text
React Frontend
      |
      | HTTP Request
      |
.NET Backend API
      |
Database
```

The frontend doesn't know how the database works, and the database doesn't know anything about React.

The **API acts as the communication contract** between them.

---

# What is REST?

**REST** stands for **Representational State Transfer**.

REST is **not a protocol**.

It is an **architectural style** introduced by **Roy Fielding** in his PhD dissertation.

It defines a set of constraints for designing scalable web APIs.

If an API follows these constraints, it is called a **RESTful API**.

```
REST
    ↓
Architectural Style

RESTful API
    ↓
API that follows REST principles
```

---

# Why was REST Introduced?

Before REST became popular, applications communicated using technologies like:

* RPC
* CORBA
* DCOM
* SOAP

Most of these approaches were:

* Complex
* Heavyweight
* Tightly coupled

As the web expanded, developers needed an architecture that was:

* Simple
* Lightweight
* Scalable
* Easy to cache
* Easy to understand

REST solved these problems.

---

# REST is Resource-Oriented

The core idea behind REST is that **everything is treated as a resource**.

Examples of resources:

* Users
* Products
* Orders
* Payments
* Categories

Every resource has its own unique URL.

Examples:

```http
GET /users

GET /users/15

GET /products

GET /orders/45
```

Notice that REST focuses on **resources**, not actions.

---

# REST Uses HTTP

REST builds on top of the HTTP protocol instead of creating a new communication protocol.

HTTP already provides methods that map naturally to CRUD operations.

| HTTP Method | Purpose        |
| ----------- | -------------- |
| GET         | Read           |
| POST        | Create         |
| PUT         | Replace        |
| PATCH       | Partial Update |
| DELETE      | Delete         |

Examples:

```http
POST /users
```

```http
GET /users/10
```

```http
PUT /users/10
```

```http
PATCH /users/10
```

```http
DELETE /users/10
```

---

# Example REST Request

Client requests a user profile:

```http
GET /users/101 HTTP/1.1
Host: api.example.com
```

Server responds:

```json
HTTP/1.1 200 OK

{
  "id": 101,
  "name": "Suraj",
  "followers": 320
}
```

---

# REST Constraints

A RESTful API follows six architectural constraints.

---

## 1. Client-Server Architecture

The client and server are completely independent.

```text
React Client
      |
   HTTP
      |
.NET Backend
      |
 Database
```

The frontend doesn't know:

* Business logic
* Database
* Authentication implementation

The backend doesn't know:

* HTML
* CSS
* React

Both can evolve independently.

---

## 2. Stateless (Most Important)

Every request must contain all the information required to process it.

The server **does not remember previous requests**.

### Stateful Example (Not REST)

Request 1

```text
Login
```

Server stores session.

Request 2

```text
Get Profile
```

Server checks stored session.

This is **stateful**.

---

### Stateless Example (REST)

Every request sends authentication information.

```http
GET /profile

Authorization: Bearer JWT_TOKEN
```

Another request:

```http
GET /orders

Authorization: Bearer JWT_TOKEN
```

The server does not remember anything from previous requests.

---

### Benefits of Statelessness

* Easier horizontal scaling
* Better fault tolerance
* Load balancers can route requests to any server
* No session synchronization required

Example:

```text
           Load Balancer
          /      |      \
      API-1   API-2   API-3
```

If API-1 goes down, another server can immediately handle the next request.

---

## 3. Cacheable

Responses should be cacheable whenever possible.

Example:

```http
GET /countries
```

Country data changes very rarely.

Browsers or CDNs can cache the response.

Benefits:

* Faster response time
* Reduced server load
* Lower network traffic

Common cache headers:

* Cache-Control
* Expires
* ETag

---

## 4. Uniform Interface

All APIs should follow a consistent structure.

Good design:

```http
GET /users

GET /users/10

POST /users

DELETE /users/10
```

Bad design:

```text
getUsers()

deleteUserNow()

createCustomer()

updateEmployee()
```

A uniform interface makes APIs predictable and easier to understand.

---

## 5. Layered System

Clients should not know how many layers exist between them and the server.

Example:

```text
React Client
      |
Load Balancer
      |
API Gateway
      |
Authentication
      |
Microservice
      |
Database
```

The client simply sends:

```http
GET /users
```

It has no knowledge of:

* Reverse proxies
* API gateways
* Firewalls
* Caching layers
* Internal microservices

---

## 6. Code on Demand (Optional)

A server may send executable code to the client.

Example:

* JavaScript downloaded by a web browser.

This constraint is optional and rarely discussed in interviews.

---

# REST Request Structure

Typical GET request:

```http
GET /users/5

Headers:
Authorization: Bearer TOKEN
Accept: application/json
```

GET requests usually have no body.

---

POST request:

```http
POST /users

Content-Type: application/json

{
  "name": "Suraj",
  "age": 25
}
```

---

# Common HTTP Status Codes

| Code | Meaning               |
| ---- | --------------------- |
| 200  | OK                    |
| 201  | Created               |
| 204  | No Content            |
| 400  | Bad Request           |
| 401  | Unauthorized          |
| 403  | Forbidden             |
| 404  | Not Found             |
| 409  | Conflict              |
| 500  | Internal Server Error |

These status codes are an important part of REST APIs.

---

# What is SOAP?

SOAP stands for **Simple Object Access Protocol**.

Unlike REST, SOAP is a **protocol** with strict standards.

---

# SOAP Architecture

```text
Client
   |
SOAP Request (XML)
   |
SOAP Server
   |
SOAP Response (XML)
```

SOAP messages are always written in XML.

Example request:

```xml
<soap:Envelope>
    <soap:Body>
        <GetUser>
            <Id>10</Id>
        </GetUser>
    </soap:Body>
</soap:Envelope>
```

Response:

```xml
<soap:Envelope>
    <soap:Body>
        <User>
            <Id>10</Id>
            <Name>Suraj</Name>
        </User>
    </soap:Body>
</soap:Envelope>
```

---

# SOAP is Operation-Oriented

SOAP focuses on operations (methods).

Examples:

```text
CreateUser()

DeleteUser()

ApproveLoan()

TransferMoney()
```

REST focuses on resources.

```http
POST /users

DELETE /users/10
```

---

# WSDL

SOAP services expose a contract called **WSDL (Web Services Description Language).**

WSDL defines:

* Available operations
* Request format
* Response format
* Data types
* Service endpoint

Client code can be generated automatically from a WSDL file.

---

# WS-* Standards

SOAP supports enterprise standards like:

* WS-Security
* WS-ReliableMessaging
* WS-AtomicTransaction

These provide:

* Encryption
* Digital Signatures
* Reliable Message Delivery
* Distributed Transactions

This is why SOAP is still used in enterprise systems such as banking and government applications.

---

# REST vs SOAP

| Feature        | REST                                 | SOAP                        |
| -------------- | ------------------------------------ | --------------------------- |
| Type           | Architectural Style                  | Protocol                    |
| Data Format    | JSON (Mostly)                        | XML Only                    |
| Performance    | Fast                                 | Slower                      |
| Payload Size   | Small                                | Large                       |
| Learning Curve | Easy                                 | Complex                     |
| HTTP Methods   | GET, POST, PUT, PATCH, DELETE        | Usually POST                |
| Stateless      | Yes                                  | Can be Stateful             |
| Caching        | Supported                            | Limited                     |
| Security       | HTTPS, JWT, OAuth                    | WS-Security                 |
| Contract       | Optional (OpenAPI/Swagger)           | WSDL                        |
| Best For       | Web APIs, Mobile Apps, Microservices | Banking, Enterprise Systems |

---

# Which One Should You Use?

### REST

Choose REST when building:

* Web applications
* Mobile applications
* Microservices
* Cloud-native systems
* Public APIs

---

### SOAP

Choose SOAP when:

* Strong contracts are required
* Enterprise messaging is needed
* Distributed transactions are important
* Working with legacy enterprise systems
* Integrating with banking or government services

---

# Interview Takeaways

You should be able to answer the following questions confidently:

* What is REST?
* Why is REST stateless?
* Why does statelessness improve scalability?
* Why is REST resource-oriented?
* Difference between PUT and PATCH?
* What makes an API truly RESTful?
* Why are HTTP status codes important?
* What is WSDL?
* When should SOAP be preferred over REST?
* Why is REST more popular for modern microservices?

---

# Quick Summary

## REST

* Architectural style
* Resource-oriented
* Uses HTTP methods
* Lightweight
* Mostly JSON
* Stateless
* Easy to scale
* Best for modern web applications

---

## SOAP

* Communication protocol
* Operation-oriented
* XML only
* Strict standards
* WSDL contract
* Enterprise security features
* Better suited for legacy enterprise applications


# GraphQL - Complete Notes for System Design Interviews

---

# Table of Contents

1. What is GraphQL?
2. Why was GraphQL Created?
3. Problems with REST APIs
4. REST vs GraphQL
5. GraphQL Architecture
6. GraphQL Request Flow
7. GraphQL Operations
   - Query
   - Mutation
   - Subscription
8. GraphQL Schema
9. GraphQL Types
10. GraphQL Resolvers
11. Nested Queries
12. Variables
13. Fragments
14. Aliases
15. Directives
16. Error Handling
17. Authentication & Authorization
18. Caching
19. N+1 Query Problem
20. Pagination
21. File Uploads
22. Performance Optimization
23. Advantages
24. Disadvantages
25. REST vs GraphQL Comparison
26. When to Use GraphQL
27. Interview Questions

---

# What is GraphQL?

GraphQL is a **Query Language for APIs** and a **runtime for executing those queries**.

It allows the client to request **exactly the data it needs**.

Unlike REST, the server does **not decide** the response structure.

The client decides.

Think of GraphQL like SQL for APIs.

Instead of writing

```sql
SELECT *
FROM Users
```

you write

```graphql
query{
    user(id:1){
        name
        age
    }
}
```

The server only returns

```json
{
  "data":{
      "user":{
          "name":"Suraj",
          "age":24
      }
  }
}
```

Nothing extra.

---

# Why was GraphQL Created?

GraphQL was created by Facebook in 2012 and open sourced in 2015.

It solved two major REST problems:

- Over Fetching
- Under Fetching

---

# Problem 1 - Over Fetching

Suppose we call

```
GET /users/1
```

REST Response

```json
{
 id:1,
 name:"Suraj",
 age:24,
 phone:"999999",
 email:"abc@gmail.com",
 address:"India",
 followers:100,
 following:50,
 company:"ABC"
}
```

Our UI only needs

- name
- followers

Still, REST returns everything.

This wastes:

- Network bandwidth
- Parsing time
- Memory

This is called

> Over Fetching

---

# Problem 2 - Under Fetching

Suppose Profile Page needs

- User
- Posts
- Followers

REST

```
GET /users/1

GET /users/1/posts

GET /users/1/followers
```

Three network requests.

This is

> Under Fetching

because one endpoint doesn't provide everything required.

---

# REST vs GraphQL

REST

```
GET /users

GET /posts

GET /orders

GET /comments
```

Multiple URLs.

GraphQL

```
POST /graphql
```

Only one endpoint.

---

REST

Server decides response.

GraphQL

Client decides response.

---

REST

```
GET /users/1
```

Response

```json
{
 name,
 age,
 email,
 phone,
 address
}
```

GraphQL

```graphql
query{

    user(id:1){

        name
        age

    }

}
```

Response

```json
{
 "data":{
    "user":{
        "name":"Suraj",
        "age":24
    }
 }
}
```

---

# GraphQL Architecture

```
                Client

                  │

          POST /graphql

                  │

          GraphQL Server

                  │

             Resolver

                  │

          Business Logic

                  │

             Database
```

Notice

GraphQL does NOT replace backend.

It simply changes how clients ask for data.

---

# Request Flow

Step 1

Client sends

```graphql
query{

    user(id:1){

        name

    }

}
```

↓

GraphQL parses query

↓

Validates schema

↓

Calls Resolver

↓

Resolver queries database

↓

Returns response

↓

GraphQL formats response

↓

Client receives

```json
{
 "data":{
    "user":{
       "name":"Suraj"
    }
 }
}
```

---

# GraphQL Operations

GraphQL supports three operations.

## 1. Query

Used to read data.

```graphql
query{

    user(id:1){

        name

    }

}
```

Equivalent to REST GET.

---

## 2. Mutation

Used to create/update/delete.

```graphql
mutation{

    createUser(

        name:"Suraj"

        age:24

    ){

        id

        name

    }

}
```

Equivalent to

- POST
- PUT
- PATCH
- DELETE

---

## 3. Subscription

Used for real-time communication.

Examples

- WhatsApp
- Live Chat
- Stock Prices
- Notifications

Example

```graphql
subscription{

    messageAdded{

        text

    }

}
```

Usually implemented using

- WebSockets
- Server Sent Events

---

# Schema

Schema defines every object in GraphQL.

Example

```graphql
type User{

    id:ID!

    name:String!

    age:Int!

}
```

The server now knows

User contains

- id
- name
- age

---

# Query Schema

```graphql
type Query{

    user(id:ID!):User

}
```

Meaning

```
Query

↓

user()

↓

returns User
```

---

# Mutation Schema

```graphql
type Mutation{

    createUser(

        name:String!

        age:Int!

    ):User

}
```

---

# Common GraphQL Types

```
String

Int

Float

Boolean

ID
```

Lists

```graphql
[User]
```

Means

Array of Users

---

Non Null

```graphql
String!
```

Means

Cannot be null.

---

# Resolvers

Resolvers fetch actual data.

Example

```javascript
const resolvers={

 Query:{

    user(parent,args){

        return User.findById(args.id);

    }

 }

}
```

Flow

```
GraphQL Query

↓

Resolver

↓

Database

↓

Response
```

Resolvers are similar to Controllers in REST.

---

# Nested Queries

Schema

```graphql
type User{

    name:String!

    posts:[Post]

}
```

Query

```graphql
query{

 user(id:1){

     name

     posts{

         title

     }

 }

}
```

One request.

Nested response.

---

# Variables

Instead of hardcoding values

```graphql
query{

 user(id:1){

     name

 }

}
```

Use variables.

```graphql
query GetUser($id:ID!){

 user(id:$id){

    name

 }

}
```

Variables

```json
{
 "id":1
}
```

Benefits

- Cleaner
- Reusable
- More secure

---

# Fragments

Avoid duplicate fields.

Without fragment

```graphql
user{

 name

 age

 profilePic

}

friend{

 name

 age

 profilePic

}
```

With fragment

```graphql
fragment UserFields on User{

 name

 age

 profilePic

}
```

Usage

```graphql
user{

 ...UserFields

}

friend{

 ...UserFields

}
```

---

# Aliases

Query same field twice.

Example

```graphql
query{

 user1:user(id:1){

    name

 }

 user2:user(id:2){

    name

 }

}
```

Response

```json
{
 user1:{...},
 user2:{...}
}
```

---

# Directives

Conditionally include fields.

```graphql
query{

 user{

   name

   phone @include(if:true)

 }

}
```

Common directives

- @include
- @skip
- @deprecated

---

# Error Handling

Even when some fields fail

GraphQL still returns partial data.

Example

```json
{
 "data":{
    "user":{
       "name":"Suraj",
       "posts":null
    },
 "errors":[]
}
```

Unlike REST

GraphQL can return

Both

- data
- errors

Together.

---

# Authentication

GraphQL itself does NOT provide authentication.

Usually

```
Authorization

Bearer Token
```

Headers

```
POST /graphql
```

Resolver checks token.

---

# Authorization

Authentication

Who are you?

Authorization

What can you access?

Usually checked inside Resolver.

Example

```javascript
if(user.role!="Admin")

throw Error("Forbidden");
```

---

# Caching

REST

Easy

```
GET /users/1
```

HTTP Cache

CDN Cache

Browser Cache

GraphQL

Harder

Because every query is different.

Solutions

- Apollo Cache
- Redis
- Persisted Queries
- CDN for persisted queries

---

# N+1 Query Problem

Very common interview question.

Suppose

```graphql
query{

 users{

    name

    posts{

        title

    }

 }

}
```

100 users

GraphQL may execute

```
SELECT * FROM Users

100 Queries

SELECT * FROM Posts WHERE userId=?
```

Total

101 queries

Very slow.

---

Solution

DataLoader

Instead

```
SELECT *

FROM Posts

WHERE userId IN (...)

```

Only

2 database queries.

Huge performance improvement.

---

# Pagination

Never fetch

100000 records.

Use Cursor Pagination.

Example

```graphql
query{

 posts(

   first:10

   after:"cursor"

 ){

   edges{

      node{

         title

      }

   }

 }

}
```

Preferred over

Offset Pagination

for large datasets.

---

# File Upload

GraphQL doesn't support files by default.

Usually use

Multipart Request Specification

or

Separate REST endpoint.

---

# Performance Optimization

Use

- DataLoader
- Query Depth Limit
- Query Complexity Analysis
- Redis Cache
- Persisted Queries
- Cursor Pagination
- Lazy Loading
- Batching

---

# Advantages

✅ No Over Fetching

Only requested fields returned.

---

✅ No Under Fetching

Nested data in one request.

---

✅ Strongly Typed

Schema validation.

---

✅ Self Documenting

Schema generates documentation automatically.

---

✅ Single Endpoint

Easy API management.

---

✅ Better Mobile Performance

Less bandwidth.

---

# Disadvantages

❌ Complex Server

Resolvers require careful design.

---

❌ Hard Caching

HTTP caching less effective.

---

❌ Query Complexity

Clients can request deeply nested data.

---

❌ N+1 Problem

Needs batching.

---

❌ Learning Curve

Schema and resolver concepts take time.

---

# REST vs GraphQL

| Feature | REST | GraphQL |
|----------|------|----------|
| Endpoints | Multiple | Single |
| Data Fetching | Fixed | Client Controlled |
| Over Fetching | Yes | No |
| Under Fetching | Yes | No |
| Versioning | Common | Rare |
| HTTP Methods | GET POST PUT DELETE | Mostly POST |
| Caching | Easy | Difficult |
| Documentation | Swagger/OpenAPI | Auto Generated Schema |
| Strong Typing | Optional | Built-in |

---

# When to Use GraphQL

Choose GraphQL when

- Mobile Apps
- Dashboards
- Multiple frontend teams
- Complex relationships
- Public APIs
- Different clients require different data

Choose REST when

- Simple CRUD
- Internal microservices
- File uploads
- Heavy HTTP caching
- Small projects

---

# Real World Companies Using GraphQL

- Facebook (Creator)
- GitHub
- Shopify
- Airbnb
- Netflix (selected services)
- Pinterest
- Twitter (selected services)

---

# Best Practices

✅ Keep schema simple.

✅ Use DataLoader.

✅ Limit query depth.

✅ Use pagination.

✅ Validate input.

✅ Cache expensive queries.

✅ Avoid business logic inside resolvers.

✅ Use persisted queries for production.

---

# Common Interview Questions

### Basic

1. What is GraphQL?
2. Why was GraphQL created?
3. Difference between REST and GraphQL?
4. What is Over Fetching?
5. What is Under Fetching?

---

### Intermediate

6. What are Queries?
7. What are Mutations?
8. What are Subscriptions?
9. What are Resolvers?
10. What is a Schema?

---

### Advanced

11. Explain N+1 Query Problem.
12. What is DataLoader?
13. How is caching handled?
14. How does authentication work?
15. How would you secure a GraphQL API?
16. Explain query complexity analysis.
17. Explain persisted queries.
18. Explain cursor pagination.
19. How does GraphQL execute nested queries?
20. When would you choose REST instead of GraphQL?

---

# Final Summary

GraphQL is not a database and not a replacement for REST. It is a query language for APIs that allows clients to request exactly the data they need. It solves REST's over-fetching and under-fetching problems through a strongly typed schema, resolvers, and a single endpoint. While GraphQL offers flexibility and efficient data retrieval, it introduces challenges such as caching complexity, the N+1 query problem, and query cost management. Understanding schemas, resolvers, DataLoader, pagination, caching, authentication, and security is essential for production systems and system design interviews.


# gRPC Notes (Part 1) – Introduction to gRPC

> These notes are designed for **Software Engineering and System Design interviews**. They focus on understanding **what gRPC is, why it exists, and how it differs from REST**.

---

# Table of Contents

1. What is gRPC?
2. Why was gRPC Created?
3. What is RPC?
4. Local Procedure Call vs Remote Procedure Call
5. REST vs gRPC Communication
6. Why Developers Like gRPC
7. REST vs gRPC Mindset
8. Advantages of gRPC
9. When to Use gRPC
10. Key Interview Takeaways

---

# 1. What is gRPC?

**gRPC (Google Remote Procedure Call)** is a high-performance, open-source **Remote Procedure Call (RPC)** framework developed by Google.

It allows one application to **call methods on another application running on a different machine as if they were local methods**.

Instead of interacting with REST endpoints such as:

```http
GET /users/123
POST /orders
DELETE /products/5
```

you directly call methods like:

```csharp
userClient.GetUser(request);

orderClient.CreateOrder(request);

paymentClient.ProcessPayment(request);
```

The networking details are completely hidden from the developer.

---

# Definition

> gRPC is a framework that enables applications to communicate by invoking methods on remote servers as though they were local function calls.

---

# 2. Why was gRPC Created?

Google operates thousands of microservices communicating with each other millions of times every second.

Traditional REST APIs introduced overhead because every request required:

* HTTP headers
* JSON serialization
* JSON parsing
* UTF-8 encoding
* Multiple HTTP connections

Although REST is excellent for public APIs, Google required something that was:

* Faster
* Smaller payload size
* Strongly typed
* Streaming capable
* Efficient for internal communication

To solve these problems, Google developed **gRPC**.

---

# 3. What is RPC?

RPC stands for **Remote Procedure Call**.

The idea is simple:

Instead of sending HTTP requests manually, you invoke a function.

The framework automatically handles:

* Network communication
* Serialization
* Request transmission
* Response reception
* Error handling

The developer simply calls a method.

---

## Local Procedure Call

Everything executes inside the same process.

```csharp
public class Calculator
{
    public int Add(int a, int b)
    {
        return a + b;
    }
}

Calculator calculator = new Calculator();

int result = calculator.Add(2, 3);
```

Execution flow:

```
Application
      │
      ▼
Calculator.Add()
      │
      ▼
Returns Result
```

No network is involved.

---

## Remote Procedure Call

Suppose the Calculator service exists on another machine.

Your application still writes:

```csharp
calculatorClient.Add(new AddRequest
{
    A = 2,
    B = 3
});
```

However, internally:

```
Client

    │

Serialize Request

    │

HTTP/2

    │

Network

    │

Server

    │

Execute Method

    │

Serialize Response

    │

Network

    │

Client Receives Result
```

To the developer, it still feels like:

```csharp
Add(2, 3);
```

This is the core idea behind RPC.

---

# 4. REST vs gRPC Communication

## REST Approach

Request:

```http
POST /add
```

Request Body

```json
{
    "a": 2,
    "b": 3
}
```

Response

```json
{
    "result": 5
}
```

Developer must think about:

* URL
* HTTP Method
* Headers
* JSON
* Serialization
* Deserialization
* Status Codes

---

## gRPC Approach

```csharp
var response = calculatorClient.Add(
    new AddRequest
    {
        A = 2,
        B = 3
    });
```

The networking details are abstracted away.

---

# 5. Why Developers Like gRPC

Developers think in terms of **functions**, not HTTP requests.

Instead of remembering endpoints like:

```
POST /payment/process

GET /orders

DELETE /users/10
```

They simply call:

```csharp
paymentClient.ProcessPayment();

orderClient.GetOrders();

userClient.DeleteUser();
```

This improves:

* Productivity
* Readability
* Type safety
* Maintainability

---

# 6. The Hidden Work Done by gRPC

Although it looks like a normal method call, internally the following happens:

```
Application

      │

Method Call

      │

Serialize Request

      │

HTTP/2

      │

Network

      │

Server

      │

Execute Method

      │

Serialize Response

      │

Network

      │

Deserialize

      │

Return Object
```

The framework hides all networking complexity.

---

# 7. REST vs gRPC Mindset

## REST thinks in Resources

REST models everything as resources.

Examples:

```
GET /users/5

GET /orders/10

DELETE /users/5
```

Focus:

> "What resource am I accessing?"

---

## gRPC thinks in Functions

Examples:

```text
CreateUser()

DeleteUser()

GenerateInvoice()

CalculateSalary()

GetRecommendations()
```

Focus:

> "What operation do I want to perform?"

---

# 8. Advantages of gRPC

* High performance
* Smaller payloads
* Strongly typed contracts
* Automatic client/server code generation
* HTTP/2 support
* Bi-directional streaming
* Better suited for microservices
* Language independent
* Excellent developer experience

---

# 9. When Should You Use gRPC?

gRPC is ideal for:

* Internal microservice communication
* High-performance systems
* Low latency applications
* Real-time communication
* Streaming data
* Large-scale distributed systems

REST is generally better for:

* Public APIs
* Browser-based applications
* Third-party integrations
* APIs consumed by external clients

Many modern systems use both:

```
                 Browser
                     │
                     │ REST
                     ▼
              API Gateway
                     │
      ┌──────────────┴──────────────┐
      │                             │
      ▼                             ▼
 Order Service  ←────gRPC────→ Inventory Service
      │
      ▼
 Payment Service
```

REST is used externally, while gRPC is used internally between services.

---

# 10. Key Interview Takeaways

### What is gRPC?

A high-performance RPC framework developed by Google that allows applications to invoke methods on remote servers as if they were local function calls.

---

### Why was gRPC created?

To enable faster, more efficient communication between Google's internal microservices by reducing network overhead and improving performance.

---

### What is RPC?

RPC (Remote Procedure Call) allows a program to execute methods on another machine without manually handling network communication.

---

### REST vs gRPC

| REST                | gRPC                        |
| ------------------- | --------------------------- |
| Resource-oriented   | Function-oriented           |
| Uses HTTP endpoints | Uses remote methods         |
| JSON                | Protocol Buffers (Binary)   |
| Mostly HTTP/1.1     | HTTP/2                      |
| Easy for browsers   | Excellent for microservices |
| Human-readable      | Optimized for machines      |

---

# Summary

* gRPC stands for **Google Remote Procedure Call**.
* It allows applications to invoke methods on remote machines just like local methods.
* The framework hides serialization, networking, and communication details.
* Google created gRPC to improve communication between thousands of internal microservices.
* Unlike REST, which is resource-oriented, gRPC is operation/function-oriented.
* gRPC is typically used for **internal service-to-service communication**, while REST is preferred for **public-facing APIs**.

---

## What's Next?

The next topic is:

> **Why is gRPC significantly faster than REST?**

We'll explore:

* HTTP/1.1 vs HTTP/2
* Binary Protocol Buffers vs JSON
* Multiplexing
* Header Compression
* Streaming
* Connection Reuse
* Benchmarks
* Internal request lifecycle

This is one of the most frequently discussed topics in backend and system design interviews.
