---
tags:
  - moc
  - adapter
  - postgresql
aliases:
  - PostgreSQL Adapter
related:
  - "[[Adapters MOC]]"
  - "[[Event Store Interface]]"
  - "[[Adapter Comparison]]"
package: emmett-postgresql
---

# PostgreSQL Adapter

The PostgreSQL adapter (`@event-driven-io/emmett-postgresql`) is the most full-featured database adapter in Emmett. It provides an event store backed by PostgreSQL with global position tracking, optimistic concurrency, schema auto-migration, distributed processor locks, Pongo document projections, and full consumer/processor/reactor/workflow support.

## Installation

```typescript
npm install @event-driven-io/emmett-postgresql
# Peer dependencies:
npm install @event-driven-io/emmett @event-driven-io/dumbo @event-driven-io/pongo
```

## Notes in This Section

- [[PostgreSQL Event Store]] -- Factory function, connection modes, schema migration, core operations, sessions
- [[PostgreSQL Projections]] -- Pongo (single/multi stream), raw SQL (per-event/batch), generic projections, reading projected data
- [[PostgreSQL Consumer]] -- Polling-based consumer, async projectors, reactors, workflow processors, handler context
- [[PostgreSQL Distributed Locking]] -- Advisory locks for processors and inline projection coordination
- [[PostgreSQL Schema]] -- Database tables, SQL functions, global sequence, partitioning, migration history
- [[PostgreSQL Rebuilding Projections]] -- Replaying all events to rebuild read models from scratch
- [[PostgreSQL Testing]] -- BDD-style `PostgreSQLProjectionSpec`, SQL assertions, Pongo assertion helpers

## Key Concepts

The PostgreSQL adapter introduces several concepts beyond the core [[Event Store Interface]]:

- ==Global position tracking== via a PostgreSQL sequence (`emt_global_message_position`), enabling ordered cross-stream consumption
- ==Inline projections== that execute within the `appendToStream` transaction for strong read-after-write consistency
- ==Async projections== via a polling-based [[PostgreSQL Consumer|consumer]] with checkpoint-based resumption
- ==Distributed locking== via PostgreSQL advisory locks for safe multi-instance deployments
- ==Pongo document projections== -- a MongoDB-like API over PostgreSQL JSONB via `@event-driven-io/pongo`
- ==Table partitioning== for multi-tenancy support on all schema tables

## See Also

- [[Event Store Interface]] -- The common interface all adapters implement
- [[Projection Concepts]] -- How projections work across all adapters
- [[Consumer Architecture]] -- The consumer/processor model
- [[Concurrency Control]] -- Optimistic concurrency with expected versions
- [[Adapter Comparison]] -- Feature matrix across all adapters
