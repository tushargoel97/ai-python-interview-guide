# Distributed Processing & Schedulers

## 📑 Table of Contents

**Message brokers & search**

1. [1. Message Brokers & Search Engines — Kafka, RabbitMQ, Elasticsearch](#1-message-brokers--search-engines--kafka-rabbitmq-elasticsearch)
2. [2. Apache Kafka — Distributed Event Streaming](#2-apache-kafka--distributed-event-streaming)
3. [3. RabbitMQ — Traditional Message Broker](#3-rabbitmq--traditional-message-broker)
4. [4. Kafka vs RabbitMQ — The Decision Framework](#4-kafka-vs-rabbitmq--the-decision-framework)
5. [5. Elasticsearch — Distributed Search & Analytics](#5-elasticsearch--distributed-search--analytics)
6. [6. Tech Lead Interview Q&A — Most Asked Questions](#6-tech-lead-interview-qa--most-asked-questions)
7. [7. 📌 The Master TLDR for Message Brokers & Search](#7--the-master-tldr-for-message-brokers--search)

**Distributed task processing**

8. [8. Celery Deep Dive & Distributed Task Processing](#8-celery-deep-dive--distributed-task-processing)
9. [9. Celery Architecture & How It Works](#9-celery-architecture--how-it-works)
10. [10. Setting Up Celery — Step by Step](#10-setting-up-celery--step-by-step)
11. [11. Creating & Calling Tasks — Every Way](#11-creating--calling-tasks--every-way)
12. [12. Task Workflows — Chaining, Groups, Chords](#12-task-workflows--chaining-groups-chords)
13. [13. Celery Beat — Periodic/Scheduled Tasks](#13-celery-beat--periodicscheduled-tasks)
14. [14. Task Routing & Queue Management](#14-task-routing--queue-management)
15. [15. Production Patterns — Idempotency, Safety & Monitoring](#15-production-patterns--idempotency-safety--monitoring)
16. [16. Celery Configuration Cheat Sheet](#16-celery-configuration-cheat-sheet)
17. [17. Distributed Task Processing — The Complete Architecture](#17-distributed-task-processing--the-complete-architecture)
18. [18. Celery Interview FAQ](#18-celery-interview-faq)

## 1. Message Brokers & Search Engines — Kafka, RabbitMQ, Elasticsearch

> **📣 Interview-ready definition:** _"Message brokers decouple services by enabling asynchronous, reliable communication. Kafka is a distributed event-streaming platform (append-only log, pull-based, massive throughput); RabbitMQ is a traditional message broker (push-based, smart routing, lower latency). Elasticsearch is a distributed search and analytics engine built on Apache Lucene's inverted index. All three are infrastructure cornerstones that tech leads must understand at an architectural level."_

---

### 2. Apache Kafka — Distributed Event Streaming

> **📣 Definition:** _"Kafka is an append-only, distributed commit log. Producers write messages to topic partitions; consumers pull messages at their own pace using offsets. It's designed for massive throughput (millions of msgs/sec), replay-ability (consumers can rewind), and durability (replicated across brokers). The key mental model: Kafka is a 'dumb broker, smart consumer' — the broker just stores the log, consumers track their own position."_

**Architecture:**

```
Producer → Topic ──┬── Partition 0 ── [msg0, msg1, msg2, msg3, ...]  → Consumer Group A
                   ├── Partition 1 ── [msg0, msg1, msg2, ...]        → Consumer Group A
                   └── Partition 2 ── [msg0, msg1, ...]              → Consumer Group B

Each partition:  append-only log  │  ordered within partition  │  replicated across brokers
```

**Core Concepts:**

| Concept                | What It Means                                   | Why It Matters                                             |
| ---------------------- | ----------------------------------------------- | ---------------------------------------------------------- |
| **Topic**              | Logical channel for messages (like a table)     | Organize events by domain (`orders`, `payments`)           |
| **Partition**          | Ordered, immutable log shard of a topic         | Unit of parallelism — more partitions = more throughput    |
| **Offset**             | Sequential ID of a message within a partition   | Consumer's "bookmark" — can rewind/replay                  |
| **Consumer Group**     | Set of consumers sharing work on a topic        | Each partition assigned to exactly 1 consumer in the group |
| **Replication Factor** | How many broker copies of each partition        | RF=3 → survives 2 broker failures                          |
| **ISR**                | In-Sync Replicas — replicas caught up to leader | `min.insync.replicas=2` + `acks=all` = no data loss        |

```python
# === Producer (Python — confluent-kafka) ===
from confluent_kafka import Producer

producer = Producer({
    "bootstrap.servers": "kafka:9092",
    "acks": "all",                    # wait for ALL ISR replicas
    "enable.idempotence": True,       # prevent duplicates (PID + sequence number)
    "retries": 5,
})

def delivery_report(err, msg):
    if err:
        print(f"Delivery failed: {err}")
    else:
        print(f"Delivered to {msg.topic()}[{msg.partition()}] @ offset {msg.offset()}")

producer.produce(
    topic="orders",
    key="order-123",                  # key determines partition (hash(key) % num_partitions)
    value='{"item": "laptop", "qty": 1}',
    callback=delivery_report,
)
producer.flush()


# === Consumer ===
from confluent_kafka import Consumer

consumer = Consumer({
    "bootstrap.servers": "kafka:9092",
    "group.id": "order-processor",
    "auto.offset.reset": "earliest",  # start from beginning if no committed offset
    "enable.auto.commit": False,      # manual commit for at-least-once
})
consumer.subscribe(["orders"])

while True:
    msg = consumer.poll(1.0)
    if msg is None:
        continue
    if msg.error():
        continue

    process_order(msg.value())
    consumer.commit(msg)              # commit AFTER processing → at-least-once
```

**Key Interview Topics:**

**Exactly-Once Semantics (EOS):**

```
Idempotent Producer    → prevents duplicate writes (PID + sequence number per partition)
Transactional Producer → atomic writes across multiple partitions/topics
EOS Consumer           → read_committed isolation + atomic offset commit
Full EOS               → idempotent producer + transactions + consumer read_committed
Trade-off              → ~20% throughput reduction, higher latency
```

**Consumer Group Rebalancing:**

```
Trigger:  consumer joins/leaves/crashes, new partitions added
Problem:  during rebalance, ALL consumers stop processing (stop-the-world)
Fix:      CooperativeStickyAssignor (incremental rebalance — only affected partitions move)
Tuning:   session.timeout.ms (heartbeat death detection)
          max.poll.interval.ms (processing time before considered dead)
          heartbeat.interval.ms (how often to send heartbeats)
```

**ZooKeeper → KRaft (2025+):**

```
Old: ZooKeeper managed broker metadata, controller election, topic configs
New: KRaft (Kafka Raft) — metadata managed by Kafka itself
Why: fewer moving parts, faster controller failover, simpler ops
```

📌 **TLDR:** "Kafka = append-only distributed log. Producers write to partitions (key → partition hash). Consumers pull at their own pace via offsets. Consumer groups share work (1 partition = 1 consumer). `acks=all` + `min.insync.replicas=2` = no data loss. Idempotent producer + transactions = exactly-once. KRaft replaces ZooKeeper."

---

### 3. RabbitMQ — Traditional Message Broker

> **📣 Definition:** _"RabbitMQ is a push-based message broker that routes messages through exchanges to queues. It's a 'smart broker, dumb consumer' — the broker handles all routing logic via exchange types (direct, fanout, topic, headers). Messages are deleted after acknowledgment. Best for: task queues, complex routing, request-reply patterns, and lower-latency scenarios where you don't need Kafka's replay or massive throughput."_

**Architecture:**

```
Producer → Exchange ──routing──→ Queue 1 → Consumer A
                    └─routing──→ Queue 2 → Consumer B
                    └─routing──→ Queue 3 → Consumer C

Exchange types determine HOW messages reach queues.
```

**Exchange Types (most-asked RabbitMQ question):**

| Exchange    | Routing Logic                      | Use Case                            | Example                                        |
| ----------- | ---------------------------------- | ----------------------------------- | ---------------------------------------------- |
| **Direct**  | Exact match on routing key         | Task distribution, specific routing | key=`payment.success` → payment queue          |
| **Fanout**  | Broadcast to ALL bound queues      | Notifications, events               | Order created → notify email, SMS, analytics   |
| **Topic**   | Wildcard pattern matching          | Flexible event filtering            | `orders.*.created` matches `orders.us.created` |
| **Headers** | Match on message headers (not key) | Complex attribute-based routing     | Content-type based routing                     |

```python
# === Producer ===
import pika

connection = pika.BlockingConnection(pika.ConnectionParameters("localhost"))
channel = connection.channel()

# Declare exchange and queue
channel.exchange_declare(exchange="orders", exchange_type="topic", durable=True)
channel.queue_declare(queue="payment_processor", durable=True)
channel.queue_bind(exchange="orders", queue="payment_processor", routing_key="order.*.payment")

# Publish with persistence
channel.basic_publish(
    exchange="orders",
    routing_key="order.us.payment",
    body='{"order_id": "123", "amount": 99.99}',
    properties=pika.BasicProperties(delivery_mode=2),  # persistent message
)


# === Consumer with manual ACK ===
def callback(ch, method, properties, body):
    try:
        process_payment(body)
        ch.basic_ack(delivery_tag=method.delivery_tag)        # ACK after success
    except Exception:
        ch.basic_nack(delivery_tag=method.delivery_tag, requeue=False)  # → DLQ

channel.basic_qos(prefetch_count=10)     # don't overwhelm consumer
channel.basic_consume(queue="payment_processor", on_message_callback=callback)
channel.start_consuming()
```

**Dead Letter Queues (DLQ) — built-in in RabbitMQ:**

```
Message fails (rejected / TTL expired / queue full)
    → automatically routed to Dead Letter Exchange (DLX)
    → lands in Dead Letter Queue for inspection / manual retry

Setup: declare queue with arguments:
  x-dead-letter-exchange: "dlx"
  x-dead-letter-routing-key: "failed"
  x-message-ttl: 60000          # optional: auto-expire after 60s
```

📌 **TLDR:** "RabbitMQ = smart broker with exchange-based routing. Direct (exact key), Fanout (broadcast), Topic (wildcards). Messages deleted after ACK. Built-in DLQ via Dead Letter Exchange. Use for: task queues, complex routing, request-reply. Lighter ops than Kafka for simpler use cases."

---

### 4. Kafka vs RabbitMQ — The Decision Framework

| Dimension               | Kafka                                     | RabbitMQ                              |
| ----------------------- | ----------------------------------------- | ------------------------------------- |
| **Model**               | Distributed log (pull)                    | Message queue (push)                  |
| **Broker Intelligence** | Dumb broker, smart consumer               | Smart broker, dumb consumer           |
| **Message Retention**   | Retained (days/weeks/forever)             | Deleted after ACK                     |
| **Replay**              | ✅ Consumers can rewind offsets           | ❌ Once consumed, gone                |
| **Ordering**            | Per-partition guaranteed                  | Per-queue guaranteed                  |
| **Throughput**          | Millions/sec                              | Tens of thousands/sec                 |
| **Latency**             | Higher (batching)                         | Lower (per-message push)              |
| **Routing**             | Simple (topic + key → partition)          | Rich (exchanges, bindings, wildcards) |
| **DLQ**                 | Manual (consumer responsibility)          | Built-in (Dead Letter Exchange)       |
| **Consumer Groups**     | Built-in (partition assignment)           | Competing consumers on a queue        |
| **Use Case**            | Event sourcing, CDC, streaming, analytics | Task queues, notifications, RPC       |
| **Ops Complexity**      | High (brokers, KRaft, partitions)         | Moderate (simpler cluster)            |

**Decision tree:**

```
Need event replay / audit log?           → Kafka
Need multiple consumers on same events?  → Kafka (consumer groups read independently)
Need complex routing (fanout/topic)?     → RabbitMQ
Need request-reply pattern?              → RabbitMQ
Need massive throughput (>100K msg/s)?   → Kafka
Simple background job processing?        → RabbitMQ (or even Celery/Redis)
Event sourcing / CQRS?                   → Kafka
```

---

### 5. Elasticsearch — Distributed Search & Analytics

> **📣 Definition:** _"Elasticsearch is a distributed, RESTful search engine built on Apache Lucene. It stores documents as JSON, indexes them using an inverted index (maps terms → documents, enabling sub-millisecond full-text search), and distributes data across shards for horizontal scaling. Key concepts: mappings (schema), analyzers (text processing pipeline), relevance scoring (BM25), and aggregations (analytics). It's the 'E' in the ELK stack (Elasticsearch + Logstash + Kibana)."_

**How the Inverted Index Works:**

```
Documents:
  doc1: "The quick brown fox"
  doc2: "The quick blue car"
  doc3: "A brown fox jumps"

Inverted Index:
  "the"    → [doc1, doc2]       Forward Index (DB):  doc1 → ["the","quick","brown","fox"]
  "quick"  → [doc1, doc2]       Inverted Index (ES): "fox" → [doc1, doc3]
  "brown"  → [doc1, doc3]
  "fox"    → [doc1, doc3]       Search "brown fox" → intersection of [doc1,doc3] ∩ [doc1,doc3] = [doc1, doc3]
  "blue"   → [doc2]             Result in milliseconds even with billions of docs.
  "car"    → [doc2]
  "jumps"  → [doc3]
```

**Architecture:**

```
Cluster → Nodes → Indices → Shards → Segments (immutable Lucene files)

Index "products" (5 primary shards, 1 replica each):
  Node A: [P0] [P1] [R2]
  Node B: [P2] [R0] [R3]
  Node C: [P3] [P4] [R1] [R4]

Search query → scatter to ALL primary shards → each shard searches locally → gather & merge results
```

**Mappings (Schema) — `text` vs `keyword`:**

```json
{
  "mappings": {
    "properties": {
      "title": { "type": "text" }, // analyzed → full-text search ("quick brown fox")
      "status": { "type": "keyword" }, // NOT analyzed → exact match, sorting, aggregations
      "price": { "type": "float" },
      "created_at": { "type": "date" },
      "description": {
        "type": "text",
        "analyzer": "english", // stemming: "running" → "run"
        "fields": {
          "raw": { "type": "keyword" } // multi-field: search + exact match
        }
      }
    }
  }
}
```

**Analysis Pipeline (how text is processed for indexing):**

```
Input text: "The Quick-Brown FOX!"
    ↓ Character Filter  → "The Quick-Brown FOX!"     (e.g., strip HTML)
    ↓ Tokenizer         → ["The", "Quick-Brown", "FOX!"]   (split into tokens)
    ↓ Token Filters      → ["the", "quick", "brown", "fox"]  (lowercase, split on hyphen, remove punctuation)

Standard Analyzer:  lowercase + standard tokenizer (good default)
English Analyzer:   + stemming ("running" → "run") + stop words removal ("the", "a")
Custom Analyzer:    build your own pipeline for domain-specific needs
```

**Key Query Types:**

```json
// Full-text search (analyzed, scored by relevance)
{ "match": { "title": "quick brown fox" } }

// Exact match (keyword fields, filters)
{ "term": { "status": "published" } }

// Boolean combination
{
  "bool": {
    "must":     [{ "match": { "title": "elasticsearch" } }],
    "filter":   [{ "range": { "price": { "lte": 100 } } }],
    "must_not": [{ "term": { "status": "draft" } }],
    "should":   [{ "match": { "tags": "tutorial" } }]
  }
}

// Aggregations (analytics)
{
  "size": 0,
  "aggs": {
    "avg_price": { "avg": { "field": "price" } },
    "by_status": {
      "terms": { "field": "status" },
      "aggs": { "avg_price_per_status": { "avg": { "field": "price" } } }
    }
  }
}
```

**Relevance Scoring — BM25 (default):**

```
Score = TF × IDF × field-length-norm

TF  (Term Frequency):    how often the term appears in the document (more = higher)
IDF (Inverse Doc Freq):  how rare the term is across all documents (rarer = higher)
Field-length norm:       shorter fields score higher (title match > body match)

Tuning: boost fields, function_score, script_score, rescore
```

**Production Concerns:**

| Topic                | Best Practice                                                                 |
| -------------------- | ----------------------------------------------------------------------------- |
| **Shard sizing**     | 30-50 GB per shard; avoid oversharding (too many small shards → overhead)     |
| **Replicas**         | At least 1 replica for HA; 0 replicas during bulk indexing for speed          |
| **Refresh interval** | Default 1s (near-real-time); increase to 30s during bulk loads                |
| **Mapping**          | Always define explicit mappings; dynamic mapping → mapping explosions         |
| **ILM**              | Hot → Warm → Cold → Frozen tiers for cost-effective retention                 |
| **Cluster health**   | Green (all shards assigned), Yellow (replicas missing), Red (primary missing) |
| **Heap**             | 50% of RAM, max 32GB (compressed OOPs threshold)                              |

📌 **TLDR:** "Elasticsearch = inverted index on Lucene. `text` fields are analyzed (full-text search); `keyword` fields are exact match. BM25 scoring: TF × IDF × field-length. Shards distribute data; replicas provide HA. Use `bool` queries for complex searches, aggregations for analytics. Always set explicit mappings. Monitor cluster health: Green/Yellow/Red."

---

### 6. Tech Lead Interview Q&A — Most Asked Questions

**Kafka:**

> **Q: How do you guarantee no message loss in Kafka?**
> A: `acks=all` (producer waits for all ISR replicas) + `min.insync.replicas=2` (at least 2 replicas must confirm) + `replication.factor=3` + `enable.idempotence=true` (prevents duplicates). On consumer side: manual offset commit AFTER processing (at-least-once) + idempotent processing downstream.

> **Q: How do you handle consumer lag?**
> A: Monitor lag via `kafka-consumer-groups.sh --describe`. Solutions: add more partitions + consumers (scale horizontally), optimize processing time, use `max.poll.records` to control batch size, check for GC pauses or slow downstream dependencies.

> **Q: How many partitions should a topic have?**
> A: `Target throughput ÷ throughput per consumer = min partitions`. Typical: start with `max(num_consumers, throughput_requirement)`. Avoid too many (increases rebalance time, broker memory, controller load). Can only increase, never decrease.

> **Q: Kafka vs a regular database for event sourcing?**
> A: Kafka is the event log (append-only, immutable, ordered). DB is the materialized view (current state). Kafka guarantees ordering per partition and allows replay. DB provides random access and queries. They complement each other — Kafka is the source of truth, DB is the read model.

**RabbitMQ:**

> **Q: How do you handle poison messages (messages that always fail)?**
> A: Configure Dead Letter Exchange (DLX). After N retry attempts (tracked via header `x-death`), route to DLQ. Monitor DLQ, alert on growth, provide manual retry tooling. Never `requeue=True` infinitely.

> **Q: RabbitMQ message ordering guarantees?**
> A: Ordered within a single queue with a single consumer. With competing consumers on the same queue, ordering is NOT guaranteed. If you need ordering: use a single consumer, or partition by key (using consistent-hash exchange plugin).

**Elasticsearch:**

> **Q: How do you handle a Yellow cluster status?**
> A: Yellow = all primaries assigned but some replicas are not. Common causes: not enough nodes for replica allocation, disk watermark exceeded, shard allocation filtering. Fix: add nodes, increase disk, check `_cluster/allocation/explain`.

> **Q: How do you optimize slow search queries?**
> A: 1) Use `filter` context instead of `query` where possible (filters are cached, no scoring). 2) Avoid wildcard queries on `text` fields. 3) Use `keyword` for exact match. 4) Reduce shard count if oversharded. 5) Use `routing` to target specific shards. 6) Profile with `_search?explain=true` and `_search/profile`.

> **Q: When would you NOT use Elasticsearch?**
> A: Primary data store (no ACID transactions), strong consistency requirements, frequent updates to the same document (segment merging overhead), small datasets where PostgreSQL full-text search suffices.

### 7. 📌 The Master TLDR for Message Brokers & Search

> "**Kafka** = distributed commit log for high-throughput event streaming. Use for: event sourcing, CDC, streaming analytics, multi-consumer replay. Key settings: `acks=all`, `min.insync.replicas=2`, idempotent producer. **RabbitMQ** = smart broker with exchange-based routing. Use for: task queues, complex routing, lower-latency messaging, built-in DLQ. **Elasticsearch** = inverted index for sub-millisecond full-text search and analytics. Use for: search, logging (ELK), aggregations. Don't use as primary DB. The tech lead skill: knowing WHEN to use which, not just HOW."

---

## 8. Celery Deep Dive & Distributed Task Processing

> **📣 Interview-ready definition:** _"Celery is a distributed task queue for Python — it lets you run code asynchronously (background jobs) and on a schedule (periodic tasks). Tasks are serialized and sent to a message broker (Redis or RabbitMQ), consumed by worker processes, and results optionally stored in a backend. For distributed task processing at scale, the architecture combines Celery for task execution, Kafka/RabbitMQ for messaging, retry systems with exponential backoff, idempotency for safe re-execution, and dead letter queues for unprocessable messages."_

---

### 9. Celery Architecture & How It Works

```
┌─────────────┐    task.delay()     ┌─────────────┐    consume     ┌──────────────┐
│   Django /   │ ──────────────────→ │   BROKER     │ ────────────→ │   WORKER(s)  │
│   FastAPI    │                     │ (Redis/RMQ)  │               │ (separate    │
│   (Producer) │                     │              │               │  processes)  │
└─────────────┘                     └─────────────┘               └──────┬───────┘
                                                                         │
                                         ┌───────────────────────────────┘
                                         ▼
                                   ┌──────────────┐
                                   │ RESULT       │
                                   │ BACKEND      │
                                   │ (Redis/DB)   │
                                   └──────────────┘

Key insight: Producer, Broker, and Worker are SEPARATE processes.
Your web server never executes the task — it just puts it on the queue.
```

### 10. Setting Up Celery — Step by Step

```python
# === Step 1: celery.py (project root, next to settings.py) ===
import os
from celery import Celery

os.environ.setdefault("DJANGO_SETTINGS_MODULE", "myproject.settings")

app = Celery("myproject")
app.config_from_object("django.conf:settings", namespace="CELERY")
app.autodiscover_tasks()   # auto-finds tasks.py in each app

# === Step 2: __init__.py (project root) ===
from .celery import app as celery_app
__all__ = ("celery_app",)

# === Step 3: settings.py ===
CELERY_BROKER_URL = "redis://localhost:6379/0"          # or "amqp://guest:guest@localhost//"
CELERY_RESULT_BACKEND = "redis://localhost:6379/1"
CELERY_ACCEPT_CONTENT = ["json"]
CELERY_TASK_SERIALIZER = "json"
CELERY_RESULT_SERIALIZER = "json"
CELERY_TIMEZONE = "UTC"
CELERY_TASK_TRACK_STARTED = True                        # track STARTED state
CELERY_TASK_TIME_LIMIT = 300                            # hard kill after 5 min
CELERY_TASK_SOFT_TIME_LIMIT = 240                       # raise SoftTimeLimitExceeded at 4 min
CELERY_WORKER_PREFETCH_MULTIPLIER = 1                   # for fair task distribution
CELERY_TASK_ACKS_LATE = True                            # ACK after completion, not receipt

# === Step 4: Run the worker ===
# celery -A myproject worker -l info -c 4                # 4 concurrent workers
# celery -A myproject beat -l info                       # periodic task scheduler
# celery -A myproject flower                             # web monitoring UI
```

### 11. Creating & Calling Tasks — Every Way

```python
# === tasks.py (in any Django app) ===
from celery import shared_task

# --- Basic task ---
@shared_task
def add(x, y):
    return x + y

# --- Task with retry and binding ---
@shared_task(bind=True, max_retries=3, default_retry_delay=60)
def send_email(self, user_id, subject, body):
    """bind=True gives access to 'self' for retry, request info, etc."""
    try:
        user = User.objects.get(id=user_id)
        email_service.send(user.email, subject, body)
    except ConnectionError as exc:
        raise self.retry(exc=exc, countdown=2 ** self.request.retries)
        # countdown: 1s, 2s, 4s (exponential backoff)

# --- Auto-retry (cleaner syntax) ---
@shared_task(autoretry_for=(ConnectionError, TimeoutError),
             retry_backoff=True,         # exponential backoff
             retry_backoff_max=600,      # max 10 min between retries
             retry_jitter=True,          # add randomness to prevent thundering herd
             max_retries=5)
def fetch_external_api(url):
    return requests.get(url, timeout=10).json()

# --- Task with custom queue routing ---
@shared_task(queue="high_priority")
def process_payment(payment_id):
    ...

# --- Task with rate limiting ---
@shared_task(rate_limit="10/m")          # max 10 executions per minute
def send_sms(phone, message):
    ...
```

**Calling tasks — all the ways:**

```python
# 1. .delay() — shortcut for .apply_async()
send_email.delay(user_id=42, subject="Welcome", body="Hi!")

# 2. .apply_async() — full control
send_email.apply_async(
    args=[42, "Welcome", "Hi!"],
    countdown=60,                        # delay 60 seconds before execution
    eta=datetime(2026, 5, 6, 10, 0),     # execute at specific time
    queue="emails",                      # route to specific queue
    expires=3600,                        # discard if not picked up within 1 hour
    priority=9,                          # 0-9, higher = more important (RabbitMQ only)
)

# 3. .s() — signature (for chaining)
add.s(2, 3)                              # creates a "signature" — a lazy task reference

# 4. Direct call (for testing — runs synchronously, no broker)
result = send_email(42, "Welcome", "Hi!")   # NOT recommended in production

# 5. Getting results
result = add.delay(4, 6)
result.id                                # task UUID
result.status                            # PENDING → STARTED → SUCCESS/FAILURE
result.get(timeout=10)                   # blocks until result (avoid in web requests!)
result.ready()                           # True if done
result.successful()                      # True if succeeded
```

### 12. Task Workflows — Chaining, Groups, Chords

```python
from celery import chain, group, chord, chunks

# === chain — sequential pipeline (output of one → input of next) ===
workflow = chain(
    fetch_data.s(url),                   # step 1: fetch
    parse_data.s(),                      # step 2: parse (receives fetch result)
    save_to_db.s(),                      # step 3: save (receives parse result)
)
workflow.apply_async()

# === group — parallel execution ===
results = group(
    process_file.s(f) for f in file_list
).apply_async()
results.get()                            # [result1, result2, ...]

# === chord — group + callback (parallel → then aggregate) ===
chord(
    group(fetch_price.s(symbol) for symbol in ["AAPL", "GOOGL", "MSFT"]),
)(calculate_portfolio.s())               # runs after ALL fetches complete

# === chunks — split large list into batches ===
process_item.chunks(zip(item_ids), 100).apply_async()  # 100 items per task
```

### 13. Celery Beat — Periodic/Scheduled Tasks

```python
# === settings.py ===
from celery.schedules import crontab

CELERY_BEAT_SCHEDULE = {
    "cleanup-expired-sessions": {
        "task": "accounts.tasks.cleanup_sessions",
        "schedule": crontab(hour=3, minute=0),            # daily at 3 AM
    },
    "send-daily-digest": {
        "task": "notifications.tasks.daily_digest",
        "schedule": crontab(hour=9, minute=0, day_of_week="1-5"),  # weekdays 9 AM
    },
    "sync-inventory": {
        "task": "inventory.tasks.sync_from_warehouse",
        "schedule": 300.0,                                # every 5 minutes
    },
    "monthly-report": {
        "task": "reports.tasks.generate_monthly",
        "schedule": crontab(day_of_month=1, hour=6),      # 1st of each month, 6 AM
    },
}

# For dynamic schedules (from DB), use django-celery-beat:
# pip install django-celery-beat
# CELERY_BEAT_SCHEDULER = "django_celery_beat.schedulers:DatabaseScheduler"
# Then manage schedules via Django Admin UI
```

### 14. Task Routing & Queue Management

```python
# === Route tasks to different queues based on type ===
# settings.py
CELERY_TASK_ROUTES = {
    "payments.tasks.*": {"queue": "critical"},
    "emails.tasks.*": {"queue": "emails"},
    "reports.tasks.*": {"queue": "low_priority"},
    "default": {"queue": "default"},
}

# Run specialized workers per queue:
# celery -A myproject worker -Q critical -c 2 --max-tasks-per-child=100
# celery -A myproject worker -Q emails -c 8
# celery -A myproject worker -Q low_priority -c 1
# celery -A myproject worker -Q default -c 4

# Why separate queues:
# - Critical tasks (payments) get dedicated workers — never starved
# - Email tasks can be scaled independently
# - Low-priority tasks don't block critical work
# - Different concurrency per queue (CPU-heavy = low, I/O = high)
```

### 15. Production Patterns — Idempotency, Safety & Monitoring

```python
# === CRITICAL: Always use transaction.on_commit with Celery ===
from django.db import transaction

def create_order(request):
    with transaction.atomic():
        order = Order.objects.create(...)
        OrderItem.objects.bulk_create(...)

        # WRONG: if transaction rolls back, task runs on non-existent data
        # send_confirmation.delay(order.id)  # ❌ DON'T

        # RIGHT: task only enqueued AFTER transaction commits
        transaction.on_commit(lambda: send_confirmation.delay(order.id))  # ✅

    return JsonResponse({"order_id": order.id})


# === Idempotent task design (MUST for acks_late) ===
@shared_task(bind=True, acks_late=True)
def charge_customer(self, payment_id):
    payment = Payment.objects.get(id=payment_id)

    if payment.status == "completed":
        return "already processed"         # idempotent — safe to re-run

    if payment.status != "pending":
        return f"unexpected status: {payment.status}"

    try:
        result = stripe.charge(payment.amount, payment.token)
        payment.status = "completed"
        payment.stripe_id = result.id
        payment.save()
    except stripe.CardError:
        payment.status = "failed"
        payment.save()
        # Don't retry — card was declined


# === Monitoring with Flower ===
# pip install flower
# celery -A myproject flower --port=5555
# Dashboard shows: active tasks, worker status, task history, queues, rates

# === Celery signals for observability ===
from celery.signals import task_prerun, task_postrun, task_failure

@task_failure.connect
def handle_task_failure(sender, task_id, exception, **kwargs):
    # Alert to Sentry, PagerDuty, Slack
    sentry_sdk.capture_exception(exception)
    logger.error(f"Task {sender.name}[{task_id}] failed: {exception}")

@task_postrun.connect
def log_task_completion(sender, task_id, retval, state, **kwargs):
    metrics.increment(f"celery.task.{state.lower()}", tags={"task": sender.name})
```

### 16. Celery Configuration Cheat Sheet

| Setting                                     | What                                        | Recommended                       |
| ------------------------------------------- | ------------------------------------------- | --------------------------------- |
| `CELERY_TASK_ACKS_LATE`                     | ACK after task completes (not on receive)   | `True` for critical tasks         |
| `CELERY_WORKER_PREFETCH_MULTIPLIER`         | How many tasks to pre-fetch per worker      | `1` (fair distribution)           |
| `CELERY_TASK_TIME_LIMIT`                    | Hard kill timeout (SIGKILL)                 | `300` (5 min)                     |
| `CELERY_TASK_SOFT_TIME_LIMIT`               | Raises `SoftTimeLimitExceeded`              | `240` (4 min — handle gracefully) |
| `CELERY_TASK_REJECT_ON_WORKER_LOST`         | Re-queue if worker crashes                  | `True` for critical tasks         |
| `CELERY_WORKER_MAX_TASKS_PER_CHILD`         | Restart worker after N tasks (memory leaks) | `1000`                            |
| `CELERY_TASK_ALWAYS_EAGER`                  | Run tasks synchronously (testing)           | `True` in test settings only      |
| `CELERY_TASK_EAGER_PROPAGATES`              | Propagate exceptions in eager mode          | `True`                            |
| `CELERY_BROKER_CONNECTION_RETRY_ON_STARTUP` | Retry broker connection on boot             | `True`                            |
| `CELERY_RESULT_EXPIRES`                     | TTL for stored results                      | `3600` (1 hour)                   |

### 17. Distributed Task Processing — The Complete Architecture

```
┌──────────────────────────────────────────────────────────────────┐
│                    PRODUCTION ARCHITECTURE                       │
│                                                                  │
│  Web Server                                                      │
│  ├── API receives request                                        │
│  ├── Writes to DB (within transaction.atomic)                    │
│  ├── on_commit → enqueues Celery task to Redis/RabbitMQ          │
│  └── Returns 202 Accepted immediately                            │
│                                                                  │
│  Message Broker (Redis / RabbitMQ)                               │
│  ├── Queue: "critical"    → payment tasks                        │
│  ├── Queue: "default"     → general tasks                        │
│  ├── Queue: "emails"      → notification tasks                   │
│  └── Queue: "dlq"         → failed tasks (dead letter)           │
│                                                                  │
│  Workers (separate processes/containers)                         │
│  ├── Worker pool A: -Q critical -c 2 (dedicated to payments)     │
│  ├── Worker pool B: -Q default,emails -c 8                       │
│  └── Worker pool C: -Q dlq -c 1 (manual inspection)              │
│                                                                  │
│  Retry System                                                    │
│  ├── Attempt 1: immediate                                        │
│  ├── Attempt 2: 60s delay (exponential backoff)                  │
│  ├── Attempt 3: 240s delay                                       │
│  ├── Attempt 4: 960s delay                                       │
│  └── After max_retries → route to DLQ                            │
│                                                                  │
│  Dead Letter Queue                                               │
│  ├── Stores permanently failed messages                          │
│  ├── Alert on DLQ growth (PagerDuty/Slack)                       │
│  ├── Admin UI for inspection and manual retry                    │
│  └── Never auto-retry from DLQ — requires human decision         │
└──────────────────────────────────────────────────────────────────┘
```

**Retry system patterns:**

| Pattern                 | How                                     | When                               |
| ----------------------- | --------------------------------------- | ---------------------------------- |
| **Immediate retry**     | `self.retry(countdown=0)`               | Transient network glitch           |
| **Exponential backoff** | `countdown=2 ** self.request.retries`   | Rate limits, overloaded downstream |
| **Backoff + jitter**    | `retry_backoff=True, retry_jitter=True` | Thundering herd prevention         |
| **Max retries**         | `max_retries=5` then give up            | All tasks — prevent infinite loops |
| **Conditional retry**   | Only retry on specific exceptions       | Don't retry validation errors      |
| **Dead letter**         | After max retries → route to DLQ        | Poison messages that always fail   |

**Dead Letter Queue implementation:**

```python
# === RabbitMQ DLQ (built-in) ===
# Queue declaration with DLX:
channel.queue_declare(
    queue="payments",
    arguments={
        "x-dead-letter-exchange": "dlx",
        "x-dead-letter-routing-key": "failed.payments",
        "x-message-ttl": 86400000,        # optional: 24h TTL
    },
)
# Failed messages (NACK without requeue) automatically go to DLQ

# === Celery DLQ (manual implementation) ===
@shared_task(bind=True, max_retries=3)
def process_order(self, order_id):
    try:
        do_processing(order_id)
    except Exception as exc:
        if self.request.retries >= self.max_retries:
            # Max retries exhausted → send to DLQ
            dead_letter_task.delay(
                original_task="process_order",
                args=[order_id],
                exception=str(exc),
                retries=self.request.retries,
            )
            return  # don't retry again
        raise self.retry(exc=exc, countdown=2 ** self.request.retries)

@shared_task(queue="dlq")
def dead_letter_task(original_task, args, exception, retries):
    """Store failed task for inspection. Alert ops team."""
    FailedTask.objects.create(
        task_name=original_task,
        args=args,
        exception=exception,
        retry_count=retries,
    )
    alert_ops(f"Task {original_task} permanently failed after {retries} retries")
```

**Idempotency in distributed tasks:**

```python
# === Pattern: Idempotency key stored in Redis/DB ===
import hashlib

@shared_task(bind=True, acks_late=True)
def process_webhook(self, webhook_id, payload):
    # Generate idempotency key
    idem_key = f"webhook:{webhook_id}"

    # Check if already processed
    if cache.get(idem_key):
        logger.info(f"Webhook {webhook_id} already processed, skipping")
        return "duplicate"

    # Process the webhook
    try:
        handle_webhook(payload)
        # Mark as processed (with TTL to prevent unbounded growth)
        cache.set(idem_key, "done", timeout=86400)  # 24h
    except Exception as exc:
        raise self.retry(exc=exc)

# Why idempotency matters for distributed tasks:
# 1. acks_late → broker may re-deliver after timeout
# 2. Network partition → task might be sent twice
# 3. Worker crash mid-processing → task requeued
# 4. Retry logic → by definition, runs the same task again
# Solution: every task must be safe to run N times with same result
```

### 18. Celery Interview FAQ

> **Q: How do you create and initialize a task in Celery?**
> A: Decorate a function with `@shared_task` (app-agnostic) or `@app.task` (bound to specific app). `@shared_task` is preferred for reusable Django apps. Configure Celery in `celery.py` with broker URL, call `autodiscover_tasks()` to find `tasks.py` in each app, and import the celery app in `__init__.py`. Call tasks with `.delay()` (simple) or `.apply_async()` (full control with countdown, ETA, queue, priority).

> **Q: `@shared_task` vs `@app.task` — what's the difference?**
> A: `@shared_task` doesn't need a reference to the Celery app — it registers the task to whatever app is current. Use for reusable apps. `@app.task` is bound to a specific Celery app instance. Use in your project's own code. Both produce the same result; `@shared_task` is more portable.

> **Q: What is `bind=True` and why would you use it?**
> A: `bind=True` passes the task instance as the first argument (`self`), giving access to `self.retry()`, `self.request` (task ID, retries count, delivery info), and `self.app`. Essential for retry logic: `raise self.retry(exc=exc, countdown=60)`.

> **Q: How do you handle task failures and retries?**
> A: Two approaches: (1) Manual: `bind=True` + try/except + `self.retry(exc=exc, countdown=backoff)`. (2) Auto: `autoretry_for=(ExceptionClass,)` + `retry_backoff=True` + `retry_jitter=True`. Always set `max_retries` to prevent infinite loops. After max retries, route to DLQ.

> **Q: What happens if a Celery worker crashes mid-task?**
> A: With default settings (`acks_early`): task is lost forever. With `acks_late=True` + `reject_on_worker_lost=True`: task is re-queued to the broker and picked up by another worker. This means tasks MUST be idempotent.

> **Q: How do you ensure a Celery task only runs after the DB transaction commits?**
> A: Use `transaction.on_commit(lambda: my_task.delay(obj.id))`. Without this, if the transaction rolls back, the worker tries to process data that doesn't exist. This is the #1 Celery production bug.

> **Q: How do you monitor Celery in production?**
> A: Flower (web UI for real-time monitoring), Prometheus + celery-exporter (metrics), Sentry (error tracking via `task_failure` signal), custom logging via `task_prerun`/`task_postrun` signals. Key metrics: queue depth, task latency, failure rate, worker count.

> **Q: Celery vs Kafka vs RabbitMQ — when to use which?**
> A: Celery is a **task queue** (Python-specific, execute code in workers). Kafka is an **event log** (language-agnostic, replay, streaming). RabbitMQ is a **message broker** (routing, pub/sub). Celery often USES Redis/RabbitMQ as its broker. Use Celery when you need Python background jobs. Use Kafka when you need event streaming/replay across services. Use RabbitMQ when you need complex message routing.

> **Q: How do you handle priority tasks in Celery?**
> A: Two approaches: (1) Separate queues: `@shared_task(queue="critical")` with dedicated workers. (2) Priority within a queue: `task.apply_async(priority=9)` (requires RabbitMQ, not Redis). Separate queues is more reliable and the recommended approach.

> **Q: What is `CELERY_WORKER_PREFETCH_MULTIPLIER` and why set it to 1?**
> A: It controls how many tasks a worker pre-fetches from the queue. Default is 4 — worker grabs 4 tasks at once. Set to 1 for fair distribution (prevents one worker from hoarding long tasks while others are idle). Set higher for very fast, uniform tasks.

📌 **TLDR:** "Celery = distributed task queue for Python. Setup: `celery.py` + broker (Redis/RMQ) + `@shared_task`. Call with `.delay()` or `.apply_async()`. Retry: `bind=True` + `self.retry()` with exponential backoff. Safety: `acks_late=True` + idempotent tasks + `transaction.on_commit()`. Routing: separate queues for critical/default/low tasks. Monitoring: Flower + Prometheus + Sentry. DLQ: route permanently failed tasks for human inspection. The #1 production rule: every task must be idempotent and only enqueued after the DB transaction commits."
