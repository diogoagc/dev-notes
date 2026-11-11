# Dev Notes

A personal engineering knowledge base for programming notes, architecture concepts, design patterns, databases, frameworks, tools, and other technical references worth keeping.

The goal is to keep notes practical, connected, and easy to revisit: what something is, why it matters, when to use it, what can go wrong, and how it relates to other concepts.

---

## Explore

- **[Languages](languages/index.md)**
  - [PHP](languages/php/index.md)
  - [Go](languages/go/index.md)
  - [Python](languages/python/index.md)
  - [C](languages/c/index.md)

- **[Frameworks](frameworks/index.md)**
  - [Laravel](frameworks/laravel/index.md)

- **[CMS](cms/index.md)**
  - [WordPress](cms/wordpress/index.md)

- **[Databases](databases/index.md)**
  - [Atomicity and Atomic Operations](databases/atomicity.md)

- **[Architecture](architecture/index.md)**
  - [Idempotency](architecture/idempotency.md)

- **[Concepts](concepts/index.md)**
  - [Concurrency & Consistency](concepts/concurrency/index.md)
    - [Transaction Isolation](concepts/concurrency/isolation.md)
    - [Optimistic Locking](concepts/concurrency/optimistic-locking.md)
    - [Pessimistic Locking](concepts/concurrency/pessimistic-locking.md)

- **[Design Patterns](design-patterns/index.md)**
  - [Transactional Outbox Pattern](design-patterns/transactional-outbox.md)
  - [Distributed Systems](design-patterns/distributed-systems/index.md)
    - [Saga Pattern](design-patterns/distributed-systems/saga.md)

- **[DevOps](devops/index.md)**

- **[Tools](tools/index.md)**

---

## Notes Structure

Notes do not need to follow a rigid template, but most should answer the questions that make them useful later:

- What is it?
- What problem does it solve?
- Why does it matter?
- How does it work?
- What can go wrong?
- When should I use it?
- When should I avoid it?
- What concepts or patterns are related?
- What should I remember six months from now?

Common sections include:

- explanation and mental model;
- practical examples;
- failure scenarios;
- trade-offs and limitations;
- implementation approaches;
- common mistakes;
- related notes;
- key takeaways;
- useful links.

---

## Conventions

### Language

Notes are written in **English**.

### File Names

Use lowercase kebab-case:

```text
transactional-outbox.md
optimistic-locking.md
pessimistic-locking.md
```

### Navigation

- Every top-level section has an `index.md`.
- Subsections that contain several notes should also have an `index.md`.
- The root `README.md` links to every section and note for fast navigation.
- Related concepts should cross-link to each other where useful.

### Organization

Prefer a small number of broad top-level sections.

More specific areas such as **Concurrency**, **Consistency**, or **Distributed Systems** should live inside those sections rather than becoming new top-level folders unless they grow large enough to justify it.

---

This repository is a living engineering reference: curated notes about concepts and practices worth understanding, not exhaustive documentation for every technology.