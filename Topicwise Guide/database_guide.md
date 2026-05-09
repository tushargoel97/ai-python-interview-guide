# Databases

### 📑 Table of Contents

1. [1. Database Choice — SQL vs NoSQL & When to Use What](#1-database-choice-sql-vs-nosql--when-to-use-what)
2. [2. ACID Properties](#2-acid-properties)
3. [3. `transaction.atomic` Deep Dive (savepoints, on_commit, select_for_update)](#3-deep-dive--transactionatomic-and-programmatic-transaction-control)
4. [4. SQL Normalization, Locks, Indexing, Deadlocks & JOINs](#4-sql-normalization-locks-indexing-deadlocks--joins)
5. [5. Database Views — Virtual Tables & Materialized Views](#5-database-views--virtual-tables--materialized-views)

## 1. Database Choice SQL vs NoSQL & When to Use What

> **📣 Definition:** _"SQL databases (PostgreSQL, MySQL) give you ACID transactions, JOINs, strong schema enforcement, and rock-solid consistency — the default for anything financial or relational. NoSQL is an umbrella for very different things: document stores (MongoDB) for flexible schemas, key-value (Redis, DynamoDB) for sub-ms reads, time-series (TimescaleDB, InfluxDB) for metrics, search (Elasticsearch) for full-text, and graph (Neo4j) for relationship traversal. The pragmatic answer for most apps in 2026: **start with PostgreSQL** (it has JSONB, full-text search, time-series extensions) and add specialized stores only when there's a clear need."_

| Use case                               | Best choice           | Why                                        |
| -------------------------------------- | --------------------- | ------------------------------------------ |
| Transactions, relationships            | PostgreSQL, MySQL     | ACID, JOINs, constraints                   |
| High-write throughput, flexible schema | MongoDB, DynamoDB     | Horizontal scaling, schema-less            |
| Time-series data                       | TimescaleDB, InfluxDB | Optimized for time-ordered inserts/queries |
| Caching, sessions                      | Redis                 | In-memory, sub-ms latency                  |
| Search                                 | Elasticsearch         | Full-text search, aggregations             |
| Graph relationships                    | Neo4j                 | Traversals, recommendations                |
| Event streaming                        | Kafka                 | Distributed log, replay, high throughput   |

**📌 TLDR**: "PostgreSQL is the default for most backend work (ACID, JSONB, full-text search built-in). Add Redis for caching, Elasticsearch for search, Kafka for event streaming. NoSQL when you need horizontal scaling and flexible schemas."

---

## 2. ACID Properties

> **📣 Interview-ready definition:** _"ACID is the four guarantees a database makes for reliable transactions. **A**tomicity — all operations in a transaction succeed together or roll back together. **C**onsistency — the database moves from one valid state to another, with all constraints satisfied. **I**solation — concurrent transactions don't interfere with each other (controlled by isolation levels). **D**urability — once committed, data survives crashes (via write-ahead logs flushed to disk). In distributed microservices, true ACID across services is hard, so we use BASE (Basically Available, Soft-state, Eventually consistent) with patterns like SAGA."_

### 🧑‍🍳 Layman Explanation

ACID is like a **bank transfer promise**:

- **A (Atomicity)**: Transferring ₹500 either fully happens (debit + credit) or doesn't happen at all. No half-transfers.
- **C (Consistency)**: Your bank balance can't go below ₹0 (if that's a rule). Every transaction respects the rules.
- **I (Isolation)**: If you and your brother both withdraw from the same account at the same time, you won't see each other's half-finished work.
- **D (Durability)**: Once the transfer is confirmed, even if the bank's power goes out 1 second later, it's permanent.

### 🔧 Technical Explanation

| Property        | Meaning                                                                     | Mechanism                           |
| --------------- | --------------------------------------------------------------------------- | ----------------------------------- |
| **Atomicity**   | All-or-nothing. Transaction fully commits or fully rolls back.              | Write-ahead log (WAL), undo logs    |
| **Consistency** | DB moves from one valid state to another. Constraints are always satisfied. | Constraints, triggers, foreign keys |
| **Isolation**   | Concurrent transactions don't interfere with each other.                    | Isolation levels, locks, MVCC       |
| **Durability**  | Committed data survives crashes.                                            | WAL flushed to disk, replication    |

---

#### A — Atomicity in Practice

```sql
-- Bank transfer: BOTH operations succeed or NEITHER does
BEGIN;
    UPDATE accounts SET balance = balance - 500 WHERE id = 1;   -- debit
    -- 💥 Imagine power loss here
    UPDATE accounts SET balance = balance + 500 WHERE id = 2;   -- credit
COMMIT;

-- Without atomicity: ₹500 disappears! Account 1 charged, account 2 not credited.
-- With atomicity: power loss → DB recovers → no commit found → both UPDATEs rolled back.
```

**In Python (Django):**

```python
from django.db import transaction

# ❌ WRONG: Not atomic, partial state possible
def transfer_bad(from_id, to_id, amount):
    Account.objects.filter(id=from_id).update(balance=F('balance') - amount)
    # 💥 If exception here, money disappears
    Account.objects.filter(id=to_id).update(balance=F('balance') + amount)

# ✅ RIGHT: Atomic transaction
def transfer_good(from_id, to_id, amount):
    with transaction.atomic():
        Account.objects.filter(id=from_id).update(balance=F('balance') - amount)
        Account.objects.filter(id=to_id).update(balance=F('balance') + amount)
    # If ANY exception inside the block → entire transaction rolled back
```

---

#### C — Consistency in Practice

Consistency means **constraints are always satisfied**: foreign keys, unique constraints, check constraints, triggers.

```sql
-- Schema-level consistency rules
CREATE TABLE accounts (
    id SERIAL PRIMARY KEY,
    balance NUMERIC(10, 2) CHECK (balance >= 0),    -- balance can't go negative
    user_id INT REFERENCES users(id) ON DELETE RESTRICT
);

CREATE TABLE orders (
    id SERIAL PRIMARY KEY,
    user_id INT REFERENCES users(id),
    total NUMERIC(10, 2) CHECK (total > 0),
    status VARCHAR(20) CHECK (status IN ('pending', 'paid', 'shipped'))
);

-- This update FAILS — DB enforces consistency
UPDATE accounts SET balance = -100 WHERE id = 1;
-- ERROR: new row for relation "accounts" violates check constraint "accounts_balance_check"
```

**Application-level consistency:** Use database constraints + Django model validators + transaction blocks. Don't rely solely on application code — DB constraints are the last line of defense.

---

#### I — Isolation in Practice (with Examples of Each Anomaly)

```sql
-- ============================================
-- DIRTY READ (prevented by Read Committed+)
-- ============================================
-- Txn A                     Txn B
BEGIN;                       BEGIN;
UPDATE products
SET stock = 0 WHERE id = 1;
                             SELECT stock FROM products WHERE id = 1;
                             -- ❌ Reads 0 (uncommitted!)
                             -- Makes business decision based on dirty data
ROLLBACK;                    -- A rolled back, but B already acted on bad data!

-- ============================================
-- NON-REPEATABLE READ (prevented by Repeatable Read+)
-- ============================================
-- Txn A                     Txn B
BEGIN;                       BEGIN;
SELECT balance FROM accts
WHERE id = 1;
-- Returns 1000
                             UPDATE accts SET balance = 500 WHERE id = 1;
                             COMMIT;
SELECT balance FROM accts
WHERE id = 1;
-- ❌ Returns 500! Same query, different answer in same transaction.

-- ============================================
-- PHANTOM READ (prevented by Serializable)
-- ============================================
-- Txn A                     Txn B
BEGIN;                       BEGIN;
SELECT count(*) FROM orders
WHERE status = 'pending';
-- Returns 5
                             INSERT INTO orders (status) VALUES ('pending');
                             COMMIT;
SELECT count(*) FROM orders
WHERE status = 'pending';
-- ❌ Returns 6! New "phantom" row appeared.
```

**Isolation Levels (from weak to strong):**

| Level            | Dirty Read   | Non-Repeatable Read | Phantom Read                  | Performance |
| ---------------- | ------------ | ------------------- | ----------------------------- | ----------- |
| Read Uncommitted | ✅ Possible  | ✅ Possible         | ✅ Possible                   | Fastest     |
| Read Committed   | ❌ Prevented | ✅ Possible         | ✅ Possible                   | Fast        |
| Repeatable Read  | ❌           | ❌ Prevented        | ✅ Possible (depends on impl) | Medium      |
| Serializable     | ❌           | ❌                  | ❌ Prevented                  | Slowest     |

PostgreSQL default: **Read Committed**. MySQL/InnoDB default: **Repeatable Read** (and InnoDB's RR also prevents phantom reads via gap locks — stricter than the SQL standard).

**Setting isolation level:**

```sql
-- PostgreSQL: per-transaction
BEGIN;
SET TRANSACTION ISOLATION LEVEL SERIALIZABLE;
-- ... your queries ...
COMMIT;

-- Or per-session
SET DEFAULT_TRANSACTION_ISOLATION = 'serializable';
```

```python
# Django
from django.db import transaction
from django.db.transaction import atomic

with atomic(durable=True):
    # On PostgreSQL, set isolation level via the connection
    with connection.cursor() as cursor:
        cursor.execute("SET TRANSACTION ISOLATION LEVEL SERIALIZABLE")
    # ... your ORM queries ...
```

---

#### D — Durability in Practice

Once `COMMIT` returns success, the data MUST survive crashes.

**How databases achieve durability:**

1. **Write-Ahead Log (WAL)**: Before modifying data files, the DB writes the change to a sequential log on disk. On crash recovery, replay the log.
2. **fsync()**: At commit time, force the WAL to be flushed from OS buffer to physical disk. (`fsync_commit = on` in PostgreSQL.)
3. **Replication**: Synchronous replication to a standby server before acknowledging the commit (multiple disks → survives a single disk failure).

```
Commit Flow (simplified):
  1. Application: COMMIT
  2. DB writes change to WAL buffer (in memory)
  3. DB calls fsync() → WAL forced to disk
  4. (Optional) DB waits for replica to confirm
  5. Return COMMIT success to app
  6. (Later) WAL changes applied to actual data files

If crash happens between step 3 and 6:
  → On restart, replay WAL to recover.
  → No data loss.
```

**Tunable trade-offs:**

- `synchronous_commit = off` (PostgreSQL) — faster, but risks losing last few transactions on crash.
- `innodb_flush_log_at_trx_commit = 2` (MySQL) — flush every second instead of every commit.
- These trade durability for performance — only acceptable for non-critical data.

---

#### ACID in Distributed Systems

True ACID is hard across services because there's no global transaction coordinator. That's why microservices use **BASE**:

- **B**asically **A**vailable: respond to every request, even if degraded
- **S**oft-state: state may change without input (eventual consistency happening)
- **E**ventual consistency: given enough time, all replicas converge

**Distributed transaction patterns:**

- **2PC (Two-Phase Commit)**: Strong consistency but blocks on coordinator failure. Rarely used in microservices.
- **SAGA**: Sequence of local transactions + compensating actions. (See [SAGA pattern](coding_principle_guide.md#3-saga-pattern--other-design-patterns).)
- **Outbox Pattern**: Atomically write business data + event to outbox table; separate process publishes to message broker.

---

#### 3. Deep Dive — `transaction.atomic` and Programmatic Transaction Control

This is the workhorse pattern for atomicity in Django (and any Python ORM). Interviewers love it because the surface API looks simple but the edge cases are where senior engineers get separated from juniors.

##### The Core Pattern

```python
from django.db import transaction

# Style 1: Context manager (recommended — explicit boundary)
def transfer_money(from_id, to_id, amount):
    with transaction.atomic():
        from_acc = Account.objects.select_for_update().get(id=from_id)
        to_acc = Account.objects.select_for_update().get(id=to_id)

        if from_acc.balance < amount:
            raise InsufficientFundsError()

        from_acc.balance -= amount
        from_acc.save()
        to_acc.balance += amount
        to_acc.save()
    # COMMIT happens here when the block exits cleanly.
    # If ANY exception → ROLLBACK automatically.

# Style 2: Decorator (wraps the whole function)
@transaction.atomic
def transfer_money(from_id, to_id, amount):
    # entire function body is one transaction
    ...
```

**What `atomic` actually does:**

- On entry: `BEGIN` (if outermost) or `SAVEPOINT` (if nested).
- On clean exit: `COMMIT` (or `RELEASE SAVEPOINT`).
- On exception: `ROLLBACK` (or `ROLLBACK TO SAVEPOINT`), then re-raises.

##### Nested `atomic` — How It Actually Works (Interview Gold)

Many engineers think nested `atomic()` blocks open new transactions. They don't. Django uses **savepoints**:

```python
@transaction.atomic                      # ← OUTER: real BEGIN/COMMIT
def process_order(order):
    order.status = 'processing'
    order.save()

    try:
        with transaction.atomic():       # ← INNER: SAVEPOINT
            charge_payment(order)
            ship_item(order)
    except PaymentFailed:
        # Inner block rolled back to its savepoint.
        # Outer transaction is STILL ALIVE and can continue.
        order.status = 'payment_failed'
        order.save()                     # this WILL be committed
        notify_customer(order)

    # outer block commits here
```

**The behavior:**

```
Outer atomic     → BEGIN
  order.save()   → INSERT/UPDATE
  Inner atomic   → SAVEPOINT sp1
    charge       → ...
    💥 raises    → ROLLBACK TO SAVEPOINT sp1
  except handler → continues (outer txn still alive!)
  order.save()   → UPDATE (will be committed)
End outer       → COMMIT
```

This is incredibly powerful — you can have "best effort" sub-operations that fail without dooming the entire transaction. **But there's a famous gotcha** ↓

##### The "Cannot Execute Queries After Failure" Trap

If a database error occurs inside an inner `atomic()`, you **cannot run more queries in the outer block** until you exit the inner one and let it roll back its savepoint:

```python
@transaction.atomic
def bad_pattern():
    Order.objects.create(...)

    try:
        # IntegrityError, e.g., violating a UNIQUE constraint
        User.objects.create(email='duplicate@example.com')
    except IntegrityError:
        # ❌ This will raise: TransactionManagementError
        # "An error occurred in the current transaction.
        #  You can't execute queries until the end of the 'atomic' block."
        Order.objects.filter(...).update(...)

# ✅ CORRECT: wrap the risky operation in its own atomic block
@transaction.atomic
def good_pattern():
    Order.objects.create(...)

    try:
        with transaction.atomic():           # creates SAVEPOINT
            User.objects.create(email='duplicate@example.com')
    except IntegrityError:
        # Inner atomic rolled back to SAVEPOINT, outer is healthy
        Order.objects.filter(...).update(...)  # ✅ works
```

**Rule of thumb:** if you might catch a database exception, wrap that operation in its own inner `atomic()`. The savepoint isolates the failure.

##### `transaction.on_commit` — Don't Fire Side Effects on Rollback

This is critical for production correctness. Interviewers love asking "what's wrong with this code":

```python
# ❌ THE BUG: side effects fire even if transaction rolls back
@transaction.atomic
def register_user(email, password):
    user = User.objects.create(email=email, password=password)
    send_welcome_email.delay(user.id)        # ← Celery task queued IMMEDIATELY
    publish_to_kafka('user.created', user)   # ← Kafka event published IMMEDIATELY

    if some_validation_fails():
        raise ValueError("Invalid")
    # 💥 Transaction rolls back, BUT:
    # - Celery worker tries to process user.id that doesn't exist!
    # - Kafka subscribers see a user that was never persisted!
```

The fix:

```python
# ✅ CORRECT: defer side effects until commit succeeds
@transaction.atomic
def register_user(email, password):
    user = User.objects.create(email=email, password=password)

    transaction.on_commit(lambda: send_welcome_email.delay(user.id))
    transaction.on_commit(lambda: publish_to_kafka('user.created', user))

    if some_validation_fails():
        raise ValueError("Invalid")
    # If this raises → on_commit callbacks NEVER FIRE. ✅
    # If transaction commits → callbacks fire AFTER commit, in registration order.
```

**Why this matters in payments:**

```python
@transaction.atomic
def process_payment(order):
    payment = Payment.objects.create(order=order, status='captured')

    # ❌ BAD: webhook publishes "payment.captured" then txn rolls back
    # publish_webhook('payment.captured', payment.id)

    # ✅ GOOD: webhook only fires after commit succeeds
    transaction.on_commit(lambda: publish_webhook('payment.captured', payment.id))
```

This is the **Outbox pattern's simpler cousin** — `on_commit` works for in-process side effects, but for true durability across crashes, you still need an outbox table (because if the server crashes after commit but before the callback fires, the callback is lost).

##### `select_for_update` — Pessimistic Locking Inside Transactions

`select_for_update` only works **inside an atomic block** — it requires an active transaction:

```python
# ❌ Will raise: TransactionManagementError
def transfer_bad(from_id, to_id, amount):
    Account.objects.select_for_update().get(id=from_id)  # no txn!

# ✅ Correct
@transaction.atomic
def transfer_good(from_id, to_id, amount):
    from_acc = Account.objects.select_for_update().get(id=from_id)
    # Row is locked until transaction commits/rolls back
    from_acc.balance -= amount
    from_acc.save()

# Advanced flags
@transaction.atomic
def claim_task():
    # SKIP LOCKED — for worker queues, skip rows other workers have locked
    task = Task.objects.select_for_update(skip_locked=True).filter(
        status='pending'
    ).first()

    # NOWAIT — fail immediately instead of waiting for lock
    task = Task.objects.select_for_update(nowait=True).get(id=42)

    # of=('table_name',) — lock only specific tables in a JOIN
    Order.objects.select_related('user').select_for_update(of=('self',)).get(id=1)
```

##### `transaction.atomic(durable=True)` — Detecting Outer-Block Bugs

When `durable=True`, the block raises `RuntimeError` if it's nested inside another atomic. This catches a common bug:

```python
# Library function — must always commit on exit
@transaction.atomic(durable=True)
def critical_idempotent_operation():
    # ... some critical write ...
    pass

@transaction.atomic
def caller():
    critical_idempotent_operation()
    # 💥 RuntimeError: A durable atomic block cannot be nested
    # This catches: caller wrapped us in their own txn,
    # so our "commit" is actually just a SAVEPOINT release.
    # If they roll back later, our "committed" data is gone.
```

Use `durable=True` for operations whose effects MUST be committed when the function returns.

##### Auto-commit vs Atomic Mode

Django's default is **autocommit mode** — every query is its own transaction:

```python
# Without atomic — each line is a separate transaction!
Account.objects.filter(id=1).update(balance=F('balance') - 500)  # COMMIT
Account.objects.filter(id=2).update(balance=F('balance') + 500)  # COMMIT
# 💥 If second call fails, first is already committed → money lost!

# With atomic — both lines are one transaction
with transaction.atomic():
    Account.objects.filter(id=1).update(balance=F('balance') - 500)
    Account.objects.filter(id=2).update(balance=F('balance') + 500)
# Both succeed together or both roll back.
```

You can flip this globally with `ATOMIC_REQUESTS = True` in settings (wraps every HTTP view in a transaction), but most teams prefer explicit per-function control.

##### Common Mistakes Senior Engineers Look For

```python
# Mistake 1: Catching exceptions inside atomic and not re-raising
@transaction.atomic
def buggy():
    Order.objects.create(...)
    try:
        risky_operation()
    except Exception:
        logger.error("oops")        # ❌ swallows exception
    # Transaction commits anyway — logical bug, broken state persists

# Mistake 2: Long-running atomic blocks holding locks
@transaction.atomic
def slow():
    user = User.objects.select_for_update().get(id=1)  # row locked
    response = requests.post('https://slow-api.com/...')  # 30 second call!
    # Other transactions waiting on this user wait 30s. Deadlock factory.

# Mistake 3: Mixing atomic with manual savepoints
@transaction.atomic
def confusing():
    sid = transaction.savepoint()
    # Mixing styles → hard to reason about. Just use nested atomic instead.

# Mistake 4: Not understanding atomic + threading
@transaction.atomic
def def_threading_bug():
    user = User.objects.create(...)
    Thread(target=lambda: User.objects.filter(id=user.id).update(...)).start()
    # ❌ The thread uses a DIFFERENT connection. It can't see uncommitted data.
    # The thread might query and get nothing, or hit the row before commit.
```

##### Quick Cheat Card

| Pattern                               | When to use                                                    |
| ------------------------------------- | -------------------------------------------------------------- |
| `with transaction.atomic():`          | Explicit, scoped transaction (most common)                     |
| `@transaction.atomic` decorator       | Whole function is one transaction                              |
| Nested `atomic()` (savepoints)        | Sub-operations that can fail without aborting parent           |
| `transaction.on_commit(callback)`     | Defer Celery tasks, emails, webhooks until commit              |
| `transaction.atomic(durable=True)`    | Operations that MUST commit on exit (catches outer-block bugs) |
| `select_for_update()`                 | Pessimistic row lock (must be inside atomic)                   |
| `select_for_update(skip_locked=True)` | Worker queue pattern — skip rows others have locked            |
| `select_for_update(nowait=True)`      | Fail fast instead of waiting for a lock                        |
| `ATOMIC_REQUESTS = True` (settings)   | Wrap every HTTP view in a transaction (global)                 |

📌 **TLDR for `transaction.atomic`:** "Wrap multi-step DB writes in `atomic()` — context manager or decorator. Nested calls use SAVEPOINTs, so inner failures can be caught without aborting the outer transaction (classic pattern for 'best effort' sub-operations). **Always use `transaction.on_commit(callback)` to defer side effects** — Celery tasks, webhooks, emails — until the transaction actually commits, otherwise rollbacks leave your async workers chasing data that doesn't exist. `select_for_update` requires an active atomic block. Use `durable=True` for operations whose commit is non-negotiable. The single biggest mistake: firing side effects inside atomic without `on_commit`."

### 📌 TLDR for Interviews

> "ACID guarantees reliable transactions: **A**tomicity (all-or-nothing — use `transaction.atomic()` in Django), **C**onsistency (DB constraints always satisfied — use CHECK, FK, UNIQUE), **I**solation (concurrent txns don't interfere — Read Committed prevents dirty reads, Repeatable Read prevents non-repeatable, Serializable prevents phantoms), **D**urability (committed = permanent — via WAL + fsync + replication). PostgreSQL defaults to Read Committed; MySQL/InnoDB to Repeatable Read. In distributed microservices, ACID is replaced by BASE + SAGA + Outbox patterns for eventual consistency."

---

## 4. SQL Normalization, Locks, Indexing, Deadlocks & JOINs

> **📣 Interview-ready definition:** _"This covers the four pillars of SQL mastery. **Normalization** (1NF→BCNF) eliminates data redundancy by splitting data into related tables. **Locks** prevent concurrent transactions from corrupting each other — row-level (`FOR UPDATE`) is granular, table-level is heavy. **Indexes** make reads fast at the cost of slower writes — B-Tree for most cases, GIN for JSON/full-text, BRIN for time-series. **Deadlocks** happen when transactions wait on each other in a cycle — prevented by consistent lock ordering, short transactions, and `NOWAIT`/`SKIP LOCKED`. **JOINs** combine rows across tables — INNER returns matches only, LEFT keeps all from the left side, FULL OUTER keeps all from both."_

### 🧑‍🍳 Layman Explanation

**Normalization** = Organizing your wardrobe. Instead of stuffing everything in one drawer (denormalized), you have separate drawers for shirts, pants, socks. Less duplication, easier to maintain — but finding a full outfit requires opening multiple drawers (JOINs).

**Locks** = Fitting room doors. Only one person can use a fitting room (row-level lock) at a time. Sometimes the whole store is closed for inventory (table-level lock).

**Deadlock** = Two people stuck in a hallway — Person A holds the bathroom key and needs the kitchen key, Person B holds the kitchen key and needs the bathroom key. Neither can move. The database manager (deadlock detector) taps one on the shoulder and says "you go back, try again later."

**JOINs** = You have a contacts book (users table) and a separate orders ledger (orders table). A JOIN is putting them side by side and matching entries — "show me each person AND their orders." Different JOINs decide what to do when someone has no orders or an order has no matching person.

**Indexing** = A book's index at the back. Instead of reading every page to find "Python," you look up "Python → page 42" and go directly there.

### 🔧 Technical Explanation

---

#### Normalization Forms

| Form     | Rule                                                                                     | Example violation                                                                     |
| -------- | ---------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------- |
| **1NF**  | Atomic values, no repeating groups                                                       | `phone: "123, 456"` → should be separate rows                                         |
| **2NF**  | 1NF + no partial dependencies (every non-key column depends on the _whole_ primary key)  | In `(student_id, course_id) → student_name` — student_name depends only on student_id |
| **3NF**  | 2NF + no transitive dependencies (non-key columns don't depend on other non-key columns) | `zip_code → city` — city depends on zip, not on the primary key directly              |
| **BCNF** | 3NF + every determinant is a candidate key                                               | Stricter version of 3NF for edge cases                                                |

**Interview-ready normalization walkthrough:**

```
UNNORMALIZED (violates 1NF):
┌────┬──────────┬────────────────────┬────────────────┐
│ id │ name     │ phones             │ city, zip      │
├────┼──────────┼────────────────────┼────────────────┤
│ 1  │ Tushar   │ 7009299432, 98765  │ Ambala, 133001 │
│ 2  │ Aman     │ 9876543210         │ Delhi, 110001  │
└────┴──────────┴────────────────────┴────────────────┘
Problems: phones has multiple values, city+zip are mashed together.

1NF (atomic values):
┌────┬──────────┬────────────┬──────────┬────────┐
│ id │ name     │ phone      │ city     │ zip    │
├────┼──────────┼────────────┼──────────┼────────┤
│ 1  │ Tushar   │ 7009299432 │ Ambala   │ 133001 │
│ 1  │ Tushar   │ 98765      │ Ambala   │ 133001 │
│ 2  │ Aman     │ 9876543210 │ Delhi    │ 110001 │
└────┴──────────┴────────────┴──────────┴────────┘
Problem: name repeats (partial dependency on composite key id+phone)

2NF (separate phones table):
users: (id PK, name, city, zip)
phones: (id, phone) → FK to users.id
Problem: city depends on zip, not on id (transitive dependency)

3NF (separate zip→city mapping):
users: (id PK, name, zip FK)
phones: (user_id FK, phone)
locations: (zip PK, city)
✅ No redundancy. Each fact stored once.
```

**When to denormalize**: Read-heavy workloads, reporting/analytics, caching layers, CQRS read models. Trade storage + update complexity for query speed.

---

#### SQL JOINs — Complete Guide with Examples

**Sample tables for all examples:**

```sql
-- users table                    -- orders table
┌────┬──────────┬──────────┐     ┌──────────┬─────────┬────────┐
│ id │ name     │ city     │     │ order_id │ user_id │ amount │
├────┼──────────┼──────────┤     ├──────────┼─────────┼────────┤
│ 1  │ Tushar   │ Ambala   │     │ 101      │ 1       │ ₹500   │
│ 2  │ Aman     │ Delhi    │     │ 102      │ 1       │ ₹300   │
│ 3  │ Priya    │ Mumbai   │     │ 103      │ 2       │ ₹700   │
│ 4  │ Rahul    │ Pune     │     │ 104      │ NULL    │ ₹200   │
└────┴──────────┴──────────┘     └──────────┴─────────┴────────┘
Note: Rahul (id=4) has NO orders. Order 104 has NO matching user.
```

**1. INNER JOIN** — Only matching rows from BOTH tables

```sql
SELECT u.name, o.order_id, o.amount
FROM users u
INNER JOIN orders o ON u.id = o.user_id;

-- Result:
-- Tushar | 101 | ₹500
-- Tushar | 102 | ₹300
-- Aman   | 103 | ₹700
-- ❌ Rahul excluded (no orders)
-- ❌ Order 104 excluded (no matching user)
```

**When to use:** You want ONLY rows where both sides have matching data. Most common JOIN.

**2. LEFT JOIN (LEFT OUTER JOIN)** — All rows from LEFT table + matching from right

```sql
SELECT u.name, o.order_id, o.amount
FROM users u
LEFT JOIN orders o ON u.id = o.user_id;

-- Result:
-- Tushar | 101  | ₹500
-- Tushar | 102  | ₹300
-- Aman   | 103  | ₹700
-- Priya  | NULL | NULL   ← included with NULLs (no orders)
-- Rahul  | NULL | NULL   ← included with NULLs (no orders)
-- ❌ Order 104 still excluded (no matching user in LEFT table)
```

**When to use:** "Show me ALL users, even those with no orders." Reports, dashboards, finding orphaned records.

**3. RIGHT JOIN (RIGHT OUTER JOIN)** — All rows from RIGHT table + matching from left

```sql
SELECT u.name, o.order_id, o.amount
FROM users u
RIGHT JOIN orders o ON u.id = o.user_id;

-- Result:
-- Tushar | 101 | ₹500
-- Tushar | 102 | ₹300
-- Aman   | 103 | ₹700
-- NULL   | 104 | ₹200   ← order with no matching user
-- ❌ Priya and Rahul excluded (not in orders table)
```

**When to use:** Rare in practice — you can always rewrite as LEFT JOIN by swapping table order. But useful for "show me all orders, even orphaned ones."

**4. FULL OUTER JOIN** — All rows from BOTH tables, NULLs where no match

```sql
SELECT u.name, o.order_id, o.amount
FROM users u
FULL OUTER JOIN orders o ON u.id = o.user_id;

-- Result:
-- Tushar | 101  | ₹500
-- Tushar | 102  | ₹300
-- Aman   | 103  | ₹700
-- Priya  | NULL | NULL   ← user with no orders
-- Rahul  | NULL | NULL   ← user with no orders
-- NULL   | 104  | ₹200   ← order with no user
```

**When to use:** Data reconciliation, finding mismatches between two datasets, migration audits.

**5. CROSS JOIN** — Every row in A × every row in B (Cartesian product)

```sql
SELECT u.name, o.order_id
FROM users u
CROSS JOIN orders o;

-- Result: 4 users × 4 orders = 16 rows
-- Tushar | 101, Tushar | 102, Tushar | 103, Tushar | 104,
-- Aman   | 101, Aman   | 102, ... (every combination)
```

**When to use:** Generating all combinations — e.g., all sizes × all colors for a product matrix. Rarely used in application queries.

**6. SELF JOIN** — Table joined to itself

```sql
-- employees table: (id, name, manager_id)
-- Find each employee's manager name
SELECT e.name AS employee, m.name AS manager
FROM employees e
LEFT JOIN employees m ON e.manager_id = m.id;

-- Result:
-- Tushar  | NULL    (CEO, no manager)
-- Aman    | Tushar
-- Priya   | Tushar
-- Rahul   | Aman
```

**When to use:** Hierarchical data (org charts, category trees), comparing rows within the same table.

**7. NATURAL JOIN** — Auto-matches columns with same name

```sql
SELECT * FROM users NATURAL JOIN orders;
-- Joins on columns that share the same name (e.g., 'id' = 'id')
-- ⚠️ DANGEROUS: fragile if column names change. Avoid in production.
```

**8. LATERAL JOIN (PostgreSQL)** — Subquery that references the outer table

```sql
-- Get each user's top 3 most recent orders (top-N per group)
SELECT u.name, recent.*
FROM users u
LEFT JOIN LATERAL (
    SELECT order_id, amount, created_at
    FROM orders o
    WHERE o.user_id = u.id
    ORDER BY created_at DESC
    LIMIT 3
) recent ON true;
```

**When to use:** Top-N per group, correlated subqueries that need to return multiple rows. Very powerful in PostgreSQL.

**JOIN performance visual cheat sheet:**

```
INNER JOIN:      LEFT JOIN:       FULL OUTER JOIN:    CROSS JOIN:
  ┌───┐            ┌───┐            ┌───┐             ┌───┐
 ╔│ A ┼╗          ╔│ A │╗          ╔│ A │╗            │ A │ × │ B │
 ║└─┬─┘║         ╔╝└─┬─┘║         ╔╝└─┬─┘╗           └───┘   └───┘
 ║ ╔╧╗  ║        ║  ╔╧╗  ║        ║  ╔╧╗  ║          = |A| × |B| rows
 ║ ║●║  ║        ║  ║●║  ║        ║  ║●║  ║
 ║ ╚╤╝  ║        ║  ╚╤╝ ╔╝        ╚╗ ╚╤╝ ╔╝
 ║┌─┴─┐║         ║┌─┴─┐╝          ╚┌─┴─┐╝
 ╚│ B ┼╝         ║│ B │            ║│ B │║
  └───┘           └───┘             └───┘
  Only            All A,           All A,
  overlap         overlap+NULL     All B,
                                   overlap
```

---

#### Locks — Deep Dive

| Lock Type              | Scope               | Use case                                                          |
| ---------------------- | ------------------- | ----------------------------------------------------------------- |
| **Row-level lock**     | Single row          | `SELECT ... FOR UPDATE` — lock specific rows during a transaction |
| **Table-level lock**   | Entire table        | DDL operations, bulk operations                                   |
| **Shared lock (S)**    | Multiple readers OK | `SELECT ... FOR SHARE` — others can read but not write            |
| **Exclusive lock (X)** | One writer only     | `UPDATE`, `DELETE` — blocks all other access                      |
| **Advisory lock**      | Application-defined | Custom mutual exclusion (`pg_advisory_lock`)                      |

**Lock compatibility matrix:**

```
             Requesting Lock
             Shared(S)   Exclusive(X)
Held Lock:
  None        ✅ Grant     ✅ Grant
  Shared(S)   ✅ Grant     ❌ Wait
  Exclusive(X) ❌ Wait     ❌ Wait

Key insight: Shared locks are compatible with each other (multiple readers OK)
             Exclusive locks conflict with everything (one writer only)
```

**Optimistic vs Pessimistic Locking:**

```sql
-- Pessimistic: Lock the row, then update
-- Use when: conflicts are COMMON (e.g., inventory with high concurrency)
BEGIN;
SELECT * FROM products WHERE id = 1 FOR UPDATE;  -- locks the row
UPDATE products SET stock = stock - 1 WHERE id = 1;
COMMIT;
-- Other transactions WAIT until this one finishes

-- Optimistic: Use a version column, no lock
-- Use when: conflicts are RARE (e.g., editing a user profile)
UPDATE products
SET stock = stock - 1, version = version + 1
WHERE id = 1 AND version = 5;
-- Returns "0 rows affected" if version changed → app retries

-- SELECT ... FOR UPDATE SKIP LOCKED (queue processing pattern)
-- Use when: multiple workers processing tasks from a table
BEGIN;
SELECT * FROM tasks
WHERE status = 'pending'
ORDER BY created_at
LIMIT 1
FOR UPDATE SKIP LOCKED;  -- skip rows locked by other workers
-- process the task...
UPDATE tasks SET status = 'done' WHERE id = ?;
COMMIT;
```

---

#### Deadlocks — What, Why, and Prevention

**What is a deadlock?**
Two (or more) transactions are stuck waiting for each other to release locks, creating a circular dependency. Neither can proceed.

```
Timeline of a deadlock:

Transaction A                    Transaction B
─────────────                    ─────────────
BEGIN;                           BEGIN;
UPDATE accounts SET balance =    UPDATE products SET stock =
  balance - 500                    stock - 1
  WHERE id = 1;                    WHERE id = 42;
  ✅ Locks row accounts.id=1       ✅ Locks row products.id=42

UPDATE products SET stock =      UPDATE accounts SET balance =
  stock - 1                        balance + 500
  WHERE id = 42;                   WHERE id = 1;
  ⏳ WAITING for B's lock          ⏳ WAITING for A's lock
      on products.id=42                on accounts.id=1

💀 DEADLOCK! A waits for B, B waits for A. Circular dependency.
```

**How databases handle deadlocks:**

- PostgreSQL/MySQL run a **deadlock detector** periodically.
- When detected, one transaction is chosen as the **victim** and rolled back with an error.
- PostgreSQL: `ERROR: deadlock detected` (error code `40P01`).
- MySQL/InnoDB: checks every time a lock wait occurs (via a wait-for graph).

**Deadlock Prevention Techniques:**

**1. Consistent Lock Ordering (Most Important!)**
Always lock resources in the **same global order** across all transactions.

```sql
-- BAD: Txn A locks accounts first, Txn B locks products first
-- Txn A: UPDATE accounts ... → UPDATE products ...
-- Txn B: UPDATE products ... → UPDATE accounts ...
-- 💀 Deadlock possible!

-- GOOD: ALWAYS lock in alphabetical/ID order: accounts THEN products
-- Txn A: UPDATE accounts ... → UPDATE products ...
-- Txn B: UPDATE accounts ... → UPDATE products ...   ← same order!
-- Txn B waits for A to finish accounts, then proceeds. No cycle possible.
```

**Interview explanation:** "If every transaction acquires locks in the same order, circular waits become impossible. Like a rule that everyone enters doors in alphabetical order — no two people will block each other."

**2. Lock Timeout (`lock_timeout`)**
Don't wait forever — give up after a timeout and retry.

```sql
-- PostgreSQL
SET lock_timeout = '5s';  -- give up waiting after 5 seconds
BEGIN;
SELECT * FROM products WHERE id = 1 FOR UPDATE;
-- If locked by another txn for >5s → ERROR: canceling statement due to lock timeout
COMMIT;

-- In application code:
try:
    execute_transaction()
except LockTimeoutError:
    retry_with_backoff()
```

**3. Use `NOWAIT` or `SKIP LOCKED`**

```sql
-- NOWAIT: fail immediately if row is locked
SELECT * FROM products WHERE id = 1 FOR UPDATE NOWAIT;
-- ERROR instantly if locked (no waiting at all)

-- SKIP LOCKED: skip locked rows, process unlocked ones
SELECT * FROM tasks WHERE status = 'pending'
FOR UPDATE SKIP LOCKED LIMIT 10;
-- Perfect for worker queues — each worker grabs different rows
```

**4. Keep Transactions Short**
The longer a transaction holds locks, the higher the deadlock probability.

```python
# BAD: Long transaction holding locks
async def bad_checkout(order):
    async with db.transaction():
        await db.execute("UPDATE inventory ... FOR UPDATE")
        await call_payment_api(order)        # ⚠️ External API call inside txn!
        await call_shipping_api(order)       # ⚠️ Another external call!
        await db.execute("UPDATE orders ...")
    # Locks held for seconds while waiting on external APIs 💀

# GOOD: Minimize lock duration
async def good_checkout(order):
    # Step 1: Do external calls OUTSIDE the transaction
    payment = await call_payment_api(order)
    shipping = await call_shipping_api(order)

    # Step 2: Short transaction for DB writes only
    async with db.transaction():
        await db.execute("UPDATE inventory ...")
        await db.execute("INSERT INTO orders ...")
    # Locks held for milliseconds, not seconds ✅
```

**5. Use Optimistic Locking Where Possible**
No locks at all → no deadlocks possible.

```sql
-- Version-based optimistic locking
UPDATE products
SET stock = stock - 1, version = version + 1
WHERE id = 42 AND version = 7;

-- If 0 rows affected → someone else updated it → retry
-- No locks, no waits, no deadlocks. Just retries.
```

**6. Reduce Lock Granularity**
Lock only what you need. Row locks > table locks.

```sql
-- BAD: Locks entire table
LOCK TABLE products IN EXCLUSIVE MODE;

-- GOOD: Lock only the row you need
SELECT * FROM products WHERE id = 42 FOR UPDATE;
```

**7. Deadlock Retry Pattern (Application Level)**

```python
import asyncio
from asyncpg.exceptions import DeadlockDetectedError

MAX_RETRIES = 3

async def execute_with_retry(db, operation):
    for attempt in range(MAX_RETRIES):
        try:
            async with db.transaction():
                return await operation()
        except DeadlockDetectedError:
            if attempt == MAX_RETRIES - 1:
                raise  # give up after max retries
            wait = (2 ** attempt) * 0.1  # exponential backoff: 0.1s, 0.2s, 0.4s
            await asyncio.sleep(wait + random.uniform(0, 0.1))  # + jitter
```

**Deadlock prevention summary for interviews:**

```
Technique                  | When to use
───────────────────────────┼──────────────────────────────────────
Consistent lock ordering   | ALWAYS (design principle)
Lock timeout               | When you can tolerate retries
NOWAIT / SKIP LOCKED       | Worker queues, non-blocking patterns
Short transactions         | ALWAYS (minimize lock hold time)
Optimistic locking         | Low-conflict scenarios (profiles, configs)
Retry with backoff         | ALWAYS have this in your app layer
Advisory locks             | Application-level coordination (cron jobs)
```

---

#### Indexing

| Index Type           | How it works                         | Best for                                                     |
| -------------------- | ------------------------------------ | ------------------------------------------------------------ |
| **B-Tree** (default) | Balanced tree structure              | Equality (`=`), range (`<`, `>`), sorting, `LIKE 'abc%'`     |
| **Hash**             | Hash table                           | Exact equality only (`=`) — faster than B-tree for this      |
| **GIN**              | Generalized inverted index           | Full-text search, JSONB, arrays                              |
| **GiST**             | Generalized search tree              | Geometric data, ranges, full-text                            |
| **BRIN**             | Block range index                    | Very large tables with naturally ordered data (timestamps)   |
| **Composite**        | Multi-column B-tree                  | Queries filtering on multiple columns (leftmost prefix rule) |
| **Partial**          | Index with a WHERE clause            | Only index rows matching a condition                         |
| **Covering**         | Includes extra columns via `INCLUDE` | Index-only scans (no table lookup)                           |

```sql
-- Composite index: works for queries on (status), (status, created_at), NOT (created_at) alone
CREATE INDEX idx_orders_status_date ON orders(status, created_at);

-- Partial index: only index active users (smaller, faster)
CREATE INDEX idx_active_users ON users(email) WHERE is_active = true;

-- Covering index: avoid table lookup entirely
CREATE INDEX idx_orders_cover ON orders(user_id) INCLUDE (total, status);
```

**Leftmost prefix rule (critical for composite indexes):**

```sql
-- Given index: CREATE INDEX idx ON orders(a, b, c);
-- ✅ Uses index: WHERE a = 1
-- ✅ Uses index: WHERE a = 1 AND b = 2
-- ✅ Uses index: WHERE a = 1 AND b = 2 AND c = 3
-- ❌ Cannot use index: WHERE b = 2 (skips 'a')
-- ❌ Cannot use index: WHERE c = 3 (skips 'a' and 'b')
-- ⚠️ Partial use: WHERE a = 1 AND c = 3 (uses index for 'a', then scans for 'c')
```

**When NOT to index:**

- Small tables (full scan is faster than index lookup)
- Columns with very low cardinality (e.g., boolean `is_active` — unless used in a partial index)
- Tables with heavy writes and rare reads (indexes slow down INSERT/UPDATE/DELETE)
- Columns rarely used in WHERE, JOIN, or ORDER BY

**`EXPLAIN ANALYZE`** — Your best friend for understanding query plans:

```sql
EXPLAIN ANALYZE SELECT * FROM orders WHERE status = 'pending' AND created_at > '2026-01-01';
-- Shows: Seq Scan vs Index Scan, estimated vs actual rows, execution time

-- What to look for:
-- ✅ "Index Scan" or "Index Only Scan" = index is being used
-- ⚠️ "Bitmap Heap Scan" = index used but also hitting table (OK for many rows)
-- ❌ "Seq Scan" on a large table = missing index or query can't use existing ones
-- ❌ "Sort" with high cost = consider adding index for ORDER BY column
-- Key metric: "actual time" and "rows" — compare estimated vs actual
```

### 📌 TLDR for Interviews

> "Normalization (1NF→BCNF) eliminates data redundancy; denormalize for read-heavy workloads. **JOINs**: INNER = only matches, LEFT = all from left + matches, FULL OUTER = everything from both, CROSS = cartesian product, LATERAL = correlated subquery per row. **Deadlock prevention**: always lock in consistent order, keep transactions short, use `NOWAIT`/`SKIP LOCKED`, implement retry with exponential backoff, prefer optimistic locking for low-conflict scenarios. **Locking**: pessimistic (`FOR UPDATE`) when conflicts are common, optimistic (version column) when rare. **Indexing**: B-Tree covers most cases; GIN for JSONB/full-text, BRIN for time-series. Composite indexes follow the leftmost prefix rule. Always validate with `EXPLAIN ANALYZE`."

---

### 5. Database Views — Virtual Tables & Materialized Views

> **📣 Definition:** _"A view is a saved SQL query that acts like a virtual table — it doesn't store data, it re-executes the query every time you SELECT from it. A materialized view stores the query result physically on disk and must be refreshed manually or on a schedule. Views simplify complex queries, enforce security (expose only certain columns), and provide stable APIs over changing schemas. Materialized views trade freshness for speed — use them for expensive aggregations and dashboards."_

```sql
-- === Regular View (virtual — no data stored) ===
CREATE VIEW active_users AS
SELECT id, email, full_name, last_login
FROM users
WHERE is_active = true AND last_login > NOW() - INTERVAL '30 days';

-- Use it like a table:
SELECT * FROM active_users WHERE email LIKE '%@company.com';
-- Every query re-executes the underlying SELECT (always fresh, but can be slow)

-- === Materialized View (physical — data stored on disk) ===
CREATE MATERIALIZED VIEW monthly_sales AS
SELECT
    DATE_TRUNC('month', created_at) AS month,
    product_id,
    SUM(amount) AS total_sales,
    COUNT(*) AS order_count
FROM orders
GROUP BY month, product_id;

-- Query is instant (reads stored data):
SELECT * FROM monthly_sales WHERE month = '2026-04-01';

-- But data is STALE until refreshed:
REFRESH MATERIALIZED VIEW monthly_sales;                -- blocks reads during refresh
REFRESH MATERIALIZED VIEW CONCURRENTLY monthly_sales;   -- no blocking (needs unique index)
-- Schedule refresh via pg_cron or Celery Beat


-- === View for Security (column-level access control) ===
-- Expose only non-sensitive columns:
CREATE VIEW public_users AS
SELECT id, full_name, avatar_url, created_at
FROM users;
-- GRANT SELECT ON public_users TO readonly_role;
-- The `readonly_role` can't see email, password_hash, phone, etc.


-- === Updatable Views (with restrictions) ===
-- Simple views (single table, no aggregation) can be updated:
CREATE VIEW draft_products AS
SELECT * FROM products WHERE status = 'draft';

INSERT INTO draft_products (name, price, status) VALUES ('New', 9.99, 'draft');  -- ✅ works
UPDATE draft_products SET price = 12.99 WHERE id = 5;                            -- ✅ works
-- Inserts/updates go to the underlying `products` table
```

**Comparison:**

| Aspect            | Regular View                         | Materialized View                           |
| ----------------- | ------------------------------------ | ------------------------------------------- |
| **Data stored?**  | ❌ No (re-executes query)            | ✅ Yes (on disk)                            |
| **Always fresh?** | ✅ Yes                               | ❌ No (stale until refreshed)               |
| **Speed**         | Same as underlying query             | Fast (pre-computed)                         |
| **Indexable?**    | ❌ No                                | ✅ Yes (can create indexes on it)           |
| **Use for**       | Simplification, security, stable API | Dashboards, reports, expensive aggregations |
| **Refresh**       | Automatic (every query)              | Manual: `REFRESH MATERIALIZED VIEW`         |

**In Django:**

```python
# Django doesn't natively support views, but you can:
# 1. Create the view via a migration:
class Migration(migrations.Migration):
    operations = [
        migrations.RunSQL(
            sql="CREATE VIEW active_users AS SELECT ...",
            reverse_sql="DROP VIEW active_users",
        ),
    ]

# 2. Map an unmanaged model to the view:
class ActiveUser(models.Model):
    email = models.EmailField()
    full_name = models.CharField(max_length=200)
    last_login = models.DateTimeField()

    class Meta:
        managed = False           # Django won't create/alter this table
        db_table = "active_users" # maps to the view

# 3. Query it like any model:
ActiveUser.objects.filter(email__endswith="@company.com")
```

📌 **TLDR:** "A view = saved query acting as a virtual table (always fresh, but re-executes every time). Materialized view = cached query result on disk (fast reads, but stale until `REFRESH`). Use views for simplification and security. Use materialized views for expensive aggregations/dashboards with scheduled refresh. In Django, create via `RunSQL` migration + unmanaged model."

---

# Most Likely Questions

These are the exact phrasings interviewers use. Practice answering each in 60-90 seconds.

### Databases

**Q: "Two transactions are deadlocking on your orders table. How do you fix it?"**
A: First, identify the lock pattern from logs — usually two transactions acquiring the same rows in different orders. Fix at the design level: enforce a consistent locking order across all code paths (e.g., always lock by ascending ID). Add lock_timeout to fail fast, application-level retry with exponential backoff, and consider optimistic locking if conflicts are rare.

**Q: "Your query is slow. Walk me through your debugging process."**
A: Run `EXPLAIN ANALYZE`. Check for: (1) Seq Scan on a large table → missing index. (2) Estimated rows ≠ actual rows → run `ANALYZE` to refresh statistics. (3) Sort with high cost → add index on ORDER BY column. (4) Nested Loop on millions of rows → likely needs a Hash Join or Merge Join, often fixable by adding indexes on JOIN keys. Then check connection pooling, query caching, and whether the workload should be denormalized.

**Q: "When would you use a LEFT JOIN vs an INNER JOIN?"**
A: INNER JOIN when you only want rows that exist in both tables — most common case. LEFT JOIN when you want every row from the left side regardless of matches — typical for reports like "all users with their order count" where you need to include users who haven't ordered yet. The classic anti-pattern check: `LEFT JOIN ... WHERE right.col IS NULL` finds orphaned records.

**Q: "What isolation level would you use for an e-commerce checkout?"**
A: Read Committed (PostgreSQL default) is fine for most operations. For inventory decrement specifically, I'd use `SELECT ... FOR UPDATE` to take a row lock on the product, ensuring no two checkouts oversell. If using Serializable, the database enforces it but you need retry logic for serialization failures.
