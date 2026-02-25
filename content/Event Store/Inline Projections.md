---
tags:
  - event-store
  - projections
aliases:
  - Inline Projection Registration
related:
  - "[[Projection Concepts]]"
  - "[[InMemory Projections]]"
  - "[[Pongo Document Projections]]"
  - "[[Raw SQL Projections]]"
  - "[[MongoDB Inline Projections]]"
  - "[[Appending Events]]"
package: emmett
---

# Inline Projections

Inline projections run synchronously as part of [[Appending Events|appendToStream]], after events are stored but before the [[After-Commit Hooks|after-commit hook]] fires. They update read models within the same operation, providing ==read-after-write consistency==.

> [!abstract]
> This note covers the general concept of inline projections and their registration. For adapter-specific implementations, see [[InMemory Projections]], [[Pongo Document Projections]], [[Raw SQL Projections]], and [[MongoDB Inline Projections]].

## Registration

Register projections when creating the event store using the `inlineProjections()` helper:

```typescript
import {
  getInMemoryEventStore,
  inlineProjections,
  inMemorySingleStreamProjection,
} from '@event-driven-io/emmett';

const eventStore = getInMemoryEventStore({
  projections: inlineProjections([
    myProjection,
  ]),
});
```

Or use the raw registration format:

```typescript
getInMemoryEventStore({
  projections: [
    { type: 'inline', projection: myProjectionDefinition },
  ],
});
```

## Projection Types (In-Memory)

The [[In-Memory Event Store]] provides three inline projection factories. For full details, see [[InMemory Projections]].

### Single-Stream Projections

Maps events from one stream to a single document. The document ID defaults to the stream name.

```typescript
import { inMemorySingleStreamProjection } from '@event-driven-io/emmett';

type ShoppingCartDetails = {
  productItems: PricedProductItem[];
  totalAmount: number;
  status: string;
};

const shoppingCartDetailsProjection = inMemorySingleStreamProjection<
  ShoppingCartDetails,
  ShoppingCartEvent
>({
  canHandle: ['ProductItemAdded', 'DiscountApplied'],
  collectionName: 'shoppingCartDetails',
  evolve: (document, event) => {
    switch (event.type) {
      case 'ProductItemAdded':
        return {
          productItems: [...(document?.productItems ?? []), event.data.productItem],
          totalAmount:
            (document?.totalAmount ?? 0) +
            event.data.productItem.price * event.data.productItem.quantity,
          status: 'open',
        };
      case 'DiscountApplied':
        return {
          ...document!,
          totalAmount: document!.totalAmount * (1 - event.data.percent / 100),
        };
    }
  },
});
```

Custom document ID:

```typescript
const projection = inMemorySingleStreamProjection({
  canHandle: ['ProductItemAdded'],
  collectionName: 'products',
  getDocumentId: (event) => event.data.productItem.productId,
  evolve: (document, event) => { /* ... */ },
});
```

### Multi-Stream Projections

Aggregates events from multiple streams into shared documents. The `getDocumentId` function is required:

```typescript
import { inMemoryMultiStreamProjection } from '@event-driven-io/emmett';

const productSalesProjection = inMemoryMultiStreamProjection<
  { totalSold: number },
  ProductItemAdded
>({
  canHandle: ['ProductItemAdded'],
  collectionName: 'productSales',
  getDocumentId: (event) => event.data.productItem.productId,
  evolve: (document, event) => ({
    totalSold: (document?.totalSold ?? 0) + event.data.productItem.quantity,
  }),
});
```

### Custom Projections

For full control, use `inMemoryProjection` with a custom `handle` function:

```typescript
import { inMemoryProjection } from '@event-driven-io/emmett';

const customProjection = inMemoryProjection<ShoppingCartEvent>({
  canHandle: ['ProductItemAdded', 'DiscountApplied'],
  handle: async (events, { database }) => {
    for (const event of events) {
      const collection = database.collection('myCollection');
      await collection.handle(event.metadata.streamName, (doc) => ({
        ...doc,
        lastUpdated: new Date().toISOString(),
      }));
    }
  },
});
```

## Evolve Patterns

Single-stream and multi-stream projections support two evolve signatures:

**Nullable document (no `initialState`)** -- the document is `null` for the first event:

```typescript
evolve: (document: DocumentType | null, event: ReadEvent<EventType>) => DocumentType | null;
```

**Non-null document (with `initialState`)** -- the document always starts from `initialState()`:

```typescript
{
  initialState: () => ({ items: [], count: 0 }),
  evolve: (document: DocumentType, event: ReadEvent<EventType>) => DocumentType | null,
}
```

In both cases, returning `null` from `evolve` ==deletes== the document from the collection.

## Processing Behavior

- Only projections whose `canHandle` array includes at least one of the appended event types will execute
- Events are processed one at a time within a projection
- Projections run synchronously after append, before [[After-Commit Hooks|onAfterCommit]]

> [!warning]
> If a projection throws, the [[After-Commit Hooks|after-commit hook]] will not fire, but the events are already stored.

## Adapter-Specific Implementations

| Adapter | Projection Types | Notes |
|---------|-----------------|-------|
| InMemory | Single-stream, multi-stream, custom | Updates [[In-Memory Database]] |
| PostgreSQL | Pongo, raw SQL, generic | Runs inside DB transaction via internal [[Before-Commit Hooks\|before-commit hook]] |
| MongoDB | Embedded in event document | Stored under `projections` field |
| SQLite | Pongo, raw SQL | Runs inside DB transaction via [[Before-Commit Hooks\|onBeforeCommit]] |

## See Also

- [[Projection Concepts]] -- What projections are and how they work
- [[InMemory Projections]] -- Full details on in-memory projection types
- [[Appending Events]] -- Where inline projections fit in the execution order
- [[Async Projections]] -- Alternative: eventual consistency via consumer/projector
- [[Testing Projections]] -- BDD-style testing for projections
