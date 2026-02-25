---
tags:
  - projections
  - sql
aliases:
  - Raw SQL Projection
  - postgreSQLRawSQLProjection
related:
  - "[[Projection Concepts]]"
  - "[[PostgreSQL Event Store]]"
  - "[[SQLite Event Store]]"
package: emmett-postgresql
---

# Raw SQL Projections

> [!abstract]
> Raw SQL projections give you ==full control over the SQL== executed for each event. Available in per-event and batch modes for both PostgreSQL and SQLite.

## PostgreSQL: Per-Event SQL

Each event produces one or more SQL statements:

```typescript
import { postgreSQLRawSQLProjection } from '@event-driven-io/emmett-postgresql';
import { sql } from '@event-driven-io/dumbo';

const shortInfoProjection = postgreSQLRawSQLProjection<ShoppingCartEvent>({
  name: 'shopping_cart_short_info',
  canHandle: ['ProductItemAdded', 'ShoppingCartConfirmed'],
  evolve: (event, context) => {
    switch (event.type) {
      case 'ProductItemAdded':
        return sql(
          `INSERT INTO shopping_cart_short_info (id, product_count)
           VALUES (%L, %L)
           ON CONFLICT (id) DO UPDATE SET product_count = product_count + %L`,
          event.metadata.streamName,
          event.data.productItem.quantity,
          event.data.productItem.quantity,
        );
      case 'ShoppingCartConfirmed':
        return sql(
          `DELETE FROM shopping_cart_short_info WHERE id = %L`,
          event.metadata.streamName,
        );
      default:
        return sql('SELECT 1'); // no-op
    }
  },
  init: () =>
    sql(`CREATE TABLE IF NOT EXISTS shopping_cart_short_info (
      id TEXT PRIMARY KEY,
      product_count INTEGER NOT NULL DEFAULT 0
    )`),
});
```

The `evolve` function receives a single event and can return `SQL`, `SQL[]`, `Promise<SQL>`, or `Promise<SQL[]>`.

### The `init` Function

The `init` function runs during schema migration and can return SQL for table creation or other DDL:

```typescript
init: () =>
  sql(`CREATE TABLE IF NOT EXISTS shopping_cart_short_info (
    id TEXT PRIMARY KEY,
    product_count INTEGER NOT NULL DEFAULT 0
  )`)
```

> [!warning]
> `init()` behavior differs by adapter:
> - **PostgreSQL**: Called during `schema.migrate()` within a transaction, AND automatically registers the projection in the [[Projection Management|`emt_projections` table]]
> - **SQLite**: Called during migration but does ==not== register in a management table

## PostgreSQL: Batch SQL

Receives all events at once, returning a batch of SQL statements:

```typescript
import { postgreSQLRawBatchSQLProjection } from '@event-driven-io/emmett-postgresql';

const batchProjection = postgreSQLRawBatchSQLProjection<ShoppingCartEvent>({
  name: 'shopping_cart_batch',
  canHandle: ['ProductItemAdded', 'ProductItemRemoved'],
  evolve: (events, context) => {
    // Receive all events at once, return SQL[]
    return events.map((event) =>
      sql(`UPDATE ... WHERE ...`, event.data),
    );
  },
});
```

> [!tip]
> Batch projections are useful when you need to optimize SQL execution across multiple events, such as combining multiple inserts into a single statement.

## SQLite: Raw SQL

SQLite provides equivalent raw SQL projection factories:

### Per-Event

```typescript
import { sqliteRawSQLProjection } from '@event-driven-io/emmett-sqlite';

const rawProjection = sqliteRawSQLProjection<ShoppingCartEvent>({
  name: 'shopping_cart_short_info',
  canHandle: ['ProductItemAdded'],
  evolve: (event, context) => {
    // Return SQL statement(s)
  },
});
```

### Batch

```typescript
import { sqliteRawBatchSQLProjection } from '@event-driven-io/emmett-sqlite';
```

## Options

| Option | Type | Description |
|---|---|---|
| `name` | `string` | Required projection name |
| `canHandle` | `string[]` | Event type names this projection handles |
| `evolve` | Per-event: `(event, context) => SQL \| SQL[]`; Batch: `(events, context) => SQL[]` | SQL-generating transform |
| `kind` | `string` | Optional projection kind tag |
| `version` | `number` | Optional version number |
| `init` | `() => SQL` | DDL to run during schema migration |

## The `sql` Helper

Raw SQL projections use the `sql()` tagged template or function from `@event-driven-io/dumbo`:

```typescript
import { sql } from '@event-driven-io/dumbo';

// Parameterized query (safe from SQL injection)
sql(`INSERT INTO table (id, value) VALUES (%L, %L)`, id, value);
```

## Differences Between PostgreSQL and SQLite

| Feature | PostgreSQL | SQLite |
|---|---|---|
| Advisory locking | Yes | No |
| Projection management table | Yes | No |
| Activate/deactivate | Yes | No |
| `init()` registers projection | Yes | No |
| API shape | Identical | Identical |

## See Also

- [[Projection Concepts]] -- Core concepts: evolve, single-stream vs multi-stream
- [[Pongo Document Projections]] -- The document-oriented alternative
- [[PostgreSQL Event Store]] -- Event store hosting PostgreSQL projections
- [[SQLite Event Store]] -- Event store hosting SQLite projections
- [[Testing Projections]] -- `PostgreSQLProjectionSpec` with SQL assertions
