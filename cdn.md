# 🚀 CDN (Content Delivery Network) -- System Design Notes

## 📌 What is a CDN?

A CDN (Content Delivery Network) is a globally distributed network of
servers (edge locations) that cache and deliver content closer to users.

👉 Acts as a distributed caching layer in front of your backend.

------------------------------------------------------------------------

## 🌍 How CDN Works

1.  User requests resource (e.g., /image.png)
2.  DNS routes request to nearest CDN edge
3.  CDN checks cache:
    -   HIT → returns response
    -   MISS → fetches from origin server
4.  Response is cached for future requests

------------------------------------------------------------------------

## 🧠 CDN in Architecture

User → CDN → Load Balancer → App Servers → Database

------------------------------------------------------------------------

## 🔥 Why CDN is Important

### 🚀 Latency Reduction

-   Serves content from nearest location

### 📉 Backend Load Reduction

-   Offloads static traffic

### 🌍 Global Scalability

-   No need for multi-region infra initially

### 🛡️ Security

-   DDoS protection, WAF, rate limiting

------------------------------------------------------------------------

## 🧩 Types of Content

### Cacheable

-   Images, CSS, JS, Fonts

### Semi-Dynamic

-   API responses (with cache headers)

### Not Ideal

-   Personalized / real-time data

------------------------------------------------------------------------

## ⚙️ Caching Strategies

### Cache-Control

Cache-Control: public, max-age=3600

### TTL

-   High → faster, stale
-   Low → fresh, more origin hits

### Invalidation

-   Manual purge
-   Versioning (file?v=2)

### Cache Keys

-   URL, headers, query params

------------------------------------------------------------------------

## 🏗️ CDN Patterns

### Pull CDN

-   Fetch from origin when needed

### Push CDN

-   Upload content directly

### Multi-CDN

-   Use multiple providers for reliability

------------------------------------------------------------------------

## ⚡ Real Use Cases

### SaaS Apps

-   UI assets
-   Reports

### Streaming Apps

-   Images, videos

------------------------------------------------------------------------

## ⚠️ Cons of CDN

### ❗ Cache Invalidation

-   Hard to manage stale data

### ❗ Cost

-   Data transfer + requests

### ❗ Debugging Complexity

-   Multiple caching layers

### ❗ Not for Dynamic Data

-   Personalized responses

### ❗ Cold Cache

-   First request slow

### ❗ Consistency Issues

-   Different regions, different cache

------------------------------------------------------------------------

## 🔐 Advanced Concepts

### Edge Computing

-   Run logic at CDN edge

### Signed URLs

-   Secure access

### Origin Shielding

-   Extra cache layer before origin

### Geo Routing

-   Location-based delivery

------------------------------------------------------------------------

## 🧠 When to Use CDN

✅ Global users\
✅ High traffic\
✅ Static-heavy apps

------------------------------------------------------------------------

## 🚫 When Not to Use

❌ Internal tools\
❌ Fully dynamic apps\
❌ Low traffic

------------------------------------------------------------------------

## 🏁 Final Thought

CDN = Global Cache + Performance Booster + Security Layer
