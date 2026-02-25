---
tags:
  - event-store
  - operation
aliases:
  - readStream
related:
  - "[[Event Store Interface]]"
  - "[[Schema Versioning]]"
  - "[[Event Metadata]]"
package: emmett
---

# Reading Streams

The ==`readStream`== method reads events from a specific stream. It supports filtering by position range and maximum count, optional concurrency checks, and [[Schema Versioning|upcasting]] of stored events.

## Type Definitions

```typescript
type ReadStreamOptions<EventType extends Event = Event, EventPayloadType extends Event = EventType> = {
  from?: StreamPosition;                          // start reading from this position
  to?: StreamPosition;                            // read up to this position (exclusive)
  maxCount?: bigint;                              // maximum number of events to return
  expectedStreamVersion?: ExpectedStreamVersion;  // concurrency check before reading
  schema?: EventStoreReadSchemaOptions<EventType, EventPayloadType>;  // upcasting options
};

type ReadStreamResult<EventType extends Event, ReadEventMetadataType> = {
  currentStreamVersion: StreamPosition;  // total number of events in the stream
  events: ReadEvent<EventType, ReadEventMetadataType>[];
  streamExists: boolean;
};
```
^readstream-types

## Basic Usage

```typescript
import type { Event } from '@event-driven-io/emmett';

type ProductItemAdded = Event<'ProductItemAdded', { productItem: PricedProductItem }>;
type DiscountApplied = Event<'DiscountApplied', { percent: number; couponId: string }>;
type ShoppingCartEvent = ProductItemAdded | DiscountApplied;

// Read all events from a stream
const result = await eventStore.readStream<ShoppingCartEvent>(
  'shoppingCart-123',
);

console.log(result.currentStreamVersion); // e.g. 3n
console.log(result.events.length);        // e.g. 3
console.log(result.streamExists);         // true
```

## Reading a Range of Events

```typescript
// Read events from position 0 up to (but not including) position 2
const result = await eventStore.readStream<ShoppingCartEvent>(
  'shoppingCart-123',
  { from: 0n, to: 2n },
);
// result.events contains the first 2 events

// Read at most 5 events
const result = await eventStore.readStream<ShoppingCartEvent>(
  'shoppingCart-123',
  { maxCount: 5n },
);
```

> [!warning]
> Stream positions are ==1-based== (`streamPosition: 1n` for the first event), but `from` and `to` use ==0-based slice indices==. In the in-memory implementation, `readStream` uses `Array.slice(from, to)`. So `from: 0n, to: 1n` returns the first event, but that event's `streamPosition` is `1n`.

## Nonexistent Streams

When a stream does not exist, `readStream` returns a result with an empty events array, `streamExists: false`, and the default stream version (typically `0n`):

```typescript
const result = await eventStore.readStream('nonexistent-stream');
// result.currentStreamVersion === 0n
// result.events === []
// result.streamExists === false
```

No error is thrown for nonexistent streams.

## Upcasting on Read

If your stored events have an older schema, you can transform them during read via the `schema.versioning.upcast` option:

```typescript
const { events } = await eventStore.readStream<ShoppingCartEvent>(
  shoppingCartId,
  {
    schema: { versioning: { upcast } },
  },
);
// events are now typed as ReadEvent<ShoppingCartEvent>[]
```

See [[Schema Versioning]] for full details on defining upcast functions.

## See Also

- [[Event Store Interface]] -- The four-method interface
- [[Aggregating Streams]] -- Folding events into state (uses `readStream` internally)
- [[Event Metadata]] -- What metadata is available on each `ReadEvent`
- [[Concurrency Control]] -- The `expectedStreamVersion` option
