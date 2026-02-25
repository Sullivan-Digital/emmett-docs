---
tags:
  - event-store
  - operation
aliases:
  - appendToStream
related:
  - "[[Event Store Interface]]"
  - "[[Concurrency Control]]"
  - "[[Inline Projections]]"
  - "[[After-Commit Hooks]]"
  - "[[Event Metadata]]"
package: emmett
---

# Appending Events

The ==`appendToStream`== method appends one or more events to the end of a stream. If the stream does not exist, it is created automatically. Multiple events in a single call are stored atomically.

## Type Definitions

```typescript
type AppendToStreamOptions<EventType extends Event = Event, EventPayloadType extends Event = EventType> = {
  expectedStreamVersion?: ExpectedStreamVersion;  // concurrency check before appending
  schema?: EventStoreAppendSchemaOptions<EventType, EventPayloadType>;  // downcasting options
};

type AppendToStreamResult = {
  nextExpectedStreamVersion: StreamPosition;  // stream position of the last appended event
  createdNewStream: boolean;                  // true if this append created the stream
};
```
^appendtostream-types

## Basic Usage

```typescript
import type { Event } from '@event-driven-io/emmett';

type ProductItemAdded = Event<'ProductItemAdded', { productItem: PricedProductItem }>;
type ShoppingCartEvent = ProductItemAdded;

const result = await eventStore.appendToStream<ShoppingCartEvent>(
  'shoppingCart-123',
  [{ type: 'ProductItemAdded', data: { productItem } }],
);

console.log(result.nextExpectedStreamVersion); // e.g. 1n
console.log(result.createdNewStream);          // true (if this was the first append)
```

## Atomic Multi-Event Writes

```typescript
const result = await eventStore.appendToStream<ShoppingCartEvent>(
  'shoppingCart-123',
  [
    { type: 'ProductItemAdded', data: { productItem: item1 } },
    { type: 'ProductItemAdded', data: { productItem: item2 } },
    { type: 'DiscountApplied', data: { percent: 10, couponId: 'SAVE10' } },
  ],
);
// All three events are stored atomically
// result.nextExpectedStreamVersion is the position of the last event
```

## Execution Order

When `appendToStream` is called, the following steps execute in order:

1. **Expected version check** -- throws `ExpectedVersionConflictError` on mismatch (see [[Concurrency Control]])
2. **Events are stored** with system-assigned [[Event Metadata|metadata]]
3. **[[Inline Projections]]** run (if any are registered)
4. **[[After-Commit Hooks|onAfterCommit hook]]** fires (fire-and-forget)

> [!warning]
> If a projection throws, the [[After-Commit Hooks|after-commit hook]] will not fire, but the events are already stored.

## Metadata Assignment

When events are appended, the store automatically assigns metadata to each event:

- `messageId` -- UUID v4, auto-generated
- `streamPosition` -- 1-based position within the stream
- `globalPosition` -- 1-based position across all streams
- `checkpoint` -- branded string derived from `globalPosition`, used by processors
- `streamName` -- the stream being appended to
- `kind` -- set to `'Event'` if not already provided

User-provided `metadata` on events is merged with system metadata. See [[Event Metadata]] for details.

> [!warning]
> System metadata takes precedence. If your event includes a `streamName` field in its metadata, the system will overwrite it.

## Downcasting on Write

If you need to transform events to a different storage format, use the `schema.versioning.downcast` option:

```typescript
await eventStore.appendToStream<ProductItemAddedV2, ProductItemAddedV1>(
  'shoppingCart-123',
  [currentFormatEvent],
  {
    schema: {
      versioning: {
        downcast: (event: ProductItemAddedV2): ProductItemAddedV1 => ({
          type: 'ProductItemAdded',
          data: {
            productId: event.data.productItem.productId,
            quantity: event.data.productItem.quantity,
          },
        }),
      },
    },
  },
);
```

> [!note]
> Downcast happens ==after metadata assignment==. The downcast function receives the fully enriched `ReadEvent`, not the raw event.

See [[Schema Versioning]] for full details.

## See Also

- [[Event Store Interface]] -- The four-method interface
- [[Concurrency Control]] -- Preventing conflicting writes
- [[Inline Projections]] -- Synchronous read model updates during append
- [[After-Commit Hooks]] -- Fire-and-forget side effects after append
- [[Event Metadata]] -- System-assigned metadata fields
