---
tags:
  - core
  - type-system
aliases:
  - Event Type
  - Event Definition
related:
  - "[[Commands]]"
  - "[[Messages]]"
  - "[[Recorded Messages]]"
  - "[[The Decider Pattern]]"
  - "[[Evolve Function]]"
package: emmett
---

# Events

Events represent ==things that have already happened== in your domain. They are immutable facts, named in past tense (e.g., `ProductItemAdded`, `ShoppingCartConfirmed`). Events are the fundamental building block of event sourcing in Emmett.

All event types use a **discriminated union** pattern based on a `type` string literal field, making them safe to use with `switch` statements and TypeScript's narrowing.

## Defining Events

Use the `Event<Type, Data, MetaData>` generic type to define events:

```typescript
import type { Event } from '@event-driven-io/emmett';

// Basic event with type and data
type ProductItemAdded = Event<
  'ProductItemAdded',
  { productItem: { productId: string; quantity: number; price: number } }
>;

// Event with custom metadata
type OrderPlaced = Event<
  'OrderPlaced',
  { orderId: string; placedAt: Date },
  { userId: string }
>;
```

^event-type-def

**Generic parameters:**

| Parameter | Constraint | Default | Description |
|---|---|---|---|
| `EventType` | `extends string` | `string` | String literal for the `type` discriminant |
| `EventData` | `extends DefaultRecord` | `DefaultRecord` | The event payload |
| `EventMetaData` | `extends DefaultRecord \| undefined` | `undefined` | Optional metadata; when `undefined`, the `metadata` property is omitted entirely |

The resulting type is `Readonly` -- all properties are immutable. Events also carry an optional `kind?: 'Event'` discriminant that can distinguish them from [[Commands|commands]] at runtime.

> [!tip] Two ways to define events
> You can also define events as plain object types without the `Event<>` generic. Both approaches produce structurally compatible types:
>
> ```typescript
> // Using the Event<> generic (preferred for simple cases)
> type ProductItemAdded = Event<'ProductItemAdded', { productItem: PricedProductItem }>;
>
> // Using a plain object type (equivalent)
> type ProductItemAdded = {
>   type: 'ProductItemAdded';
>   data: { productItem: PricedProductItem };
> };
> ```

To define a union of events for a stream (the most common pattern):

```typescript
type ShoppingCartEvent =
  | Event<'ProductItemAdded', { productItem: PricedProductItem }>
  | Event<'DiscountApplied', { percent: number; couponId: string }>;
```

^event-union-example

## The `event()` Factory Function

The ==`event()`== factory creates event instances at runtime and sets the `kind: 'Event'` discriminant:

```typescript
import { event, type Event } from '@event-driven-io/emmett';

type ProductItemAdded = Event<
  'ProductItemAdded',
  { productItem: PricedProductItem }
>;

const myEvent = event<ProductItemAdded>(
  'ProductItemAdded',
  { productItem: { productId: 'p1', quantity: 2, price: 9.99 } }
);
// Result: { type: 'ProductItemAdded', data: { productItem: ... }, kind: 'Event' }
```

When an event type includes metadata, the factory requires it as the third argument:

```typescript
type OrderPlaced = Event<
  'OrderPlaced',
  { orderId: string },
  { userId: string }
>;

const myEvent = event<OrderPlaced>(
  'OrderPlaced',
  { orderId: 'order-123' },
  { userId: 'user-456' }
);
```

> [!note] How the factory overload works
> The `event()` function uses variadic conditional tuple types for parameter overloading. If the event type has no metadata, it accepts `(type, data)`. If it has metadata, it requires `(type, data, metadata)`. The `kind: 'Event'` field is attached automatically at runtime.

## Event Helper Types

| Type | Description |
|---|---|
| `AnyEvent` | `Event<any, any, any>` -- loosened type for generic contexts |
| `EventTypeOf<T>` | Extracts the `type` string literal from an event type |
| `EventDataOf<T>` | Extracts the `data` type from an event type |
| `EventMetaDataOf<T>` | Extracts metadata type; returns `undefined` if no metadata |
| `CreateEventType<Type, Data, MetaData>` | Alias identical to `Event<>` for clarity in type creation contexts |

^event-helper-types

## Stream Position Types

Defined alongside events, these types represent ordering within event streams:

```typescript
type StreamPosition = bigint;  // Position within a single stream
type GlobalPosition = bigint;  // Global ordering across all streams
```

See [[Utility Types]] for more on these and other foundational types.

> [!warning] The `kind` field behavior
> The `kind` field is **optional** at the type level (`kind?: 'Event'`). It is only set at runtime by the `event()` factory function. You generally do not need to check `kind` in application code. However, when events are persisted and read back as [[Recorded Messages|RecordedMessage]], the `kind` field becomes **required** (non-optional).

## See Also

- [[Commands]] -- The intent counterpart to events
- [[Messages]] -- The union type encompassing both events and commands
- [[Recorded Messages]] -- What events look like after persistence (with system metadata)
- [[The Decider Pattern]] -- How events are produced by `decide` and consumed by `evolve`
- [[Evolve Function]] -- The pure state reducer that processes events
