---
tags:
  - adapter
  - mongodb
  - consumer
aliases:
  - MongoDB Consumer
  - mongoDBEventStoreConsumer
related:
  - "[[Consumer Architecture]]"
  - "[[Reactors]]"
  - "[[Projectors]]"
  - "[[MongoDB Checkpointing]]"
  - "[[MongoDB Event Store]]"
package: emmett-mongodb
---

# MongoDB Change Stream Consumer

The `mongoDBEventStoreConsumer()` uses [MongoDB Change Streams](https://www.mongodb.com/docs/manual/changeStreams/) to react to events asynchronously. This is the MongoDB adapter's mechanism for building read models in separate collections, sending notifications, triggering side effects, or integrating with external systems.

> [!warning] Requirements
> - **Replica set** -- change streams do not work on standalone `mongod` instances. For local development, use a single-node replica set.
> - **MongoDB 5+** -- version 6+ preferred for `fullDocument: 'whenAvailable'` support; version 5 uses `fullDocument: 'updateLookup'`; below version 5 throws `EmmettError`.

## Setting Up a Consumer

```typescript
import {
  mongoDBEventStoreConsumer,
} from '@event-driven-io/emmett-mongodb';

const consumer = mongoDBEventStoreConsumer({
  connectionString: 'mongodb://localhost:27017/',
  clientOptions: { directConnection: true },
});

// Or with an existing client:
const consumer = mongoDBEventStoreConsumer({ client: mongoClient });
```

## Registering Processors

Register [[Reactors|reactors]] (general-purpose message handlers) or [[Projectors|projectors]] (projection-focused handlers) before starting the consumer.

### Reactor

A reactor handles messages with arbitrary side effects:

```typescript
consumer.reactor<ShoppingCartEvent>({
  processorId: 'email-notification-reactor',
  connectionOptions: { client: mongoClient },
  eachMessage: async (event) => {
    if (event.type === 'ShoppingCartConfirmed') {
      await sendConfirmationEmail(event.data);
    }
  },
});
```

### Projector

A projector is semantically identical to a reactor but communicates the intent to build a read model:

```typescript
consumer.projector<ShoppingCartEvent>({
  projectionName: 'shopping-cart-summary',
  connectionOptions: { connectionString: 'mongodb://localhost:27017/' },
  eachMessage: async (event, { client }) => {
    const db = client.db('read-models');
    await db.collection('cart_summaries').updateOne(
      { _id: event.metadata.streamName },
      { $set: { lastEventType: event.type } },
      { upsert: true },
    );
  },
});
```

Projectors default their `processorId` to `projection:{projectionName}`.

> [!note] Handler Context
> Both reactor and projector `eachMessage` handlers receive a `MongoDBProcessorHandlerContext` with a `client: MongoClient` property. Each processor creates its own client if a `connectionString` is provided, or uses the provided `client`.

## Consumer Lifecycle

```typescript
// Start consuming (requires at least one registered processor)
await consumer.start();

// Check status
console.log(consumer.isRunning); // true

// Stop consuming (closes the change stream)
await consumer.stop();

// Close the consumer (stops + closes the MongoDB client if internally created)
await consumer.close();
```

> [!warning] At Least One Processor Required
> Starting a consumer without any registered processors throws `EmmettError('Cannot start consumer without at least a single processor')`.

### Lifecycle Details

1. `start()` -- connects, calls `start()` on all processors to get their last checkpoints, determines the earliest checkpoint via `zipMongoDBMessageBatchPullerStartFrom()`, opens a change stream from that position
2. The change stream watches all collections matching `^emt:` (excluding `emt:processors`)
3. Each change is processed by extracting events and passing batches to processors
4. `stop()` -- signals the change stream to close, waits for completion
5. `close()` -- stops + closes the MongoDB client if it was internally created

## What the Consumer Watches

The change stream consumer watches ==all collections matching `^emt:`== at the database level, excluding `emt:processors` (which stores [[MongoDB Checkpointing|checkpoints]]). It does not filter by specific stream types.

This means every registered processor receives events from **all** stream types. Processors must filter for the event types they care about via `canHandle` or conditional logic in `eachMessage`.

### Change Stream Pipeline

The internal pipeline matches:
- Collection names matching `/^emt:/` (and not `emt:processors`)
- Only `insert` and `update` operation types

Event extraction depends on the change type:
- **Insert** -- all events from `fullDocument.messages`
- **Update** -- new events from `updateDescription.updatedFields` where keys start with `messages.`

## Resilience

The consumer automatically retries on transient errors with exponential backoff:

| Setting | Default |
|---|---|
| Retries | Forever |
| Minimum timeout | 100ms |
| Backoff factor | 1.5x |
| Stops on | Unrecoverable errors (DNS failures, closed topology) |

Customize retry behavior:

```typescript
const consumer = mongoDBEventStoreConsumer({
  connectionString: 'mongodb://localhost:27017/',
  resilience: {
    resubscribeOptions: {
      forever: true,
      minTimeout: 500,
      factor: 2,
    },
  },
});
```

> [!note] Unrecoverable Errors
> The consumer stops retrying for errors matching `isDatabaseUnavailableError()`: DNS resolution failures (`getaddrinfo ENOTFOUND`, `getaddrinfo EAI_AGAIN`) and `'Topology is closed'`. All other errors trigger automatic resubscription.

## Consumer Options

```typescript
type MongoDBConsumerOptions<ConsumerMessageType> = {
  consumerId?: string;                 // auto-generated UUID if not provided
  processors?: MessageProcessor[];     // pre-registered processors
  resilience?: {
    resubscribeOptions?: AsyncRetryOptions;
  };
} & (
  | { connectionString: string; clientOptions?: MongoClientOptions }
  | { client: MongoClient }
);
```

## Consumer API

```typescript
type MongoDBEventStoreConsumer<ConsumerMessageType> = {
  consumerId: string;
  isRunning: boolean;
  processors: MessageProcessor[];
  reactor<MessageType>(options): MongoDBProcessor<MessageType>;
  projector<EventType>(options): MongoDBProcessor<EventType>;
  start(): Promise<void>;
  stop(): Promise<void>;
  close(): Promise<void>;
};
```

## Idempotency Notes

- Multiple `start()` calls return the same promise (idempotent)
- Multiple `stop()` calls are safe (idempotent)
- If `consumerId` is not provided, a UUID is auto-generated

> [!info] No Workflow Processor
> Unlike [[PostgreSQL Consumer|PostgreSQL]] and [[SQLite Consumer|SQLite]], the MongoDB consumer does ==not== support `workflowProcessor()`. See [[Workflow Processor]] for details on which adapters support workflows.
