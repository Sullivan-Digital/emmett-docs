---
tags:
  - projections
  - mongodb
  - inline
aliases:
  - mongoDBInlineProjection
  - MongoDB Read Model
related:
  - "[[Projection Concepts]]"
  - "[[MongoDB Event Store]]"
  - "[[MongoDB Storage Strategies]]"
  - "[[MongoDB Testing]]"
  - "[[Inline Projections Overview]]"
package: emmett-mongodb
---

# MongoDB Inline Projections

Inline projections are read models computed ==atomically during `appendToStream`== and stored inside the stream document under the `projections` field. They provide strongly consistent read models with zero additional infrastructure -- no separate collections, no eventual consistency, no change stream consumer required.

> [!abstract]
> Unlike other adapters that store projections in separate tables/collections, MongoDB's inline projections live **inside** the event stream document. The projection evolve function runs during the same `updateOne` operation that appends events.

## Defining Projections

Use `mongoDBInlineProjection()` to define a projection, then register it when creating the [[MongoDB Event Store|event store]]:

```typescript
import { projections } from '@event-driven-io/emmett';
import {
  getMongoDBEventStore,
  mongoDBInlineProjection,
} from '@event-driven-io/emmett-mongodb';

type ShoppingCartSummary = {
  productItemsCount: number;
  totalAmount: number;
};

const summaryProjection = mongoDBInlineProjection<
  ShoppingCartSummary,
  ShoppingCartEvent
>({
  canHandle: ['ProductItemAdded', 'ShoppingCartConfirmed'],
  evolve: (doc, event) => {
    switch (event.type) {
      case 'ProductItemAdded':
        return {
          productItemsCount:
            doc.productItemsCount + event.data.productItem.quantity,
          totalAmount:
            doc.totalAmount +
            event.data.productItem.price * event.data.productItem.quantity,
        };
      default:
        return doc;
    }
  },
  initialState: () => ({ productItemsCount: 0, totalAmount: 0 }),
});

const eventStore = getMongoDBEventStore({
  client,
  projections: projections.inline([summaryProjection]),
});
```

## Two Evolve Patterns

### With `initialState` (Recommended)

The `evolve` function always receives a non-null document. On the first event for a new stream, `initialState()` creates the seed value.

```typescript
mongoDBInlineProjection({
  canHandle: ['ProductItemAdded', 'DiscountApplied'],
  evolve: (doc, event) => {
    // doc is always non-null -- guaranteed by initialState
    return { ...doc, count: doc.count + 1 };
  },
  initialState: () => ({ count: 0, total: 0 }),
});
```

### Without `initialState`

The `evolve` function receives `null` when no prior projection data exists for the stream. You handle initialization manually.

```typescript
mongoDBInlineProjection({
  canHandle: ['ProductItemAdded', 'ShoppingCartCancelled'],
  evolve: (doc, event) => {
    // doc may be null for a new stream
    const current = doc ?? { count: 0, total: 0 };
    switch (event.type) {
      case 'ShoppingCartCancelled':
        return null; // soft delete
      default:
        return { ...current, count: current.count + 1 };
    }
  },
});
```

> [!info]
> See [[Evolve Function]] and [[Projection Concepts]] for the general nullable vs non-nullable evolve pattern used across all projection types.

## Named Projections

By default, projections use the name `'_default'` (`MongoDBDefaultInlineProjectionName`). You can register multiple projections per stream by giving them distinct names:

```typescript
const detailsProjection = mongoDBInlineProjection({
  // name defaults to '_default'
  canHandle: ['ProductItemAdded', 'ShoppingCartConfirmed'],
  evolve: detailsEvolve,
});

const shortInfoProjection = mongoDBInlineProjection({
  name: 'shoppingCartShortInfo',
  canHandle: ['ProductItemAdded', 'ShoppingCartConfirmed'],
  evolve: shortInfoEvolve,
  initialState: () => ({ productItemsCount: 0, totalAmount: 0 }),
});

const eventStore = getMongoDBEventStore({
  client,
  projections: projections.inline([detailsProjection, shortInfoProjection]),
});
```

> [!warning] Duplicate Name Detection
> Registering two projections with the same name (including two unnamed projections that both default to `'_default'`) throws an `EmmettError`. Deduplication is handled by the core `filterProjections()` utility from `@event-driven-io/emmett`.

