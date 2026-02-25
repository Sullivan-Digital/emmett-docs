---
tags:
  - moc
  - projections
aliases:
  - Projections
related:
  - "[[Projection Concepts]]"
  - "[[Inline Projections Overview]]"
  - "[[Async Projections]]"
  - "[[Projection Rebuilding]]"
  - "[[Projection Management]]"
  - "[[InMemory Projections]]"
  - "[[Pongo Document Projections]]"
  - "[[Raw SQL Projections]]"
  - "[[MongoDB Inline Projections]]"
  - "[[Testing Projections]]"
package: emmett
---

# Projections

Projections transform event streams into ==read models== -- query-optimized views of your data. Instead of replaying events every time you need to answer a query, projections maintain up-to-date denormalized views that can be read directly. Emmett supports two processing modes: **inline** (synchronous, same transaction) and **async** (eventual consistency via consumer/processor).

## Concepts

- [[Projection Concepts]] -- The `evolve` function, single-stream vs multi-stream, deletion semantics, and the `ProjectionDefinition` interface
- [[Inline Projections Overview]] -- Synchronous projections that run inside `appendToStream`, guaranteeing read-after-write consistency
- [[Async Projections]] -- Decoupled projection processing via the consumer/projector pattern

## Adapter-Specific Projections

- [[InMemory Projections]] -- `inMemorySingleStreamProjection()` and `inMemoryMultiStreamProjection()` for the in-memory event store
- [[Pongo Document Projections]] -- MongoDB-like document projections over PostgreSQL/SQLite JSONB via Pongo
- [[Raw SQL Projections]] -- Full SQL control with per-event and batch projection modes
- [[MongoDB Inline Projections]] -- Unique embedded architecture where read models live inside event stream documents

## Operations

- [[Projection Rebuilding]] -- Truncate-and-replay strategies for rebuilding projection data
- [[Projection Management]] -- PostgreSQL-specific activate/deactivate and metadata tracking

## Quality

- [[Testing Projections]] -- BDD-style given/when/then specs for every adapter

## See Also

- [[Event Store MOC]] -- The event store interface that projections build on
- [[Consumers MOC]] -- The consumer/processor architecture that powers async projections
- [[Projectors]] -- Consumer-side projector registration
- [[Evolve Function]] -- The core state reducer shared between deciders and projections
