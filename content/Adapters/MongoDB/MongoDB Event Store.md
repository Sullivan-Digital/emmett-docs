---
tags:
  - adapter
  - mongodb
aliases:
  - getMongoDBEventStore
  - MongoDBEventStore
related:
  - "[[Event Store Interface]]"
  - "[[MongoDB Storage Strategies]]"
  - "[[MongoDB Inline Projections]]"
  - "[[Concurrency Control]]"
  - "[[Schema Versioning]]"
  - "[[After-Commit Hooks]]"
package: emmett-mongodb
---

# MongoDB Event Store

The `getMongoDBEventStore()` factory creates an [[Event Store Interface|EventStore]] backed by MongoDB. It supports two connection modes, three [[MongoDB Storage Strategies|storage strategies]], [[MongoDB Inline Projections|inline projections]], [[Schema Versioning|event versioning]], and [[After-Commit Hooks|after-commit hooks]].

**Package**: `@event-driven-io/emmett-mongodb`
**Peer dependencies**: `@event-driven-io/emmett`, `mongodb` ^6.10.0

## Getting Started

```typescript
import { getMongoDBEventStore } from '@event-driven-io/emmett-mongodb';

const eventStore = getMongoDBEventStore({
  connectionString: 'mongodb://localhost:27017/',
});

// Append events
await eventStore.appendToStream('shopping_cart:abc-123', [
  {
    type: 'ProductItemAdded',
    data: {
      productItem: { productId: 'shoes-1', quantity: 1, price: 100 },
    },
  },
]);

// Read them back
const { events } = await eventStore.readStream('shopping_cart:abc-123');

// Clean up (only needed when using connectionString)
await eventStore.close();
```

## Connection Modes

### Connection String (Event Store Manages Client)

When you provide a `connectionString`, the event store creates and owns a `MongoClient` internally. You must call `close()` when done.

```typescript
const eventStore = getMongoDBEventStore({
  connectionString: 'mongodb://localhost:27017/',
  clientOptions: { directConnection: true }, // optional MongoClientOptions
});

// eventStore implements Closeable
await eventStore.close();
```

### External Client (You Manage Client)

When you provide an existing `MongoClient`, the event store does ==not== close it. The `close()` method is removed from the returned instance entirely.

```typescript
import { MongoClient } from 'mongodb';

const client = new MongoClient('mongodb://localhost:27017/');
await client.connect();

const eventStore = getMongoDBEventStore({ client });

// eventStore does NOT have close() -- you manage the client
await client.close();
```

> [!tip] Auto-Connect
> You do not need to call `client.connect()` before passing a client. The event store calls `connect()` internally on each operation, and the MongoDB driver handles this idempotently.

### Database Selection

