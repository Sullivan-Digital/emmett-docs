---
tags:
  - adapter
  - postgresql
  - consumer
aliases:
  - PostgreSQL Consumer
  - postgreSQLEventStoreConsumer
related:
  - "[[Consumer Architecture]]"
  - "[[Reactors]]"
  - "[[Projectors]]"
  - "[[PostgreSQL Distributed Locking]]"
  - "[[Checkpointing]]"
package: emmett-postgresql
---

# PostgreSQL Consumer

The PostgreSQL consumer polls for new events and dispatches them to processors (projectors, reactors, or workflow processors). Each batch of events is processed within a PostgreSQL transaction, ensuring ==atomicity between event handling and checkpoint updates==.

## Creating a Consumer

```typescript
// From the event store
const consumer = eventStore.consumer({
  pulling: {
    batchSize: 100,            // default: 100
    pullingFrequencyInMs: 50,  // default: 50ms
  },
  stopWhen: { noMessagesLeft: true }, // stop when caught up (useful for rebuilds)
});

// Or standalone
import { postgreSQLEventStoreConsumer } from '@event-driven-io/emmett-postgresql';

const consumer = postgreSQLEventStoreConsumer({
  connectionString,
});
```

## Consumer Lifecycle

```typescript
await consumer.start();   // initializes processors, starts polling
// ... consumer runs, processing events ...
await consumer.stop();    // stops polling, waits for in-flight batches
await consumer.close();   // stops + closes the connection pool
```

The internal lifecycle follows these steps:

1. `start()` initializes all processors and starts the message puller
2. The message puller polls `emt_messages` in batches using `readMessagesBatch()`
3. Each batch is dispatched to all active processors in parallel via `Promise.allSettled()`
4. If all processors return `STOP`, the consumer stops
5. `stop()` signals the puller to stop, waits for completion, and closes processors
6. `close()` calls `stop()` and then closes the connection pool

> [!warning]
> Consumer and processor pools are separate. Each consumer creates its own connection pool unless one is provided. Processors can optionally use a different pool. Be mindful of connection limits.

## Async Projectors

```typescript
consumer.projector({
  projection: shoppingCartDetailsProjection,
  startFrom: 'CURRENT',       // resume from last checkpoint
  lock: {
    acquisitionPolicy: { type: 'fail' },  // default
    timeoutSeconds: 300,                   // default: 5 minutes
  },
  truncateOnStart: false,      // set true to clear projection data first
});

await consumer.start();
```

`startFrom` options:

| Value | Behavior |
|---|---|
| `'CURRENT'` | Resume from the processor's last saved checkpoint |
| `'BEGINNING'` | Process all events from global position 0 |
| `'END'` | Start from the current latest global position |

The projector's `processorId` is auto-generated from the projection name. Additional options include `stopAfter` (predicate to stop processing), `connectionOptions` (separate pool), `migrationOptions`, and lifecycle `hooks`.

See [[Projectors]] for the general projector concept and [[PostgreSQL Distributed Locking]] for lock details.

## Reactors

[[Reactors]] handle events with custom logic (side effects, sending emails, calling APIs):

```typescript
consumer.reactor({
  processorId: 'orderNotifier',  // required for reactors
  eachMessage: async (message, context) => {
    if (message.type === 'OrderPlaced') {
      await sendOrderConfirmationEmail(message.data);
    }
  },
  startFrom: 'CURRENT',
});
```

Or handle events in batches:

```typescript
consumer.reactor({
  processorId: 'batchReporter',
  eachBatch: async (messages, context) => {
    // Process all messages in the batch at once
  },
});
```

> [!note]
> Unlike projectors, reactors ==require an explicit `processorId`== since there is no projection name to derive it from.

## Workflow Processors

Workflow processors combine a [[Workflow Pattern|workflow definition]] with event-driven processing:

```typescript
consumer.workflowProcessor({
  workflow: myWorkflowDefinition,
  startFrom: 'CURRENT',
});
```

See [[Workflow Processor]] for the full workflow processing model.

## Processor Handler Context

All processor handlers receive a rich context:

```typescript
type PostgreSQLProcessorHandlerContext = {
  partition: string;
  execute: SQLExecutor;           // run raw SQL in the current transaction
  connection: {
    connectionString: string;
    client: PgClient;
    transaction: PgTransaction;
    pool: Dumbo;
    messageStore: PostgresEventStore;  // scoped to this transaction
  };
};
```

The ==`messageStore`== in the context is a full `PostgresEventStore` scoped to the processor's transaction connection. This allows processors to append events within the same transaction:

```typescript
consumer.reactor({
  processorId: 'orderSaga',
  eachMessage: async (message, context) => {
    if (message.type === 'PaymentReceived') {
      // Append to a different stream within the same transaction
      await context.connection.messageStore.appendToStream(
        `order-${message.data.orderId}`,
        [{ type: 'OrderFulfilled', data: { ... } }],
      );
    }
  },
});
```

> [!tip]
> Because the `messageStore` shares the processor's transaction, any events you append are atomic with the checkpoint update. This gives you exactly-once processing semantics for event-to-event workflows.

## Processing Scope

Each message batch runs within a PostgreSQL transaction:

```typescript
pool.withTransaction(async (transaction) => {
  const client = await transaction.connection.open();
  return handler({ execute: transaction.execute, connection: { ... } });
});
```

This ensures atomicity: checkpoint update + projection/reactor update happen in the same transaction.

## Pulling Configuration

```typescript
const consumer = eventStore.consumer({
  pulling: {
    batchSize: 200,             // events per batch (default: 100)
    pullingFrequencyInMs: 100,  // ms between polls when events found (default: 50)
  },
});
```

**Backoff behavior:** When no events are available, the poll interval doubles (starting at 100ms) up to a maximum of 1000ms. When events are found, it resets to `pullingFrequencyInMs`.

**Message visibility:** The consumer only reads committed, visible transactions using PostgreSQL's transaction snapshot mechanism:

```sql
WHERE transaction_id < pg_snapshot_xmin(pg_current_snapshot())
```

> [!warning]
> Events from long-running transactions are invisible to consumers until the transaction commits. This is by design to prevent dirty reads, but it means there can be a delay between an append and when a consumer sees the event.

**Multiple processors:** When multiple processors have different `startFrom` positions, the consumer uses the earliest (minimum) position so all processors see all their needed events.

## Multiple Consumers

You can create multiple consumers from the same event store. Each polls independently. [[PostgreSQL Distributed Locking|Processor locks]] prevent the same processor from running concurrently across instances.

## See Also

- [[Consumer Architecture]] -- The general consumer/processor model
- [[Reactors]] -- General reactor concept
- [[Projectors]] -- General projector concept
- [[PostgreSQL Distributed Locking]] -- Advisory lock mechanics for safe multi-instance deployments
- [[Checkpointing]] -- How processor positions are tracked
- [[Workflow Processor]] -- Workflow processing details
