---
tags:
  - adapter
  - postgresql
  - projections
aliases:
  - PostgreSQL Inline Projections
related:
  - "[[Pongo Document Projections]]"
  - "[[Raw SQL Projections]]"
  - "[[PostgreSQL Event Store]]"
  - "[[Projection Concepts]]"
package: emmett-postgresql
---

# PostgreSQL Projections

The PostgreSQL adapter provides five projection types for building read models from event streams. All can be used as [[Inline Projections Overview|inline projections]] (running within the `appendToStream` transaction) or as [[Async Projections|async projections]] (via a [[PostgreSQL Consumer|consumer]]).

## Registering Inline Projections

Register projections when creating the event store with [[PostgreSQL Event Store|`getPostgreSQLEventStore()`]]:

```typescript
import { projections } from '@event-driven-io/emmett';
import { getPostgreSQLEventStore } from '@event-driven-io/emmett-postgresql';

const eventStore = getPostgreSQLEventStore(connectionString, {
  projections: projections.inline([
    myPongoProjection,
    myRawSQLProjection,
  ]),
});
```

Inline projections execute within the `appendToStream` transaction, ensuring the projected read model is ==always consistent with the written events==.

## Pongo Single Stream Projection

Projects events from a single stream into a [[Pongo Document Projections|Pongo document]], keyed by the stream name:

```typescript
import { pongoSingleStreamProjection } from '@event-driven-io/emmett-postgresql';

type ShoppingCartDetails = {
  clientId: string;
  productItems: PricedProductItem[];
  totalAmount: number;
  status: 'Opened' | 'Confirmed' | 'Cancelled';
};

const shoppingCartDetailsProjection = pongoSingleStreamProjection({
  collectionName: 'shoppingCartDetails',
  evolve: (document: ShoppingCartDetails | null, event: ShoppingCartEvent) => {
    switch (event.type) {
      case 'ProductItemAddedToShoppingCart': {
        const doc = document ?? {
          status: 'Opened',
          productItems: [],
          totalAmount: 0,
        };
        return {
          ...doc,
          clientId: event.data.clientId,
          productItems: [...doc.productItems, event.data.productItem],
          totalAmount:
            doc.totalAmount +
            event.data.productItem.unitPrice * event.data.productItem.quantity,
        };
      }
      case 'ShoppingCartConfirmed':
        return { ...document!, status: 'Confirmed' };
      case 'ShoppingCartCancelled':
        return { ...document!, status: 'Cancelled' };
      default:
        return document;
    }
  },
  canHandle: [
    'ProductItemAddedToShoppingCart',
    'ProductItemRemovedFromShoppingCart',
    'ShoppingCartConfirmed',
    'ShoppingCartCancelled',
  ],
});
```

### With `initialState`

When you provide `initialState`, the [[Evolve Function|evolve]] function receives a non-null document (the initial state is used for the first event in a new stream):

```typescript
const shortInfoProjection = pongoSingleStreamProjection({
  collectionName: 'shoppingCartShortInfo',
  evolve: (document: ShoppingCartShortInfo, event: ShoppingCartEvent) => {
    // document is never null when initialState is provided
    switch (event.type) {
      case 'ProductItemAddedToShoppingCart':
        return {
          totalAmount:
            document.totalAmount +
            event.data.productItem.unitPrice * event.data.productItem.quantity,
          productItemsCount:
            document.productItemsCount + event.data.productItem.quantity,
        };
      case 'ShoppingCartConfirmed':
      case 'ShoppingCartCancelled':
        return null; // returning null deletes the document
      default:
        return document;
    }
  },
  canHandle: [
    'ProductItemAddedToShoppingCart',
    'ProductItemRemovedFromShoppingCart',
    'ShoppingCartConfirmed',
    'ShoppingCartCancelled',
  ],
  initialState: () => ({ productItemsCount: 0, totalAmount: 0 }),
});
```

> [!tip]
> Returning `null` from the `evolve` function ==deletes the document== from the Pongo collection. This is useful for projections like short-lived summaries that should be removed when a cart is confirmed or cancelled.

### Collection Versioning

Set `version` to append a `_v{N}` suffix to the collection name, allowing multiple projection versions to coexist:

```typescript
const projection = pongoSingleStreamProjection({
  collectionName: 'shoppingCartDetails',
  version: 2,
  // ...
});
// Actual collection name: 'shoppingCartDetails_v2'
```

> [!warning]
> When `version` is set, your read queries must use the versioned collection name (`shoppingCartDetails_v2`), not the base name.

## Pongo Multi Stream Projection

Aggregates events from multiple streams into a single document, keyed by a custom ID. The `getDocumentId` function is required:

