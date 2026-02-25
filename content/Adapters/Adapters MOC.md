---
tags:
  - moc
  - adapter
aliases:
  - Database Adapters
related:
  - "[[Event Store Interface]]"
  - "[[Adapter Comparison]]"
  - "[[Consumers MOC]]"
package: emmett
---

# Adapters

All database adapters implement the common [[Event Store Interface]] and provide `consumer()` with `projector()` / `reactor()` support. Each adapter has its own strengths and trade-offs.

## Adapter MOCs

- [[PostgreSQL MOC]] -- Full-featured: global position, streaming, distributed locks, schema migrations, Pongo projections
- [[MongoDB MOC]] -- Change stream consumers, inline projections embedded in stream documents, flexible storage strategies
- [[SQLite MOC]] -- Multi-driver (native sqlite3, Cloudflare D1), global position tracking, lightweight
- [[EventStoreDB MOC]] -- Thin adapter over `@eventstore/db-client`, native subscription-based consumers

## Quick Comparison

| Feature | PostgreSQL | MongoDB | SQLite | EventStoreDB |
|---|---|---|---|---|
| Consumer mechanism | Polling | Change streams | Polling | Native subscriptions |
| Global position | Yes (sequence) | No | Yes (auto-increment) | Yes (native) |
| Inline projections | Pongo + Raw SQL | Embedded in document | Pongo + Raw SQL | No |
| Distributed locking | Advisory locks | No | No | No |
| Workflow processor | Yes | No | Yes | No |
| Schema management | Auto-migration | Auto-index | Auto-migration | None needed |
| Default stream version | `0n` | `0n` | `0n` | `-1n` |

> [!info] See [[Adapter Comparison]] for a comprehensive feature matrix comparing all adapters including the [[In-Memory Event Store]].

## See Also

- [[Event Store Interface]] -- The common interface all adapters implement
- [[Concurrency Control]] -- Per-adapter differences in version checking
- [[Consumer Architecture]] -- How consumers work across adapters
