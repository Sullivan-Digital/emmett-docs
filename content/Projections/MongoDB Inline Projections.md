---
tags:
  - projections
  - mongodb
aliases:
  - mongoDBInlineProjection
  - MongoDB Read Model
related:
  - "[[Projection Concepts]]"
  - "[[MongoDB Event Store]]"
  - "[[MongoDB Storage Strategies]]"
  - "[[MongoDB Testing]]"
package: emmett-mongodb
---

# MongoDB Inline Projections

> [!abstract]
> MongoDB projections have a unique architecture: read models are stored ==inside the event stream document== under a `projections` field, rather than in separate collections. This is fundamentally different from all other adapters.

## Embedded Architecture

Unlike PostgreSQL, SQLite, and InMemory projections which store read models in separate collections or tables, MongoDB inline projections embed the read model ==directly in the event stream document==:

```
{
  streamName: "shopping_cart-123",
  messages: [...],
  metadata: {...},
  projections: {
    "shoppingCartShortInfo": {
      _metadata: { streamId: "...", name: "...", schemaVersion: 1, streamPosition: 5n },
      productItemsCount: 3,
      totalAmount: 45.50
    }
  }
}
```

Each projection is stored under a key matching its `name` in the `projections` field.

## Defining a Projection

```typescript
import { mongoDBInlineProjection } from '@event-driven-io/emmett-mongodb';

const shortInfoProjection = mongoDBInlineProjection<
  ShoppingCartShortInfo,
  ShoppingCartEvent
>({
  name: 'shoppingCartShortInfo',
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

## Options

| Option | Type | Default | Description |
|---|---|---|---|
| `name` | `string` | `'_default'` | Projection name (key in the `projections` field) |
| `schemaVersion` | `number` | `1` | Schema version stored in read model metadata |
| `canHandle` | `string[]` | -- | Event type names this projection handles |
| `evolve` | `(doc, event) => Doc \| null` | -- | Transform function |
| `initialState` | `() => Doc` | -- | Factory for initial document (optional if evolve accepts nullable) |

> [!warning]
> If you omit the `name` option, it defaults to ==`'_default'`==. Multiple unnamed projections on the same stream would overwrite each other. Always provide a unique name.

## Registration

```typescript
import { getMongoDBEventStore } from '@event-driven-io/emmett-mongodb';
import { projections } from '@event-driven-io/emmett';

const eventStore = getMongoDBEventStore({
  projections: projections.inline([shortInfoProjection]),
  client: mongoClient,
});
```

## Querying Projections

Since projections are embedded in stream documents, you ==cannot query them with standard MongoDB collection queries==. Use the event store's built-in helpers:

### Find by Stream Name

```typescript
const result = await eventStore.projections.inline.findOne<ShoppingCartShortInfo>({
  streamName: 'shopping_cart-123',
  projectionName: 'shoppingCartShortInfo',
});
```

### Find by Stream Type

```typescript
const results = await eventStore.projections.inline.find<ShoppingCartShortInfo>({
  streamType: 'shopping_cart',
  projectionName: 'shoppingCartShortInfo',
});
```

### Count

```typescript
const count = await eventStore.projections.inline.count<ShoppingCartShortInfo>({
  streamType: 'shopping_cart',
  projectionName: 'shoppingCartShortInfo',
});
```

## Read Model Metadata

Each embedded projection includes a `_metadata` field with tracking information:

```typescript
type MongoDBReadModelMetadata = {
  streamId: string;
  name: string;
  schemaVersion: number;
  streamPosition: bigint;
};
```

## Deletion Semantics

When `evolve` returns `null`, MongoDB ==sets the projection field to `null`== in the stream document using `$set`:

```typescript
updates.$set['projections.shoppingCartShortInfo'] = null;
```

This is a soft delete -- the key remains in the document with a null value, unlike other adapters which fully remove the document.

## Limitations

> [!warning] MongoDB-Specific Constraints
> - **Inline only**: MongoDB has ==no native async projection handler==. For async processing, use `inMemoryProjector` with a MongoDB consumer.
> - **No `init()` support**: MongoDB projections do not support the `init()` function.
> - **Embedded storage**: Projection data is coupled to the stream document lifecycle. If the stream document grows large, projection data contributes to that size.
> - **No separate query**: You must use the event store's query helpers (`findOne`, `find`, `count`) -- standard MongoDB collection queries won't find embedded projections.

## See Also

- [[Projection Concepts]] -- Core concepts: evolve, single-stream vs multi-stream, deletion
- [[MongoDB Event Store]] -- The event store that hosts these projections
- [[MongoDB Storage Strategies]] -- How stream documents are organized
- [[Testing Projections]] -- `MongoDBInlineProjectionSpec` for testing
