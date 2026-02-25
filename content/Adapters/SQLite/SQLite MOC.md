---
tags:
  - moc
  - adapter
  - sqlite
aliases:
  - SQLite Adapter
related:
  - "[[Adapters MOC]]"
  - "[[Event Store Interface]]"
  - "[[Adapter Comparison]]"
package: emmett-sqlite
---

# SQLite Adapter

The SQLite adapter (`@event-driven-io/emmett-sqlite`) provides a full-featured [[Event Store Interface|EventStore]] implementation backed by SQLite. It supports two database drivers (native `sqlite3` and Cloudflare D1), global position tracking via auto-increment `INTEGER PRIMARY KEY`, polling-based consumers with reactors/projectors/workflow processors, inline and async projections (including Pongo document projections), and schema auto-migration.

## Installation

```typescript
npm install @event-driven-io/emmett-sqlite

// Pick a driver:
npm install sqlite3                     // Node.js native
npm install @cloudflare/workers-types   // Cloudflare D1
```

## Notes in This Section

- [[SQLite Event Store]] -- Factory function, configuration, lifecycle hooks, core operations, schema management
- [[SQLite Drivers]] -- The `EventStoreDriver` abstraction, `sqlite3EventStoreDriver`, `d1EventStoreDriver`
- [[SQLite Projections]] -- Inline projections: Pongo (single/multi/generic), raw SQL (per-event/batch)
- [[SQLite Consumer]] -- Polling-based consumer, reactors, projectors, workflow processors, adaptive backoff
- [[SQLite Testing]] -- BDD-style `SQLiteProjectionSpec`, Pongo assertion helpers, raw SQL assertions

## Key Concepts

The SQLite adapter shares many patterns with the [[PostgreSQL MOC|PostgreSQL adapter]] while being lightweight and embeddable:

- ==Multi-driver architecture== -- swap between Node.js `sqlite3` and Cloudflare D1 via the `EventStoreDriver` abstraction
- ==Global position tracking== via SQLite's `INTEGER PRIMARY KEY` on the `emt_messages` table (auto-increment ROWID alias)
- ==Inline projections== that execute within the `appendToStream` transaction for strong read-after-write consistency
- ==Polling-based consumer== with adaptive backoff, checkpoint-based resumption, and persistent checkpointing in `emt_processors`
- ==Pongo document projections== -- a MongoDB-like document API on top of SQLite via `@event-driven-io/pongo`
- ==Workflow processors== for multi-step orchestration with double-hop or single-operation modes
- ==Schema auto-migration== with `'CreateOrUpdate'` (default) or `'None'` for manual control

## See Also

- [[Event Store Interface]] -- The common interface all adapters implement
- [[Projection Concepts]] -- How projections work across all adapters
- [[Consumer Architecture]] -- The consumer/processor model
- [[Concurrency Control]] -- Optimistic concurrency with expected versions
- [[Adapter Comparison]] -- Feature matrix across all adapters
- [[PostgreSQL MOC]] -- The most similar adapter in terms of feature set
