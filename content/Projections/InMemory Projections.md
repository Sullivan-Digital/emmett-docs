---
tags:
  - projections
  - in-memory
aliases:
  - inMemorySingleStreamProjection
  - inMemoryMultiStreamProjection
related:
  - "[[Projection Concepts]]"
  - "[[In-Memory Event Store]]"
  - "[[In-Memory Database]]"
  - "[[Testing Projections]]"
package: emmett
---

# InMemory Projections

> [!abstract]
> The in-memory adapter stores read models in an `InMemoryDatabase` with named collections. It provides `inMemorySingleStreamProjection()` and `inMemoryMultiStreamProjection()` for the two most common patterns, plus a lower-level `inMemoryProjection()` for custom logic.

## Single-Stream Projection

Creates one read model document per event stream. The document ID ==defaults to `event.metadata.streamName`==.

```typescript
import {
  inMemorySingleStreamProjection,
  type InMemoryProjectionDefinition,
} from '@event-driven-io/emmett';

type ShoppingCartShortInfo = {
  productItemsCount: number;
  totalAmount: number;
};

const shortInfoProjection: InMemoryProjectionDefinition<ShoppingCartEvent> =
  inMemorySingleStreamProjection<ShoppingCartShortInfo, ShoppingCartEvent>({
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

You can override the document ID with a custom `getDocumentId` function if needed.

## Multi-Stream Projection

Creates documents with custom IDs that aggregate data across multiple streams. The ==`getDocumentId` function is required==.

```typescript
import { inMemoryMultiStreamProjection } from '@event-driven-io/emmett';

const productSalesProjection = inMemoryMultiStreamProjection<
  ProductSales,
  ShoppingCartEvent
>({
  collectionName: 'productSales',
  canHandle: ['ProductItemAdded', 'ProductItemRemoved'],
  getDocumentId: (event) => event.data.productItem.productId,
  evolve: (document, event) => {
    // aggregate sales data across all shopping carts
    // ...
  },
  initialState: () => ({ totalSold: 0, revenue: 0 }),
});
```

## Options

| Option | Type | Description |
|---|---|---|
| `collectionName` | `string` | Name of the in-memory collection |
| `canHandle` | `string[]` | Event type names this projection handles |
| `evolve` | `(doc, event) => Doc \| null` | Transform function (see [[Projection Concepts#Two Evolve Variants|evolve variants]]) |
| `initialState` | `() => Doc` | Factory for initial document state (required with non-nullable evolve) |
| `getDocumentId` | `(event) => string` | Maps event to document ID (required for multi-stream; defaults to stream name for single-stream) |

### Evolve Variants

InMemory projections support both evolve signatures:
- `InMemoryWithNotNullDocumentEvolve`: `(document: Doc, event) => Doc | null` -- paired with `initialState`
- `InMemoryWithNullableDocumentEvolve`: `(document: Doc | null, event) => Doc | null` -- handles null yourself

## Registration

```typescript
import { getInMemoryEventStore, projections } from '@event-driven-io/emmett';

const eventStore = getInMemoryEventStore({
  projections: projections.inline([
    shortInfoProjection,
    productSalesProjection,
  ]),
});
```

## Querying Read Models

In-memory projection results are stored in the [[In-Memory Database]]. Access them via the event store's `database` property:

```typescript
const db = eventStore.database;
const collection = db.collection<ShoppingCartShortInfo>('shoppingCartShortInfo');

const shortInfo = collection.findOne('shopping_cart-123');
```

## Truncation

The in-memory truncate implementation iterates all documents in the collection and deletes them one by one. There is no bulk delete operation.

> [!note]
> The `init()` function is ==not automatically called== for in-memory projections. If your projection defines `init()`, you must call it manually if needed.

## See Also

- [[Projection Concepts]] -- Core concepts: evolve, single-stream vs multi-stream, deletion
- [[In-Memory Event Store]] -- The event store that hosts these projections
- [[In-Memory Database]] -- The document store for querying read models
- [[Testing Projections]] -- `InMemoryProjectionSpec` for testing these projections
