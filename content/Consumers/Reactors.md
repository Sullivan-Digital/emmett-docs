---
tags:
  - consumer
  - reactor
aliases:
  - Reactor
  - consumer.reactor
related:
  - "[[Consumer Architecture]]"
  - "[[Message Handlers]]"
  - "[[Checkpointing]]"
package: emmett
---

# Reactors

> [!abstract]
> A reactor is the most fundamental processor type. It receives messages one at a time and executes arbitrary logic -- sending emails, calling APIs, publishing to other systems, or any side effect you need.

## Creating a Reactor

Register a reactor on a consumer using `consumer.reactor()`:

```typescript
import { postgreSQLEventStoreConsumer } from '@event-driven-io/emmett-postgresql';
import type { Event } from '@event-driven-io/emmett';

type GuestCheckedIn = Event<'GuestCheckedIn', { guestId: string }>;
type GuestCheckedOut = Event<'GuestCheckedOut', { guestId: string }>;
type GuestStayEvent = GuestCheckedIn | GuestCheckedOut;

const consumer = postgreSQLEventStoreConsumer({
  connectionString: 'postgresql://localhost:5432/mydb',
});

consumer.reactor<GuestStayEvent>({
  processorId: 'guest-notifications',
  eachMessage: async (message, context) => {
    if (message.type === 'GuestCheckedIn') {
      await sendWelcomeEmail(message.data.guestId);
    }
    // context.execute, context.connection.pool available for PG
  },
});

await consumer.start();
```

> [!warning] Processor ID stability
> The `processorId` must be a ==stable string==. It is the key for [[Checkpointing|checkpoint storage]]. Changing it causes the processor to reprocess all messages from the beginning.

## Filtering Events with `canHandle`

Use `canHandle` to restrict which event types a processor receives:

```typescript
consumer.reactor<GuestStayEvent>({
  processorId: 'checkin-only-reactor',
  canHandle: ['GuestCheckedIn'],
  eachMessage: async (message) => {
    // Only GuestCheckedIn events arrive here
  },
});
```

> [!note] Checkpoint advancement
> When `canHandle` is set, the processor still receives all messages in the batch from the consumer, but internally skips events whose type is not in the list. The [[Checkpointing|checkpoint]] still advances past skipped events.

## Controlling Start Position

The `startFrom` option determines where a processor begins reading:

```typescript
import { bigIntProcessorCheckpoint } from '@event-driven-io/emmett';

// Start from the very beginning (default for new processors)
consumer.reactor({ processorId: 'r', startFrom: 'BEGINNING', eachMessage: handler });

// Start from the end (only process new events)
consumer.reactor({ processorId: 'r', startFrom: 'END', eachMessage: handler });

// Resume from stored checkpoint, or BEGINNING if none exists
consumer.reactor({ processorId: 'r', startFrom: 'CURRENT', eachMessage: handler });

// Start from a specific position
consumer.reactor({
  processorId: 'r',
  startFrom: { lastCheckpoint: bigIntProcessorCheckpoint(42n) },
  eachMessage: handler,
});
```

> [!tip]
> `'CURRENT'` is the most common choice for production. On first run it behaves like `'BEGINNING'`. After the processor has stored checkpoints, it resumes from where it left off.

## Stopping a Processor

Use `stopAfter` to automatically stop the processor when a condition is met:

```typescript
consumer.reactor<GuestStayEvent>({
  processorId: 'one-time-processor',
  stopAfter: (message) =>
    message.metadata.globalPosition === targetPosition,
  eachMessage: handler,
});
```

When the predicate returns `true`, the processor signals `STOP` to the consumer. When all processors stop, the consumer's `start()` promise resolves.

## Lifecycle Hooks

Processors support `onInit`, `onStart`, and `onClose` hooks:

```typescript
consumer.reactor({
  processorId: 'my-reactor',
  eachMessage: handler,
  hooks: {
    onInit: async (context) => {
      // Called once during consumer.init() or first start
    },
    onStart: async (context) => {
      // Called each time the processor starts (after checkpoint read)
    },
    onClose: async (context) => {
      // Called on stop or shutdown
    },
  },
});
```

| Hook | When | Use Case |
|---|---|---|
| `onInit` | Once during initialization | Schema setup, one-time configuration |
| `onStart` | Each time the processor starts | Acquiring resources, logging start position |
| `onClose` | On stop or shutdown | Releasing resources, cleanup |

## Handler Context by Adapter

The `context` parameter in `eachMessage` varies by adapter:

| Adapter | Context Properties |
|---|---|
| **PostgreSQL** | `context.execute` (SQL executor in transaction), `context.connection.pool`, `context.connection.client`, `context.connection.transaction`, `context.connection.messageStore` |
| **SQLite** | `context.execute` (SQL executor in transaction), `context.connection` (SQLite connection) |
| **MongoDB** | `context.client` (MongoClient instance) |
| **EventStoreDB** | `context.database` (InMemoryDatabase) |

> [!info] Transaction wrapping
> PostgreSQL and SQLite wrap each `handle()` call in a ==database transaction==. MongoDB and EventStoreDB do not provide transaction wrapping.

## `eachBatch` Caveat

> [!warning]
> The `HandlerOptions` type includes both `eachMessage` and `eachBatch`, but the `reactor()` implementation only uses `eachMessage`. Providing only `eachBatch` results in a ==no-op handler== -- the batch handler is never called.

## See Also

- [[Consumer Architecture]] -- How consumers coordinate processors
- [[Projectors]] -- A specialized reactor for read model building
- [[Checkpointing]] -- How position tracking works
- [[Graceful Shutdown]] -- Automatic shutdown handling
- [[Message Handlers]] -- The full handler type taxonomy