By default, the event store uses the database from your connection string (the MongoDB driver's default `client.db()` behavior). Override this per storage strategy using the `databaseName` option -- see [[MongoDB Storage Strategies]].

## Factory Overloads

The factory has two TypeScript overloads that control the return type:

```typescript
// With client: returns MongoDBEventStore (no close method)
function getMongoDBEventStore(
  options: MongoDBEventStoreOptions & { client: MongoClient },
): MongoDBEventStore;

// With connection string: returns MongoDBEventStore & Closeable
function getMongoDBEventStore(
  options: MongoDBEventStoreOptions & { connectionString: string },
): MongoDBEventStore & Closeable;
```

The `MongoDBEventStore` type extends the common `EventStore` interface with:

- `projections` -- query helpers for [[MongoDB Inline Projections|inline projections]] (`findOne`, `find`, `count`)
- `collectionFor` -- direct access to the underlying MongoDB collection

## Full Options

```typescript
type MongoDBEventStoreOptions = {
  projections?: ProjectionRegistration[];
  storage?: MongoDBEventStoreStorageOptions;
} & MongoDBEventStoreConnectionOptions
  & DefaultEventStoreOptions<MongoDBEventStore>;
```

> [!example]- Full Configuration Example
> ```typescript
> import { projections } from '@event-driven-io/emmett';
> import {
>   getMongoDBEventStore,
>   mongoDBInlineProjection,
> } from '@event-driven-io/emmett-mongodb';
>
> const eventStore = getMongoDBEventStore({
>   connectionString: 'mongodb://localhost:27017/',
>   storage: 'COLLECTION_PER_STREAM_TYPE',
>   projections: projections.inline([
>     mongoDBInlineProjection({
>       canHandle: ['ProductItemAdded', 'DiscountApplied'],
>       evolve: myEvolveFunction,
>       initialState: () => ({ count: 0, total: 0 }),
>     }),
>   ]),
>   hooks: {
>     onAfterCommit: (events) => {
>       console.log(`Committed ${events.length} events`);
>     },
>   },
> });
> ```

## Core Operations

### Appending Events

`appendToStream` adds events to a stream, creating it if it does not exist (upsert).

```typescript
import type { Event } from '@event-driven-io/emmett';

type ProductItemAdded = Event<
  'ProductItemAdded',
  { productItem: { productId: string; quantity: number; price: number } }
>;
type ShoppingCartEvent = ProductItemAdded | ShoppingCartConfirmed;

const result = await eventStore.appendToStream<ShoppingCartEvent>(
  'shopping_cart:abc-123',
  [
    {
      type: 'ProductItemAdded',
      data: {
        productItem: { productId: 'shoes-1', quantity: 2, price: 100 },
      },
    },
  ],
);

console.log(result.nextExpectedStreamVersion); // 1n
console.log(result.createdNewStream);          // true
```

Under the hood, the adapter uses MongoDB's `updateOne` with `upsert: true`. Events are pushed via `$push` with `$each`, and each event gets a UUID `messageId` and an incrementing `streamPosition`. [[MongoDB Inline Projections|Inline projections]] are computed and written in the ==same update operation==.

#### Optimistic Concurrency

Use `expectedStreamVersion` to guard against concurrent writes:

```typescript
import { STREAM_DOES_NOT_EXIST } from '@event-driven-io/emmett';

// Require the stream to not exist yet
await eventStore.appendToStream('shopping_cart:abc-123', events, {
  expectedStreamVersion: STREAM_DOES_NOT_EXIST,
});

// Require a specific version
await eventStore.appendToStream('shopping_cart:abc-123', moreEvents, {
  expectedStreamVersion: 2n,
});
// Throws ExpectedVersionConflictError if the current version does not match
```

The adapter includes `metadata.streamPosition` in the MongoDB `updateOne` filter. If another write incremented the position first, the update matches zero documents and raises `ExpectedVersionConflictError`. See [[Concurrency Control]] for more detail.

### Reading Events

```typescript
const { events, currentStreamVersion, streamExists } =
  await eventStore.readStream<ShoppingCartEvent>('shopping_cart:abc-123');

// Read a slice (0-based indices via MongoDB $slice)
const { events: slice } = await eventStore.readStream<ShoppingCartEvent>(
  'shopping_cart:abc-123',
  { from: 0n, to: 4n },
);
```

If the stream does not exist, the result is `{ events: [], currentStreamVersion: 0n, streamExists: false }` -- no error is thrown.

You can also assert the expected version on read:

```typescript
const result = await eventStore.readStream('shopping_cart:abc-123', {
  expectedStreamVersion: 5n,
});
// Throws if the stream is not at version 5
```

### Aggregating State

`aggregateStream` reads events and folds them into a state object using an `evolve` function, matching the [[The Decider Pattern|Decider pattern]].

```typescript
const { state, currentStreamVersion, streamExists } =
  await eventStore.aggregateStream('shopping_cart:abc-123', {
    evolve: (state, event) => {
      switch (event.type) {
        case 'ProductItemAdded':
          return {
            ...state,
            items: [...state.items, event.data.productItem],
          };
        case 'ShoppingCartConfirmed':
          return { ...state, status: 'confirmed' };
        default:
          return state;
      }
    },
    initialState: () => ({
      items: [] as Array<{ productId: string; quantity: number; price: number }>,
      status: 'open' as const,
    }),
  });
```

### Checking Stream Existence

```typescript
const exists: boolean = await eventStore.streamExists('shopping_cart:abc-123');
```

Uses `countDocuments` with `limit: 1` for efficiency.

### Direct Collection Access

For advanced queries beyond what the event store API provides:

```typescript
const collection =
  await eventStore.collectionFor<ShoppingCartEvent>('shopping_cart');

// Standard MongoDB Collection<EventStream<ShoppingCartEvent>>
const doc = await collection.findOne(
  { streamName: 'shopping_cart:abc-123' },
  { useBigInt64: true },
);
```

## Event Versioning

The adapter supports both upcasting (on read) and downcasting (on write). See [[Schema Versioning]] for the general pattern.

```typescript
// Upcasting on read
const { state } = await eventStore.aggregateStream('shopping_cart:abc-123', {
  evolve,
  initialState,
  read: {
    schema: {
      versioning: {
        upcast: (event) => {
          if (event.type === 'ShoppingCartOpened') {
            return {
              ...event,
              data: { ...event.data, loyaltyPoints: event.data.loyaltyPoints ?? 0 },
            };
          }
          return event;
        },
      },
    },
  },
});

// Downcasting on write
await eventStore.appendToStream('shopping_cart:abc-123', events, {
  schema: {
    versioning: {
      downcast: (event) => event, // transform to storage format
    },
  },
});
```

## After-Commit Hooks

Register a hook that runs after each successful `appendToStream`:

```typescript
const eventStore = getMongoDBEventStore({
  client,
  hooks: {
    onAfterCommit: (events) => {
      for (const event of events) {
        console.log(`Committed: ${event.type}`);
      }
    },
  },
});
```

> [!warning] Silent Failure
> If the hook throws, the error is ==silently swallowed==. Events are already committed and will not be rolled back. Use this for non-critical side effects like logging or in-memory bus forwarding.

## Important Caveats

> [!warning] No Global Position
> Unlike [[PostgreSQL Event Store|PostgreSQL]] and [[SQLite Event Store|SQLite]], the MongoDB adapter does **not** track a global event position. Events have `streamPosition` (within their stream) but no cross-stream ordering. `MongoDBReadEventMetadata` is `ReadEventMetadataWithoutGlobalPosition`.

> [!warning] BigInt Throughout
> All MongoDB operations use `useBigInt64: true`. Stream positions and versions are `bigint` values. The default stream version for a non-existent stream is `0n`.

> [!warning] Stream Name Format
> Stream names must follow `{streamType}:{streamId}`. The `streamType` portion determines which collection stores the stream. See [[Stream Naming Conventions]] for details.
