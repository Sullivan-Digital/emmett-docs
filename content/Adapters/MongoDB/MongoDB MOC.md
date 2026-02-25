---
tags:
  - moc
  - adapter
  - mongodb
aliases:
  - MongoDB Adapter
related:
  - "[[Adapters MOC]]"
  - "[[MongoDB Event Store]]"
  - "[[MongoDB Storage Strategies]]"
  - "[[MongoDB Inline Projections]]"
  - "[[MongoDB Change Stream Consumer]]"
  - "[[MongoDB Checkpointing]]"
  - "[[MongoDB Testing]]"
package: emmett-mongodb
---

# MongoDB

The `@event-driven-io/emmett-mongodb` package implements the [[Event Store Interface]] using a ==single-document-per-stream== storage model. Each stream is a single MongoDB document containing all events in a `messages` array, along with metadata and inline projection results. This design trades the 16MB document size limit for atomic, consistent reads and writes without multi-document transactions.

## Notes in This Folder

- [[MongoDB Event Store]] -- `getMongoDBEventStore()` factory, two connection modes, core operations, lifecycle management
- [[MongoDB Storage Strategies]] -- Three collection layout options: per-stream-type (default), single collection, or custom routing
- [[MongoDB Inline Projections]] -- Read models computed atomically during append, stored inside the stream document
- [[MongoDB Change Stream Consumer]] -- Async event processing via MongoDB change streams with reactor and projector support
- [[MongoDB Checkpointing]] -- Resume token-based checkpoint storage in the `emt:processors` collection
- [[MongoDB Testing]] -- BDD projection specs with `MongoDBInlineProjectionSpec` and dummy MongoDB stubs

## Key Characteristics

| Feature | MongoDB |
|---|---|
| Default stream version | `0n` |
| Global position | No -- uses change stream resume tokens instead |
| Consumer mechanism | Change streams (requires replica set) |
| Inline projections | Embedded in stream document (`projections` field) |
| Workflow processor | Not supported |
| Schema management | Automatic unique index on `streamName` |

> [!warning] 16MB Document Size Limit
> All events for a stream live in a single MongoDB document. Long-lived or high-volume streams can exceed MongoDB's 16MB document limit. Design stream boundaries accordingly -- prefer shorter-lived streams.

## See Also

- [[Adapters MOC]] -- Overview of all database adapters
- [[Adapter Comparison]] -- Feature matrix across all backends
- [[Event Store Interface]] -- The common interface this adapter implements
- [[Concurrency Control]] -- Optimistic concurrency (MongoDB uses `metadata.streamPosition` in the update filter)
- [[Schema Versioning]] -- Upcasting and downcasting support