```typescript
import { pongoMultiStreamProjection } from '@event-driven-io/emmett-postgresql';

const clientShoppingSummaryProjection = pongoMultiStreamProjection({
  collectionName: 'ClientShoppingSummary',
  getDocumentId: (event) => event.metadata.clientId,
  evolve: (document: ClientShoppingSummary | null, event: ShoppingCartEvent) => {
    const summary = document ?? {
      clientId: event.metadata.clientId,
      pending: undefined,
      confirmed: { cartsCount: 0, productItemsCount: 0, totalAmount: 0 },
      cancelled: { cartsCount: 0, productItemsCount: 0, totalAmount: 0 },
    };
    // ... build summary across multiple shopping cart streams
    return summary;
  },
  canHandle: [
    'ProductItemAddedToShoppingCart',
    'ProductItemRemovedFromShoppingCart',
    'ShoppingCartConfirmed',
    'ShoppingCartCancelled',
  ],
});
```

## Raw SQL Projection

Execute raw SQL statements per event. Use `init` to create the target table:

```typescript
import { postgreSQLRawSQLProjection } from '@event-driven-io/emmett-postgresql';

const projection = postgreSQLRawSQLProjection({
  name: 'orderTotals',
  canHandle: ['OrderPlaced', 'OrderCancelled'],
  init: (context) => sql`
    CREATE TABLE IF NOT EXISTS order_totals (
      order_id TEXT PRIMARY KEY,
      total NUMERIC NOT NULL
    )
  `,
  evolve: (event, context) => {
    switch (event.type) {
      case 'OrderPlaced':
        return sql`INSERT INTO order_totals (order_id, total)
                   VALUES (${event.data.orderId}, ${event.data.total})`;
      case 'OrderCancelled':
        return sql`DELETE FROM order_totals
                   WHERE order_id = ${event.data.orderId}`;
      default:
        return [];
    }
  },
});
```

See also [[Raw SQL Projections]] for the general concept across adapters.

## Raw Batch SQL Projection

Receive all events in a batch at once (useful for bulk operations):

```typescript
import { postgreSQLRawBatchSQLProjection } from '@event-driven-io/emmett-postgresql';

const projection = postgreSQLRawBatchSQLProjection({
  name: 'eventLog',
  canHandle: ['OrderPlaced', 'OrderCancelled'],
  evolve: (events, context) => {
    return events.map((event) =>
      sql`INSERT INTO event_log (event_type, event_data)
          VALUES (${event.type}, ${JSON.stringify(event.data)})`
    );
  },
});
```

## Generic Projection

For complete control over the projection logic, use `postgreSQLProjection()`:

```typescript
import { postgreSQLProjection } from '@event-driven-io/emmett-postgresql';

const projection = postgreSQLProjection({
  name: 'customProjection',
  canHandle: ['OrderPlaced'],
  init: async (options) => {
    // Run setup SQL, create tables, etc.
  },
  truncate: async (context) => {
    // Clean up projection data (called during rebuild)
  },
  handle: async (events, context) => {
    // Full access to context.execute (SQL executor),
    // context.connection.client, context.connection.transaction
    for (const event of events) {
      await context.execute(sql`INSERT INTO ...`);
    }
  },
});
```

The generic projection wraps the core `projection()` function and automatically registers itself in the `emt_projections` table during initialization.

## Projection Handler Context

All PostgreSQL projections receive a rich context:

```typescript
type PostgreSQLProjectionHandlerContext = {
  execute: SQLExecutor;
  connection: {
    connectionString: string;
    client: PgClient;
    transaction: PgTransaction;
    pool: Dumbo;
  };
};
```

Pongo projections additionally receive a `PongoClient` scoped to the current transaction:

```typescript
type PongoProjectionHandlerContext = PostgreSQLProjectionHandlerContext & {
  pongo: PongoClient;
};
```

> [!note]
> The Pongo client is created per-invocation using the transaction's connection and closed after the handler runs. You do not need to manage its lifecycle.

## Reading Projected Data

Pongo projections store documents in PostgreSQL using JSONB. Query them with the Pongo client:

```typescript
import { pongoClient } from '@event-driven-io/pongo';

const pongo = pongoClient(connectionString);
const db = pongo.db();

// Find by ID (stream name for single-stream projections)
const cart = await db
  .collection<ShoppingCartDetails>('shoppingCartDetails')
  .findOne({ _id: 'shopping_cart-abc123' });

// Query with filters
const openCarts = await db
  .collection<ShoppingCartDetails>('shoppingCartDetails')
  .find({ status: 'Opened' })
  .toArray();
```

## See Also

- [[Projection Concepts]] -- General projection concepts including evolve patterns and deletion semantics
- [[Pongo Document Projections]] -- Pongo projections in detail (also available on SQLite)
- [[Raw SQL Projections]] -- Raw SQL projections across adapters
- [[PostgreSQL Rebuilding Projections]] -- How to rebuild projections from scratch
- [[PostgreSQL Testing]] -- BDD-style testing for projections
- [[Inline Projections Overview]] -- How inline projections work during `appendToStream`
