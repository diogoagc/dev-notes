# Concurrency & Consistency

Notes about keeping data correct when multiple operations happen at the same time.

- [Transaction Isolation](isolation.md) — what concurrent transactions may observe and how isolation levels change those guarantees.
- [Optimistic Locking](optimistic-locking.md) — detect stale writes with versions or compare-and-swap when conflicts are relatively rare.
- [Pessimistic Locking](pessimistic-locking.md) — reserve data before acting when conflicts are likely or costly.

## Quick Comparison

| Concern | Main question | Typical mechanism |
| --- | --- | --- |
| [Atomicity](../../databases/atomicity.md) | What if one operation fails halfway? | Transaction / atomic write |
| [Isolation](isolation.md) | What if transactions overlap? | Isolation level / MVCC |
| [Optimistic Locking](optimistic-locking.md) | Did someone change this since I read it? | Version / conditional update |
| [Pessimistic Locking](pessimistic-locking.md) | Can I reserve this before I decide? | Row / explicit lock |
