---
tags:
  - moc
  - event-store
aliases:
  - Event Store
related:
  - "[[Event Store Interface]]"
  - "[[Core MOC]]"
  - "[[Adapters MOC]]"
  - "[[Projections MOC]]"
package: emmett
---

# Event Store

The ==`EventStore`== interface is the central abstraction in Emmett. It provides four methods for reading, writing, and aggregating event streams with built-in concurrency control. Every database adapter implements this same interface, so the patterns you learn here apply across all backends.

## Notes in This Folder

- [[Event Store Interface]] -- The common four-method interface all adapters implement
- [[Reading Streams]] -- `readStream` method, options, and result types
- [[Appending Events]] -- `appendToStream` method, atomic writes, execution order
- [[Aggregating Streams]] -- `aggregateStream` method, evolve/initialState, time-travel queries
- [[Concurrency Control]] -- Optimistic concurrency with expected version sentinels and exact matching
- [[Event Metadata]] -- System-assigned metadata on read events (`messageId`, `streamPosition`, `globalPosition`)
- [[Schema Versioning]] -- Upcasting (read-side) and downcasting (write-side) event transforms
- [[After-Commit Hooks]] -- Fire-and-forget hooks after successful appends
- [[Before-Commit Hooks]] -- Transactional hooks that run inside the database commit
- [[Inline Projections]] -- Synchronous read model updates during `appendToStream`
- [[In-Memory Event Store]] -- `getInMemoryEventStore()` for development, testing, and prototyping
- [[In-Memory Database]] -- Document store for querying inline projection results
- [[Sessions]] -- Transactional session abstraction with `withSession` callback
- [[Global Subscription Events]] -- Internal events for subscription coordination

## See Also

- [[Core MOC]] -- Type system foundations (Events, Commands, Messages)
- [[The Decider Pattern]] -- The decide/evolve/initialState pattern that `aggregateStream` implements
- [[Projections MOC]] -- Full projection system (inline and async) across all adapters
- [[Adapters MOC]] -- Database-specific implementations of this interface
- [[Command Handling]] -- Wiring Deciders to the event store