## Soft Deletes

When an `evolve` function returns `null`, the projection value is set to `null` in the document. The query helpers automatically filter out `null` projections, so soft-deleted items will not appear in `find`, `findOne`, or `count` results.

```typescript
const projection = mongoDBInlineProjection({
  canHandle: ['ProductItemAdded', 'ShoppingCartCancelled'],
  evolve: (doc, event) => {
    if (event.type === 'ShoppingCartCancelled') return null; // soft delete
    return doc ?? { itemCount: 0 };
  },
});
```

## Querying Projections

The event store exposes `projections.inline` with three query methods: `findOne`, `find`, and `count`. All return `MongoDBReadModel<Doc>` -- your document type extended with a `_metadata` field.

### `findOne` -- Single Projection

```typescript
// By stream name (most common)
const result = await eventStore.projections.inline.findOne<ShoppingCartSummary>({
  streamName: 'shopping_cart:abc-123',
});
// Returns MongoDBReadModel<ShoppingCartSummary> | null

// Named projection
const result = await eventStore.projections.inline.findOne<ShoppingCartShortInfo>(
  {
    streamName: 'shopping_cart:abc-123',
    projectionName: 'shoppingCartShortInfo',
  },
);

// By streamType + streamId
const result = await eventStore.projections.inline.findOne<ShoppingCartSummary>({
  streamType: 'shopping_cart',
  streamId: 'abc-123',
});
```

### `find` -- Multiple Projections

```typescript
// All projections for a stream type
const results = await eventStore.projections.inline.find<ShoppingCartSummary>({
  streamType: 'shopping_cart',
});

// Specific stream IDs
const results = await eventStore.projections.inline.find<ShoppingCartSummary>({
  streamType: 'shopping_cart',
  streamIds: ['id1', 'id2', 'id3'],
});

// By explicit stream names
const results = await eventStore.projections.inline.find<ShoppingCartSummary>({
  streamNames: ['shopping_cart:id1', 'shopping_cart:id2'],
});

// With MongoDB filter and pagination
const results = await eventStore.projections.inline.find<ShoppingCartSummary>(
  { streamType: 'shopping_cart' },
  { totalAmount: { $gte: 100 }, productItemsCount: { $eq: 5 } },
  {
    skip: 20,
    limit: 10,
    sort: [['_metadata.streamId', 1]],
  },
);
```

### `count` -- Count Matching Projections

```typescript
const count = await eventStore.projections.inline.count<ShoppingCartSummary>(
  { streamType: 'shopping_cart' },
  { totalAmount: { $gte: 50 } },
);
```

### Filter Transformation

Projection query filters are automatically prefixed with `projections.{projectionName}.` by the `prependMongoFilterWithProjectionPrefix()` utility. You write filters as if querying the projection document directly:

- `{ totalAmount: { $gte: 20 } }` becomes `{ 'projections._default.totalAmount': { $gte: 20 } }`
- You can also filter on metadata: `{ '_metadata.schemaVersion': { $eq: 1 } }`

### Projection Metadata

Each projection result includes a `_metadata` field of type `MongoDBReadModelMetadata`:

```typescript
const result = await eventStore.projections.inline.findOne<ShoppingCartSummary>({
  streamName: 'shopping_cart:abc-123',
});

if (result) {
  console.log(result._metadata.streamId);       // "abc-123"
  console.log(result._metadata.name);            // "_default"
  console.log(result._metadata.schemaVersion);   // 1
  console.log(result._metadata.streamPosition);  // 5n (bigint)
}
```

## Query Edge Cases

> [!warning] Empty Arrays in Queries
> When calling `projections.inline.find()`:
> - An empty `streamNames` array (`[]`) returns an empty result immediately (short-circuit)
> - An empty `streamIds` array is ==not== applied as a filter -- it returns all projections for that stream type

## Types

```typescript
type MongoDBReadModelMetadata = {
  streamId: string;
  name: string;
  schemaVersion: number;
  streamPosition: bigint;
};

type MongoDBReadModel<Doc extends Document = Document> = Doc & {
  _metadata: MongoDBReadModelMetadata;
};
```
