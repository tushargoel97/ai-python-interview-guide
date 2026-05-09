## Table of Contents

1. [1. Event-Driven Architecture (EDA)](#1-event-driven-architecture-eda)
2. [2. CAP Theorem & Consistency Models](#2-cap-theorem--consistency-models)
3. [3. Caching Strategies](#3-caching-strategies)
4. [4. Rate Limiting & API Security](#4-rate-limiting--api-security)
5. [5. Observability — Logging, Metrics, Tracing](#5-observability--logging-metrics-tracing)
6. [6. System Design](#6-system-design)
7. [7. Architecture](#7-architecture)

### 1. Event-Driven Architecture (EDA)

> **📣 Definition:** _"Event-Driven Architecture is a pattern where services communicate by emitting and reacting to events through a message broker (Kafka, RabbitMQ, Redis Streams) instead of calling each other directly. Producers don't know who's listening; consumers subscribe to event types they care about. This decouples services, enables eventual consistency, and supports replay for debugging — but adds complexity around ordering, deduplication, and dead-letter queues."_

**Layman**: Instead of calling your friend to ask for updates every hour (polling), your friend just texts you whenever something happens (events).

**Technical**: Producers emit events to a message broker (Kafka, RabbitMQ, Redis Streams). Consumers subscribe and react asynchronously. Decouples services, enables eventual consistency, supports replay.

**Key patterns within EDA**: Pub/Sub, Event Sourcing, CQRS, Outbox Pattern, Dead Letter Queue, Competing Consumers.

**📌 TLDR**: "EDA decouples services through asynchronous events via a message broker. Combined with CQRS and Event Sourcing, it enables scalable, auditable microservices. Kafka for high-throughput streaming, RabbitMQ for flexible routing."

---

### 2. CAP Theorem & Consistency Models

> **📣 Definition:** _"The CAP theorem says a distributed database can guarantee at most two of three properties: Consistency (everyone reads the same data at the same time), Availability (every request gets a response), and Partition tolerance (the system keeps working even if the network splits). Since network partitions are inevitable in real distributed systems, the real choice is between CP (sacrifice availability during partitions) and AP (sacrifice strict consistency for eventual consistency). MongoDB, HBase, Zookeeper lean CP; Cassandra, DynamoDB lean AP."_

**Layman**: A distributed database can only guarantee 2 out of 3: **C**onsistency (everyone reads the same data), **A**vailability (always responds), **P**artition tolerance (works even if network splits). Since partitions are inevitable, you always choose between CP or AP.

**Technical**:

- **CP systems**: MongoDB (strong consistency mode), HBase, Zookeeper — sacrifice availability during partitions.
- **AP systems**: Cassandra, DynamoDB, CouchDB — sacrifice consistency (eventual consistency).
- **Real-world**: It's a spectrum, not a binary choice. Tunable consistency (Cassandra: `QUORUM` reads/writes).

**📌 TLDR**: "CAP: pick 2 of 3 (Consistency, Availability, Partition tolerance). Since partitions are inevitable, it's really CP vs AP. Most microservices choose AP with eventual consistency. Know tunable consistency and when to choose strong vs eventual."

---

### 3. Caching Strategies

> **📣 Definition:** _"Caching stores frequently accessed data in fast storage (memory, Redis) to avoid recomputing or re-querying. Common patterns: **cache-aside** (app checks cache → miss → query DB → write to cache; the most common), **write-through** (write to cache + DB simultaneously), **write-behind** (cache first, async DB write — fast but risky). The hard problem isn't caching — it's invalidation. Use TTL-based expiry combined with event-based invalidation. Cache at multiple layers: CDN, application, and database query cache."_

**Layman**: A restaurant keeps the most popular dishes pre-made in a warming drawer (cache) instead of cooking from scratch every time.

**Technical patterns**:

- **Cache-aside (Lazy)**: App checks cache → miss → query DB → write to cache.
- **Write-through**: Write to cache AND DB simultaneously.
- **Write-behind**: Write to cache first, async write to DB later (risky but fast).
- **Cache invalidation**: TTL-based, event-based, versioned keys.
- **Tools**: Redis, Memcached. Redis supports data structures (sorted sets, hashes, streams).

**📌 TLDR**: "Cache-aside is the most common pattern. Use Redis for structured caching with TTL. Invalidation is the hard part — prefer TTL + event-based invalidation. Cache at multiple layers: application, CDN, database query cache."

---

### 4. Rate Limiting & API Security

> **📣 Definition:** _"Rate limiting protects APIs from abuse, runaway clients, and accidental DoS. The most common algorithm is **token bucket** — a bucket holds N tokens that refill at R tokens/second; each request costs a token, and an empty bucket returns 429 Too Many Requests. **Sliding window** counters are more precise but heavier. For distributed systems, implement with Redis + Lua scripts for atomic check-and-decrement. Combine rate limiting with OAuth2/JWT for auth, CORS for browser security, and input validation to prevent SQL injection and XSS."_

**Technical**:

- **Token Bucket**: Bucket holds N tokens, refills at R/sec. Each request costs 1 token. Empty bucket = 429.
- **Sliding Window**: Count requests in a sliding time window (more precise than fixed window).
- **Distributed rate limiting**: Redis + Lua scripts for atomic check-and-decrement.
- **API Security**: OAuth2 + JWT, API keys, CORS, input validation, SQL injection prevention, HTTPS everywhere.

**📌 TLDR**: "Token bucket is the go-to algorithm for rate limiting. Implement with Redis for distributed systems. Combine with JWT/OAuth2 for auth, CORS for browser security, and input validation for injection prevention."

---

### 5. Observability — Logging, Metrics, Tracing

> **📣 Definition:** _"Observability is the ability to understand a running system from the outside, built on three pillars. **Logs** = structured records of discrete events (use JSON, not free-form). **Metrics** = aggregated numerical data over time (counters, gauges, histograms — Prometheus). **Distributed traces** = following a single request across multiple services with a shared trace ID (OpenTelemetry → Jaeger). All three should share correlation IDs so you can pivot between them. You can't debug production microservices without all three."_

**The three pillars**:

1. **Logs**: Structured JSON logs (not print statements). ELK Stack / Loki.
2. **Metrics**: Counters, gauges, histograms. Prometheus + Grafana.
3. **Distributed Tracing**: Follow a request across services. OpenTelemetry → Jaeger/Zipkin. Trace ID propagation via headers.

**📌 TLDR**: "Observability = Logs + Metrics + Traces. Use structured logging, Prometheus for metrics, OpenTelemetry for distributed tracing. Correlate all three with a shared trace ID. This is essential for debugging microservices."

---



### Scaling REST APIs for Traffic Surges

> **📣 Definition:** _"Scaling APIs for traffic surges is a multi-layered strategy: rate limiting protects the backend, caching reduces load at every tier, async offloading moves slow work off the request path, horizontal scaling adds capacity, circuit breakers prevent cascade failures, and queue-based load leveling absorbs burst traffic. The tech-lead answer isn't 'add more servers' — it's knowing which layer to optimize first based on the bottleneck."_

**The 7 Layers of API Scaling:**

```
┌───────────────────────────────────────────────────────────┐
│  Layer 1: CDN / Edge Cache (Cloudflare, CloudFront)      │  ← static + cacheable responses
│  Layer 2: Load Balancer (Nginx, ALB, HAProxy)            │  ← distribute across instances
│  Layer 3: Rate Limiting / Throttling                      │  ← protect backend from abuse
│  Layer 4: Application Cache (Redis)                       │  ← avoid recomputation
│  Layer 5: Async Offloading (Celery, queues)              │  ← move slow work off hot path
│  Layer 6: Database Optimization                           │  ← read replicas, connection pooling
│  Layer 7: Queue-Based Load Leveling (Kafka, RabbitMQ)    │  ← absorb burst, process at own pace
└───────────────────────────────────────────────────────────┘
```

**1. Rate Limiting & Throttling (first line of defense):**

```python
# Token Bucket Algorithm (most common — allows bursts)
# - Bucket has capacity (e.g., 100 tokens)
# - Tokens refill at fixed rate (e.g., 10/sec)
# - Each request consumes 1 token
# - If bucket empty → 429 Too Many Requests

# Sliding Window (more precise, higher memory)
# - Track timestamps of requests in a rolling window
# - Count requests in last N seconds

# Implementation: Redis + Lua script (atomic)
import redis, time

r = redis.Redis()

def rate_limit(user_id: str, max_requests: int = 100, window_sec: int = 60) -> bool:
    key = f"ratelimit:{user_id}"
    pipe = r.pipeline()
    now = time.time()
    pipe.zremrangebyscore(key, 0, now - window_sec)   # remove old entries
    pipe.zadd(key, {str(now): now})                    # add current request
    pipe.zcard(key)                                     # count requests in window
    pipe.expire(key, window_sec)
    _, _, count, _ = pipe.execute()
    return count <= max_requests

# Response headers (always include):
# X-RateLimit-Limit: 100
# X-RateLimit-Remaining: 42
# X-RateLimit-Reset: 1620000000   (Unix timestamp)
# Retry-After: 30                 (seconds, on 429)
```

**2. Caching Strategy (biggest bang for the buck):**

```python
# === Cache Hierarchy ===
# L1: CDN / Edge (seconds-minutes)     → GET /products (public, cacheable)
# L2: Reverse Proxy (Nginx/Varnish)    → full response caching
# L3: Application Cache (Redis)         → computed results, DB query results
# L4: Database Query Cache              → Postgres query plan cache

# Cache-Control headers for HTTP caching:
@app.get("/products/{id}")
async def get_product(id: int):
    product = await cache_or_fetch(id)
    return JSONResponse(
        content=product,
        headers={
            "Cache-Control": "public, max-age=300",       # CDN/browser: 5 min
            "ETag": hashlib.md5(json.dumps(product).encode()).hexdigest(),
        },
    )

# Cache invalidation patterns:
# 1. TTL-based (simplest)         → cache expires after N seconds
# 2. Event-driven (best)          → invalidate on write (publish to Redis pub/sub)
# 3. Cache-aside (read-through)   → check cache → miss → fetch DB → populate cache
# 4. Write-through                → write to cache AND DB simultaneously
```

**3. Async Offloading (keep the hot path fast):**

```python
# Rule: if it takes > 200ms, don't do it in the request
# Move to background: emails, image processing, PDF generation, webhooks, analytics

# Synchronous (blocks the response):
@app.post("/orders")
async def create_order(order: OrderCreate):
    result = await save_order(order)
    await send_confirmation_email(order)     # ← 500ms! User waits
    await update_inventory(order)            # ← 300ms! User waits more
    return result                            # total: 800ms+

# Async offloaded (fast response):
@app.post("/orders", status_code=202)        # 202 Accepted
async def create_order(order: OrderCreate):
    result = await save_order(order)
    celery_send_email.delay(order.id)        # ← queued, returns immediately
    celery_update_inventory.delay(order.id)  # ← queued, returns immediately
    return {"order_id": result.id, "status": "processing"}   # total: 50ms
```

**4. Horizontal Scaling & Load Balancing:**

```
# Stateless API instances behind a load balancer:
# - Each instance is identical and disposable
# - No local state (sessions in Redis, files in S3)
# - Load balancer distributes: round-robin, least-connections, or IP-hash

# Auto-scaling rules:
# Scale UP:   CPU > 70% for 3 min, or request queue depth > 100
# Scale DOWN: CPU < 30% for 10 min
# Min instances: 2 (always, for HA)
# Max instances: based on budget and DB connection limits

# Connection math:
# 10 instances × 4 Gunicorn workers × 20 DB pool = 800 connections
# PostgreSQL default max_connections = 100  ← PROBLEM
# Solution: PgBouncer in front of PostgreSQL (connection multiplexer)
```

**5. Circuit Breaker (prevent cascade failures):**

```python
# When a downstream service is failing, stop calling it
# instead of letting timeouts pile up and crash your service

# States: CLOSED (normal) → OPEN (failing, reject calls) → HALF-OPEN (test recovery)

import circuitbreaker  # pip install circuitbreaker

@circuitbreaker.circuit(failure_threshold=5, recovery_timeout=30)
async def call_payment_service(order_id: str):
    """After 5 failures, circuit opens for 30s. Returns fallback during outage."""
    response = await httpx.post(PAYMENT_URL, json={"order_id": order_id}, timeout=5.0)
    response.raise_for_status()
    return response.json()

# In production: use with retry + exponential backoff
# Libraries: circuitbreaker, tenacity, pybreaker
```

**6. Queue-Based Load Leveling (absorb burst traffic):**

```
# Problem: flash sale → 100K requests/sec → DB can handle 5K/sec → crash
# Solution: accept requests into a queue, process at sustainable rate

Client → API (fast accept, 202) → Kafka/RabbitMQ → Workers (process at DB-safe rate)
                                                         ↓
                                                    Database (5K/sec max)

# The API becomes a thin "intake" layer:
# 1. Validate request
# 2. Publish to queue
# 3. Return 202 Accepted + job_id
# 4. Client polls GET /jobs/{id} for status

# This is how every high-traffic system works:
# - Uber: ride requests queued → matched asynchronously
# - Stripe: payment intents → processed via state machine
# - Any flash sale: orders queued → inventory checked async
```

**Traffic Surge Decision Matrix:**

| Bottleneck              | Symptom                         | Fix                                          |
| ----------------------- | ------------------------------- | -------------------------------------------- |
| **Too many requests**   | 429s, high CPU                  | Rate limiting, auto-scaling, CDN             |
| **Slow responses**      | High latency, timeouts          | Caching (Redis), async offloading            |
| **DB overload**         | Connection errors, slow queries | Read replicas, PgBouncer, query optimization |
| **Downstream failures** | Cascade timeouts                | Circuit breakers, retry with backoff         |
| **Burst traffic**       | Queue overflow, OOM             | Queue-based load leveling (Kafka/RabbitMQ)   |
| **Static content**      | Bandwidth saturation            | CDN (CloudFront, Cloudflare)                 |
| **Memory pressure**     | OOM kills, swapping             | Pagination, streaming, `iterator()`          |

**The interview answer framework:**

```
1. "First, I'd identify the bottleneck" → monitoring (Prometheus, OpenTelemetry)
2. "Protect the backend"               → rate limiting, circuit breakers
3. "Reduce the load"                   → caching at every tier, CDN
4. "Move slow work off the hot path"   → async (Celery, queues)
5. "Scale horizontally"                → stateless instances, auto-scaling
6. "Absorb the burst"                  → queue-based load leveling
7. "Prevent cascade failure"           → circuit breakers, bulkheads, timeouts
```

### 📌 TLDR for Interviews

> "Idempotency means repeating a request produces the same result. GET, PUT, DELETE are idempotent by spec. For POST, use an `Idempotency-Key` header + server-side caching (Redis). REST design: resources as nouns, HTTP verbs for actions, proper status codes, pagination, versioning, and consistent error responses. **Scaling for surges**: rate-limit first (token bucket), cache aggressively (CDN → Redis → DB), async-offload anything >200ms, scale horizontally behind a load balancer, use circuit breakers for downstream failures, and queue-based load leveling for burst absorption. The right answer is never 'add more servers' — it's 'find the bottleneck first, then apply the right layer'."

---


Most Likely Questions

These are the exact phrasings interviewers use. Practice answering each in 60-90 seconds.

### 6. System Design

**Q: "Design a URL shortener like bit.ly."**
A: Functional: shorten URL → unique short code → redirect. (1) **API**: POST /shorten, GET /:code. (2) **Encoding**: base62 (a-zA-Z0-9) of an auto-increment ID gives 6 chars for ~56 billion URLs. (3) **Storage**: PostgreSQL for durability + Redis cache for hot reads (99% of traffic is reads). (4) **Scale**: read-heavy → CDN + caching, eventual consistency OK. (5) **Analytics**: async write to Kafka → ClickHouse for click counts. (6) **Failure modes**: cache miss → DB; rate-limit creation to prevent abuse.

**Q: "Design a payment system with idempotency."**
A: Client generates a UUID `Idempotency-Key` per logical transaction. Server checks Redis (with 24h TTL) — if key exists, return cached response. Otherwise, start a DB transaction: insert idempotency record + process payment + commit atomically. On retry, the unique constraint on the key catches duplicates. Combine with Outbox Pattern: write the payment event to an outbox table in the same transaction, separate worker publishes to Kafka.

**Q: "How do you handle a service that needs to call 5 downstream services?"**
A: (1) Parallelize independent calls with `asyncio.gather()`. (2) Wrap each in a circuit breaker — fail fast if a service is down. (3) Add timeouts (never call without one). (4) Retry with exponential backoff + jitter for transient failures. (5) Idempotency keys for non-idempotent operations. (6) Distributed tracing (OpenTelemetry) so you can debug across services. (7) Bulkhead pattern — separate connection pools per downstream so one slow service doesn't exhaust all connections.

### 7. Architecture

**Q: "Monolith vs microservices — when do you choose what?"**
A: Start with a monolith. Microservices when you have: independent scaling needs (one service has 100x traffic), independent deployment cadence (team autonomy), or different tech requirements (ML service in Python, video service in Go). Costs: distributed tracing, network failures, eventual consistency, ops overhead. Premature microservices is a top architecture mistake — the operational burden kills small teams.

**Q: "Walk me through how you'd add a new feature to a large production system."**
A: (1) Read the existing code and find the natural seam. (2) Write the design doc — API contract, data model changes, failure modes, rollback plan. (3) Feature flag from day one. (4) Add observability before logic — metrics, structured logs, traces. (5) Migration strategy if schema changes — backward-compatible changes first (add nullable column, dual-write, backfill, cut over). (6) Roll out to 1% → 10% → 100% with metrics watch. (7) Document the runbook.
