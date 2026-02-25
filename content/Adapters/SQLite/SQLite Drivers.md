---
tags:
  - adapter
  - sqlite
  - driver
aliases:
  - sqlite3EventStoreDriver
  - d1EventStoreDriver
related:
  - "[[SQLite Event Store]]"
package: emmett-sqlite
---

# SQLite Drivers

The SQLite adapter supports multiple database engines through the ==`EventStoreDriver`== abstraction. You choose a driver when creating the event store and import it from a dedicated sub-path. This design lets you use the same event store API whether you're running on Node.js or Cloudflare Workers.

## The Driver Interface

```typescript
interface EventStoreDriver<
  DatabaseDriver extends AnyDumboDatabaseDriver,
  DriverOptions extends AnyEventStoreDriverOptions,
> {
  driverType: DatabaseDriver['driverType'];
  dumboDriver: DatabaseDriver;
  mapToDumboOptions(
    driverOptions: DriverOptions,
  ): ExtractDumboDatabaseDriverOptions<DatabaseDriver>;
}
```

Each driver maps its own configuration format into `dumbo` (Emmett's database abstraction library) options.

## sqlite3 (Native Node.js)

For Node.js environments using the `sqlite3` npm package:

```typescript
import { getSQLiteEventStore } from '@event-driven-io/emmett-sqlite';
import { sqlite3EventStoreDriver } from '@event-driven-io/emmett-sqlite/sqlite3';

const eventStore = getSQLiteEventStore({
  driver: sqlite3EventStoreDriver,
  fileName: './events.db',
});
```

The `fileName` option accepts a file system path or the special in-memory constant:

```typescript
import { InMemorySQLiteDatabase } from '@event-driven-io/dumbo/sqlite3';

const eventStore = getSQLiteEventStore({
  driver: sqlite3EventStoreDriver,
  fileName: InMemorySQLiteDatabase,
});
```

> [!tip]
> `InMemorySQLiteDatabase` is ideal for unit and integration tests where you don't need persistence between test runs.

### Configuration Type

```typescript
type SQLite3EventStoreDriverOptions = {
  fileName: string;
  connectionOptions?: Omit<SQLite3DumboOptions, 'fileName' | 'connectionString'>;
};
```

> [!note]
> The sqlite3 driver sets `transactionOptions: { allowNestedTransactions: true }` in the dumbo config, enabling [[SQLite Projections|inline projections]] to run inside the append transaction.

## Cloudflare D1

For Cloudflare Workers using the D1 database binding:

```typescript
import { getSQLiteEventStore } from '@event-driven-io/emmett-sqlite';
import { d1EventStoreDriver } from '@event-driven-io/emmett-sqlite/cloudflare';

// Inside a Cloudflare Worker handler
export default {
  async fetch(request: Request, env: Env): Promise<Response> {
    const eventStore = getSQLiteEventStore({
      driver: d1EventStoreDriver,
      database: env.MY_D1_DATABASE,
    });

    // Use eventStore...
  },
};
```

The `database` property accepts a `D1Database` reference from your Cloudflare Worker bindings.

### Configuration Type

```typescript
type D1EventStoreDriverOptions = {
  database: D1Database;
  connectionOptions?: Omit<D1PoolOptions, 'database'>;
};
```

> [!note]
> The D1 driver sets `transactionOptions: { allowNestedTransactions: true, mode: 'session_based' }`. Session-based transactions are required by D1's execution model. Tests for this driver use `Miniflare` to simulate the D1 environment.

## Package Entry Points

| Import path | What it provides |
|---|---|
| `@event-driven-io/emmett-sqlite` | Main entry: event store, projections, schema, consumers |
| `@event-driven-io/emmett-sqlite/sqlite3` | `sqlite3EventStoreDriver` and related types |
| `@event-driven-io/emmett-sqlite/cloudflare` | `d1EventStoreDriver` and related types |

## See Also

- [[SQLite Event Store]] -- Creating and configuring the event store
- [[SQLite Consumer]] -- Consumer system that works with either driver
- [[Adapter Comparison]] -- How SQLite compares to other adapters
