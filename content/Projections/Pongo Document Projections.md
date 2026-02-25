---
tags:
  - projections
  - pongo
aliases:
  - pongoSingleStreamProjection
  - pongoMultiStreamProjection
related:
  - "[[Projection Concepts]]"
  - "[[PostgreSQL Event Store]]"
  - "[[SQLite Event Store]]"
  - "[[Projection Rebuilding]]"
package: emmett-postgresql
---

# Pongo Document Projections

> [!abstract]
> Pongo projections use [Pongo](https://github.com/event-driven-io/pongo) -- a MongoDB-like document-database layer on top of PostgreSQL JSONB. This is the ==most common pattern== for PostgreSQL projections. Pongo projections are also available on SQLite.

## PostgreSQL: Single-Stream

Creates one read model document per event stream, stored in a Pongo collection (which becomes a PostgreSQL table):

```typescript
import { pongoSingleStreamProjection } from '@event-driven-io/emmett-postgresql';

const shortInfoProjection = pongoSingleStreamProjection<
  ShoppingCartShortInfo,
  ShoppingCartEvent
>({
  collectionName: 'shoppingCartShortInfo',
  canHandle: [
    'ProductItemAdded',
    'ProductItemRemoved',
    'ShoppingCartConfirmed',
    'ShoppingCartCancelled',
  ],
  evolve,
  initialState: () => ({ productItemsCount: 0, totalAmount: 0 }),
});
```

## PostgreSQL: Multi-Stream

Creates documents with custom IDs that aggregate data across multiple streams:

```typescript
import { pongoMultiStreamProjection } from '@event-driven-io/emmett-postgresql';

const productSalesProjection = pongoMultiStreamProjection<
  ProductSales,
  ShoppingCartEvent
>({
  collectionName: 'productSales',
  canHandle: ['ProductItemAdded', 'ProductItemRemoved'],
  getDocumentId: (event) => event.data.productItem.productId,
  evolve: salesEvolve,
  initialState: () => ({ totalSold: 0, revenue: 0 }),
});
```

## Options

| Option | Type | Description |
|---|---|---|
| `collectionName` | `string` | Pongo collection name (becomes a PostgreSQL table) |
| `canHandle` | `string[]` | Event type names this projection handles |
| `evolve` | `(doc, event) => Doc \| null` | Transform function |
| `initialState` | `() => Doc` | Factory for initial document (optional if evolve accepts nullable documents) |
| `getDocumentId` | `(event) => string` | Maps event to document ID (required for multi-stream; defaults to stream name for single-stream) |
| `version` | `number` | Version number. When > 0, appends `_v{version}` to the collection name |
| `collectionOptions` | `PongoDBCollectionOptions` | Pongo-specific collection settings (schema config, etc.) |

### Version Suffixing

> [!warning]
> When `version` is set to a value ==greater than 0==, Pongo projections append `_v{version}` to the collection name. For example, `collectionName: 'shoppingCartShortInfo'` with `version: 2` becomes `shoppingCartShortInfo_v2`. Version `0` or `undefined` uses the raw collection name.

## Registration

```typescript
import {
  getPostgreSQLEventStore,
  pongoSingleStreamProjection,
} from '@event-driven-io/emmett-postgresql';
import { projections } from '@event-driven-io/emmett';

const eventStore = getPostgreSQLEventStore(connectionString, {
  projections: projections.inline([shortInfoProjection]),
});
```

## SQLite: Pongo Projections

Pongo projections are also available on SQLite with the same API shape:

```typescript
import {
  sqlitePongoSingleStreamProjection,
} from '@event-driven-io/emmett-sqlite';

const shortInfoProjection = sqlitePongoSingleStreamProjection<
  ShoppingCartShortInfo,
  ShoppingCartEvent
>({
  collectionName: 'shoppingCartShortInfo',
  canHandle: ['ProductItemAdded', 'ProductItemRemoved'],
  evolve,
  initialState: () => ({ productItemsCount: 0, totalAmount: 0 }),
});
```

> [!note]
> SQLite Pongo projections use the same API as PostgreSQL but without advisory locks or a projection management table. There is no activate/deactivate functionality on SQLite.

## Generic `pongoProjection`

For full control, use the lower-level `pongoProjection()` factory, which accepts a custom `handle` function instead of the `evolve`/`initialState` pattern:

```typescript
import { pongoProjection } from '@event-driven-io/emmett-postgresql';
```

## How Pongo Handles Deletion

When `evolve` returns `null`, Pongo's `collection.handle(id, handler)` method detects the null return and deletes the document. This is handled automatically by the single-stream and multi-stream wrappers.

## Querying Projected Data

Pongo collections are regular PostgreSQL tables with JSONB storage. You can query them using a Pongo client or raw SQL:

```typescript
const pongoClient = pongoClient(connectionString);
const collection = pongoClient.db().collection<ShoppingCartShortInfo>('shoppingCartShortInfo');
const doc = await collection.findOne({ _id: 'shopping_cart-123' });
```

## See Also

- [[Projection Concepts]] -- Core concepts: evolve, single-stream vs multi-stream, deletion
- [[Raw SQL Projections]] -- Alternative when you need full SQL control
- [[PostgreSQL Event Store]] -- Event store that hosts PostgreSQL projections
- [[SQLite Event Store]] -- Event store that hosts SQLite projections
- [[Testing Projections]] -- `PostgreSQLProjectionSpec` and Pongo assertion helpers
