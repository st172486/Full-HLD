# Client--Server Architecture (HLD Deep Dive)

------------------------------------------------------------------------

## 1. Problem Statement

Design a scalable client-server architecture where multiple clients
interact with centralized backend services.

### Functional Requirements

-   Clients can send HTTP requests (GET, POST, etc.)
-   Server processes requests and returns responses
-   Support web and mobile clients

### Non-Functional Requirements

-   Scalability: Handle millions of users
-   Availability: 99.9% uptime
-   Latency: \<100ms
-   Security: HTTPS, authentication

------------------------------------------------------------------------

## 2. High-Level Overview

Client communicates with server via request-response model.

### High-Level Architecture Diagram

``` mermaid
graph TD
Client --> CDN
CDN --> LB
LB --> API
API --> Cache
API --> DB
```

------------------------------------------------------------------------

## 3. Core Components

-   Client (Browser/Mobile)
-   CDN (Cloudflare)
-   Load Balancer (NGINX)
-   API Server (.NET/Node.js)
-   Cache (Redis)
-   Database (Postgres/MongoDB)

------------------------------------------------------------------------

## 4. Detailed Architecture

``` mermaid
sequenceDiagram
Client->>LB: Request
LB->>API: Forward
API->>Cache: Check
Cache-->>API: Hit/Miss
API->>DB: Fetch if needed
DB-->>API: Data
API-->>Client: Response
```

------------------------------------------------------------------------

## 5. Data Flow

``` mermaid
graph TD
A[Client] --> B[DNS]
B --> C[CDN]
C --> D[LB]
D --> E[API]
E --> F{Cache}
F -->|Hit| G[Response]
F -->|Miss| H[DB]
H --> E
E --> G
```

------------------------------------------------------------------------

## 6. Database Design

Users Table: - id (PK) - email (indexed) - name - created_at

------------------------------------------------------------------------

## 7. Scalability Design

``` mermaid
graph TD
Client --> LB
LB --> S1
LB --> S2
LB --> S3
```

-   Horizontal scaling
-   Redis caching
-   CDN usage

------------------------------------------------------------------------

## 8. Trade-offs

Pros: - Scalable - Maintainable

Cons: - Complexity - Network latency

------------------------------------------------------------------------

## 9. Failure Handling

-   Retry with backoff
-   Circuit breaker
-   Rate limiting

------------------------------------------------------------------------

## 10. Real-world Example

-   Netflix: CDN + microservices
-   Uber: Real-time systems with Kafka

------------------------------------------------------------------------

## 11. Interview Discussion Points

-   How to scale?
-   Stateless vs Stateful
-   Cache invalidation

------------------------------------------------------------------------

## 12. Quick Revision

Client → LB → API → Cache/DB → Response

------------------------------------------------------------------------
