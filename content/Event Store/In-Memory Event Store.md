---
tags:
  - event-store
  - adapter
aliases:
  - InMemoryEventStore
  - getInMemoryEventStore
related:
  - "[[Event Store Interface]]"
  - "[[In-Memory Database]]"
  - "[[InMemory Projections]]"
package: emmett
---

# In-Memory Event Store

The ==`getInMemoryEventStore()`== factory creates an in-memory implementation of the [[Event Store Interface]]. It is useful for development, testing, and prototyping -- no external database required.

## Creating an Instance

```typescript
import { getInMemoryEventStore } from '@event-driven-io/emmett';

const eventStore = getInMemoryEventStore();
```

### With Options

```typescript
import {
  getInMemoryEventStore,
  getInMemoryDatabase,
  inlineProjections,
  forwardToMessageBus,
} from '@event-driven-io/emmett';

const database = getInMemoryDatabase();

const eventStore = getInMemoryEventStore({
  database,                                    // optional: share a database instance
  projections: inlineProjections([myProjection]), // optional: inline projections
  hooks: {
    onAfterCommit: forwardToMessageBus(messageBus), // optional: after-commit hook
  },
});
```

## Types

```typescript
type InMemoryEventStore = EventStore<ReadEventMetadataWithGlobalPosition> & {
  database: InMemoryDatabase;
};

type InMemoryReadEventMetadata = ReadEventMetadataWithGlobalPosition;
// Expands to: {
//   messageId: string;
//   streamPosition: bigint;
//   streamName: string;
//   checkpoint?: ProcessorCheckpoint | null;
//   globalPosition: bigint;
// }

type InMemoryEventStoreOptions = DefaultEventStoreOptions<InMemoryEventStore> & {
  projections?: ProjectionRegistration<'inline', InMemoryReadEventMetadata, InMemoryProjectionHandlerContext>[];
  database?: InMemoryDatabase;
};

type InMemoryProjectionHandlerContext = {
  eventStore?: InMemoryEventStore;
  database?: InMemoryDatabase;
};
```
^inmemory-eventstore-types

## Internal Storage

The in-memory store uses a `Map<string, ReadEvent[]>` internally -- a Map from stream name to array of read events.

### Stream Positions

Stream positions are ==1-based==. The first event in a stream has `streamPosition: 1n`. The default (empty) stream version is:

```typescript
const InMemoryEventStoreDefaultStreamVersion = 0n;
```

### Global Position

Global position is computed on-the-fly by summing all event counts across all streams:

```typescript
Array.from(streams.values()).map(s => s.length).reduce(...)
```

With multiple events in a batch, each gets a sequential global position (`totalCount + index + 1`).

## The `database` Property

The in-memory event store exposes its [[In-Memory Database]] directly via the `database` property. This allows querying [[Inline Projections|inline projection]] results:

```typescript
const details = await eventStore.database
  .collection<ShoppingCartDetails>('shoppingCartDetails')
  .findOne((doc) => doc._id === 'shoppingCart-123');
```

You can also provide an external database instance at creation time to share it between the event store and your application code:

```typescript
const database = getInMemoryDatabase();
const eventStore = getInMemoryEventStore({ database });
// Both reference the same database instance
```

## Limitations

> [!warning]
> - **No built-in concurrency control beyond version checks.** The in-memory store is single-process only with no locking mechanism. Concurrent `appendToStream` calls could theoretically interleave if the JS event loop yields between the version check and storage update.
> - **No persistence.** All data is lost when the process exits.
> - **No sessions.** The in-memory store does not natively support transactional [[Sessions|sessions]]. Use `nulloSessionFactory()` if a session-based API is required.

## See Also

- [[Event Store Interface]] -- The four-method interface this store implements
- [[In-Memory Database]] -- The document store exposed via `database`
- [[InMemory Projections]] -- Inline projection types for this store
- [[Sessions]] -- `nulloSessionFactory()` fallback for session-based APIs
- [[Adapters MOC]] -- Other database-backed implementations
