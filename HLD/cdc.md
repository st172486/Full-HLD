# CDC - change data capture

Perfect. **CDC (Change Data Capture)** is one of the most important concepts in distributed systems, especially when we want to keep **PostgreSQL and Elasticsearch in sync**.

Let's do a deep dive.

---

# What is CDC (Change Data Capture)?

CDC is a mechanism that captures every change happening in your database and publishes those changes as events.

Instead of saying

> "Every 5 minutes read all records and update Elasticsearch"

we say

> "Whenever a row changes, immediately tell me."

So CDC converts database changes into an event stream.

```
INSERT
UPDATE
DELETE

↓

Event
```

---

# Without CDC

Suppose your Ticket table is

```
Events

Id     Name           City
-----------------------------
1      Coldplay       Delhi
2      IPL Final      Mumbai
```

User updates

```
UPDATE Events
SET City='Hyderabad'
WHERE Id=1
```

Now PostgreSQL becomes

```
Id     Name           City
-----------------------------
1      Coldplay       Hyderabad
2      IPL Final      Mumbai
```

But Elasticsearch still has

```
Id     Name           City
-----------------------------
1      Coldplay       Delhi
```

Search becomes inconsistent.

---

## Traditional Solution

Every minute

```
SELECT *
FROM Events
WHERE UpdatedAt > LastSyncTime
```

Then update ES.

Problems

* expensive queries
* delay
* may miss updates
* polling wastes CPU

---

# CDC Solution

Instead

```
UPDATE Event

↓

Database notices change

↓

CDC captures it

↓

Kafka Event

↓

Indexer updates Elasticsearch
```

No polling.

No periodic jobs.

Almost real time.

---

# How does PostgreSQL know a row changed?

This is where things become interesting.

Whenever PostgreSQL performs

```
INSERT
UPDATE
DELETE
```

it writes those operations into something called the

## WAL

**Write Ahead Log**

Every database has transaction logs.

Before updating actual data

Postgres first writes

```
UPDATE Event
ID=1

City changed

Old=Delhi

New=Hyderabad
```

inside WAL.

Then only transaction commits.

---

Imagine

```
Client

↓

UPDATE Event

↓

WAL
(write operation)

↓

Actual Table
```

Everything first goes to WAL.

Originally WAL exists for

* crash recovery
* replication
* backup

CDC simply reads this WAL.

It never scans tables.

---

# Example

Current row

```
Id=10

Artist=Coldplay

Seats=5000
```

User books tickets.

```
Seats = 4998
```

WAL contains something like

```
UPDATE Events

PrimaryKey =10

Old Seats=5000

New Seats=4998
```

CDC reads this.

---

# Where does CDC run?

Usually

```
                PostgreSQL
                     |
                 WAL Logs
                     |
               CDC Connector
               (Debezium)
                     |
                   Kafka
                     |
          Search Indexer Service
                     |
              Elasticsearch
```

Most companies use

* **Debezium**
* **Maxwell** (MySQL)
* AWS DMS
* Logical Replication

---

# Example Timeline

### Time 10:00

```
INSERT

Event Id=100

Concert=Adele
```

Immediately

CDC publishes

```
{
   operation : INSERT,

   table : Events,

   id :100,

   Concert :"Adele"
}
```

Indexer

↓

ES inserts document.

---

### 10:05

Admin changes venue.

```
UPDATE

Venue

Mumbai

↓

Delhi
```

CDC publishes

```
{
  operation:"UPDATE",

  before:{
      venue:"Mumbai"
  },

  after:{
      venue:"Delhi"
  }
}
```

Indexer updates ES.

---

### 10:07

Delete event.

```
DELETE

Id=100
```

CDC

```
{
 operation:"DELETE",

 id:100
}
```

Indexer removes document.

---

Everything is automatic.

---

# What exactly does CDC publish?

Suppose database row

```
Id=5

Artist=Coldplay

City=Delhi

Price=5000
```

User updates

```
Price=6500
```

CDC Event

```
{
  "before":{

      "id":5,

      "price":5000
  },

  "after":{

      "id":5,

      "price":6500
  },

  "operation":"UPDATE",

  "timestamp":1712355123
}
```

Indexer uses

```
after
```

to update ES.

---

# Why Kafka?

Imagine

```
1000 updates/sec
```

Without Kafka

```
Database

↓

Indexer
```

If Indexer crashes

Events are lost.

---

With Kafka

```
Database

↓

Kafka

↓

Indexer
```

Kafka stores events.

Indexer can restart later.

Nothing is lost.

---

# Example

10:00

```
100,000 updates
```

Indexer crashes.

Kafka still contains

```
Offset 1

Offset 2

Offset 3

...

Offset 100000
```

Indexer comes back.

Starts reading from

```
Offset 45210
```

Continues processing.

No data loss.

---

# Why not update Elasticsearch inside the Event Service?

For example

```
Create Event

↓

Save PostgreSQL

↓

Update Elasticsearch
```

Looks simple.

But imagine

```
Postgres

SUCCESS

Elasticsearch

FAILED
```

Now database says

```
Concert exists
```

Search says

```
Not found
```

You now need retries, error handling, and compensation logic inside your business service.

With CDC

```
Application

↓

Only writes PostgreSQL

↓

Transaction commits

↓

CDC guarantees change is emitted

↓

Indexer retries until ES succeeds
```

Application stays simple.

---

# How does CDC know transaction committed?

Suppose

```
BEGIN

UPDATE Seats

ROLLBACK
```

Should ES update?

No.

CDC only emits committed transactions.

Example

```
BEGIN

Seat=4999

COMMIT
```

↓

CDC publishes.

If

```
BEGIN

Seat=4999

ROLLBACK
```

↓

Nothing published.

This keeps Elasticsearch consistent with committed database state.

---

# End-to-End Flow in TicketMaster

```
User books ticket

        │
        ▼

Booking Service

        │

UPDATE Seats

        │
        ▼

PostgreSQL

        │

WAL Entry Generated

        │
        ▼

Debezium CDC

        │

Kafka Topic

        │
        ▼

Search Indexer

        │

Transform Document

        │
        ▼

Elasticsearch

        │
        ▼

User searches

        │
        ▼

Search Service

        │
        ▼

Elasticsearch
```

---

## One important interview question

**Q: What happens if the Search Indexer crashes for 30 minutes?**

**Answer:**

* PostgreSQL continues accepting writes.
* CDC continues reading committed changes and publishing them to Kafka.
* Kafka retains those events (based on its retention policy).
* When the Search Indexer restarts, it resumes consuming from its last committed offset and replays all missed events.
* Elasticsearch eventually catches up, giving **eventual consistency** without losing updates.

This decoupling between the database, event stream, and indexer is one of the biggest advantages of using CDC with Kafka in large-scale systems like Ticketmaster.
