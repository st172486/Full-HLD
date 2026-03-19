# 📘 API & REST API - Detailed Notes

## 🔹 1. What is an API?

API (Application Programming Interface) is a mechanism that allows two software systems to communicate with each other.

### 📌 Real-world analogy
- Client (User / Frontend) → Request
- API → Mediator
- Server → Processes request and sends response

---

## 🔹 2. REST API (Representational State Transfer)

REST is an architectural style for designing APIs.

### 🔑 Principles:
- Resource-based (nouns, not verbs)
- Stateless
- Use HTTP methods properly
- Standard status codes

---

## 🔹 3. HTTP Methods

| Method | Purpose |
|--------|--------|
| GET | Retrieve data |
| POST | Create data |
| PUT | Update full resource |
| PATCH | Update partial resource |
| DELETE | Remove resource |

---

## 🔹 4. Example: Candidate API

### Get all candidates
GET /candidates

### Get candidate by ID
GET /candidates/{id}

### Create candidate
POST /candidates

### Update candidate
PATCH /candidates/{id}

### Delete candidate
DELETE /candidates/{id}

---

## 🔹 5. Good vs Bad API Design

### ❌ Bad
GET /getAllCandidates  
POST /createCandidate  

### ✅ Good
GET /candidates  
POST /candidates  

---

## 🔹 6. Advanced Concepts

### 📌 Filtering, Sorting, Pagination
GET /candidates?skill=React&page=1&limit=10

### 📌 Versioning
/api/v1/candidates

### 📌 Nested Resources
GET /jobs/{id}/candidates

---

## 🔹 7. Authentication

Use JWT or OAuth

Authorization: Bearer <token>

---

## 🔹 8. Error Handling

```json
{
  "error": {
    "code": "INVALID_INPUT",
    "message": "Email is required"
  }
}
```

---

## 🔹 9. Best Practices

- Use nouns in endpoints
- Keep APIs stateless
- Use proper HTTP methods
- Implement pagination
- Add versioning
- Handle errors properly

---

## 🔹 10. System Design Perspective

- APIs are contracts between frontend and backend
- Should be scalable and consistent
- Design for future extensibility

---

## 🔹 11. Common Mistakes

- Verb-based URLs
- No pagination
- Poor error handling
- No versioning

---

## 🔹 12. Summary

Frontend → API → Backend → Database

API = Contract  
REST = Rules for designing API
