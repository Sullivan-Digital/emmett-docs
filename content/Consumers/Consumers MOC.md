---
tags:
  - moc
  - consumer
aliases:
  - Consumers
  - Processors
related:
  - "[[Consumer Architecture]]"
  - "[[Reactors]]"
  - "[[Projectors]]"
  - "[[Checkpointing]]"
  - "[[Graceful Shutdown]]"
  - "[[Workflows MOC]]"
package: emmett
---

# Consumers

Emmett's async processing model follows a ==consumer-processor== architecture. A **consumer** manages a message source -- polling loop, change stream, or native subscription -- and dispatches message batches to one or more **processors**. Each processor handles messages independently, maintains its own [[Checkpointing|checkpoint]], and auto-registers for [[Graceful Shutdown|graceful shutdown]].

## Notes in This Folder

- [[Consumer Architecture]] -- The consumer-processor model: one consumer, many processors, parallel dispatch via `Promise.allSettled`
- [[Reactors]] -- General-purpose message handlers with `eachMessage`, `canHandle` filtering, `startFrom` options, and lifecycle hooks
- [[Projectors]] -- Specialized reactors for building read models, wrapping a `ProjectionDefinition` with `truncateOnStart` support
- [[Checkpointing]] -- Position tracking via `ProcessorCheckpoint` branded strings, storage by adapter, and reset strategies
- [[Graceful Shutdown]] -- `onShutdown()` for automatic `SIGTERM`/`SIGINT` handling across Node.js, Bun, and Deno

## Processor Types at a Glance

| Type | Factory | Purpose |
|---|---|---|
| **Reactor** | `consumer.reactor()` | General-purpose side effects: emails, API calls, publishing |
| **Projector** | `consumer.projector()` | Read model building with projection definitions |
| **Workflow Processor** | `consumer.workflowProcessor()` | Stateful multi-step orchestration (see [[Workflows MOC]]) |

## See Also

- [[Workflows MOC]] -- Stateful orchestration built on the consumer-processor model
- [[Async Projections]] -- How projections use the consumer/projector pattern
- [[PostgreSQL Consumer]] -- Polling-based consumer with distributed locking
- [[MongoDB Change Stream Consumer]] -- Change stream-based consumer
- [[SQLite Consumer]] -- Polling-based consumer for SQLite
- [[EventStoreDB Consumer]] -- Native subscription-based consumer with in-memory checkpoints
