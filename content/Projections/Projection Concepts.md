---
tags:
  - projections
  - concept
aliases:
  - Read Model
  - Projection Definition
related:
  - "[[Evolve Function]]"
  - "[[Inline Projections Overview]]"
  - "[[Async Projections]]"
package: emmett
---

# Projection Concepts

> [!abstract]
> Projections transform event streams into read models using an `evolve` function. They can be single-stream (one document per aggregate) or multi-stream (custom IDs aggregating across streams), and support deletion by returning `null` from evolve.

## What Are Projections?

In event sourcing, the event store is the source of truth, but querying raw event streams for every read operation is impractical. ==Projections== maintain denormalized, query-optimized views (read models) that stay up-to-date as new events are appended.

All projection definitions across every adapter share a common `ProjectionDefinition` interface from the core `@event-driven-io/emmett` package.

## The `evolve` Function

The central concept in Emmett projections is the ==`evolve`== function. It takes the current state of a read model document and an event, then returns the updated document (or `null` to delete it).

```typescript
const evolve = (
  document: ShoppingCartShortInfo,
  { type, data }: ShoppingCartEvent,
): ShoppingCartShortInfo | null => {
  switch (type) {
    case 'ProductItemAdded':
      return {
        ...document,
        productItemsCount:
          document.productItemsCount + data.productItem.quantity,
        totalAmount:
          document.totalAmount +
          data.productItem.quantity * data.productItem.price,
      };
    case 'ProductItemRemoved':
      return {
        ...document,
        productItemsCount:
          document.productItemsCount - data.productItem.quantity,
        totalAmount:
          document.totalAmount -
          data.productItem.quantity * data.productItem.price,
      };
    case 'ShoppingCartConfirmed':
    case 'ShoppingCartCancelled':
      return null; // Delete the read model
    default:
      return document;
  }
};
```

> [!info]
> The projection `evolve` function follows the same pattern as the [[Evolve Function]] in the [[The Decider Pattern|Decider pattern]], but operates on read model documents instead of aggregate state.

### Two Evolve Variants

- **Non-nullable** -- `(document: Doc, event) => Doc | null`: The document parameter is always non-null. Paired with an `initialState` factory that provides the starting document when none exists yet.
- **Nullable** -- `(document: Doc | null, event) => Doc | null`: The document can be `null` (first event for this ID, or after deletion). You handle the null case yourself.

## Single-Stream vs Multi-Stream

### Single-Stream Projections

Create ==one read model document per event stream==. The document ID defaults to the stream name (e.g., `shopping_cart-123`). Use these when your read model maps 1:1 with an aggregate.

### Multi-Stream Projections

Create documents with ==custom IDs== that can aggregate data across multiple streams. You must provide a `getDocumentId` function that maps an event to the target document ID:

```typescript
getDocumentId: (event) => event.data.productItem.productId
```

This enables projections like "total sales per product" that aggregate events from many shopping cart streams into a single document per product.

## Deletion Semantics

Returning `null` from `evolve` signals that the read model document should be deleted. The exact behavior depends on the adapter:

| Adapter | Deletion Behavior |
|---|---|
| **InMemory** | Calls `deleteOne` on the in-memory collection |
| **PostgreSQL/SQLite (Pongo)** | Pongo's `collection.handle()` deletes the document when the handler returns `null` |
| **MongoDB** | Sets the projection field to `null` in the stream document (`$set` operation) |

## The `ProjectionDefinition` Interface

All adapter-specific projection factories produce objects implementing this core interface:

```typescript
interface ProjectionDefinition<EventType, EventMetaDataType, ProjectionHandlerContext, EventPayloadType> {
  name?: string;
  version?: number;
  kind?: string;
  canHandle: CanHandle<EventType>;
  handle: ProjectionHandler<EventType, EventMetaDataType, ProjectionHandlerContext>;
  truncate?: TruncateProjection<ProjectionHandlerContext>;
  init?: (options: ProjectionInitOptions<ProjectionHandlerContext>) => void | Promise<void>;
  eventsOptions?: {
    schema?: EventStoreReadSchemaOptions<EventType, EventPayloadType>;
  };
}
```

| Field | Description |
|---|---|
| `name` | Projection identifier. Used for locking (PostgreSQL), naming (MongoDB), and duplicate detection |
| `version` | Version number (defaults to 1). Used for lock keys and collection name suffixing |
| `kind` | String tag for the projection type (e.g., `'emt:projections:postgresql:pongo:single_stream'`) |
| `canHandle` | Array of event type strings this projection handles |
| `handle` | The core handler -- receives a batch of events and adapter-specific context |
| `truncate` | Clears all projection data. Used during [[Projection Rebuilding|rebuilds]] |
| `init` | Called during schema migration / first registration. Used to create tables or collections |
| `eventsOptions.schema` | Event [[Schema Versioning|versioning/upcasting]] options |

> [!warning]
> Although `name` is optional on `ProjectionDefinition`, you should ==always provide a name==. Without one, PostgreSQL skips advisory locking entirely and `filterProjections` cannot distinguish unnamed projections. Two projections with `undefined` names will throw an `EmmettError`.

## Registration Helpers

The core package provides convenience functions for registering projections:

```typescript
import { projections } from '@event-driven-io/emmett';

// Register as inline (synchronous, same transaction)
const eventStore = getEventStore({
  projections: projections.inline([myProjectionA, myProjectionB]),
});
```

> [!bug]
> The `projections.async()` / `asyncProjections()` helper currently returns `type: 'inline'` instead of `type: 'async'`. This appears to be a bug in the source code. Async projections are typically registered via `consumer.projector()` instead.

## The `canHandle` Filter

The `canHandle` array specifies which event types a projection processes. When events are appended, the system checks if any event type in the batch matches `canHandle`. If at least one matches, ==all events from the batch are passed to the handler== -- the `evolve` wrappers handle per-event filtering internally.

> [!tip]
> If you write a custom `handle` function instead of using the adapter-specific factories, you may need to filter events yourself since the batch may contain event types your projection does not handle.

## See Also

- [[Inline Projections Overview]] -- How inline projections run inside `appendToStream`
- [[Async Projections]] -- How async projections run via consumers
- [[Evolve Function]] -- The core state reducer pattern shared with deciders
- [[Shopping Cart Example]] -- Complete example using projections
