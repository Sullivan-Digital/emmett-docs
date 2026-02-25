---
tags:
  - adapter
  - sqlite
aliases:
  - getSQLiteEventStore
  - SQLiteEventStore
related:
  - "[[Event Store Interface]]"
  - "[[SQLite Drivers]]"
  - "[[SQLite Projections]]"
  - "[[Concurrency Control]]"
  - "[[Before-Commit Hooks]]"
package: emmett-sqlite
---

# SQLite Event Store

The SQLite event store is created via ==`getSQLiteEventStore()`==, which returns a full [[Event Store Interface]] implementation backed by SQLite. It requires a [[SQLite Drivers|driver]] to determine which SQLite engine to use.

## Quick Start

```typescript
import { getSQLiteEventStore } from '@event-driven-io/emmett-sqlite';
import { sqlite3EventStoreDriver } from '@event-driven-io/emmett-sqlite/sqlite3';
import type { Event } from '@event-driven-io/emmett';

type ProductItemAdded = Event<
  'ProductItemAdded',
  { productItem: { productId: string; quantity: number; price: number } }
>;
type DiscountApplied = Event<
  'DiscountApplied',
  { percent: number; couponId: string }
>;
type ShoppingCartEvent = ProductItemAdded | DiscountApplied;

const eventStore = getSQLiteEventStore({
  driver: sqlite3EventStoreDriver,
  fileName: './my-database.db',
});

const streamName = `shopping_cart-${cartId}`;
const result = await eventStore.appendToStream<ShoppingCartEvent>(
  streamName,
  [{ type: 'ProductItemAdded', data: { productItem } }],
);

const { events, currentStreamVersion } =
  await eventStore.readStream(streamName);

await eventStore.close();
```

## Configuration

The factory function accepts `SQLiteEventStoreOptions`:

```typescript
const eventStore = getSQLiteEventStore({
  driver: sqlite3EventStoreDriver,   // Required: which driver to use
  fileName: './events.db',           // Driver-specific option
  schema: { autoMigration: 'CreateOrUpdate' },  // Default
  projections: [{ type: 'inline', projection }], // Optional
  hooks: { /* ... */ },              // Optional lifecycle hooks
  pool: existingDumboPool,           // Optional shared connection pool
});
```

### Schema Auto-Migration

By default, the event store creates and migrates its schema automatically on first use (`'CreateOrUpdate'`). Set to `'None'` to manage schema manually:

```typescript
const eventStore = getSQLiteEventStore({
  driver: sqlite3EventStoreDriver,
  fileName: './events.db',
  schema: { autoMigration: 'None' },
});

// Run migrations when ready
await eventStore.schema.migrate();
```

### Lifecycle Hooks

```typescript
const eventStore = getSQLiteEventStore({
  driver: sqlite3EventStoreDriver,
  fileName: './events.db',
  hooks: {
    onBeforeSchemaCreated: async ({ connection }) => {
      // Runs before event store schema tables are created
    },
    onAfterSchemaCreated: async () => {
      // Runs after schema creation completes
    },
    onBeforeCommit: (messages, { connection }) => {
      // Runs inside the append transaction, before commit
      // Useful for transactional consistency with side effects
    },
  },
});
```

> [!tip]
> The `onBeforeCommit` hook fires inside the same transaction as `appendToStream`. This is where [[SQLite Projections|inline projections]] execute, giving you strong read-after-write consistency.

### Sharing a Connection Pool

If you already have a `dumbo` connection pool, pass it to avoid creating a new one:

```typescript
import { dumbo } from '@event-driven-io/dumbo';

const pool = dumbo({ /* your config */ });

const eventStore = getSQLiteEventStore({
  driver: sqlite3EventStoreDriver,
  fileName: './events.db',
  pool,
});
```

## Core Operations

### Appending Events

```typescript
// Without concurrency check
const result = await eventStore.appendToStream<ShoppingCartEvent>(
  'shopping_cart-abc123',
  [{ type: 'ProductItemAdded', data: { productItem } }],
);
// result: { nextExpectedStreamVersion, lastEventGlobalPosition, createdNewStream }

// With optimistic concurrency
const result2 = await eventStore.appendToStream<ShoppingCartEvent>(
  'shopping_cart-abc123',
  [{ type: 'DiscountApplied', data: { percent: 10, couponId: 'SAVE10' } }],
  { expectedStreamVersion: result.nextExpectedStreamVersion },
);
```

Internally, `appendToStream` wraps everything in a transaction: checks the current stream position, inserts/updates the stream record, batch-inserts messages, runs [[SQLite Projections|inline projections]] via the `onBeforeCommit` hook, then commits.

### Reading a Stream

```typescript
const { events, currentStreamVersion, streamExists } =
  await eventStore.readStream('shopping_cart-abc123');

// With options
const { events: recentEvents } = await eventStore.readStream(
  'shopping_cart-abc123',
  { from: 5n, maxCount: 10 },
);
```

If the stream does not exist, the result has `events: []`, `streamExists: false`, and `currentStreamVersion` set to ==`SQLiteEventStoreDefaultStreamVersion`== (`0n`).

### Aggregating a Stream

Fold events into state using an [[Evolve Function|evolve]] function:

