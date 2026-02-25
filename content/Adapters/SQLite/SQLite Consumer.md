---
tags:
  - adapter
  - sqlite
  - consumer
aliases:
  - SQLite Consumer
  - sqliteEventStoreConsumer
related:
  - "[[Consumer Architecture]]"
  - "[[Reactors]]"
  - "[[Projectors]]"
  - "[[Workflow Processor]]"
  - "[[SQLite Event Store]]"
package: emmett-sqlite
---

# SQLite Consumer

The SQLite consumer provides ==polling-based== asynchronous event processing. It groups one or more processors (reactors, projectors, workflow processors) that each handle event batches independently. Checkpoints are persisted in the `emt_processors` table, so processing resumes where it left off after restarts.

## Creating a Consumer

From the [[SQLite Event Store|event store]] directly:

```typescript
const consumer = eventStore.consumer({
  pulling: {
    batchSize: 100,           // default: 100
    pullingFrequencyInMs: 50, // default: 50ms
  },
});
```

Or standalone (useful when you only need the consumer, not the full event store):

```typescript
import { sqliteEventStoreConsumer } from '@event-driven-io/emmett-sqlite';
import { sqlite3EventStoreDriver } from '@event-driven-io/emmett-sqlite/sqlite3';

const consumer = sqliteEventStoreConsumer({
  driver: sqlite3EventStoreDriver,
  fileName: './events.db',
});
```

## Polling and Adaptive Backoff

The consumer uses a polling-based batch puller that reads from `emt_messages` ordered by `global_position`:

- Starts polling at ==100ms== intervals
- When no messages are found, ==doubles the wait time== up to 1000ms
- Resets to `pullingFrequencyInMs` (default: 50ms) when messages arrive

```typescript
const consumer = eventStore.consumer({
  pulling: {
    batchSize: 200,
    pullingFrequencyInMs: 100,
  },
  stopWhen: {
    noMessagesLeft: true, // Stops when no more messages to process
  },
});
```

> [!tip]
> The `stopWhen.noMessagesLeft` option is useful for batch processing and [[Projection Rebuilding|projection rebuilds]] where you want the consumer to exit after processing all existing events.

## Reactors

Reactors handle each event with a callback. They are the most general-purpose processor type:

```typescript
type GuestCheckedIn = Event<'GuestCheckedIn', { guestId: string }>;
type GuestCheckedOut = Event<'GuestCheckedOut', { guestId: string }>;
type GuestStayEvent = GuestCheckedIn | GuestCheckedOut;

const consumer = eventStore.consumer();

consumer.reactor<GuestStayEvent>({
  processorId: 'guest-notifications',
  eachMessage: async (event) => {
    switch (event.type) {
      case 'GuestCheckedIn':
        await sendWelcomeEmail(event.data.guestId);
        break;
      case 'GuestCheckedOut':
        await sendFeedbackRequest(event.data.guestId);
        break;
    }
  },
});

await consumer.start();
```

### Resuming from a Position

```typescript
import { bigIntProcessorCheckpoint } from '@event-driven-io/emmett';

consumer.reactor<GuestStayEvent>({
  processorId: 'guest-notifications',
  startFrom: {
    lastCheckpoint: bigIntProcessorCheckpoint(lastKnownPosition),
  },
  eachMessage: async (event) => {
    // Process event...
  },
});
```

Use `startFrom: 'CURRENT'` to check for a stored checkpoint and resume from there, or start from the beginning if no checkpoint exists.

> [!note]
> When multiple processors have different start positions, the consumer's batch puller starts from the ==smallest checkpoint== across all processors (via `zipSQLiteEventStoreMessageBatchPullerStartFrom()`).

## Projectors

Projectors build read models from events using SQLite-backed checkpoint storage:

```typescript
consumer.projector({
  processorId: 'shopping-cart-summary',
  projection: myProjection, // A SQLiteProjectionDefinition
});
```

The projector calls `projection.init()` during initialization if defined, and wraps each batch in a transaction.

See [[SQLite Projections]] for details on defining projections.

## Workflow Processors

Workflow processors orchestrate multi-step processes. The SQLite adapter supports two modes:

- **Double-hop** (`separateInputInboxFromProcessing: true`): Input events are first stored in a workflow-specific stream with prefixed types (e.g., `GroupCheckoutWorkflow:InitiateGroupCheckout`), then processed in a second poll cycle. This provides at-least-once delivery guarantees.
- **Single-operation** (`separateInputInboxFromProcessing: false`): Input and outputs are appended together in a single operation.

Workflow stream names follow the pattern `workflow-{workflowName}-{workflowId}`.

> [!note]
> The workflow processor creates an ==internal `SQLiteEventStore`== instance with `autoMigration: 'None'` that shares the same connection pool. The schema must already exist.

See [[Workflow Processor]] and [[Separated Inbox Mode]] for detailed workflow documentation.

## Consumer Lifecycle

```typescript
const consumer = eventStore.consumer();
consumer.reactor({ /* ... */ });

// Start processing
await consumer.start();

// Stop processing gracefully
await consumer.stop();

// Stop and close the connection pool
await consumer.close();
```

> [!warning]
> Consumers require at least one processor. Calling `start()` without any processors throws an `EmmettError`.

## Checkpointing

Checkpoints are stored in the `emt_processors` table as ==zero-padded 19-digit strings==. The checkpointer uses optimistic concurrency on `last_processed_checkpoint`:

- On update: `UPDATE ... WHERE last_processed_checkpoint = {currentValue}`
- On first write: `INSERT` with unique constraint handling for idempotency
- Returns `{ success: true, newCheckpoint }` or `{ success: false, reason: 'IGNORED' | 'MISMATCH' }`

> [!tip]
> Use `bigIntProcessorCheckpoint()` and `parseBigIntProcessorCheckpoint()` from core `emmett` if you interact with checkpoint values directly.

## Processor Handler Context

All SQLite processors receive `SQLiteProcessorHandlerContext`:

```typescript
type SQLiteProcessorHandlerContext = {
  execute: SQLExecutor;
  connection: AnySQLiteConnection;
};
```

Each batch is wrapped in a SQLite transaction via `sqliteProcessingScope()`.

## See Also

- [[Consumer Architecture]] -- The consumer/processor model shared across adapters
- [[Reactors]] -- General-purpose event handler pattern
- [[Projectors]] -- Async projection processing pattern
- [[Checkpointing]] -- How checkpoint-based resumption works
- [[Graceful Shutdown]] -- Automatic shutdown handling for consumers
