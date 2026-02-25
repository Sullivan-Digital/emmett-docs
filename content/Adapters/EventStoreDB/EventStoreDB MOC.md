---
tags:
  - moc
  - adapter
  - eventstoredb
aliases:
  - EventStoreDB Adapter
  - ESDB
related:
  - "[[Adapters MOC]]"
  - "[[Event Store Interface]]"
  - "[[Adapter Comparison]]"
package: emmett-esdb
---

# EventStoreDB Adapter

The EventStoreDB adapter (`@event-driven-io/emmett-esdb`) is a thin wrapper around the `@eventstore/db-client` SDK, mapping Emmett's [[Event Store Interface]] onto EventStoreDB's gRPC-based API. It is the lightest adapter -- no schema management, no connection pooling, no inline projections -- just a clean bridge between Emmett's types and EventStoreDB's native capabilities.

## Installation

```typescript
npm install @event-driven-io/emmett-esdb @event-driven-io/emmett @eventstore/db-client
```

Both `@event-driven-io/emmett` and `@eventstore/db-client` (v6.2.1+) are required peer dependencies.

## Notes in This Section

- [[EventStoreDB Event Store]] -- Factory function, core operations, concurrency mapping, event versioning (upcast + downcast)
- [[EventStoreDB Consumer]] -- Push-based streaming subscriptions, subscription sources (`$all`, named stream, category), in-memory processors
- [[EventStoreDB Event Mapping]] -- `mapFromESDBEvent()`, field-by-field mapping, checkpoint semantics
- [[EventStoreDB Reconnection]] -- Exponential backoff, gRPC error handling, custom retry configuration

## Key Concepts

The EventStoreDB adapter differs significantly from the database-backed adapters ([[PostgreSQL MOC|PostgreSQL]], [[SQLite MOC|SQLite]]):

- ==No schema management== -- EventStoreDB handles storage internally, no tables or migrations to manage
- ==Push-based subscriptions== instead of polling -- events are delivered as they arrive via gRPC streaming
- ==In-memory processors only== -- reactors and projectors use in-memory state; checkpoints are lost on restart
- ==No workflow processor== support -- for orchestration, use PostgreSQL or SQLite
- ==No inline projections== -- read models must be built asynchronously via the consumer
- ==Caller manages client lifecycle== -- the event store has no `close()` method
- ==Default stream version is `-1n`== -- EventStoreDB's native convention, unlike `0n` for PostgreSQL/SQLite
- ==Downcast support== -- the only adapter that supports write-time event transformation alongside read-time upcasting

## See Also

- [[Event Store Interface]] -- The common interface all adapters implement
- [[Consumer Architecture]] -- The consumer/processor model
- [[Concurrency Control]] -- Optimistic concurrency with expected versions
- [[Adapter Comparison]] -- Feature matrix across all adapters
- [[Schema Versioning]] -- Upcasting and downcasting concepts