```typescript
const { state, currentStreamVersion } = await eventStore.aggregateStream(
  'shopping_cart-abc123',
  {
    evolve: (state, event) => {
      switch (event.type) {
        case 'ProductItemAdded':
          return {
            totalAmount:
              state.totalAmount +
              event.data.productItem.price * event.data.productItem.quantity,
            productItemsCount:
              state.productItemsCount + event.data.productItem.quantity,
          };
        default:
          return state;
      }
    },
    initialState: () => ({ productItemsCount: 0, totalAmount: 0 }),
  },
);
```

### Checking Stream Existence

```typescript
const exists = await eventStore.streamExists('shopping_cart-abc123');
```

## Concurrency Control

The adapter uses optimistic concurrency via the stream position. On version mismatch, it throws `ExpectedVersionConflictError`:

```typescript
import {
  ExpectedVersionConflictError,
  NO_CONCURRENCY_CHECK,
} from '@event-driven-io/emmett';

try {
  await eventStore.appendToStream(streamName, events, {
    expectedStreamVersion: 2n,
  });
} catch (error) {
  if (error instanceof ExpectedVersionConflictError) {
    // Handle conflict -- reload and retry
  }
}

// Skip concurrency check entirely
await eventStore.appendToStream(streamName, events, {
  expectedStreamVersion: NO_CONCURRENCY_CHECK,
});
```

> [!warning]
> `STREAM_DOES_NOT_EXIST` and `STREAM_EXISTS` expected version constants are ==not enforced== in the SQLite adapter. They are mapped to `null` internally (marked as TODOs in the source). Only explicit `bigint` values and `NO_CONCURRENCY_CHECK` work as expected. See [[Concurrency Control]] for cross-adapter differences.

## Event Upcasting

Transform events at read time for [[Schema Versioning|schema evolution]]:

```typescript
const upcast = (event: Event): ShoppingCartOpenedV2 => {
  if (event.type === 'ShoppingCartOpened') {
    const e = event as ShoppingCartOpenedV1;
    return {
      ...e,
      data: {
        openedAt: new Date(e.data.openedAt),
        loyaltyPoints: BigInt(e.data.loyaltyPoints),
      },
    };
  }
  return event as ShoppingCartOpenedV2;
};

const { state } = await eventStore.aggregateStream(streamName, {
  evolve: myEvolve,
  initialState: () => myInitialState,
  read: { schema: { versioning: { upcast } } },
});
```

## Stream Naming Convention

Stream names follow the `{streamType}-{id}` pattern. The text before the first `-` is extracted as the stream type:

```
shopping_cart-abc123  -> stream type: "shopping_cart"
guestStay-guest-42    -> stream type: "guestStay"
noHyphen              -> stream type: "emt:unknown"
```

> [!warning]
> Stream type extraction splits on the ==first== `-` only. A stream named `my-entity-123` has type `my`, not `my-entity`. Plan stream names accordingly. See [[Stream Naming Conventions]] for details.

## Database Schema

The adapter creates four tables with the `emt_` prefix:

| Table | Purpose |
|---|---|
| `emt_streams` | Stream metadata: `stream_id`, `stream_position`, `stream_type`, `partition` |
| `emt_messages` | Events: `global_position` (INTEGER PK), `stream_id`, `stream_position`, `message_data`, `message_type` |
| `emt_processors` | Consumer checkpoints: `processor_id`, `last_processed_checkpoint`, `status` |
| `emt_projections` | Projection metadata: `name`, `version`, `kind`, `status` |

> [!note]
> The `global_position` column is the `INTEGER PRIMARY KEY` of `emt_messages`, making it an alias for SQLite's ROWID. This auto-increments and provides a ==total ordering== across all streams, used by the [[SQLite Consumer|consumer]] for checkpoint-based polling.

### Schema Utilities

```typescript
const ddl = eventStore.schema.sql();     // Get raw SQL DDL as string
eventStore.schema.print();               // Print DDL to console
await eventStore.schema.migrate();       // Run migrations manually
```

### Schema Migrations

- **v0.41.0**: Initial schema with `emt_streams`, `emt_messages`, `emt_subscriptions`
- **v0.42.0**: Renamed `emt_subscriptions` to `emt_processors` (added `status`, `processor_instance_id`), added `emt_projections` table. Checkpoint data migrated with `printf('%019d', last_processed_position)` format conversion.

## Default Stream Version

The SQLite adapter uses ==`0n`== for non-existing streams (`SQLiteEventStoreDefaultStreamVersion`). This differs from [[EventStoreDB Event Store|EventStoreDB]] which uses `-1n`.

## Closing

```typescript
await eventStore.close(); // Stops consumers and closes the connection pool
```

## Gotchas

> [!warning] Caveats
> - **Empty events array throws**: `appendToStream([])` surfaces as `ExpectedVersionConflictError`
> - **D1 performance trade-off**: The adapter always queries current stream position before appending (even for `sqlite3`), as a workaround for D1 compatibility
> - **`createdNewStream` may be inaccurate**: Due to a comparison bug, this flag can return `true` for subsequent appends, not just the first
> - **Default stream version is `0n`**: Unlike EventStoreDB (`-1n`)
>
> See [[Gotchas]] for the full list.
