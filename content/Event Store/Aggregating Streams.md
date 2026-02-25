---
tags:
  - event-store
  - operation
aliases:
  - aggregateStream
related:
  - "[[Event Store Interface]]"
  - "[[Evolve Function]]"
  - "[[The Decider Pattern]]"
package: emmett
---

# Aggregating Streams

The ==`aggregateStream`== method reads events from a stream and reduces them into a state object using an `evolve` function and `initialState`. This is the primary way to rebuild the current state of an entity from its event history.

## Type Definitions

```typescript
type AggregateStreamOptions<State, EventType extends Event, ReadEventMetadataType, EventPayloadType extends Event = EventType> = {
  evolve: Evolve<State, EventType, ReadEventMetadataType>;
  initialState: () => State;
  read?: ReadStreamOptions<EventType, EventPayloadType>;  // same filtering options as readStream
};

type AggregateStreamResult<State> = {
  currentStreamVersion: StreamPosition;
  state: State;
  streamExists: boolean;
};
```
^aggregatestream-types

The [[Evolve Function|`Evolve`]] type supports three signatures:

```typescript
type Evolve<State, EventType, ReadEventMetadataType> =
  | ((currentState: State, event: EventType) => State)                                        // raw event only
  | ((currentState: State, event: ReadEvent<EventType, ReadEventMetadataType>) => State)      // with full metadata
  | ((currentState: State, event: ReadEvent<EventType>) => State);                            // with default metadata
```
^evolve-three-signatures

## Basic Usage

```typescript
import type { Event } from '@event-driven-io/emmett';

type ShoppingCart = {
  productItems: PricedProductItem[];
  totalAmount: number;
};

const evolve = (
  state: ShoppingCart,
  { type, data }: ShoppingCartEvent,
): ShoppingCart => {
  switch (type) {
    case 'ProductItemAdded':
      return {
        productItems: [...state.productItems, data.productItem],
        totalAmount: state.totalAmount + data.productItem.price * data.productItem.quantity,
      };
    case 'DiscountApplied':
      return {
        ...state,
        totalAmount: state.totalAmount * (1 - data.percent / 100),
      };
  }
};

const initialState = (): ShoppingCart => ({
  productItems: [],
  totalAmount: 0,
});

const result = await eventStore.aggregateStream<ShoppingCart, ShoppingCartEvent>(
  'shoppingCart-123',
  { evolve, initialState },
);

console.log(result.state);          // { productItems: [...], totalAmount: 54 }
console.log(result.streamExists);   // true
```

## Time-Travel Queries

Restrict the event range to read state at a specific point in time:

```typescript
// Get state after the first 2 events only
const pastState = await eventStore.aggregateStream<ShoppingCart, ShoppingCartEvent>(
  'shoppingCart-123',
  { evolve, initialState, read: { to: 2n } },
);
```

This uses the same `from`, `to`, and `maxCount` options as [[Reading Streams|readStream]].

## Nonexistent Streams

When a stream does not exist, `aggregateStream` returns the initial state:

```typescript
const result = await eventStore.aggregateStream<ShoppingCart, ShoppingCartEvent>(
  'nonexistent-stream',
  { evolve, initialState },
);
// result.state === { productItems: [], totalAmount: 0 }
// result.streamExists === false
```

## Upcasting on Aggregate

When using `aggregateStream`, the [[Schema Versioning|schema options]] are nested under a `read` property:

```typescript
const { state } = await eventStore.aggregateStream<
  ShoppingCartState,
  ShoppingCartEvent
>(shoppingCartId, {
  evolve,
  initialState,
  read: {
    schema: { versioning: { upcast } },
  },
});
```

> [!warning] currentStreamVersion Gotcha
> In the in-memory implementation, `aggregateStream` sets `currentStreamVersion` to `BigInt(filteredEvents.length)` -- the count of ==returned events==, not the actual stream version. If you use `to: 2n` on a stream with 5 events, you get `currentStreamVersion: 2n`. This differs from [[Reading Streams|readStream]], which always returns the actual total stream length.

## See Also

- [[Event Store Interface]] -- The four-method interface
- [[Evolve Function]] -- The pure state reducer function
- [[The Decider Pattern]] -- The decide/evolve/initialState pattern
- [[Command Handling]] -- Uses `aggregateStream` for the read-then-decide-then-write flow
- [[Reading Streams]] -- The underlying read operation
