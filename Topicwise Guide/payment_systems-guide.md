# Payment Systems (Interview Specialty)

## 📑 Table of Contents

1. [1. Payment Systems — Concepts Interviewers Drill On](#1-payment-systems--concepts-interviewers-drill-on)
2. [2. The Failure Modes That Cause Double Charges](#2-the-failure-modes-that-cause-double-charges)
3. [3. Idempotency — Beyond the Buzzword (How Stripe & Razorpay Actually Do It)](#3-idempotency--beyond-the-buzzword-how-stripe--razorpay-actually-do-it)
4. [4. The Payment State Machine — Why No Record Is Ever Updated](#4-the-payment-state-machine--why-no-record-is-ever-updated)
5. [5. Double-Entry Bookkeeping & The Ledger](#5-double-entry-bookkeeping--the-ledger)
6. [6. The Authorize-then-Capture Flow — Two-Phase Payments](#6-the-authorize-then-capture-flow--two-phase-payments)
7. [7. Webhooks — The Async Confirmation Channel](#7-webhooks--the-async-confirmation-channel)
8. [8. Reconciliation — The Safety Net Against Everything](#8-reconciliation--the-safety-net-against-everything)
9. [9. Distributed Transaction Routing & Deduplication at Scale](#9-distributed-transaction-routing--deduplication-at-scale)
10. [10. Money Representation — Never Use Float](#10-money-representation--never-use-float)
11. [11. Refunds, Partial Refunds & Chargebacks](#11-refunds-partial-refunds--chargebacks)
12. [12. Payment System Interview Q&A](#12-payment-system-interview-qa)
13. [13. 📌 The Master TLDR for Payment Systems](#13--the-master-tldr-for-payment-systems)

## 1. Payment Systems — Concepts Interviewers Drill On

> **📣 Interview-ready definition:** _"Payment systems are distributed systems where money is an invariant — duplicates and partial failures matter infinitely more than latency. Because true exactly-once delivery is impossible in distributed systems, real payment systems combine **at-least-once delivery + idempotency + reconciliation** to achieve effective exactly-once. The core building blocks are: idempotency keys at multiple layers (gateway, app, PSP), an immutable state machine for payment lifecycle (created → authorized → captured → settled), double-entry bookkeeping in an append-only ledger (debits always equal credits), webhooks for async PSP confirmations, and continuous reconciliation against PSP and bank statements. Money is stored as BIGINT in smallest currency units (paise/cents) — never as float."_

Payment systems are a special category of interview because **money is an invariant** — duplicates and partial failures are infinitely worse than latency. Interviewers love this domain because it forces you to reason about distributed failure modes, state machines, and consistency guarantees in ways that "design Twitter" doesn't.

The single most important sentence to internalize: **Exactly-once delivery is a myth in distributed systems. We achieve effective exactly-once via at-least-once delivery + idempotency + reconciliation.**

---

### 2. The Failure Modes That Cause Double Charges

> **📣 Definition:** _"There are exactly three failure modes that can lead to a customer being double-charged. **(1) Pre-server failure** — the request never reaches the server (safe to retry naively). **(2) Mid-processing failure** — request reaches the server, the PSP charged the card, but the server crashed before writing to the DB; client retries → double charge. **(3) Post-processing failure** — everything succeeded but the response was lost in transit; client retries → double charge. Saying 'we use idempotency' without naming these three sounds junior; leading with the failure modes signals senior."_

Before discussing solutions, an interviewer wants to know **you understand the actual failure modes**. There are three:

```
1. PRE-SERVER FAILURE (request never reaches server)
   Client ─[network drop]─X    Server
   ✅ Safe to retry naively — server never processed anything.

2. MID-PROCESSING FAILURE (request arrives, processing starts, fails midway)
   Client ──→ Server ──→ PSP (Stripe) ✅ charge succeeded
                ↓
              💥 server crashed before writing to DB
              ↓
              Client times out → retries
   ❌ Without idempotency: customer charged twice (PSP doesn't know it's a retry)

3. POST-PROCESSING FAILURE (success on server, response lost in transit)
   Client ──→ Server ──→ PSP ✅ charged ──→ DB ✅ written ──→ Response ─[network drop]─X
   Client times out → retries
   ❌ Without idempotency: customer charged twice + two DB rows
```

**The customer's experience is always the same: "I clicked Pay, got an error, clicked Pay again."** Your job is to make that retry safe.

📌 **Interviewer test:** If you say "we use idempotency" without naming these three failure modes, you sound junior. Lead with the failure modes, then walk through the solution.

---

### 3. Idempotency — Beyond the Buzzword (How Stripe & Razorpay Actually Do It)

> **📣 Definition:** _"Production payment systems enforce idempotency at three layers, not one. **Layer 1** — API gateway/edge: Redis cache for fast retries, returns cached response on hit. **Layer 2** — payment service: Postgres UNIQUE constraint on idempotency_keys table, inserted in the same transaction as the business write so it's atomic and durable. **Layer 3** — PSP: forward the same idempotency key to Stripe/Razorpay so even if our retry logic fires, the external charge is deduped. The key is generated client-side per logical attempt (UUID), survives retries, includes a request fingerprint to detect client-side reuse with different params, and **caches failures** as well as successes to prevent silent state divergence."_

Yes, you mentioned not to "just say idempotency" — and rightly so. The interviewer wants the **actual mechanics**.

#### The Three-Layer Idempotency Stack

Production payment systems enforce idempotency at THREE layers, not one:

```
┌─────────────────────────────────────────────────────────┐
│  Layer 1: API Gateway / Edge                           │
│  • Client sends Idempotency-Key (UUIDv4) per request   │
│  • Gateway checks Redis cache for recent keys           │
│  • Returns cached response if hit                       │
│  • Purpose: cut down obvious retries cheaply            │
└─────────────────────────────────────────────────────────┘
                         ↓
┌─────────────────────────────────────────────────────────┐
│  Layer 2: Payment Service (durable check)              │
│  • Inserts row into idempotency_keys table              │
│  • UNIQUE constraint on key column                      │
│  • If insert fails (key exists) → return stored result  │
│  • Purpose: durable dedup even if Redis is wiped        │
└─────────────────────────────────────────────────────────┘
                         ↓
┌─────────────────────────────────────────────────────────┐
│  Layer 3: PSP (Stripe/Razorpay) call                   │
│  • Forward the SAME idempotency key to the PSP          │
│  • Stripe caches the response for 24h with that key     │
│  • Even if our service retries, PSP never double-charges│
│  • Purpose: bulletproof external safety net             │
└─────────────────────────────────────────────────────────┘
```

#### Stripe's actual contract (memorize this)

1. Client generates a UUID per **logical** payment attempt (NOT per HTTP request — the same UUID survives retries).
2. Send it as an HTTP header: `Idempotency-Key: <uuid>`
3. Stripe caches the **full response** (success, failure, validation error) for **24 hours** keyed on that UUID.
4. Sending the same key with **different params** returns 400 — keys are commitments, not labels.
5. Stripe also caches **failures**. If your first attempt got a 500, retries with the same key return that same 500. Why? Because if your code "thinks" the call failed but it actually succeeded, the consistent failure-on-retry forces you to investigate, instead of letting state silently diverge.

Razorpay uses the same pattern with the `x-idempotency-key` header.

#### The actual implementation — Postgres approach (interviewer gold)

```sql
-- The idempotency table — co-located with payment data so it's in the same transaction
CREATE TABLE idempotency_keys (
    key VARCHAR(255) PRIMARY KEY,
    request_fingerprint VARCHAR(64) NOT NULL,    -- SHA256 of request body
    response_status INT,
    response_body JSONB,
    payment_id BIGINT REFERENCES payments(id),
    state VARCHAR(20) NOT NULL,                   -- 'in_progress' | 'completed'
    locked_at TIMESTAMP,
    expires_at TIMESTAMP NOT NULL,
    created_at TIMESTAMP DEFAULT NOW()
);

CREATE INDEX idx_idempotency_expires ON idempotency_keys(expires_at);
```

```python
import hashlib, json
from django.db import transaction, IntegrityError

def process_payment(idempotency_key: str, request_body: dict):
    fingerprint = hashlib.sha256(
        json.dumps(request_body, sort_keys=True).encode()
    ).hexdigest()

    # ─── Step 1: Try to claim the key (atomic insert) ────────────
    try:
        with transaction.atomic():
            record = IdempotencyKey.objects.create(
                key=idempotency_key,
                request_fingerprint=fingerprint,
                state='in_progress',
                locked_at=timezone.now(),
                expires_at=timezone.now() + timedelta(hours=24),
            )
    except IntegrityError:
        # ─── Step 2: Key already exists — handle the 4 cases ─────
        existing = IdempotencyKey.objects.get(key=idempotency_key)

        # Case A: same key, DIFFERENT request → reject (Stripe behavior)
        if existing.request_fingerprint != fingerprint:
            raise HTTPError(422, "Idempotency key reused with different params")

        # Case B: completed → return cached response
        if existing.state == 'completed':
            return existing.response_body, existing.response_status

        # Case C: in_progress + recent lock → another worker is on it
        if existing.locked_at > timezone.now() - timedelta(seconds=30):
            raise HTTPError(409, "Request still in progress, retry later")

        # Case D: in_progress + stale lock → recover (worker crashed)
        # Update lock time and proceed (or query PSP to see what happened)
        existing.locked_at = timezone.now()
        existing.save()
        record = existing

    # ─── Step 3: Do the actual work ──────────────────────────────
    try:
        # CRITICAL: pass same key to PSP so it dedups too (3rd layer!)
        psp_response = stripe.PaymentIntent.create(
            amount=request_body['amount'],
            currency='inr',
            idempotency_key=idempotency_key,   # ← forward to Stripe
        )

        payment = Payment.objects.create(
            psp_reference=psp_response.id,
            amount=request_body['amount'],
            status='authorized',
        )

        # ─── Step 4: Finalize the idempotency record ─────────────
        record.state = 'completed'
        record.response_status = 200
        record.response_body = {'payment_id': payment.id, 'status': 'authorized'}
        record.payment_id = payment.id
        record.save()

        return record.response_body, 200

    except Exception as e:
        # Don't delete the key — Stripe-style: cache the failure too
        record.state = 'completed'
        record.response_status = 500
        record.response_body = {'error': str(e)}
        record.save()
        raise
```

**Key design decisions to call out in interviews:**

1. **`UNIQUE` constraint on `key`** — the database is the source of truth, not application logic. If two threads try to insert the same key simultaneously, **exactly one wins** at the DB level. No race condition possible.

2. **Request fingerprint** — guards against client bugs reusing keys with different bodies. Stripe rejects with 400/422 in this case.

3. **`locked_at` field** — handles the case where the first attempt crashed mid-processing. Without this, the key would be "stuck" forever in `in_progress`.

4. **Cache failures, not just successes** — counter-intuitive but critical. Prevents silent state divergence between client and server.

5. **Same key forwarded to PSP** — even if your retry logic fires twice, Stripe deduplicates. Three layers of safety.

#### Redis vs Postgres for Idempotency Storage

| Aspect                        | Redis                              | Postgres                  |
| ----------------------------- | ---------------------------------- | ------------------------- |
| Latency                       | ~1ms                               | ~5-10ms                   |
| Durability                    | Memory-first (configurable AOF)    | Disk + WAL                |
| Atomicity with business write | Separate system → split brain risk | Same transaction → atomic |
| TTL handling                  | Native                             | Cron job needed           |
| **Verdict for payments**      | Layer 1 cache only                 | **Source of truth**       |

> **Interviewer follow-up: "Why not just use Redis?"**
> Answer: "Redis is great for the gateway layer to short-circuit retries cheaply. But for the durable layer, Postgres lets us insert the idempotency record + business data in the SAME transaction — atomicity that Redis can't give us across system boundaries. If Redis crashes and loses keys, we're protected by the DB. The trade-off is ~5ms of latency, which is acceptable for payments where correctness > speed."

📌 **TLDR for interviews:** "Idempotency in payments is a three-layer stack: edge cache (Redis) for cheap dedup, payment service (Postgres unique constraint, same transaction as business write) for durable dedup, and the PSP idempotency key as the final safety net. The key is generated client-side per logical attempt (not per HTTP request), survives retries, and includes a request fingerprint to catch client-side key reuse with different params. Cache failures, not just successes."

---

### 4. The Payment State Machine — Why No Record Is Ever Updated

> **📣 Definition:** _"A payment is a **state machine, not a row**. Use two tables: a `payments` table with the current denormalized status (for fast reads), and an append-only `payment_events` table that records every state transition as an immutable row. Status flow: `created → processing → authorized → captured → settled` (with `failed`, `cancelled`, `refunded` as alternative paths). Enforce valid transitions in code — you can't refund a payment that was never captured. The append-only log gives you a perfect audit trail for chargebacks and disputes, and lets you rebuild current state if denormalization drifts."_

The second most-asked question after idempotency: "How do you model a payment?"

The wrong answer: a single row that gets UPDATEd as status changes. The right answer: an **immutable state machine + event log**.

#### The 7 (or so) Canonical States

```
                                   ┌──────────────┐
                                   │   CREATED    │
                                   │ (intent only)│
                                   └──────┬───────┘
                                          │ user submits
                                          ▼
                                   ┌──────────────┐
                                   │  PROCESSING  │
                                   │ (sent to PSP)│
                                   └──┬─────────┬─┘
                          PSP success │         │ PSP failure
                                      ▼         ▼
                              ┌─────────────┐  ┌──────────────┐
                              │ AUTHORIZED  │  │   FAILED     │
                              │ (funds held)│  └──────────────┘
                              └──────┬──────┘
                       merchant/auto │
                       capture       │
                                     ▼
                              ┌──────────────┐
                              │  CAPTURED    │
                              │ (we got $$)  │
                              └──────┬───────┘
                       T+1 settle   │
                                    ▼
                              ┌──────────────┐
                              │   SETTLED    │
                              │(in our bank) │
                              └──────┬───────┘
                                    │ refund requested
                                    ▼
                              ┌──────────────┐
                              │   REFUNDED   │
                              └──────────────┘

Special branch: any captured payment can become DISPUTED → CHARGEBACK_LOST
```

#### Why never UPDATE — append-only event log

```sql
-- ❌ WRONG: mutable status column
CREATE TABLE payments (
    id BIGINT PRIMARY KEY,
    status VARCHAR(20),       -- gets overwritten on each transition
    amount NUMERIC(20, 4)
);
-- Problem: if there's a bug, you've lost history. Audit impossible.

-- ✅ RIGHT: payment + immutable event log
CREATE TABLE payments (
    id BIGINT PRIMARY KEY,
    amount BIGINT NOT NULL,           -- in smallest currency unit (paise/cents)
    currency CHAR(3) NOT NULL,
    customer_id BIGINT NOT NULL,
    psp_reference VARCHAR(255) UNIQUE,
    -- current_status is a DERIVED field — denormalized for fast reads
    current_status VARCHAR(20) NOT NULL,
    created_at TIMESTAMP NOT NULL DEFAULT NOW()
);

CREATE TABLE payment_events (
    id BIGSERIAL PRIMARY KEY,
    payment_id BIGINT NOT NULL REFERENCES payments(id),
    event_type VARCHAR(50) NOT NULL,    -- 'authorized', 'captured', 'failed', 'refunded'
    from_state VARCHAR(20),
    to_state VARCHAR(20) NOT NULL,
    metadata JSONB,                      -- PSP response, error codes, user_id, etc.
    created_at TIMESTAMP NOT NULL DEFAULT NOW(),
    -- THIS table is APPEND-ONLY. No UPDATE, no DELETE, ever.
    CONSTRAINT no_updates CHECK (true)
);
CREATE INDEX idx_events_payment ON payment_events(payment_id, created_at);
```

#### Enforce valid transitions in code

```python
# Define the allowed transitions
VALID_TRANSITIONS = {
    'created':     {'processing', 'cancelled'},
    'processing':  {'authorized', 'failed'},
    'authorized':  {'captured', 'cancelled', 'expired'},
    'captured':    {'settled', 'refund_pending'},
    'settled':     {'refund_pending', 'disputed'},
    'failed':      set(),         # terminal
    'cancelled':   set(),         # terminal
    'refunded':    set(),         # terminal
}

class InvalidTransition(Exception):
    pass

@transaction.atomic
def transition_payment(payment_id: int, to_state: str, metadata: dict):
    payment = Payment.objects.select_for_update().get(id=payment_id)

    if to_state not in VALID_TRANSITIONS[payment.current_status]:
        raise InvalidTransition(
            f"Cannot go from {payment.current_status} → {to_state}"
        )

    # 1. Append the immutable event
    PaymentEvent.objects.create(
        payment_id=payment.id,
        event_type=to_state,
        from_state=payment.current_status,
        to_state=to_state,
        metadata=metadata,
    )

    # 2. Update the denormalized current state
    payment.current_status = to_state
    payment.save()

    # 3. Publish via outbox pattern (see 16.6)
```

**Why interviewers love this design:**

- Full audit trail for chargebacks, disputes, debugging
- State can be **rebuilt from events** if `current_status` ever drifts (event sourcing)
- Invalid transitions blocked at code level — can't refund a payment that was never captured
- Each event is timestamped → you can answer "when was this payment authorized?" precisely

📌 **TLDR for interviews:** "A payment is a state machine, not a row. Use an append-only `payment_events` table to record every transition, with a denormalized `current_status` on the payment row for fast reads. Enforce valid transitions in code. This gives you a perfect audit trail for chargebacks and disputes, and lets you rebuild state if the denormalized field ever drifts."

---

### 5. Double-Entry Bookkeeping & The Ledger

> **📣 Definition:** _"Double-entry bookkeeping is the accounting principle every serious payment system uses: every transaction creates matched **debit** and **credit** entries across multiple accounts, and the invariant `SUM(debits) = SUM(credits)` must always hold. The ledger is append-only — never UPDATE or DELETE. Continuous reconciliation jobs verify the invariant; if it ever drifts, you have a bug worth waking the on-call engineer at 3 AM. The ledger is the only system that can't be rebuilt from anything else; everything else can be reconstructed from it. This is how Stripe processes billions without losing pennies — if you don't mention this in a payments interview, you're not signaling senior."_

> "Double-entry bookkeeping is not optional — it's how you detect bugs before they cost millions." — Stripe staff engineer

This is the secret sauce of every serious payment system. Coming from a Stripe interview perspective, if you don't mention this, you're not a senior payments engineer.

#### The Core Invariant

For every transaction, **the sum of debits equals the sum of credits**. Always. No exceptions.

```
Customer pays ₹500 for a product:

┌────────────────────────────────────────────────────────────┐
│  Account                       │  Debit (₹) │  Credit (₹) │
├────────────────────────────────┼────────────┼─────────────┤
│  Customer's bank (their bank)  │            │     500     │
│  Our receivables (Razorpay)    │     500    │             │
│  Merchant payable              │            │     500     │
│  Our merchant-payout liability │     500    │             │
└────────────────────────────────┴────────────┴─────────────┘
                          SUM:        1000          1000  ← MUST be equal!
```

If `SUM(debits) ≠ SUM(credits)` for any time window → **you have a bug**. This is a real-time invariant check that runs continuously.

#### Schema

```sql
CREATE TABLE ledger_accounts (
    id BIGSERIAL PRIMARY KEY,
    name VARCHAR(100) NOT NULL UNIQUE,
    type VARCHAR(20) NOT NULL,      -- 'asset', 'liability', 'revenue', 'expense'
    -- Examples:
    -- ('Customer Receivables', 'asset')
    -- ('Merchant Payouts', 'liability')
    -- ('Processing Fees', 'revenue')
);

CREATE TABLE ledger_entries (
    id BIGSERIAL PRIMARY KEY,
    transaction_id UUID NOT NULL,           -- groups debits + credits of one event
    account_id BIGINT NOT NULL REFERENCES ledger_accounts(id),
    amount BIGINT NOT NULL,                  -- always positive
    direction CHAR(1) NOT NULL CHECK (direction IN ('D', 'C')),  -- D=debit, C=credit
    payment_id BIGINT REFERENCES payments(id),
    created_at TIMESTAMP NOT NULL DEFAULT NOW()
    -- IMMUTABLE: no UPDATE, no DELETE.
);

-- Critical index for the invariant check
CREATE INDEX idx_ledger_txn ON ledger_entries(transaction_id);
```

#### Posting a transaction (atomic)

```python
import uuid

@transaction.atomic
def post_payment_to_ledger(payment: Payment):
    txn_id = uuid.uuid4()

    entries = [
        # When customer pays ₹500
        LedgerEntry(
            transaction_id=txn_id,
            account_id=RECEIVABLES_ACCOUNT_ID,
            amount=payment.amount,
            direction='D',                # debit our receivables
            payment_id=payment.id,
        ),
        LedgerEntry(
            transaction_id=txn_id,
            account_id=MERCHANT_PAYABLE_ACCOUNT_ID,
            amount=payment.amount,
            direction='C',                # credit merchant payable
            payment_id=payment.id,
        ),
    ]

    # Verify the invariant BEFORE writing
    debits = sum(e.amount for e in entries if e.direction == 'D')
    credits = sum(e.amount for e in entries if e.direction == 'C')
    assert debits == credits, f"Imbalanced entries: D={debits} C={credits}"

    LedgerEntry.objects.bulk_create(entries)
```

#### The Continuous Invariant Check

A background job runs every few minutes (or on every commit, in extreme systems):

```sql
-- This MUST always return zero. If not, alert immediately.
SELECT
    SUM(CASE WHEN direction = 'D' THEN amount ELSE 0 END)
  - SUM(CASE WHEN direction = 'C' THEN amount ELSE 0 END)
    AS imbalance
FROM ledger_entries
WHERE created_at >= NOW() - INTERVAL '1 hour';
```

If the imbalance is ever non-zero → page the on-call engineer. This is how Stripe processes billions without losing pennies.

📌 **TLDR for interviews:** "Every payment creates matched debit + credit ledger entries. The invariant SUM(debits) = SUM(credits) must always hold. Continuous reconciliation jobs check this — if it ever drifts, you have a bug worth waking someone up at 3 AM. The ledger is append-only and is the ONLY rebuildable source of truth — every other system can be reconstructed from it."

---

### 6. The Authorize-then-Capture Flow — Two-Phase Payments

> **📣 Definition:** _"Card payments are two-phase: **authorize** holds funds on the card without transferring them (typically valid for 7-30 days), and **capture** actually transfers the money. This separation is essential for e-commerce (auth at checkout, capture on ship), hotels (auth a deposit, capture on checkout), and refund-friendly flows — voiding an auth shows no charge on the customer's statement, while capture-then-refund creates two visible entries. The classic interview question 'what if the server crashes between auth and DB write' is solved by **pull-based reconciliation**: a worker periodically queries the PSP for stuck payments and updates the DB. The PSP is the source of truth."_

Real payment systems (especially card networks) are **two-phase**:

1. **Authorize** — "Hold ₹500 on this card." Funds reserved but not transferred. Typically held for 7-30 days.
2. **Capture** — "Now actually take that ₹500." Money moves.

Why two phases?

- **E-commerce**: Authorize at checkout, capture only when order ships. If product is out of stock, void the auth — customer never charged.
- **Hotels/car rentals**: Authorize a "deposit" amount. Capture only the actual usage on checkout.
- **Refund handling**: An authorized-but-not-captured payment can be **voided** (no record on customer statement) instead of being captured-then-refunded (creates two confusing entries).

```python
# Step 1: Authorize at checkout
intent = stripe.PaymentIntent.create(
    amount=50000,
    currency='inr',
    payment_method='pm_xxx',
    capture_method='manual',          # ← key flag: don't auto-capture
    confirm=True,
    idempotency_key='order_123_auth',
)
# Funds are now held. payment.status = 'authorized'.

# ... later, when order ships ...

# Step 2: Capture
stripe.PaymentIntent.capture(
    intent.id,
    idempotency_key='order_123_capture',  # different key for the capture phase
)
# payment.status = 'captured', funds actually move.

# OR if the order is cancelled before shipping:
stripe.PaymentIntent.cancel(intent.id)
# Auth is released, customer sees no charge.
```

**The classic "half-completed transaction" problem (interview gold):**

> "What happens if the server crashes between authorizing with the PSP and writing to the database?"

This is the most-asked payment systems question. Here's the answer:

```
1. Server calls Stripe → authorization succeeds → funds held on customer card
2. 💥 Server crashes BEFORE writing payment row to DB
3. Database has NO record of the payment, but Stripe has authorized funds
4. Customer sees "₹500 hold" on their statement, panics
5. Customer retries → idempotency key prevents duplicate auth at Stripe
   But our DB still has nothing. We're stuck.
```

**The solution: pull-based reconciliation.**

```python
# Background worker runs every 5 minutes
def reconcile_stuck_authorizations():
    # Find idempotency records that "started" but never finished
    stuck = IdempotencyKey.objects.filter(
        state='in_progress',
        locked_at__lt=timezone.now() - timedelta(minutes=10),
    )

    for record in stuck:
        # Ask Stripe: "Did this actually succeed?"
        psp_response = stripe.PaymentIntent.list(
            metadata={'idempotency_key': record.key}
        )

        if psp_response.data:
            intent = psp_response.data[0]
            if intent.status == 'requires_capture':
                # Stripe authorized but we never recorded it. Fix our DB.
                Payment.objects.create(
                    psp_reference=intent.id,
                    amount=intent.amount,
                    status='authorized',
                )
                record.state = 'completed'
                record.save()
            elif intent.status == 'canceled':
                record.state = 'completed'
                record.response_body = {'error': 'cancelled by PSP'}
                record.save()
        else:
            # No record at Stripe → mark as failed
            record.state = 'completed'
            record.response_body = {'error': 'never reached PSP'}
            record.save()
```

**The principle:** the PSP is the source of truth. Our DB is eventually consistent with the PSP. Reconciliation closes the loop.

📌 **TLDR for interviews:** "Card payments are two-phase: authorize holds funds (refundable as a void with no customer-visible charge), capture transfers them. This separation handles cancellation gracefully. The half-completed-transaction problem (server crash between PSP auth and DB write) is solved by pull-based reconciliation — a background worker queries the PSP for stuck records and updates our DB. The PSP is the source of truth."

---

### 7. Webhooks — The Async Confirmation Channel

> **📣 Definition:** _"Webhooks deliver async truth from the PSP — events like `payment_intent.succeeded`, `charge.dispute.created`, `refund.processed`. Three non-negotiable rules: **(1)** Verify the HMAC signature first to prevent forged events. **(2)** Deduplicate by event ID with a UNIQUE constraint on a `processed_webhooks` table — webhooks are at-least-once delivery, so duplicates WILL happen. **(3)** Process asynchronously via a queue and return 200 within seconds, otherwise the PSP retries with exponential backoff and amplifies your load. For YOUR outgoing webhooks to merchants, use the Outbox pattern: write business data + event in one DB transaction, separate worker handles delivery with retries."_

Synchronous API responses lie. The PSP authorizes a payment, returns success, then... two seconds later, the issuing bank reverses it. Or fraud team flags it. Or 3DS challenge fails. The truth comes via **webhooks** — async events from the PSP to your server.

#### The Webhook Pattern

```
PSP (Stripe)                    Your Server
─────────────                   ───────────
                                ┌─────────────────────────┐
                                │ POST /webhooks/stripe   │
                                │ Stripe-Signature: ...   │
                                │ {                       │
                                │   "id": "evt_123",      │
                                │   "type": "payment_     │
                                │           intent.       │
                                │           succeeded",   │
                                │   "data": {...}         │
                                │ }                       │
                                └─────────────────────────┘
                                          ↓
                                ┌─────────────────────────┐
                                │ 1. Verify signature     │
                                │ 2. Check event ID dedup │
                                │ 3. Process              │
                                │ 4. Return 200 quickly   │
                                └─────────────────────────┘

If you don't return 200 within ~10s, Stripe retries (up to 3 days, exponential backoff).
```

#### Three Critical Webhook Patterns

**1. Verify the signature (security)**

```python
import stripe
from django.http import HttpResponse

@csrf_exempt
def stripe_webhook(request):
    payload = request.body
    sig_header = request.META['HTTP_STRIPE_SIGNATURE']

    try:
        event = stripe.Webhook.construct_event(
            payload, sig_header, settings.STRIPE_WEBHOOK_SECRET
        )
    except (ValueError, stripe.error.SignatureVerificationError):
        return HttpResponse(status=400)   # reject — possibly forged

    # ... continue with verified event ...
```

Without signature verification, anyone can POST to your webhook endpoint and fake a `payment_intent.succeeded`. This is a real attack vector — never skip it.

**2. Deduplicate by event ID (at-least-once delivery)**

Webhooks are at-least-once: Stripe will deliver the same event multiple times if your server is slow, returns non-200, or is unreachable.

```python
@transaction.atomic
def stripe_webhook(request):
    event = construct_event(request)  # verified above

    # Atomic claim of this event ID
    try:
        ProcessedWebhook.objects.create(
            event_id=event.id,                     # UNIQUE constraint
            event_type=event.type,
            received_at=timezone.now(),
        )
    except IntegrityError:
        # Already processed — return 200 immediately
        return HttpResponse(status=200)

    # Now safe to process — only one worker reaches here per event
    handle_event(event)
    return HttpResponse(status=200)
```

**3. Process quickly, return 200 fast**

If your handler takes 30 seconds, Stripe will time out and retry — multiplying load.

```python
def stripe_webhook(request):
    event = construct_event(request)

    # Just queue it, return 200 fast
    process_webhook_event.delay(event.id, event.type, event.data)  # Celery task
    return HttpResponse(status=200)


@shared_task(bind=True, max_retries=3)
def process_webhook_event(self, event_id, event_type, data):
    # Heavy work happens here, async, with retries
    if event_type == 'payment_intent.succeeded':
        handle_payment_success(data)
    elif event_type == 'charge.dispute.created':
        handle_chargeback(data)
    # ... etc
```

#### The Outbox Pattern (Reliable Outbound Webhooks)

If YOU are sending webhooks to merchants, you face the same problem: how do you guarantee delivery without losing events?

```sql
-- Same DB transaction as your business write
BEGIN;
    INSERT INTO payments (...) VALUES (...);
    INSERT INTO outbox_events (event_type, payload, status)
        VALUES ('payment.captured', '{...}', 'pending');
COMMIT;

-- Separate worker reads from outbox and publishes to Kafka / merchant URL
-- with retry + exponential backoff
```

This guarantees: **you never have a payment in the DB without a matching pending event**, and you never publish an event that wasn't durably committed. (See [SAGA pattern](coding_principle_guide.md#3-saga-pattern--other-design-patterns) for the full Outbox pattern.)

📌 **TLDR for interviews:** "Webhooks deliver async truth from the PSP. Three rules: verify the signature (HMAC), deduplicate by event ID with a unique constraint (at-least-once delivery means duplicates), and process asynchronously via a queue (return 200 fast or PSP will retry and amplify load). For YOUR outgoing webhooks to merchants, use the Outbox pattern — co-write business data + event in one DB transaction, separate worker handles delivery with retries."

---

### 8. Reconciliation — The Safety Net Against Everything

> **📣 Definition:** _"Reconciliation is the periodic, independent comparison of your internal ledger against external sources of truth — PSP settlement reports + bank statements. It's the safety net that catches everything idempotency and webhooks miss: missed events after webhook delivery exhausted retries, fee calculation bugs, manual PSP adjustments that bypass webhooks, dev-team scripts that touched production. Build it as a first-class service with its own data store and dashboards — not a scheduled cron job. Key metric to monitor: 'unreconciled items > 24 hours.' Every payment system bigger than a demo has one."_

> "Webhooks alone are not enough. Reconciliation is the only thing that catches what idempotency and webhooks miss." — Razorpay engineer

Reconciliation is the **periodic, independent check** that compares your internal state against external sources of truth:

```
Daily reconciliation job:

1. Pull settlement report from PSP (CSV download or API)
2. Pull bank statement (or open banking API)
3. Compare against your ledger:

   Internal ledger says:    Payment 12345 captured ₹500 on Mar 5
   PSP report says:         Payment 12345 captured ₹500 on Mar 5  ✅
   Bank statement says:     +₹500 from PSP on Mar 6                ✅

4. Investigate every mismatch:
   - PSP says succeeded, our DB says failed → webhook missed → fix DB
   - PSP says ₹500, bank says ₹490 → PSP took a fee → record fee entry
   - Our DB says succeeded, PSP says nothing → phantom payment → bug!
```

#### The Schema

```sql
CREATE TABLE reconciliation_runs (
    id BIGSERIAL PRIMARY KEY,
    period_start DATE,
    period_end DATE,
    source VARCHAR(50),                  -- 'stripe', 'razorpay', 'hdfc_bank'
    matched_count INT DEFAULT 0,
    discrepancy_count INT DEFAULT 0,
    status VARCHAR(20),                  -- 'running' | 'completed' | 'failed'
    started_at TIMESTAMP DEFAULT NOW(),
    completed_at TIMESTAMP
);

CREATE TABLE reconciliation_discrepancies (
    id BIGSERIAL PRIMARY KEY,
    run_id BIGINT REFERENCES reconciliation_runs(id),
    type VARCHAR(50),                    -- 'missing_internal', 'amount_mismatch', etc.
    psp_reference VARCHAR(255),
    payment_id BIGINT,
    expected_amount BIGINT,
    actual_amount BIGINT,
    raw_data JSONB,
    resolution_status VARCHAR(20),       -- 'unresolved' | 'investigating' | 'fixed' | 'wontfix'
    resolved_at TIMESTAMP
);
```

#### Why this is non-negotiable

Even with perfect idempotency + webhooks + state machine + ledger, things go wrong:

- Webhook delivery fails after 3 days of retries (rare, but happens)
- A bug introduces an off-by-one rupee in fee calculation
- PSP issues a manual adjustment that bypasses webhooks
- Your dev accidentally runs a script in production

**Reconciliation is the cron job that catches all of these.** The metrics that matter:

- **Reconciliation lead time**: how long from settlement → matched ledger entry
- **Unreconciled items > 24h / 72h**: should always trend toward zero
- **Discrepancy resolution time**

📌 **TLDR for interviews:** "Reconciliation is the periodic comparison of your internal ledger against external sources of truth (PSP settlement reports + bank statements). It's the safety net that catches what idempotency and webhooks miss — missed events, fee calculation bugs, manual PSP adjustments. Build it as a first-class service, not a scheduled cron job. Every payment system bigger than 'demo' has one. The metric to watch: unreconciled items > 24h."

---

### 9. Distributed Transaction Routing & Deduplication at Scale

> **📣 Definition:** _"This is the two-part 'how do you scale payment processing' question. **Routing** — use **consistent hashing on the idempotency key** so retries always hit the same instance, leveraging local cache and reducing cross-instance coordination. Consistent hashing minimizes data movement when nodes scale up/down (only ~1/N keys move). **Deduplication** — use defense in depth: Redis cache → Redis SETNX lock with TTL (prevents two workers processing the same key concurrently; TTL prevents deadlock if a worker crashes) → Postgres UNIQUE constraint as the source of truth → PSP idempotency key as final external safety net. Shard the idempotency table itself when scale demands it (Airbnb's pattern)."_

Now to your second question: **"In a distributed system, how do you determine where a transaction is handled and prevent duplicates?"**

This question has two parts: **routing** (where) and **dedup** (preventing duplicates).

#### Part 1: Routing — Consistent Hashing on the Idempotency Key

When you have N payment service instances, you want **the same idempotency key to always hit the same instance** — that way, the local cache + DB connection on that instance handles dedup most efficiently.

```
                          ┌─────────────┐
                          │ API Gateway │
                          └──────┬──────┘
                                 │ hash(idempotency_key) → which instance?
                                 │
            ┌────────────────────┼────────────────────┐
            │                    │                    │
       ┌────▼────┐          ┌────▼────┐          ┌────▼────┐
       │ Pay #1  │          │ Pay #2  │          │ Pay #3  │
       │ keys:   │          │ keys:   │          │ keys:   │
       │ a-h     │          │ i-q     │          │ r-z     │
       └─────────┘          └─────────┘          └─────────┘

Why hash on idempotency_key (not customer_id)?
- A retry uses the SAME key → routes to SAME instance → instance's local cache hits
- Avoids cross-instance coordination for typical retries
```

**Why consistent hashing specifically?**

```
Naive hash: hash(key) % N
Problem: if you add a 4th instance, ALL keys remap. 75% of requests go to wrong instance.

Consistent hashing: place servers + keys on a virtual ring.
Adding a server only moves ~1/N keys. Most retries still find their data locally.
```

```python
# Simplified consistent hash ring (production: use Bloom or HRW for better distribution)
import hashlib
import bisect

class ConsistentHashRing:
    def __init__(self, replicas=150):
        self.replicas = replicas      # virtual nodes per server (smooth distribution)
        self.ring = []                # sorted list of (hash, server)
        self.hash_to_server = {}

    def add_server(self, server):
        for i in range(self.replicas):
            h = self._hash(f"{server}:{i}")
            bisect.insort(self.ring, (h, server))
            self.hash_to_server[h] = server

    def get_server(self, key):
        if not self.ring:
            return None
        h = self._hash(key)
        idx = bisect.bisect(self.ring, (h, ''))
        if idx == len(self.ring):
            idx = 0   # wrap around
        return self.ring[idx][1]

    def _hash(self, s):
        return int(hashlib.md5(s.encode()).hexdigest(), 16)

ring = ConsistentHashRing()
ring.add_server('payment-1')
ring.add_server('payment-2')
ring.add_server('payment-3')

# Same key always routes to same server
print(ring.get_server('idem_abc123'))   # → payment-2 (always)
print(ring.get_server('idem_abc123'))   # → payment-2 (always)
```

**However:** even with routing, you still need DB-level dedup as the source of truth. Routing is an optimization (cache locality, reduced contention); it's not a correctness primitive.

#### Part 2: Deduplication — Layered Defense

**Airbnb's payment dedup framework** (real production lesson):

```
┌─────────────────────────────────────────────────────────────┐
│  Defense Layer 1: API Gateway dedup cache (Redis)           │
│  • Hash(idempotency_key) → routed to instance               │
│  • Local Redis check first (sub-ms latency)                 │
│  • If hit + completed: return cached response                │
│  • Catches ~90% of retries cheaply                          │
└─────────────────────────────────────────────────────────────┘
                           ↓ miss / in-progress
┌─────────────────────────────────────────────────────────────┐
│  Defense Layer 2: Application-level lock (Redis SETNX)      │
│  • SET idempotency:<key> "processing" NX EX 60              │
│  • If returns nil → another worker has it, return 409 retry │
│  • Prevents two workers processing same key concurrently     │
└─────────────────────────────────────────────────────────────┘
                           ↓ acquired
┌─────────────────────────────────────────────────────────────┐
│  Defense Layer 3: Postgres UNIQUE constraint                │
│  • INSERT INTO idempotency_keys ... ON CONFLICT DO NOTHING  │
│  • Sharded by idempotency_key for scale                     │
│  • THE source of truth — guarantees correctness              │
│  • Same transaction as the business write                   │
└─────────────────────────────────────────────────────────────┘
                           ↓ inserted
┌─────────────────────────────────────────────────────────────┐
│  Defense Layer 4: PSP idempotency key                       │
│  • Forward same key to Stripe/Razorpay                      │
│  • Even if our retry fires past Layers 1-3, PSP dedups       │
└─────────────────────────────────────────────────────────────┘
```

**Code: the Redis SETNX lock pattern:**

```python
import redis

r = redis.Redis()

def process_with_distributed_lock(idempotency_key, request):
    lock_key = f"lock:idem:{idempotency_key}"

    # Try to acquire lock atomically — NX = only if not exists, EX = expire in 60s
    acquired = r.set(lock_key, "processing", nx=True, ex=60)

    if not acquired:
        # Another worker has this key
        existing_state = r.get(lock_key)
        if existing_state == b'processing':
            return {"error": "in progress, retry later"}, 409
        elif existing_state == b'done':
            # Cached result available
            cached = r.get(f"result:idem:{idempotency_key}")
            return json.loads(cached), 200

    try:
        # We have the lock — do the work
        result = actually_process_payment(request, idempotency_key)

        # Cache the result for future retries
        r.set(f"result:idem:{idempotency_key}", json.dumps(result), ex=86400)
        r.set(lock_key, "done", ex=86400)
        return result, 200

    except Exception as e:
        r.delete(lock_key)   # release on failure → safe retry
        raise
```

**Critical detail: TTL on the lock.** If a worker crashes while holding the lock, the TTL ensures the lock expires (instead of a permanent deadlock). This is why `ex=60` matters — a crashed worker's lock auto-releases.

#### Sharding the Idempotency Table

When the idempotency table itself becomes a bottleneck (millions of rows/day), shard it:

```python
def get_idempotency_shard(idempotency_key: str, num_shards: int = 16) -> int:
    """Hash the key to determine which DB shard stores it."""
    return int(hashlib.md5(idempotency_key.encode()).hexdigest(), 16) % num_shards

# In code
shard_id = get_idempotency_shard(idem_key)
db = get_database_connection(f"idempotency_shard_{shard_id}")
db.execute("INSERT INTO idempotency_keys ...")
```

This is what Airbnb does in their generic idempotency framework — sharded master DB, with idempotency_key as the shard key (high cardinality, even distribution).

📌 **TLDR for interviews:** "Distributed transaction handling has two parts: **routing** (use consistent hashing on idempotency_key so retries hit the same instance — cache locality, less contention; consistent hashing minimizes data movement when nodes change) and **dedup** (defense in depth: Redis cache → Redis SETNX lock with TTL → Postgres UNIQUE constraint as source of truth → PSP idempotency key as final safety net). The Redis lock has a TTL so a crashed worker doesn't deadlock the key forever. The DB constraint guarantees correctness even if all caches fail. Shard the idempotency table itself when scale demands it."

---

### 10. Money Representation — Never Use Float

> **📣 Definition:** _"Never use `float` or `double` for money — binary representation causes precision drift (`0.1 + 0.2 = 0.30000000000000004`), and accumulated errors over millions of transactions become real lost rupees. Two valid approaches: **(1) BIGINT in smallest currency unit** — store ₹500.50 as 50050 paise, integer math is always exact (used by Stripe, Square, every major processor). **(2) NUMERIC/DECIMAL** — fixed precision like `NUMERIC(20, 4)`, also exact, slightly more storage. Always store the currency code separately because some currencies have 0 decimals (JPY, KRW) and others have 3 (KWD, BHD). Use Python's `Decimal` (not float) when precision math is needed."_

This is a fast-fail interview question.

```python
# ❌ NEVER do this for money
amount = 0.1 + 0.2
print(amount)             # 0.30000000000000004  ← WTF, you just lost a paisa

balance = 100.00
balance -= 33.33
balance -= 33.33
balance -= 33.34
print(balance)            # -7.105427357601002e-15  ← should be exactly 0!
```

**Why:** floats are binary representations. Decimal fractions like 0.1 don't have exact binary representations. After many operations, error accumulates.

**The correct approaches:**

```python
# Option 1: Store as smallest currency unit (paise / cents) using BIGINT
# ₹500.50 → 50050 paise (integer math, always exact)
amount_in_paise = 50050
total_in_paise = sum(payments)         # always exact

# Option 2: Use Python's Decimal for display/calculation
from decimal import Decimal, ROUND_HALF_UP

a = Decimal('0.1')
b = Decimal('0.2')
print(a + b)              # Decimal('0.3') ✅

# Always specify precision and rounding for display
amount = Decimal('99.999')
display = amount.quantize(Decimal('0.01'), rounding=ROUND_HALF_UP)
# Decimal('100.00')

# Option 3: For high-precision needs (crypto, FX), use string representation
# during transmission and storage. Parse to Decimal only for calculation.
```

**Database column type:**

```sql
-- ❌ WRONG
CREATE TABLE payments (amount FLOAT);          -- never
CREATE TABLE payments (amount DOUBLE);         -- never

-- ✅ RIGHT for fiat currencies
CREATE TABLE payments (amount BIGINT);         -- store paise as integer

-- ✅ ALSO RIGHT (more readable, slightly more storage)
CREATE TABLE payments (amount NUMERIC(20, 4));  -- 4 decimal places, exact
```

**Currencies without decimal places:** Japanese yen has no fractional unit. Kuwaiti dinar has 3 decimal places. **Always store the currency code separately** and respect the unit:

```python
ZERO_DECIMAL_CURRENCIES = {'JPY', 'KRW', 'VND'}
THREE_DECIMAL_CURRENCIES = {'KWD', 'BHD', 'OMR'}

def format_amount(amount_smallest_unit: int, currency: str) -> str:
    if currency in ZERO_DECIMAL_CURRENCIES:
        return f"{amount_smallest_unit}"
    elif currency in THREE_DECIMAL_CURRENCIES:
        return f"{amount_smallest_unit / 1000:.3f}"
    else:
        return f"{amount_smallest_unit / 100:.2f}"
```

📌 **TLDR for interviews:** "Never use float or double for money — binary representation causes precision drift. Store amounts as BIGINT in the smallest currency unit (paise, cents, satang) — integer math is always exact. Use Python's `Decimal` (not float) when precision math is needed. Track the currency code separately because some currencies have 0 decimals (JPY) and some have 3 (KWD). Stripe, Square, and every major processor uses this pattern."

---

### 11. Refunds, Partial Refunds & Chargebacks

> **📣 Definition:** _"Refunds need their own immutable table with a CHECK constraint or trigger enforcing `SUM(refunds) ≤ payment.amount` so partial refunds never exceed the original. Use idempotency keys for refund requests too. **Chargebacks are different** — they originate from the customer's bank (not from your system), arrive via webhook (`charge.dispute.created`), and pull funds from your account immediately. You typically have ~10 days to upload evidence (proof of delivery, signatures, terms agreement) or you lose by default plus a fee. Build a chargeback workflow as a first-class feature, not an afterthought."_

#### Refund Patterns

```python
# Full refund — easy
stripe.Refund.create(
    payment_intent='pi_xxx',
    idempotency_key=f'refund_{order_id}_full',
)

# Partial refund — requires careful tracking
stripe.Refund.create(
    payment_intent='pi_xxx',
    amount=20000,                       # ₹200 of the original ₹500
    idempotency_key=f'refund_{order_id}_partial_1',
)

# Multiple partial refunds → must track total refunded
# Constraint: SUM(refunds) ≤ original_payment_amount
```

**Schema:**

```sql
CREATE TABLE refunds (
    id BIGSERIAL PRIMARY KEY,
    payment_id BIGINT REFERENCES payments(id),
    amount BIGINT NOT NULL,
    status VARCHAR(20),
    reason VARCHAR(50),                 -- 'duplicate', 'fraud', 'customer_request'
    psp_refund_id VARCHAR(255) UNIQUE,
    created_at TIMESTAMP DEFAULT NOW(),
    -- DB-level invariant: refund cannot exceed payment
    CONSTRAINT refund_within_payment CHECK (amount > 0)
);

-- Trigger to enforce: total refunds don't exceed payment amount
CREATE OR REPLACE FUNCTION check_refund_total() RETURNS TRIGGER AS $$
DECLARE
    total_refunded BIGINT;
    payment_amount BIGINT;
BEGIN
    SELECT amount INTO payment_amount FROM payments WHERE id = NEW.payment_id;
    SELECT COALESCE(SUM(amount), 0) INTO total_refunded
        FROM refunds WHERE payment_id = NEW.payment_id;
    IF total_refunded + NEW.amount > payment_amount THEN
        RAISE EXCEPTION 'Refund would exceed payment amount';
    END IF;
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trigger_check_refund_total
BEFORE INSERT ON refunds
FOR EACH ROW EXECUTE FUNCTION check_refund_total();
```

#### Chargebacks — The Adversarial Refund

A chargeback is when a customer disputes a charge through their bank (not through you). You have ~10 days to provide evidence, or you lose automatically.

```
Chargeback flow:
1. Customer disputes charge with their bank
2. Bank issues a chargeback to your PSP
3. PSP webhooks you: charge.dispute.created
4. Funds are pulled from your account immediately (you owe them)
5. You upload evidence (proof of delivery, signatures, etc.)
6. Bank decides → won (funds returned) or lost (gone + fee)

Your code:
@webhook_handler('charge.dispute.created')
def handle_chargeback(event):
    payment = Payment.objects.get(psp_reference=event['charge'])
    transition_payment(payment.id, 'disputed', metadata=event)

    # Trigger evidence collection workflow
    Dispute.objects.create(
        payment=payment,
        reason=event['reason'],
        amount=event['amount'],
        evidence_due_by=event['evidence_details']['due_by'],
    )
    # Notify ops team to gather evidence
```

📌 **TLDR for interviews:** "Refunds need their own immutable table with a CHECK constraint or trigger that enforces SUM(refunds) ≤ payment_amount. Use idempotency keys for refund requests too. Chargebacks are different — they originate from the customer's bank, hit you via webhook, and pull funds immediately. You have a deadline (often 10 days) to submit evidence or lose by default."

---

### 12. Payment System Interview Q&A

> **📣 Definition:** _"This is the curated set of payment-system interview questions interviewers actually ask — how Stripe prevents double charges, the half-completed-transaction problem, webhook deduplication, payment data model design, double-entry bookkeeping, fraud at scale, scaling to 10K TPS, authorize-vs-capture, stuck PENDING payments, and money representation. The pattern in good answers: lead with the failure mode or invariant being protected, then walk through the solution with concrete mechanism (idempotency key, state machine, ledger), then mention the trade-off. Practice each in 60-90 seconds. Mention specific vendors (Stripe, Razorpay) — it shows production exposure, not just textbook knowledge."_

These are the actual questions interviewers love. Practice 60-90 second answers.

**Q1: "How does Stripe prevent double charges?"**

> "Three layers. (1) Client generates a UUID idempotency key per logical payment attempt — same key on retry. (2) Stripe caches the response (success OR failure) for that key for 24 hours; retries get the cached response instead of a new charge. (3) Sending the same key with different request bodies returns 400 — the key is a commitment, not a label. To make our own system bulletproof, we add: Postgres UNIQUE constraint on idempotency_keys table (DB is source of truth), Redis cache as fast first check, and we forward the same key to Stripe so even our retries are deduped externally."

**Q2: "What happens if your server crashes between calling Stripe and writing to the database?"**

> "Classic half-completed-transaction problem. Stripe has authorized funds, but our DB has no record. Solution: pull-based reconciliation. A background worker scans for idempotency records stuck in `in_progress` for >10 minutes, queries Stripe for the actual status, and reconciles our DB. The PSP is the source of truth; eventual consistency is acceptable as long as we close the loop. Critical: the customer's retry uses the same idempotency key, so even before reconciliation runs, Stripe won't double-charge."

**Q3: "How do you handle webhooks that are delivered multiple times?"**

> "Stripe's webhooks are at-least-once. Three patterns: (1) Verify the HMAC signature first — protects against forged events. (2) Deduplicate by event ID with a UNIQUE constraint on a `processed_webhooks` table — atomic insert, on conflict return 200 immediately. (3) Process async via Celery — return 200 within seconds or Stripe retries with exponential backoff up to 3 days, multiplying load."

**Q4: "Design the data model for a payment."**

> "Two tables, not one. `payments` has the current denormalized state (for fast reads). `payment_events` is append-only — every state transition is a row. Status field uses a strict state machine enforced in code: created → processing → authorized → captured → settled. Refunds are their own table with a CHECK constraint that SUM(refunds) ≤ payment.amount. The append-only event log gives us perfect audit history for chargebacks and disputes, and lets us rebuild current state if denormalization drifts."

**Q5: "Why double-entry bookkeeping?"**

> "Every payment creates matched debit + credit ledger entries — sum of debits always equals sum of credits. Continuous reconciliation jobs verify this invariant. If it ever drifts, we have a bug worth waking the on-call engineer. The ledger is append-only — no UPDATE, no DELETE — and is the only system that can't be rebuilt from anything else; everything else can be rebuilt from the ledger. This is how Stripe processes billions without losing pennies."

**Q6: "How do you prevent fraud at scale?"**

> "Layered approach. (1) Rules engine: hard limits (e.g., reject >₹50K from new accounts). (2) Velocity checks: same card used 10 times in 5 minutes → flag. (3) ML risk scoring: features like card BIN, IP geo, device fingerprint, time-of-day, deviation from user history → fraud probability score. (4) Network-level: 3DS challenges for high-risk transactions, shifting liability to the issuer. Run synchronously in the auth path with strict latency budget (~200ms). Async layer feeds back into model retraining."

**Q7: "How would you scale the payment system to 10K TPS?"**

> "Routing layer with consistent hashing on idempotency_key — same key always hits same instance for cache locality. Sharded Postgres for the idempotency + payments tables, sharded by user_id or idempotency_key. Read replicas for queries. Redis for the hot dedup cache. Async webhook processing via Kafka + dedicated workers. Kafka for event streaming to downstream (analytics, fraud, notifications). The hot path stays fast because ledger writes go to a write-optimized table; analytics reads from a separate CQRS read model fed by CDC (Debezium → Kafka)."

**Q8: "What's the difference between authorize and capture?"**

> "Two-phase payment: authorize holds funds on the card without transferring them (typically valid for 7-30 days), capture actually transfers the money. Used for e-commerce (auth at checkout, capture on ship), hotels (auth a deposit, capture on checkout), and as a refund safety net — voiding an auth shows no charge on the customer's statement, vs capture-then-refund which creates two visible entries. Manual capture is `capture_method='manual'` in Stripe."

**Q9: "How do you handle a payment that's stuck in PENDING?"**

> "Reconciliation job. A background worker scans for payments in PENDING state for >N minutes, calls the PSP API to get the actual status, and updates our DB accordingly. If PSP says authorized → we update to AUTHORIZED. If PSP has no record → we mark FAILED. PSP is source of truth; we converge to it. Customer's retry uses the same idempotency key so they never get double-charged in the meantime."

**Q10: "How do you store money in your database?"**

> "Never float — binary precision drift causes accumulated errors. Two valid approaches: (1) BIGINT in smallest currency unit (paise, cents) — integer math is always exact. (2) NUMERIC/DECIMAL with explicit precision (e.g., NUMERIC(20, 4)). Always store currency code separately because some currencies have 0 decimals (JPY) and some have 3 (KWD). Stripe and every major processor uses BIGINT-in-smallest-unit."

---

### 13. 📌 The Master TLDR for Payment Systems

> "Payment systems exist to maintain money as an invariant in the face of distributed failures. Three core principles: **(1) Idempotency at every layer** — client-generated UUID flows through Redis cache → Postgres UNIQUE constraint → PSP idempotency key, with request fingerprinting to catch client-side key reuse. **(2) Immutable state machines** — payments are append-only event logs with denormalized current state for fast reads, never UPDATE-in-place. **(3) Double-entry ledger + continuous reconciliation** — every transaction creates matched debits/credits, periodic jobs compare internal ledger to PSP and bank statements. The unspoken rule: at-least-once delivery + idempotency + reconciliation is how we achieve 'effective exactly-once' in practice — true exactly-once is a myth in distributed systems. Other essentials: BIGINT-in-paise for money (never float), authorize-then-capture for refund-friendly flows, signed webhooks with event-ID dedup for async truth, and pull-based reconciliation as the safety net for half-completed transactions."

---
