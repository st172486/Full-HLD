# GraphQL - Detailed Notes

## 🚀 What is GraphQL?
GraphQL is a query language for APIs and a runtime to execute those queries.

- Client decides what data it needs
- Server returns exactly that data

---

## ❗ Problems with REST

### 1. Over-fetching
Server returns more data than needed.

### 2. Under-fetching
Multiple API calls required.

### 3. Tight coupling
Frontend depends on backend response structure.

---

## 🧠 Core Concepts

### Schema
Defines structure of API.

### Query
Used to fetch data.

### Mutation
Used to modify data.

### Resolver
Functions that fetch actual data.

---

## 🎯 Example: Recruiter Dashboard

### UI Query
```graphql
query GetDashboard {
  recruiter {
    name
  }
  jobs {
    id
    title
    candidateCount
  }
  notifications {
    message
  }
}
```

---

## ⚙️ Backend Flow

### Step 1: Request hits GraphQL server

### Step 2: Schema validation

### Step 3: Resolver execution

```js
Query: {
  recruiter: () => recruiterService.getRecruiter(),
  jobs: () => jobService.getJobs(),
  notifications: () => notificationService.getNotifications()
}
```

### Step 4: Nested resolvers

```js
Job: {
  candidateCount: (job) => {
    return candidateService.countByJobId(job.id);
  }
}
```

---

## 📦 Response

```json
{
  "data": {
    "recruiter": { "name": "Suraj" },
    "jobs": [
      { "id": "1", "title": "Frontend Developer", "candidateCount": 25 }
    ],
    "notifications": [
      { "message": "New candidate applied" }
    ]
  }
}
```

---

## 🔁 Full Flow

```
React UI → GraphQL Query → Server → Resolvers → DB/Services → Response → UI
```

---

## ⚠️ N+1 Problem

If multiple jobs:
- 1 query for jobs
- N queries for candidate count

### Solution: DataLoader
Batch queries into single request.

---

## 🎯 When to Use GraphQL

- Complex UI dashboards
- Multiple data sources
- Mobile + Web clients
- Reducing API calls

---

## ❌ When NOT to Use

- Simple CRUD APIs
- Heavy caching requirements
- File uploads/streaming

---

## 🆚 GraphQL vs REST

| Feature | REST | GraphQL |
|--------|------|--------|
| Data fetching | Fixed | Flexible |
| Requests | Multiple | Single |
| Over-fetching | Yes | No |
| Complexity | Low | Medium |

---

## 🧠 Key Takeaway

- REST = Server decides data
- GraphQL = Client decides data

---

## 🔥 Real-world Usage

- Facebook
- GitHub
- Netflix
- Shopify

---

## 🚀 Final Summary

GraphQL allows frontend to:
- Ask exactly what it needs
- Reduce API calls
- Improve performance

Backend:
- Resolves data field-by-field
- Aggregates multiple services
