# Transaction Isolation

**Transaction isolation** defines how much one transaction is allowed to observe or interfere with the work of other transactions running at the same time.

> **Key point:** Atomicity answers **"what if this transaction fails halfway?"** Isolation answers **"what can concurrent transactions see and change while this transaction is running?"**

**Related:** [Atomicity and Atomic Operations](../../databases/atomicity.md) · [Optimistic Locking](optimistic-locking.md) · [Pessimistic Locking](pessimistic-locking.md)

## Contents

- [Why Isolation Matters](#why-isolation-matters)
- [Concurrency Anomalies](#concurrency-anomalies)
- [Isolation Levels](#isolation-levels)
- [MVCC](#mvcc)
- [Isolation vs. Locking](#isolation-vs-locking)
- [Choosing an Isolation Level](#choosing-an-isolation-level)
- [Common Mistakes](#common-mistakes)
- [Key Takeaways](#key-takeaways)
- [Useful Links](#useful-links)

---

## Why Isolation Matters

Imagine two requests trying to buy the last item in stock:

```text
Stock = 1

Transaction A                  Transaction B
-------------                  -------------
READ stock -> 1                READ stock -> 1
Decides it can sell            Decides it can sell
WRITE stock = 0                WRITE stock = 0
COMMIT                         COMMIT
```

Both transactions can be perfectly **atomic** and still produce the wrong business result: two sales were accepted for one item.

The problem is concurrency, not partial failure.

Isolation controls what happens when transactions overlap.

---

## Concurrency Anomalies

Different isolation levels prevent different classes of anomalies.

### Dirty Read

A transaction reads data written by another transaction that has **not committed yet**.

```text
A: UPDATE accounts SET balance = 0
B: READ balance -> 0
A: ROLLBACK
```

Transaction B observed a value that never became real.

### Non-Repeatable Read

The same row returns different committed values inside one transaction.

```text
A: READ price -> 10
B: UPDATE price = 12; COMMIT
A: READ price -> 12
```

### Phantom Read

A query returning a set of rows returns a different set later because another transaction inserted or deleted matching rows.

```text
A: SELECT * FROM bookings WHERE day = 'Friday' -> 4 rows
B: INSERT another Friday booking; COMMIT
A: SELECT same query -> 5 rows
```

### Lost Update

Two transactions read the same value, independently calculate a replacement, and one overwrites the other's result.

```text
counter = 10

A reads 10
B reads 10
A writes 11
B writes 11

Expected: 12
Actual:   11
```

A transaction does not automatically prevent this. The application may need conditional writes, locking, version checks, or stronger isolation.

### Write Skew

Two transactions each make a valid decision based on a snapshot, but together they violate a rule spanning multiple rows.

Example: two doctors are both on call. Each transaction sees the other doctor on call and independently marks itself off call. Both changes are individually valid, but the final state has nobody on call.

---

## Isolation Levels

SQL defines four common isolation levels. The exact behavior varies between database engines, so always check the database documentation.

| Level | General idea | Typical trade-off |
| --- | --- | --- |
| **Read Uncommitted** | May allow reading uncommitted changes. | Highest concurrency, weakest guarantees. |
| **Read Committed** | Each statement sees committed data as of that statement. | Common default; good throughput, but repeated reads can change. |
| **Repeatable Read** | A transaction generally sees a stable snapshot for repeated reads. | Stronger consistency, more conflicts/retries. |
| **Serializable** | Transactions behave as if they executed one at a time in some serial order. | Strongest guarantee, but applications must handle serialization failures/retries. |

### Read Committed

Often a sensible default for ordinary CRUD work.

But this is unsafe:

```text
1. SELECT stock
2. decide in application code
3. UPDATE stock
```

Another transaction can change the row between steps 1 and 3.

Prefer a conditional write where possible:

```sql
UPDATE products
SET stock = stock - 1
WHERE id = 42 AND stock > 0;
```

The database evaluates the condition and update together.

### Repeatable Read

Useful when several reads in a transaction must be based on a consistent view of the data.

It does **not** mean "nothing else can change the database". Other transactions still run. The database controls what your transaction sees and may reject conflicting writes.

### Serializable

Serializable aims to make the result equivalent to transactions running one after another.

This does not necessarily mean the database literally executes them one at a time. Modern databases can run them concurrently and detect combinations that cannot be serialized safely.

The application must be ready for errors such as a **serialization failure** and retry the entire transaction.

---

## MVCC

Many relational databases use **Multi-Version Concurrency Control (MVCC)**.

Instead of making readers and writers block each other for every operation, the database can keep multiple row versions so a transaction reads a snapshot appropriate to its isolation level.

Conceptually:

```text
Row version 1: balance = 100
Row version 2: balance = 80

Transaction A may still see version 1
while another transaction has committed version 2.
```

MVCC improves concurrency, but it does not remove the need to understand conflicts, isolation levels, and locking.

---

## Isolation vs. Locking

They are related but different.

**Isolation** is the guarantee you want.

**Locking** is one mechanism a database or application can use to achieve safe concurrency.

For example:

- use **pessimistic locking** when a transaction must reserve a row before deciding what to do;
- use **optimistic locking** when conflicts are uncommon and you prefer detecting them at write time;
- use **Serializable** when correctness depends on a broader set of reads and writes and you want the database to reject unsafe concurrent executions.

Do not automatically add `FOR UPDATE` everywhere just because concurrency exists.

---

## Choosing an Isolation Level

Start with the database default unless the business rule needs something stronger.

Ask:

1. Can two requests read the same state and both make a decision from it?
2. Is it acceptable for repeated reads in one transaction to return different committed values?
3. Does correctness depend on several rows or a query result rather than one row?
4. Can I express the rule as a constraint or conditional update instead?
5. Can the application safely retry the whole transaction if the database reports a conflict?

Typical examples:

| Situation | Possible approach |
| --- | --- |
| Decrement stock if available | Conditional `UPDATE` |
| Enforce unique email | Unique constraint |
| Edit a record where conflicts are rare | Optimistic locking |
| Reserve a scarce resource before complex work | Pessimistic locking |
| Complex invariant across many reads/writes | Serializable transaction |

---

## Common Mistakes

- **Assuming a transaction automatically solves concurrency.** A transaction gives atomicity; its isolation level decides concurrency behavior.
- **Using read-then-write without protection.** This creates check-then-act races.
- **Increasing isolation without understanding retries.** Serializable transactions can legitimately fail and must be retried.
- **Holding transactions open while calling external APIs.** This keeps database resources and locks alive unnecessarily and cannot make the external call atomic with the database.
- **Treating database-specific behavior as universal SQL behavior.** Isolation implementations differ.

---

## Key Takeaways

- Isolation controls interactions between **concurrent** transactions.
- Atomicity and isolation solve different problems.
- Read Committed is common, but application-level read-then-write logic can still race.
- Prefer constraints and atomic conditional writes when they directly express the business rule.
- Optimistic and pessimistic locking are concurrency-control strategies, not isolation levels.
- Serializable gives the strongest general isolation guarantee, but applications must support retries.

---

## Useful Links

- [PostgreSQL: Concurrency Control](https://www.postgresql.org/docs/current/mvcc.html)
- [PostgreSQL: Transaction Isolation](https://www.postgresql.org/docs/current/transaction-iso.html)
- [PostgreSQL: Explicit Locking](https://www.postgresql.org/docs/current/explicit-locking.html)
