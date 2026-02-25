---
tags:
  - adapter
  - eventstoredb
  - consumer
aliases:
  - EventStoreDB Consumer
  - eventStoreDBEventStoreConsumer
related:
  - "[[Consumer Architecture]]"
  - "[[EventStoreDB Event Store]]"
  - "[[EventStoreDB Reconnection]]"
package: emmett-esdb
---

# EventStoreDB Consumer

The EventStoreDB consumer uses ==push-based streaming subscriptions== instead of the polling approach used by [[SQLite Consumer|SQLite]] and [[PostgreSQL Consumer|PostgreSQL]]. Events are delivered as they arrive via gRPC streaming, processed sequentially through a Node.js `Transform` stream.

A key difference: the ESDB consumer uses ==in-memory processors== only. Checkpoint state is stored in-memory and lost when the process restarts.

## Creating a Consumer

From the [[EventStoreDB Event Store|event store]]:

```typescript
const consumer = eventStore.consumer({
  from: { stream: '$all' },
});
```

Standalone with a connection string:

```typescript
import {
  eventStoreDBEventStoreConsumer,
  $all,
} from '@event-driven-io/emmett-esdb';

const consumer = eventStoreDBEventStoreConsumer({
  connectionString: 'esdb://localhost:2113?tls=false',
  from: { stream: $all },
});
```

Or with an existing client:

```typescript
const consumer = eventStoreDBEventStoreConsumer({
  client: myEventStoreDBClient,
  from: { stream: $all },
});
```

## Subscription Sources

The consumer supports three subscription modes:

### Subscribe to `$all`

Receive all events across all streams:

```typescript
import { $all } from '@event-driven-io/emmett-esdb';

const consumer = eventStoreDBEventStoreConsumer({
  connectionString,
  from: { stream: $all },
});
```

System events are automatically excluded via `excludeSystemEvents()` filter.

### Subscribe to a Named Stream

Receive events from a specific stream:

```typescript
const consumer = eventStoreDBEventStoreConsumer({
  connectionString,
  from: { stream: 'guestStay-guest42' },
});
```

### Subscribe to a Category Projection

Receive events from all streams of a type via `$ce-` projections:

```typescript
const consumer = eventStoreDBEventStoreConsumer({
  connectionString,
  from: {
    stream: '$ce-guestStay',
    options: { resolveLinkTos: true },
  },
});
```

> [!warning]
> When subscribing to `$ce-` category projections, you ==must== pass `resolveLinkTos: true`. Without it, you receive link events rather than the actual event data.

## Reactors

Reactors process each event with a callback. Unlike SQLite/PostgreSQL reactors, these use ==in-memory== processor infrastructure:

```typescript
const consumer = eventStoreDBEventStoreConsumer({
  connectionString,
  from: { stream: $all },
});

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

### Starting from a Position

```typescript
import { bigIntProcessorCheckpoint } from '@event-driven-io/emmett';

consumer.reactor<GuestStayEvent>({
  processorId: 'guest-notifications',
  startFrom: {
    lastCheckpoint: bigIntProcessorCheckpoint(lastKnownPosition),
  },
  eachMessage: async (event) => {
    // ...
  },
});
```

Use `startFrom: 'CURRENT'` to resume from the last stored checkpoint, or start from the beginning if none exists.

## Projectors

Projectors build read models using in-memory processor infrastructure:

```typescript
consumer.projector<GuestStayEvent>({
  processorId: 'guest-stay-summary',
  projection: myInMemoryProjection,
});
```

> [!info]
> Since processors are in-memory, projected state is lost on restart. For durable projections, consider using the [[SQLite Consumer|SQLite]] or [[PostgreSQL Consumer|PostgreSQL]] adapter.

## No Workflow Processor

Unlike SQLite and PostgreSQL, the ESDB adapter does ==not provide `workflowProcessor()`==. For multi-step orchestration, use a database-backed adapter. See [[Workflow Processor]].

## Consumer Lifecycle

```typescript
const consumer = eventStoreDBEventStoreConsumer({
  connectionString,
  from: { stream: $all },
});

consumer.reactor({ /* ... */ });

// Start the subscription
await consumer.start();

// Stop the subscription
await consumer.stop();

// close() is the same as stop() for ESDB
await consumer.close();
```

> [!warning]
> Consumers require at least one processor. Calling `start()` with no processors throws an `EmmettError`.

## Sequential Processing

Events are processed ==one at a time== through a `SubscriptionSequentialHandler` Transform stream. Even though a `batchSize` configuration exists (default: 100), each event is processed individually -- `eachBatch([message])` is called with a single-element array.

> [!note]
> This sequential processing means the `batchSize` config has no practical effect on event processing. Events flow through the pipeline one at a time regardless of the setting.

## Subscription Engine

Under the hood, the consumer uses `eventStoreDBSubscription()` which:

1. **Subscribes** via `client.subscribeToAll()` or `client.subscribeToStream()` with the appropriate start position
2. **Processes** via Node.js `stream.pipeline()` with a sequential Transform handler
3. **Reconnects** automatically on failure via [[EventStoreDB Reconnection|exponential backoff]]

> [!warning]
> If an `eachMessage` handler throws, it ==stops the subscription entirely==. The error is propagated and processing ends. Unlike polling-based adapters, there is no automatic retry for handler errors -- only connection-level reconnection.

## See Also

- [[Consumer Architecture]] -- The consumer/processor model shared across adapters
- [[EventStoreDB Event Mapping]] -- How ESDB events are mapped to Emmett format
- [[EventStoreDB Reconnection]] -- Retry and resilience configuration
- [[Reactors]] -- General reactor pattern
- [[Checkpointing]] -- How checkpointing works (in-memory for ESDB)
