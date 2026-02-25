---
tags:
  - consumer
  - architecture
aliases:
  - MessageConsumer
  - Consumer-Processor Model
related:
  - "[[Reactors]]"
  - "[[Projectors]]"
  - "[[Checkpointing]]"
  - "[[PostgreSQL Consumer]]"
  - "[[MongoDB Change Stream Consumer]]"
  - "[[SQLite Consumer]]"
  - "[[EventStoreDB Consumer]]"
package: emmett
---

# Consumer Architecture

> [!abstract]
> A `MessageConsumer` owns a message source and dispatches batches to one or more processors. Each adapter provides its own consumer factory with a different message delivery mechanism: polling, change streams, or native subscriptions.

## The Consumer-Processor Model

A consumer coordinates the full lifecycle of async message processing:

1. **Creates a message source** -- polling loop (PostgreSQL, SQLite), change stream (MongoDB), or native subscription (EventStoreDB)
2. **Manages processors** -- one or more [[Reactors|reactors]], [[Projectors|projectors]], or [[Workflow Processor|workflow processors]]
3. **Determines start position** -- takes the ==minimum checkpoint== across all processors so no processor misses messages
4. **Dispatches batches** -- sends each message batch to all active processors in parallel via `Promise.allSettled`
5. **Stops** -- when all processors signal `STOP` or no active processors remain

```typescript
import { type MessageConsumer } from '@event-driven-io/emmett';
```

## Consumer Interface

| Property | Type | Description |
|---|---|---|
| `consumerId` | `string` | Unique identifier (auto-generated UUID if not provided) |
| `isRunning` | `boolean` | Whether the consumer is actively processing |
| `processors` | `ReadonlyArray<MessageProcessor>` | Registered processors |
| `start()` | `Promise<void>` | Start the consumer (resolves when the consumer **stops**) |
| `stop()` | `Promise<void>` | Stop pulling/subscription and close processors |
| `close()` | `Promise<void>` | Calls `stop()` and closes the underlying connection pool |

> [!warning] `start()` resolves when the consumer stops
> The promise returned by `start()` does **not** resolve when the consumer begins processing. For polling consumers (PostgreSQL, SQLite), it resolves when polling stops. For subscription consumers (MongoDB, EventStoreDB), it resolves when the subscription ends. If you need to run code after startup, use the `onStart` [[Reactors#Lifecycle Hooks|lifecycle hook]].

## Adapter Consumer Factories

Each adapter provides its own factory function:

```typescript
import { postgreSQLEventStoreConsumer } from '@event-driven-io/emmett-postgresql';
import { mongoDBEventStoreConsumer } from '@event-driven-io/emmett-mongodb';
import { sqliteEventStoreConsumer } from '@event-driven-io/emmett-sqlite';
import { eventStoreDBEventStoreConsumer } from '@event-driven-io/emmett-esdb';
```

### Message Delivery Mechanisms

| Adapter | Mechanism | Details |
|---|---|---|
| **PostgreSQL** | Polling | Reads global messages table by `globalPosition`. Adaptive backoff. |
| **SQLite** | Polling | Same pattern as PostgreSQL. Supports native sqlite3 and Cloudflare D1. |
| **MongoDB** | Change stream | Watches all `emt:*` collections. Requires replica set. |
| **EventStoreDB** | Native subscription | Uses `subscribeToAll()` or `subscribeToStream()`. |

### Supported Processor Types

| Adapter | `reactor()` | `projector()` | `workflowProcessor()` |
|---|---|---|---|
| PostgreSQL | Yes | Yes | Yes |
| SQLite | Yes | Yes | Yes |
| MongoDB | Yes | Yes | No |
| EventStoreDB | Yes | Yes | No |

## Parallel Dispatch

When a consumer has multiple processors, every message batch goes to ==all active processors== simultaneously via `Promise.allSettled`. Each processor independently:

1. Checks its own [[Checkpointing|checkpoint]] to skip already-processed messages
2. Applies its `canHandle` filter
3. Handles the message
4. Stores its updated checkpoint

> [!note] Minimum checkpoint resolution
> The consumer starts from the earliest needed position across all processors. A processor that is caught up still receives (and skips) messages that an earlier processor needs. This is a design trade-off for simplicity -- processors do not maintain independent subscriptions.

## `stop()` vs `close()`

| Method | Behavior |
|---|---|
| `stop()` | Stops message pulling/subscription, closes all processors, but may keep the connection pool open |
| `close()` | Calls `stop()` **and** closes the underlying connection pool/client |

> [!warning] MongoDB client ownership
> If you pass an existing `MongoClient` via `options.client`, calling `consumer.close()` will **not** close that client. It only closes clients the consumer created internally.

> [!info]
> EventStoreDB consumers have no separate pool to manage -- `close()` is identical to `stop()`.

## See Also

- [[Reactors]] -- The fundamental processor type
- [[Projectors]] -- Specialized processor for read model building
- [[Checkpointing]] -- How processors track their position
- [[Graceful Shutdown]] -- Automatic shutdown signal handling
- [[Adapter Comparison]] -- Full feature matrix across all adapters
