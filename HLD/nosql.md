# NO SQL

Great question. This is actually one of the most important topics in System Design interviews.

People often say **NoSQL** as if it's one database technology, but **NoSQL is actually a family of databases**. Different databases are optimized for different kinds of problems.

There are **four major types of NoSQL databases**:

```
                NoSQL Databases
                      |
   -----------------------------------------
   |            |            |             |
Key-Value    Document     Column       Graph
 Database      Database     Family      Database
```

Let's understand each one deeply from a system design perspective.

---

# 1. Key-Value Database

Think of it as a giant HashMap.

```
Key                Value
--------------------------------
user:101      ->   {...}
user:102      ->   {...}
cart:501      ->   {...}
session:abcd  ->   {...}
```

Everything is stored as

```
Key  ---> Value
```

The database only understands the key.

It has **no idea** what's inside the value.

For example

```
Key:

User123

Value:

{
    Name : "Suraj",
    Age : 24,
    City : Hyderabad
}
```

The database simply stores

```
User123
   |
   |
   V
Binary Data
```

It never indexes

```
Age
City
Name
```

unless you build it yourself.

---

## Example Databases

* Redis
* Amazon DynamoDB (can also behave like document)
* Riak
* Aerospike

---

## Time Complexity

```
GET(Key)

O(1)
```

because internally

```
Hash Table
```

or

```
Consistent Hashing
```

is used.

---

## Advantages

Very Fast

```
Read

Write

Delete
```

All are extremely fast.

Perfect for

* Session Storage
* Cache
* Shopping Cart
* OTP
* Token Storage
* User Preferences

---

## Disadvantages

Suppose you ask

```
Find all users
whose age > 30
```

Impossible.

Why?

Because database only knows

```
Key
```

not

```
Age
```

---

## System Design Use Cases

Facebook

```
Session

sessionId

↓

User Data
```

Netflix

```
Movie Cache
```

Instagram

```
Feed Cache
```

---

# 2. Document Database

Instead of storing binary values, it stores structured documents.

Usually

```
JSON
```

or

```
BSON
```

---

Example

```
{
   "_id":101,
   "name":"Suraj",
   "age":24,
   "city":"Hyderabad",
   "skills":[
       "React",
       ".NET"
   ]
}
```

Each object is called a

```
Document
```

Collection

```
Users

    |
    |--- Document
    |--- Document
    |--- Document
```

Almost similar to

```
Table

↓

Rows
```

in SQL.

---

## Example Databases

* MongoDB
* Couchbase
* Amazon DocumentDB
* CouchDB

---

## Advantages

Flexible Schema

Today

```
{
 Name,
 Age
}
```

Tomorrow

```
{
 Name,
 Age,
 Address,
 Skills,
 Salary,
 Education
}
```

No migration required.

---

Supports queries

```
Age > 30

City = Hyderabad

Salary > 20L

Skill = React
```

Very developer friendly.

---

Nested Objects

```
{
 Address :
 {
    City,
    State,
    Country
 }
}
```

Arrays

```
Skills

[
 React,
 Docker,
 AWS
]
```

---

Perfect for

* User Profiles
* CMS
* Product Catalog
* Orders
* Blogging
* Ecommerce

---

## Disadvantages

Joins are limited compared to SQL databases, and complex multi-document transactions may be more expensive or less central to the design.

---

# 3. Column Family Database

This is the most misunderstood NoSQL database.

It is NOT

```
Excel Columns
```

Instead

Columns are grouped into

```
Column Families
```

Example

```
UserID

101

Name

Suraj

Age

24

City

Hyderabad
```

Another user

```
102

Name

Rahul

Age

NULL

Salary

20L
```

Notice

Every row can have

different columns.

No fixed schema.

---

Internally

Data is stored column-wise.

Instead of

```
Row1

Name
Age
Salary
```

It stores

```
Column

↓

Name

Suraj

Rahul

Amit

Riya
```

Age

```
24
30
22
18
```

Salary

```
20L
15L
30L
```

Reading one column becomes extremely fast.

---

## Example Databases

* Apache Cassandra
* Apache HBase
* Google Bigtable
* ScyllaDB

---

## Advantages

Extremely scalable

Handles

```
Petabytes
```

of data.

Millions of writes/sec.

High availability.

Very good for

Time-series data

Logging

IoT

Analytics

Messaging

Telemetry

---

## Disadvantages

Complex data modeling.

Not good for joins.

Requires partition key design.

---

## Real World

Instagram

Likes

Twitter

Timeline

Uber

Ride Events

Netflix

Viewing History

---

# 4. Graph Database

Instead of storing rows,

it stores

```
Nodes

and

Relationships
```

Example

```
Suraj

      Friend

Rahul

      Friend

Amit

      Friend

Riya
```

Everything becomes a graph.

---

Each node

```
User
```

Each edge

```
Friend
```

---

Example

```
Suraj

  |

Lives In

  |

Hyderabad

  |

Located In

  |

India
```

---

Example

```
Suraj

LIKES

Marvel

WATCHED

Avengers

FRIEND

Rahul
```

---

## Example Databases

* Neo4j
* Amazon Neptune
* TigerGraph
* JanusGraph

---

## Advantages

Relationship queries are extremely fast.

Example

```
Friends of friends

↓

Friends of friends

↓

People who bought this

↓

Shortest path

↓

Recommendation Engine
```

---

## Perfect For

Facebook

```
Friend Network
```

LinkedIn

```
Professional Connections
```

Google Maps

```
Shortest Route
```

Fraud Detection

```
Money Transfer Graph
```

Recommendation Systems

```
Netflix

Amazon
```

Knowledge Graphs

---

## Disadvantages

Not ideal for large-scale aggregations, reporting, or simple key-based lookups. Horizontal scaling can also be more challenging than with some other NoSQL databases.

---

# Complete Comparison

| Feature           | Key-Value               | Document              | Column Family                                          | Graph                                        |
| ----------------- | ----------------------- | --------------------- | ------------------------------------------------------ | -------------------------------------------- |
| Data Model        | Key → Value             | JSON/BSON Documents   | Rows with dynamic columns grouped into column families | Nodes and Relationships                      |
| Schema            | None                    | Flexible              | Flexible                                               | Flexible                                     |
| Query Capability  | By Key Only             | Rich document queries | Partition/range queries                                | Graph traversals                             |
| Speed             | Fastest for key lookups | Fast                  | Excellent for high write throughput                    | Excellent for relationship queries           |
| Best For          | Cache, Sessions         | User Profiles, Orders | Analytics, IoT, Logs                                   | Social Networks, Recommendations             |
| Scalability       | Excellent               | Excellent             | Excellent                                              | Good, but depends on graph size and workload |
| Relationships     | Poor                    | Limited               | Limited                                                | Excellent                                    |
| Example Databases | Redis, DynamoDB         | MongoDB, Couchbase    | Cassandra, HBase                                       | Neo4j, Amazon Neptune                        |

# How to Choose in System Design Interviews

A common interview question is: **"Which NoSQL database would you choose and why?"**

* **Need ultra-fast lookups by a unique key?** → **Key-Value** (e.g., Redis for caching sessions or tokens).
* **Need flexible records with rich querying?** → **Document** (e.g., MongoDB for user profiles or product catalogs).
* **Need to ingest huge volumes of data with high write throughput?** → **Wide-Column** (e.g., Cassandra for logs, metrics, or time-series events).
* **Need to model and traverse complex relationships?** → **Graph** (e.g., Neo4j for social networks, recommendations, or fraud detection).

As a system designer, the choice isn't about which database is "best"—it's about selecting the one whose data model and access patterns match your application's requirements.
