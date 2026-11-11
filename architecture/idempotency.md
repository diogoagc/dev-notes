# Idempotency

An operation is **idempotent** if performing it several times has the same effect as performing it once.

> **Key point:** In systems with networks and retries, an operation may be executed more than once. Idempotency is what makes those retries safe.

**Related:** [Atomicity and Atomic Operations](../databases/atomicity.md) · [Transactional Outbox Pattern](../design-patterns/transactional-outbox.md) · [Saga Pattern](../design-patterns/distributed-systems/saga.md)

## Contents

* [Why It Matters](#why-it-matters)

* [What Idempotent Means Exactly](#what-idempotent-means-exactly)

* [Idempotency in HTTP](#idempotency-in-http)

* [Making Operations Idempotent](#making-operations-idempotent)

* [Idempotency Keys](#idempotency-keys)

* [Implementing an Idempotent Endpoint](#implementing-an-idempotent-endpoint)

* [Common Pitfalls](#common-pitfalls)

* [Idempotent vs. Atomic](#idempotent-vs-atomic)

* [When to Care](#when-to-care)

* [Key Takeaways](#key-takeaways)

* [Useful Links](#useful-links)

---

## Why It Matters

A client sends a request and gets a timeout:

```text

Client sends "charge 50 EUR"

↓

... no response ...

```

The client cannot know which of these happened:

```text

1. The request never arrived          → nothing was charged

2. The server failed while processing → maybe charged

3. The server succeeded, the response was lost → charged

```

From the outside, all three look identical. The client has two options:

* **do not retry** → the customer may never be charged (lost operation);

* **retry** → the customer may be charged twice (duplicate operation).

Neither is acceptable unless the operation is idempotent. Then the answer is simple: **always retry**, because a duplicate has no additional effect.

Retries are everywhere, often invisible: HTTP clients, load balancers, message brokers redelivering, queue workers restarting, users double-clicking a button, mobile apps resending after reconnect.

---

## What Idempotent Means Exactly

Mathematically: `f(f(x)) = f(x)`.

```text

Idempotent                          Not idempotent

x = 5                               x = x + 1

SET status = 'paid'                 balance = balance + 100

DELETE user 42                      INSERT a new row

"set the thermostat to 21°"         "raise the thermostat by 1°"

press the elevator call button      append a line to a file

```

The pattern: **setting an absolute state** is idempotent; **applying a relative change** or **creating something new** is not.

Idempotency is about the **effect on state**, not about the response. Deleting a resource twice may return `204` and then `404`; the state ("resource does not exist") is the same, so the operation is idempotent.

---

## Idempotency in HTTP

HTTP defines which methods are expected to be idempotent (RFC 9110): the intended effect of several identical requests is the same as that of a single request.

| Method | Safe (read-only) | Idempotent | Notes |

| --- | --- | --- | --- |

| `GET`, `HEAD`, `OPTIONS` | Yes | Yes | Must not change state |

| `PUT` | No | Yes | Replaces the resource with the given state |

| `DELETE` | No | Yes | Gone after the first call, still gone after the second |

| `POST` | No | **No** | "Create" or "do": each call may have a new effect |

| `PATCH` | No | **Not guaranteed** | Depends on the patch: `set name` is, `increment` is not |

Why it matters in practice: clients, proxies and libraries may automatically retry idempotent methods after a connection failure, but should not automatically retry `POST`.

These are **contracts, not guarantees**. A `PUT` handler that appends to a list, or a `GET` that changes data, breaks the contract and will be retried incorrectly by infrastructure that trusts it.

---

## Making Operations Idempotent

**1. Use naturally idempotent operations**

Design the operation as "set the state to X" instead of "change the state by X".

```text

POST /orders/42/pay            →  PUT /orders/42/status {"status": "paid"}

UPDATE ... SET qty = qty + 1   →  UPDATE ... SET qty = 5

```

**2. Let the client choose the identifier**

If the client generates the ID (a UUID), creation becomes an idempotent `PUT`:

```text

PUT /orders/7f3c9a1e-...     first call creates, repeated calls change nothing

```

**3. Upsert on a business key**

```sql

INSERT INTO subscriptions (user_id, plan)

VALUES (42, 'pro')

ON CONFLICT (user_id) DO UPDATE SET plan = EXCLUDED.plan;

```

**4. Conditional updates (state or version checks)**

```sql

UPDATE orders SET status = 'shipped'

WHERE id = 42 AND status = 'paid';

```

The second execution matches zero rows. This also rejects stale or out-of-order requests.

**5. Deduplicate by a unique ID**

When the operation is inherently not idempotent ("add 100 to the balance"), record a unique ID for each logical operation and refuse to apply it twice:

```text

BEGIN TRANSACTION

Insert operation_id into processed_operations   (unique constraint)

If it already exists:

    do nothing

Otherwise:

    apply the change

COMMIT

```

The record and the change must be in the **same transaction**. For message consumers this is `event_id` deduplication, described in the [Transactional Outbox note](../design-patterns/transactional-outbox.md#event-ids-and-deduplication). For APIs it is an idempotency key.

---

## Idempotency Keys

An idempotency key turns a non-idempotent request (typically `POST`) into an idempotent one. The **client** generates a unique key per logical operation and sends it with every attempt; the **server** uses it to recognise retries.

```text

POST /payments

Idempotency-Key: 5b2f6c0e-9d1a-4a8e-b7c3-2f1d8e6a9c41

{"amount": 5000, "currency": "eur"}

First request   → payment is executed, response stored under the key

Retried request → payment is NOT executed again, stored response is returned

```

Client rules:

* One key per **logical operation**, reused for all its retries. Generating a new key per attempt defeats the purpose.

* Generate the key **before** the first attempt and persist it if the retry may happen after a restart (for example, store it with the order).

* Use a random value with enough entropy (UUID v4), or derive it from a stable identifier such as an `event_id`. Avoid sensitive data in keys.

How Stripe does it (a good reference implementation, API v1):

* all `POST` requests accept an `Idempotency-Key` header;

* the status code and body of the first request are saved and returned for retries, **including errors**;

* a retry with the same key but different parameters is rejected, to catch accidental key reuse;

* keys can be removed once they are at least 24 hours old; after that, the same key is treated as a new request.

The `Idempotency-Key` header is a widely used convention and an IETF draft, not a finished standard. Behaviour differs between providers; read their documentation.

---

## Implementing an Idempotent Endpoint

Conceptual server-side flow:

```text

1. Read the idempotency key from the request.

2. Try to insert the key (unique constraint), with a fingerprint

   of the request and status "in progress".

3. If the key already exists:

     - different request fingerprint → reject (key reused incorrectly)

     - still in progress             → reject with a conflict, client retries later

     - completed                     → return the stored response, do nothing else

4. Otherwise execute the operation.

5. Store the response under the key and mark it completed.

```

Details that matter:

* **The unique constraint is the mechanism.** "Look up the key, then insert it" is a check-then-act race; two concurrent retries would both proceed. See [Atomicity](../databases/atomicity.md#check-then-act-the-classic-bug).

* **Key and business change in one transaction** when the operation only touches the local database. Then either both exist or neither does.

* **External calls break this.** If the operation calls another system, the key cannot be committed atomically with it. Pass an idempotency key on to that system too, and treat an "in progress" key found after a crash as "unknown outcome" that needs a retry or reconciliation.

* **Scope keys** per client/account and per endpoint so that keys from different callers cannot collide.

* **Expire keys.** Keep them longer than the longest realistic retry window.

---

## Common Pitfalls

* **Idempotent does not mean "same response".** It means same resulting state. Returning the stored response is a convenience for the client, not the definition.

* **Idempotent does not mean order-independent.** `SET status = 'shipped'` is idempotent, but a delayed duplicate of an older `SET status = 'paid'` arriving afterwards overwrites newer state. Protect with a version number or a state condition.

* **Deduplication without atomicity.** Checking "already processed?" outside the transaction that applies the change leaves a window for duplicates or lost work.

* **A new key on every retry.** Common when the key is generated inside the function that sends the request instead of by the caller.

* **Dedup records that expire too early.** A retry after expiry is a new operation.

* **Side effects outside the database** (emails, external API calls) are not covered by local deduplication. See [External Side Effects](../design-patterns/transactional-outbox.md#external-side-effects).

* **Assuming "exactly-once" from the infrastructure.** In practice, exactly-once **effect** = at-least-once delivery + idempotent processing.

---

## Idempotent vs. Atomic

| | Atomic | Idempotent |

| --- | --- | --- |

| Question it answers | What if it stops halfway? | What if it runs again? |

| Guarantee | All or nothing | Repeating has no additional effect |

| Protects against | Partial failure | Duplicates and retries |

| Typical tool | Transaction, conditional write | Natural idempotency, dedup ID, idempotency key |

They are independent, and they work together:

* an atomic transfer can still run twice → needs idempotency;

* idempotency is usually **implemented with** atomicity: the dedup record and the business change committed in one transaction.

A common recipe for a reliable operation: **atomic locally, idempotent at the boundary, retried until confirmed.**

---

## When to Care

Idempotency is needed wherever something can be retried or redelivered:

* **API endpoints that create or charge** – payments, orders, sign-ups, anything a client may resend after a timeout.

* **Message and event consumers** – brokers deliver at least once.

* **Queue jobs and scheduled tasks** – workers crash and jobs are re-run; schedulers may fire twice.

* **Webhook handlers** – providers retry until they get a success response.

* **Scripts and migrations** – a script that can be re-run safely after failing halfway (`CREATE TABLE IF NOT EXISTS`, upserts).

* **Infrastructure as code** – "ensure this state" (Terraform, Ansible) rather than "run these commands".

* **User interfaces** – double-clicks and form resubmission.

It matters less for pure reads and for operations where a duplicate is harmless or acceptable. That should be a decision, not an accident.

---

## Key Takeaways

* Idempotent = **doing it again changes nothing more**.

* A timeout is ambiguous: the caller cannot know whether the operation happened. Idempotency makes "just retry" the correct answer.

* Prefer **naturally idempotent** designs: set state, upsert, client-generated IDs, conditional updates.

* For everything else, **deduplicate by a unique ID**: event IDs for messages, idempotency keys for APIs.

* The key must be **stable across retries**, and the dedup record must be committed **atomically** with the change.

* `GET`, `PUT`, `DELETE` are idempotent by contract; `POST` is not, and `PATCH` depends.

* Idempotency handles duplicates, **not ordering**, and local deduplication does **not** cover external side effects.

---

## Useful Links

* [RFC 9110 – HTTP Semantics: Idempotent Methods](https://www.rfc-editor.org/rfc/rfc9110.html#name-idempotent-methods)

* [Stripe API: Idempotent requests](https://docs.stripe.com/api/idempotent_requests)

* [Stripe blog: Designing robust and predictable APIs with idempotency](https://stripe.com/blog/idempotency)

* [IETF draft: The Idempotency-Key HTTP Header Field](https://datatracker.ietf.org/doc/draft-ietf-httpapi-idempotency-key-header/)

* [MDN: Idempotent](https://developer.mozilla.org/en-US/docs/Glossary/Idempotent)
