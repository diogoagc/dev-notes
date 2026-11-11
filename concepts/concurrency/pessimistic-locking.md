# Pessimistic Locking

**Pessimistic locking** assumes a concurrent conflict is likely enough, or expensive enough, that a transaction should reserve the relevant data before making decisions from it.

> **Key point:** Pessimistic locking prevents competing transactions from modifying locked data while the current transaction is using it, but that safety comes with blocking, lock management, and deadlock risk.

**Related:** [Transaction Isolation](isolation.md) · [Optimistic Locking](optimistic-locking.md) · [Atomicity and Atomic Operations](../../databases/atomicity.md)

## Contents

- [The Problem](#the-problem)
- [How It Works](#how-it-works)
- [Row Locks](#row-locks)
- [Lock Duration](#lock-duration)
- [Deadlocks](#deadlocks)
- [Lock Waiting and Timeouts](#lock-waiting-and-timeouts)
- [When to Use It](#when-to-use-it)
- [When Not to Use It](#when-not-to-use-it)
- [Common Mistakes](#common-mistakes)
- [Optimistic vs. Pessimistic Locking](#optimistic-vs-pessimistic-locking)
- [Key Takeaways](#key-takeaways)
- [Useful Links](#useful-links)

---

## The Problem

Suppose an application needs to reserve the last seat and then perform several database decisions based on that reservation.

Without locking:

```text
Transaction A reads seat = available
Transaction B reads seat = available
A decides to reserve it
B decides to reserve it
```

If the operation cannot be expressed safely as one conditional statement, one strategy is to lock the row before making the decision.

---

## How It Works

A common relational database pattern is:

```sql
BEGIN;

SELECT *
FROM seats
WHERE id = 42
FOR UPDATE;

-- application decides what to do

UPDATE seats
SET status = 'reserved'
WHERE id = 42;

COMMIT;
```

`FOR UPDATE` tells the database that this transaction intends to modify the selected row.

A competing transaction trying to acquire an incompatible lock generally has to wait until the first transaction commits or rolls back.

Conceptually:

```text
A acquires lock on seat 42
↓
B tries to acquire same lock
↓
B waits
↓
A commits
↓
B continues and sees the new state
```

---

## Row Locks

Use the narrowest lock that protects the business rule.

Row-level locks are usually preferable to table-level locks because unrelated rows can still be processed concurrently.

Different databases expose several lock modes. For example, PostgreSQL supports row-level modes such as `FOR UPDATE`, `FOR NO KEY UPDATE`, `FOR SHARE`, and `FOR KEY SHARE`.

The exact compatibility rules are database-specific.

---

## Lock Duration

A database lock normally lives until the transaction ends.

That means this is dangerous:

```text
BEGIN
lock row
call payment API for 8 seconds
send email
wait for another service
COMMIT
```

While the external work runs:

- another request may be blocked;
- database connections remain occupied;
- the chance of deadlocks increases;
- external failure still cannot be rolled back atomically with the database.

Keep locked transactions **short** and perform only the work that genuinely needs the protected database state.

---

## Deadlocks

A deadlock happens when transactions wait on each other in a cycle.

```text
Transaction A locks row 1
Transaction B locks row 2

A waits for row 2
B waits for row 1
```

Neither can proceed.

Databases detect deadlocks and abort one transaction so the other can continue.

Your application must therefore be prepared to retry a transaction even when using pessimistic locking.

A common prevention technique is to acquire locks in a **consistent order**:

```text
Always lock accounts by ascending account ID
```

This reduces the chance of circular waits.

---

## Lock Waiting and Timeouts

Sometimes waiting is not desirable.

Depending on the database, variants can include concepts such as:

- fail immediately if the row is locked (`NOWAIT`);
- skip currently locked rows (`SKIP LOCKED`);
- configure lock/statement timeouts.

`SKIP LOCKED` is especially useful for worker queues where multiple workers should claim different rows without waiting on each other.

Example:

```sql
SELECT id
FROM jobs
WHERE status = 'pending'
ORDER BY id
FOR UPDATE SKIP LOCKED
LIMIT 10;
```

Each worker can claim a different batch.

---

## When to Use It

Pessimistic locking is useful when:

- conflicts are frequent;
- a resource must be reserved before a multi-step database decision;
- retrying after doing substantial work would be expensive;
- correctness depends on nobody changing the row during the transaction;
- the protected transaction can remain short.

Examples:

- assigning a unique scarce resource;
- processing queue rows with multiple workers;
- updating an account based on several reads that must remain stable;
- inventory workflows where a simple conditional update is not enough.

---

## When Not to Use It

Avoid it when:

- the transaction includes slow external I/O;
- users may leave the transaction open while editing;
- conflicts are rare and [Optimistic Locking](optimistic-locking.md) would avoid blocking;
- a unique constraint or single conditional statement already solves the problem;
- very high concurrency matters and locks would serialize too much work.

A database constraint or atomic statement is often simpler and safer than manual locking.

---

## Common Mistakes

- **Locking outside a transaction.** The lock may be released immediately or not protect the intended work.
- **Holding locks while calling external services.** This creates long transactions without making the external operation atomic.
- **Locking too much data.** Broad locks reduce throughput and increase contention.
- **Inconsistent lock ordering.** This increases deadlock probability.
- **Assuming locks remove the need for retries.** Deadlocks, timeouts, and connection failures can still abort a transaction.
- **Using a lock where a constraint is enough.** Let the database enforce simple invariants directly.

---

## Optimistic vs. Pessimistic Locking

| | Optimistic | Pessimistic |
| --- | --- | --- |
| Assumption | Conflicts are rare | Conflicts are likely or costly |
| Main idea | Detect stale writes | Reserve data before acting |
| Blocking | Minimal | Possible/expected |
| Typical mechanism | Version conditional update | Row/table lock |
| Best for | Read-heavy, low-contention workflows | Short, high-contention critical sections |

Neither is universally better. Choose based on contention, retry cost, transaction duration, and the business invariant.

---

## Key Takeaways

- Pessimistic locking prevents conflicting work by acquiring locks before making protected decisions.
- Keep locked transactions as short as possible.
- Lock the smallest amount of data that protects the invariant.
- Deadlocks are normal possibilities; applications need retry handling.
- Prefer constraints or atomic conditional statements when they solve the problem directly.
- Do not use database locks to pretend external API calls are part of the same atomic transaction.

---

## Useful Links

- [PostgreSQL: Explicit Locking](https://www.postgresql.org/docs/current/explicit-locking.html)
- [PostgreSQL: SELECT locking clause](https://www.postgresql.org/docs/current/sql-select.html#SQL-FOR-UPDATE-SHARE)
- [PostgreSQL: Concurrency Control](https://www.postgresql.org/docs/current/mvcc.html)
