# Optimistic Locking

**Optimistic locking** assumes concurrent conflicts are relatively uncommon. Instead of locking a record before working with it, the application allows concurrent reads and verifies at write time that the data has not changed since it was read.

> **Key point:** Optimistic locking does not prevent another transaction from changing the row. It detects that the row changed and refuses to overwrite newer data silently.

**Related:** [Transaction Isolation](isolation.md) · [Pessimistic Locking](pessimistic-locking.md) · [Atomicity and Atomic Operations](../../databases/atomicity.md)

## Contents

- [The Problem](#the-problem)
- [How It Works](#how-it-works)
- [Version Columns](#version-columns)
- [Compare-and-Swap](#compare-and-swap)
- [Handling Conflicts](#handling-conflicts)
- [When to Use It](#when-to-use-it)
- [When Not to Use It](#when-not-to-use-it)
- [Common Mistakes](#common-mistakes)
- [Optimistic vs. Pessimistic Locking](#optimistic-vs-pessimistic-locking)
- [Key Takeaways](#key-takeaways)

---

## The Problem

Two users open the same record:

```text
Current document: version 7

User A reads version 7
User B reads version 7

User A saves changes
User B saves changes afterwards
```

Without conflict detection, User B can silently overwrite User A's update.

This is a **lost update**.

---

## How It Works

Store a version number with the record:

```text
id:      42
name:    "Draft"
version: 7
```

The client or application remembers that it read version `7`.

When saving:

```sql
UPDATE documents
SET
    name = 'Approved',
    version = 8
WHERE id = 42
  AND version = 7;
```

Then inspect the affected row count:

```text
1 row updated -> success
0 rows updated -> somebody changed the row first
```

The check and write happen in one database statement, so another writer cannot slip between them.

---

## Version Columns

A monotonically increasing integer is usually the clearest implementation:

```text
version = 1, 2, 3, 4, ...
```

A timestamp can sometimes be used, but version numbers are less ambiguous and do not depend on timestamp precision or clock behavior.

Typical flow:

```text
READ row with version = 12
↓
Modify data in memory
↓
UPDATE ... WHERE version = 12
↓
Success -> version becomes 13
Conflict -> 0 rows updated
```

ORMs often expose this as a built-in optimistic concurrency feature, but the underlying idea is still a conditional write.

---

## Compare-and-Swap

Optimistic locking is closely related to **compare-and-swap (CAS)**:

> Set the new value only if the value/version is still what I previously observed.

Conceptually:

```text
compare current version with expected version
if equal:
    write new state
else:
    report conflict
```

The database must enforce the comparison and update atomically. Do **not** implement it as:

```text
SELECT version
if version == expected:
    UPDATE row
```

That recreates a check-then-act race.

---

## Handling Conflicts

Detecting the conflict is only half the design. You also need a policy for what happens next.

Possible strategies:

### Retry Automatically

Useful when the operation can be safely recomputed from fresh data.

```text
read latest state
reapply operation
try conditional update again
```

Keep retries bounded.

### Return a Conflict to the Client

For user edits, returning a conflict is often better than guessing how to merge data.

An HTTP API might return `409 Conflict` and include the latest representation or version.

### Merge

Useful when changes are independent and can be combined safely, but merging business state is application-specific.

### Abort

For high-risk operations, rejecting stale work may be the safest behavior.

---

## When to Use It

Optimistic locking is a good fit when:

- concurrent edits are possible but **conflicts are uncommon**;
- reads may happen long before the final write;
- you do not want to hold database locks while a user thinks or an application performs other work;
- silent overwrites would be incorrect;
- retrying or surfacing a conflict is acceptable.

Common examples:

- editing a customer or product record;
- updating configuration;
- APIs where clients send a version or ETag;
- background workers that should only update the state they originally observed.

---

## When Not to Use It

It is less attractive when:

- conflicts are frequent, causing repeated retries;
- a resource must be **reserved immediately** so nobody else can take it;
- the operation involves a scarce item and the cost of losing the race is high;
- the application cannot safely repeat the operation after reloading state.

In those cases, [Pessimistic Locking](pessimistic-locking.md) or a different atomic database operation may be a better fit.

---

## Common Mistakes

- **Checking the version in application code and then issuing an unconditional update.** The comparison belongs in the `UPDATE` condition.
- **Ignoring zero affected rows.** That is the conflict signal, not a successful no-op.
- **Automatically retrying non-idempotent side effects.** Recomputing a database update may be safe; charging a card or sending an email again may not be.
- **Using timestamps with insufficient precision.** Two updates can accidentally share the same timestamp.
- **Assuming optimistic locking protects multiple records automatically.** A single row version only protects that row unless you design a broader version/invariant.

---

## Optimistic vs. Pessimistic Locking

| | Optimistic | Pessimistic |
| --- | --- | --- |
| Assumption | Conflicts are rare | Conflicts are likely or expensive |
| Lock held while working | No | Usually yes |
| Conflict behavior | Detect at write time | Other transactions block/fail earlier |
| Throughput | Usually better when conflicts are rare | Can reduce concurrency |
| Retry requirement | Common | Possible, especially after deadlocks/timeouts |
| Typical mechanism | Version/CAS conditional update | `SELECT ... FOR UPDATE` / explicit lock |

---

## Key Takeaways

- Optimistic locking **detects** conflicting writes rather than preventing them.
- A version column plus a conditional `UPDATE` is the classic implementation.
- The version check and write must be atomic.
- A conflict must lead to an explicit policy: retry, merge, reject, or abort.
- It works best when contention is low and holding locks would be wasteful.
