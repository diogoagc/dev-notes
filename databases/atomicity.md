# Atomicity and Atomic Operations

An **atomic operation** either takes effect completely or has no effect at all.

In database terminology, atomicity is primarily about failure: if an operation cannot complete, its partial effects must not remain. Whether concurrent operations can observe intermediate states is a separate concern: **isolation**.

> **Key point:** Atomicity does not automatically extend across system boundaries. A database transaction can make changes inside that database atomic, but combining a database write with a message broker, external API, another database, or another independent system requires additional coordination or patterns designed for partial failure.

**Related:** [Idempotency](../architecture/idempotency.md) · [Transaction Isolation](../concepts/concurrency/isolation.md) · [Optimistic Locking](../concepts/concurrency/optimistic-locking.md) · [Pessimistic Locking](../concepts/concurrency/pessimistic-locking.md) · [Transactional Outbox Pattern](../design-patterns/transactional-outbox.md)

## Contents

* [Why It Matters](#why-it-matters)

* [Two Meanings of "Atomic"](#two-meanings-of-atomic)

* [Atomicity in Database Transactions](#atomicity-in-database-transactions)

* [What Atomicity Does Not Give You](#what-atomicity-does-not-give-you)

* [Check-Then-Act: The Classic Bug](#check-then-act-the-classic-bug)

* [Making Operations Atomic](#making-operations-atomic)

* [Atomicity Outside the Database](#atomicity-outside-the-database)

* [Where Atomicity Ends](#where-atomicity-ends)

* [When to Care](#when-to-care)

* [Key Takeaways](#key-takeaways)

* [Useful Links](#useful-links)

---

## Why It Matters

Most business operations are several steps that only make sense together:

```text

Transfer 100 from account A to account B

1. subtract 100 from A

2. add 100 to B

```

If the process crashes after step 1, 100 has disappeared. Without atomicity, every multi-step operation needs hand-written cleanup code for every point where it could stop, and that cleanup code can fail too.

With atomicity there are only two outcomes to reason about: **it happened** or **it did not happen**.

---

## Two Meanings of "Atomic"

The word is used for two related but different guarantees:

| Meaning | Protects against | Example |

| --- | --- | --- |

| **All or nothing** (ACID atomicity) | Failures midway: crash, error, constraint violation | A transaction that rolls back completely |

| **Indivisible** (concurrency atomicity) | Other threads/requests seeing or interleaving with an intermediate state | An atomic counter increment, compare-and-swap |

In ACID, the "A" is strictly the first one. Protection from concurrent transactions is the "I" (isolation). In everyday use, "make it atomic" usually means both: nobody sees it half-done, and it never stays half-done.

---

## Atomicity in Database Transactions

A transaction groups several statements into one unit:

```sql

BEGIN;

UPDATE accounts SET balance = balance - 100 WHERE id = 'A';

UPDATE accounts SET balance = balance + 100 WHERE id = 'B';

COMMIT;

```

* If everything succeeds, `COMMIT` makes all changes permanent together.

* If the transaction fails before a successful commit, or an explicit `ROLLBACK` occurs, **all** changes are undone.

* If the connection is lost while committing, the outcome may be unknown to the client: the commit may have succeeded even though the response was never received. This is one reason idempotency matters when retrying operations.

The database achieves this with a log (write-ahead log / undo log): changes are recorded so that an incomplete transaction can be reverted during recovery.

A single SQL statement is also atomic by itself. An `UPDATE` that touches 1,000 rows and fails on row 600 leaves zero rows changed.

---

## What Atomicity Does Not Give You

**Atomicity is not isolation.** A transaction being all-or-nothing does not stop two transactions from interleaving. This is still broken, even inside a transaction, at the default isolation level of most databases (read committed):

```text

Transaction 1: SELECT stock → 1

Transaction 2: SELECT stock → 1

Transaction 1: UPDATE stock = 0, COMMIT

Transaction 2: UPDATE stock = 0, COMMIT   ← two sales, one item

```

Both transactions were perfectly atomic. The result is still wrong (a **lost update**).

**Atomicity is not durability.** All-or-nothing says nothing about surviving a power loss after commit; that is the "D".

**Atomicity is not idempotency.** Atomic means one execution is all-or-nothing. Idempotent means repeated executions have the same effect as one. A transfer can be perfectly atomic and still move the money twice if the request is retried. See [Idempotency](../architecture/idempotency.md).

---

## Check-Then-Act: The Classic Bug

The most common atomicity bug in application code: read a value, decide in the application, then write.

```text

stock = SELECT stock FROM products WHERE id = 42

if stock > 0:

    UPDATE products SET stock = stock - 1 WHERE id = 42

    create order

```

Between the check and the write, another request can do the same thing. Both pass the check. The same shape appears everywhere:

* "check if the email exists, then insert the user" → duplicate users;

* "read the counter, add one, write it back" → lost increments;

* "check if the file exists, then create it" → two writers;

* "check if the coupon was used, then apply it" → used twice.

The bug is rare under low load and in tests, and shows up in production under concurrency.

---

## Making Operations Atomic

The fix is always the same idea: **move the check and the write into one step that the system enforces**.

**1. Conditional write (check inside the statement)**

```sql

UPDATE products

SET stock = stock - 1

WHERE id = 42 AND stock > 0;

```

Then look at the affected row count: `1` means success, `0` means out of stock. The database evaluates the condition and performs the write as one step.

**2. Constraints instead of checks**

```sql

ALTER TABLE users ADD CONSTRAINT users_email_unique UNIQUE (email);

```

Do not ask "does it exist?" first. Insert and handle the constraint violation. The constraint cannot be raced.

**3. Atomic upsert**

```sql

INSERT INTO page_views (page_id, views) VALUES (7, 1)

ON CONFLICT (page_id) DO UPDATE SET views = page_views.views + 1;

```

**4. Pessimistic locking**

```sql

BEGIN;

SELECT stock FROM products WHERE id = 42 FOR UPDATE;  -- others wait here

-- decide in application code

UPDATE products SET stock = stock - 1 WHERE id = 42;

COMMIT;

```

Use when the decision needs application logic that cannot be expressed in one statement.

**5. Optimistic locking (compare-and-swap with a version)**

```sql

UPDATE products

SET stock = 4, version = 8

WHERE id = 42 AND version = 7;

```

If zero rows are affected, someone else changed the row first: reload and retry. Good when conflicts are rare.

**6. Transactions for multi-statement changes**

Wrap related writes in one transaction so they commit or roll back together. Remember this gives all-or-nothing, not protection from the interleaving above; combine with one of the techniques 1–5 when there is a read-then-write.

---

## Atomicity Outside the Database

**CPU and memory.** `counter++` is three steps (read, add, write) and loses updates between threads. Languages provide atomic types built on single hardware instructions: `sync/atomic` in Go, `std::atomic` in C/C++, `AtomicInteger` in Java. The core primitive is **compare-and-swap (CAS)**: "set the value to X only if it is still Y".

**File system.** Writing directly over a file is not atomic; a crash leaves a half-written file. The standard technique:

```text

1. write the new content to a temporary file in the same directory

2. flush it to disk (fsync)

3. rename the temporary file over the target

```

POSIX requires `rename()` to be atomic when replacing an existing file: other processes see either the old file or the new one, never a missing or partial file. The temporary file must be on the **same file system** as the target.

**Key-value stores and caches.** Use the store's atomic primitives rather than get-then-set: `INCR`, `SET key value NX` (set only if absent, the basis of simple locks), or server-side scripts/transactions.

---

## Where Atomicity Ends

A database transaction can only cover that database. These are **not** atomic, no matter how the code is arranged:

```text

write to database    +  publish to message broker

write to database    +  call an external API

write to database A  +  write to database B

write to database    +  write a file / send an email

```

One side can succeed while the other fails. This is the dual-write problem. The available tools are different in kind:

* **Avoid the second system in the critical step** – commit everything locally, do the rest asynchronously. See the [Transactional Outbox Pattern](../design-patterns/transactional-outbox.md).

* **Retries plus idempotency** – accept that a step may run more than once and make that harmless. See [Idempotency](../architecture/idempotency.md).

* **Sagas / compensation** – a sequence of local transactions, each with an undo step. Not atomic: intermediate states are visible.

* **Two-phase commit** – true distributed atomicity, but it needs every participant to support it, blocks on coordinator failure, and is rarely available for brokers and external APIs.

---

## When to Care

Ask of any operation: **"What if this stops halfway, or two of these run at the same time?"**

It matters most when:

* money, stock, quotas, or any counted resource is involved;

* several rows or tables must stay consistent with each other;

* uniqueness must hold (usernames, coupon redemptions, bookings);

* the code has a read followed by a write based on that read;

* a file or configuration is replaced while something else may read it.

It matters less when the data is derived, can be rebuilt, or small inaccuracies are acceptable (approximate view counters, caches).

---

## Key Takeaways

* Atomic = **all or nothing**, and no observable intermediate state.

* ACID atomicity is about **failure**; isolation is about **concurrency**. A transaction alone does not prevent lost updates.

* **Check-then-act** in application code is a race. Put the check into the write: conditional updates, constraints, upserts, locks, or version checks.

* A single SQL statement is atomic; several statements need a transaction.

* Outside databases: atomic CPU types and CAS for memory, write-then-rename for files, native atomic commands for key-value stores.

* Atomicity **stops at the system boundary**. Across systems, use local transactions plus outbox, retries, and idempotency.

* Atomic and idempotent are different properties; reliable systems usually need both.

---

## Useful Links

* [PostgreSQL: Transactions](https://www.postgresql.org/docs/current/tutorial-transactions.html)

* [PostgreSQL: Transaction Isolation](https://www.postgresql.org/docs/current/transaction-iso.html)

* [Atomicity (database systems) – Wikipedia](https://en.wikipedia.org/wiki/Atomicity_(database_systems))

* [POSIX `rename()` specification](https://pubs.opengroup.org/onlinepubs/9699919799/functions/rename.html)

* [Compare-and-swap – Wikipedia](https://en.wikipedia.org/wiki/Compare-and-swap)
