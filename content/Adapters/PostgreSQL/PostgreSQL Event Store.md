---
tags:
  - adapter
  - postgresql
aliases:
  - getPostgreSQLEventStore
  - PostgresEventStore
related:
  - "[[Event Store Interface]]"
  - "[[PostgreSQL Schema]]"
  - "[[PostgreSQL Projections]]"
  - "[[Concurrency Control]]"
  - "[[Sessions]]"
package: emmett-postgresql
---

# PostgreSQL Event Store

The PostgreSQL event store is created via the ==`getPostgreSQLEventStore()`== factory function. It implements the common [[Event Store Interface]] with PostgreSQL-specific extensions including global position tracking, a consumer factory, and schema management.

## Factory Function

```typescript
import { getPostgreSQLEventStore } from '@event-driven-io/emmett-postgresql';

const eventStore = getPostgreSQLEventStore(connectionString, options?);
```

The returned `PostgresEventStore` extends both `EventStore<PostgresReadEventMetadata>` and `EventStoreSessionFactory<PostgresEventStore>`.

## Options

```typescript
type PostgresEventStoreOptions = {
  // Inline projections to run within appendToStream transactions
  projections?: ProjectionRegistration<
    'inline',
    PostgresReadEventMetadata,
    PostgreSQLProjectionHandlerContext
  >[];

  // Schema migration behavior
  schema?: {
    autoMigration?: 'CreateOrUpdate' | 'None'; // default: 'CreateOrUpdate'
  };

  // Connection configuration
  connectionOptions?: PostgresEventStoreConnectionOptions;

  // Lifecycle hooks
  hooks?: {
    onBeforeSchemaCreated?: (context: PostgreSQLProjectionHandlerContext) => Promise<void> | void;
    onAfterSchemaCreated?: (context: PostgreSQLProjectionHandlerContext) => Promise<void> | void;
  };
};
```

| Option | Default |
|---|---|
| `projections` | `[]` |
| `schema.autoMigration` | `'CreateOrUpdate'` |

## Connection Modes

The adapter supports several connection modes:

**Pooled (default)** -- provide a connection string or a `pg.Pool` instance:

```typescript
// From connection string (creates a pool automatically)
const eventStore = getPostgreSQLEventStore(connectionString);

// With an existing pg.Pool
import pg from 'pg';
const pool = new pg.Pool({ connectionString });
const eventStore = getPostgreSQLEventStore(connectionString, {
  connectionOptions: { pool },
});
```

**Not pooled** -- provide a `pg.Client`, a Dumbo instance, or a raw connection:

```typescript
import { dumbo } from '@event-driven-io/dumbo';

const db = dumbo({ connectionString });
const eventStore = getPostgreSQLEventStore(connectionString, {
  connectionOptions: { dumbo: db },
});
```

## Schema Migration

By default, the event store ==auto-migrates the schema on the first operation== (lazy initialization). The schema creation is memoized -- it runs once per event store instance.

```typescript
// Auto-migration (default) -- schema is created on first read/write/append
const eventStore = getPostgreSQLEventStore(connectionString);

// Explicit migration
const eventStore = getPostgreSQLEventStore(connectionString, {
  schema: { autoMigration: 'None' },
});
await eventStore.schema.migrate();

// Migration with options
await eventStore.schema.migrate({
  dryRun: true,                      // preview without applying
  ignoreMigrationHashMismatch: true, // skip hash checks
  migrationTimeoutMs: 30000,         // timeout in ms
});
```

You can also inspect the schema SQL:

```typescript
const sql = eventStore.schema.sql();   // returns the full schema SQL string
eventStore.schema.print();              // prints it to console
```

> [!warning] Lazy Migration Gotcha
> Schema migration is lazy and memoized. The schema is created on the first operation, not when the event store is constructed. This makes the first operation slower. If migration fails, the cached failed promise is returned on subsequent attempts -- you must create a new event store instance to retry.

See [[PostgreSQL Schema]] for full details on tables, SQL functions, and migration history.

## Appending Events

```typescript
const result = await eventStore.appendToStream(streamName, events, options?);
```

The result includes PostgreSQL-specific fields:

```typescript
type AppendToStreamResultWithGlobalPosition = {
  nextExpectedStreamVersion: bigint;
  lastEventGlobalPosition: bigint;
  createdNewStream: boolean;
};
```

### Optimistic Concurrency

Pass an expected stream version to enable [[Concurrency Control|optimistic concurrency control]]:

