---
tags:
  - adapter
  - sqlite
  - projections
aliases:
  - SQLite Inline Projections
related:
  - "[[Pongo Document Projections]]"
  - "[[Raw SQL Projections]]"
  - "[[SQLite Event Store]]"
  - "[[Projection Concepts]]"
package: emmett-sqlite
---

# SQLite Projections

The SQLite adapter supports [[Inline Projections Overview|inline projections]] that run inside the `appendToStream` transaction. This gives you strong ==read-after-write consistency== -- your read model is updated atomically with the events. Three projection families are available: Pongo document projections, raw SQL projections, and generic Pongo projections.

## Registering Inline Projections

Projections are registered when creating the [[SQLite Event Store|event store]]:

```typescript
import {
  getSQLiteEventStore,
  pongoSingleStreamProjection,
} from '@event-driven-io/emmett-sqlite';

const projection = pongoSingleStreamProjection({
  collectionName: 'shoppingCartSummary',
  evolve: (document, event) => {
    switch (event.type) {
      case 'ProductItemAdded':
        return {
          ...document,
          totalAmount:
            document.totalAmount +
            event.data.productItem.price * event.data.productItem.quantity,
          productItemsCount:
            document.productItemsCount + event.data.productItem.quantity,
        };
      default:
        return document;
    }
  },
  canHandle: ['ProductItemAdded', 'DiscountApplied'],
  initialState: () => ({ productItemsCount: 0, totalAmount: 0 }),
});

const eventStore = getSQLiteEventStore({
  driver: sqlite3EventStoreDriver,
  fileName: './events.db',
  projections: [{ type: 'inline', projection }],
});
```

## Pongo Document Projections

Pongo projections provide a MongoDB-like document API on top of SQLite via `@event-driven-io/pongo`. There are three variants.

### Single-Stream

One document per stream, keyed by `streamName`:

```typescript
import { pongoSingleStreamProjection } from '@event-driven-io/emmett-sqlite';

const projection = pongoSingleStreamProjection({
  collectionName: 'shoppingCartShortInfo',
  canHandle: ['ProductItemAdded', 'DiscountApplied'],
  initialState: () => ({
    productItemsCount: 0,
    totalAmount: 0,
    appliedDiscounts: [],
  }),
  evolve: (document, event) => {
    switch (event.type) {
      case 'ProductItemAdded':
        return {
          ...document,
          totalAmount:
            document.totalAmount +
            event.data.productItem.price * event.data.productItem.quantity,
          productItemsCount:
            document.productItemsCount + event.data.productItem.quantity,
        };
      case 'DiscountApplied':
        if (document.appliedDiscounts.includes(event.data.couponId))
          return document; // idempotency
        return {
          ...document,
          totalAmount:
            (document.totalAmount * (100 - event.data.percent)) / 100,
          appliedDiscounts: [...document.appliedDiscounts, event.data.couponId],
        };
      default:
        return document;
    }
  },
});
```

The document ID defaults to `event.metadata.streamName`. The `initialState` function is called when no document exists yet.

### Multi-Stream

Custom document ID, useful for cross-stream aggregations:

```typescript
import { pongoMultiStreamProjection } from '@event-driven-io/emmett-sqlite';

const projection = pongoMultiStreamProjection({
  collectionName: 'dailyRevenue',
  canHandle: ['OrderPlaced'],
  getDocumentId: (event) => event.data.date,
  initialState: () => ({ total: 0, orderCount: 0 }),
  evolve: (document, event) => ({
    total: document.total + event.data.amount,
    orderCount: document.orderCount + 1,
  }),
});
```

### Generic

Full access to the Pongo client for arbitrary operations:

```typescript
import { pongoProjection } from '@event-driven-io/emmett-sqlite';

const projection = pongoProjection({
  collectionName: 'myCollection',
  canHandle: ['MyEvent'],
  handle: async (events, { pongo }) => {
    // Full access to the Pongo client
  },
});
```

### Versioned Collections

When a projection has `version` > 0, the collection name is automatically suffixed with `_v{version}`:

```
version 0 -> shoppingCartShortInfo
version 2 -> shoppingCartShortInfo_v2
```

> [!warning]
> Pongo projection ==kind strings reuse PostgreSQL naming==. You'll see kinds like `emt:projections:postgresql:pongo:single_stream` even when running on SQLite. This is cosmetic and does not affect behavior.

## Raw SQL Projections

For full control over the generated SQL, use raw SQL projections.

### Single-Event

Processes events one at a time and returns SQL statements:

```typescript
import { sqliteRawSQLProjection } from '@event-driven-io/emmett-sqlite';
import { SQL } from '@event-driven-io/dumbo';

const projection = sqliteRawSQLProjection({
  name: 'shoppingCartShortInfo',
  canHandle: ['ProductItemAdded', 'DiscountApplied'],
  init: () =>
    SQL`CREATE TABLE IF NOT EXISTS shoppingCartShortInfo (
      id TEXT PRIMARY KEY,
      productItemsCount INTEGER,
      totalAmount INTEGER,
      discountsApplied JSON
    )`,
  evolve: (event) => {
    switch (event.type) {
      case 'ProductItemAdded': {
        const qty = event.data.productItem.quantity;
        const amount = event.data.productItem.price * qty;
        return SQL`INSERT INTO shoppingCartShortInfo
          (id, productItemsCount, totalAmount, discountsApplied)
          VALUES (${event.metadata.streamName}, ${qty}, ${amount}, '[]')
          ON CONFLICT (id) DO UPDATE SET
            productItemsCount = productItemsCount + ${qty},
            totalAmount = totalAmount + ${amount}`;
      }
      case 'DiscountApplied':
        return SQL`UPDATE shoppingCartShortInfo
          SET totalAmount = (totalAmount * (100 - ${event.data.percent})) / 100
          WHERE id = ${event.metadata.streamName}`;
    }
  },
});
```

> [!tip]
> The `init` function runs once during projection initialization. Use it for DDL like `CREATE TABLE IF NOT EXISTS`.

### Batch

Receives the full event array at once and returns an array of SQL statements:

```typescript
import { sqliteRawBatchSQLProjection } from '@event-driven-io/emmett-sqlite';

const projection = sqliteRawBatchSQLProjection({
  name: 'orderStats',
  canHandle: ['OrderPlaced'],
  evolve: (events) => {
    return events.map(
      (event) =>
        SQL`INSERT INTO orderStats (orderId, amount)
            VALUES (${event.data.orderId}, ${event.data.amount})`,
    );
  },
});
```

## Projection Handler Context

All SQLite projections receive a `SQLiteProjectionHandlerContext`:

```typescript
type SQLiteProjectionHandlerContext = {
  execute: SQLExecutor;       // Run arbitrary SQL
  connection: AnySQLiteConnection;
  driverType: DatabaseDriverType;
};
```

## Differences from PostgreSQL

Unlike the [[PostgreSQL Projections|PostgreSQL adapter]], the SQLite adapter:
- Has no advisory lock coordination for inline projections
- Has no projection management API (`registerProjection`, `activateProjection`, etc.)
- Uses the same [[Projection Concepts|evolve pattern]] but with SQLite-specific handler context

## See Also

- [[Projection Concepts]] -- Core projection patterns
- [[Pongo Document Projections]] -- Shared Pongo concepts across adapters
- [[Raw SQL Projections]] -- Shared raw SQL concepts across adapters
- [[SQLite Testing]] -- BDD testing for projections
- [[SQLite Consumer]] -- Async projection processing via projectors
