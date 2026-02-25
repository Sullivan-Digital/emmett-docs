---
tags:
  - projections
  - async
aliases:
  - Async Projection Processing
related:
  - "[[Projection Concepts]]"
  - "[[Projectors]]"
  - "[[Consumer Architecture]]"
  - "[[Checkpointing]]"
  - "[[Projection Rebuilding]]"
package: emmett
---

# Async Projections

> [!abstract]
> Async projections are decoupled from the write path and process events asynchronously via the consumer/projector pattern. They provide ==eventual consistency== but keep the write path fast and the projection logic independent.

## Projector via Consumer

A ==projector== is a specialized [[Reactors|reactor]] that wraps a `ProjectionDefinition`. It processes events one at a time through the projection's `handle` method, manages [[Checkpointing|checkpoints]], and optionally truncates on start.

```typescript
import { postgreSQLEventStoreConsumer } from '@event-driven-io/emmett-postgresql';

const consumer = postgreSQLEventStoreConsumer({ connectionString });

consumer.projector({
  projection: shortInfoProjection,
  // Optional: custom processor ID (defaults to emt:processor:projector:<projectionName>)
  processorId: 'shopping-cart-short-info-projector',
});

// Start consuming events
await consumer.start();
```

The consumer polls for new events (PostgreSQL/SQLite) or subscribes to a change stream (MongoDB) or native subscription (EventStoreDB), then dispatches them to registered projectors. Each projector maintains its own checkpoint so it can ==resume from where it left off== after restarts.

## How Projectors Work Internally

The `projector()` function wraps a `ProjectionDefinition` into a `MessageProcessor`:

1. **On start**: Reads the last checkpoint. If `truncateOnStart` is set, calls the projection's `truncate` function.
2. **On each event**: Calls `projection.handle([event], context)` (single event wrapped in an array), then stores a new checkpoint.
3. **Filtering**: Only processes events whose type is in the projection's `canHandle` array.
4. **Upcasting**: Supports event [[Schema Versioning|versioning/upcasting]] via `eventsOptions.schema`.

```typescript
// Simplified internal implementation
const projector = (options) => {
  const { projection, processorId, truncateOnStart, ...rest } = options;
  return reactor({
    ...rest,
    type: MessageProcessorType.PROJECTOR,
    canHandle: projection.canHandle,
    processorId: processorId ?? getProjectorId({ projectionName: projection.name }),
    hooks: {
      onStart: truncateOnStart && projection.truncate
        ? async (context) => projection.truncate(context)
        : undefined,
    },
    eachMessage: async (event, context) => projection.handle([event], context),
  });
};
```

## `truncateOnStart`

When `truncateOnStart: true` is set, the projector calls the projection's `truncate` function before processing any events. This clears all existing projection data and replays from the beginning.

```typescript
consumer.projector({
  projection: shortInfoProjection,
  truncateOnStart: true, // Clear all data before replaying
});
```

The `truncate` function is part of the `ProjectionDefinition` interface. Its behavior depends on the adapter:
- **Pongo**: Calls `collection.deleteMany()`
- **InMemory**: Iterates documents and calls `deleteOne` on each
- **Raw SQL**: Executes the projection's custom truncate SQL

> [!tip]
> For a dedicated rebuild workflow with automatic stop, see [[Projection Rebuilding]].

## InMemory Projector

For testing and adapters without native projection stores (like EventStoreDB), use `inMemoryProjector`:

```typescript
import { inMemoryProjector } from '@event-driven-io/emmett';

consumer.projector(
  inMemoryProjector({
    projection: shortInfoProjection,
  }),
);
```

This creates a projector with in-memory checkpointing and processing scope -- useful for integration tests across all adapters.

## Consumer Mechanisms by Adapter

| Adapter | Mechanism | Notes |
|---|---|---|
| **PostgreSQL** | Polling | Configurable batch size and frequency, adaptive backoff |
| **SQLite** | Polling | Same pattern as PostgreSQL |
| **MongoDB** | Change streams | Requires replica set, MongoDB 5+ |
| **EventStoreDB** | Native subscriptions | Push-based streaming, in-memory processors only |

## When to Use Async Projections

Async projections are ideal when:
- The projection logic is expensive or slow
- Eventual consistency is acceptable
- You want to decouple read model updates from the write path
- You need to rebuild projections independently

For immediate read-after-write consistency, use [[Inline Projections Overview|inline projections]] instead.

## See Also

- [[Projection Concepts]] -- Core projection concepts and the `ProjectionDefinition` interface
- [[Inline Projections Overview]] -- The synchronous alternative
- [[Projection Rebuilding]] -- Truncate-and-replay strategies
- [[Consumer Architecture]] -- How consumers manage processors
- [[Projectors]] -- Consumer-side projector registration details
- [[Checkpointing]] -- How projectors track their position