```typescript
import { ExpectedVersionConflictError } from '@event-driven-io/emmett';

try {
  await eventStore.appendToStream(streamName, events, {
    expectedStreamVersion: 5n,
  });
} catch (error) {
  if (error instanceof ExpectedVersionConflictError) {
    // Another writer modified the stream concurrently
  }
}
```

The underlying SQL function uses `UPDATE ... WHERE stream_position = expected` on the `emt_streams` table. If the update affects zero rows, the append fails with a concurrency conflict.

Use `NO_CONCURRENCY_CHECK` to skip version checking:

```typescript
import { NO_CONCURRENCY_CHECK } from '@event-driven-io/emmett';

await eventStore.appendToStream(streamName, events, {
  expectedStreamVersion: NO_CONCURRENCY_CHECK,
});
```

> [!bug] Incomplete Expected Version Enforcement
> `STREAM_DOES_NOT_EXIST` and `STREAM_EXISTS` expected versions are not fully enforced. Both are currently mapped to `null` internally (equivalent to no concurrency check). This is a known limitation noted with TODO comments in the source.

### Stream Name Convention

Stream names should follow the pattern `{type}-{id}`, for example `shopping_cart-abc123`. The adapter extracts the stream type from the portion before the first hyphen.

| Stream Name | Extracted Type |
|---|---|
| `shopping_cart-abc123` | `shopping_cart` |
| `order-xyz` | `order` |
| `my-complex-stream-123` | `my` |
| `noHyphenHere` | `emt:unknown` |

> [!warning]
> Stream type extraction is naive -- it takes the portion before the first `-`. A stream named `my-complex-stream-123` gets type `my`, not `my-complex-stream`. Use underscores in the type portion: `my_complex_stream-123`.

See also [[Stream Naming Conventions]] for patterns across all adapters.

## Reading Events

### Reading a Stream

```typescript
const { events, currentStreamVersion } = await eventStore.readStream<MyEvent>(
  'shopping_cart-abc123',
);
```

Each event includes PostgreSQL-specific metadata:

```typescript
type PostgresReadEventMetadata = {
  messageId: string;
  streamName: string;
  streamPosition: bigint;
  globalPosition: bigint;
};
```

### Aggregating a Stream

Fold events into a state using [[Evolve Function|evolve]] and `initialState`:

```typescript
const { state, currentStreamVersion } = await eventStore.aggregateStream(
  'shopping_cart-abc123',
  {
    evolve: (state, event) => { /* return new state */ },
    initialState: () => ({ status: 'Empty' }),
  },
);
```

### Checking Stream Existence

```typescript
const { exists } = await eventStore.streamExists('shopping_cart-abc123');
```

## Sessions

[[Sessions]] provide a scoped event store that uses a single connection for multiple operations:

```typescript
await eventStore.withSession(async (session) => {
  await session.eventStore.appendToStream('stream-1', events1);
  await session.eventStore.appendToStream('stream-2', events2);
  // Both operations share the same connection
});
```

> [!note]
> The session's event store skips schema migration (assumes the parent already handled it) and its `close()` is a no-op since the parent manages the connection lifecycle.

## Lifecycle

The default stream version is ==`0n`== (BigInt zero). Streams that do not exist return empty results without errors.

To close the event store and release its connection pool:

```typescript
await eventStore.close();
```

## Quick Start

```typescript
import { projections } from '@event-driven-io/emmett';
import { getPostgreSQLEventStore } from '@event-driven-io/emmett-postgresql';
import { pongoClient } from '@event-driven-io/pongo';

const connectionString = 'postgresql://postgres:postgres@localhost:5432/postgres';

const eventStore = getPostgreSQLEventStore(connectionString, {
  projections: projections.inline([
    shoppingCartDetailsProjection,
    shoppingCartShortInfoProjection,
  ]),
});

// Append events
const result = await eventStore.appendToStream(
  'shopping_cart-abc123',
  [{ type: 'ProductItemAddedToShoppingCart', data: { ... } }],
);

// Read events
const { events } = await eventStore.readStream('shopping_cart-abc123');

// Read projected data via Pongo
const pongo = pongoClient(connectionString);
const cart = await pongo.db()
  .collection('shoppingCartDetails')
  .findOne({ _id: 'shopping_cart-abc123' });

await eventStore.close();
```

## See Also

- [[PostgreSQL Projections]] -- All projection types available for inline and async use
- [[PostgreSQL Consumer]] -- Async event processing with projectors, reactors, and workflows
- [[PostgreSQL Schema]] -- Database tables and migration system
- [[Event Store Interface]] -- The common interface this adapter implements
