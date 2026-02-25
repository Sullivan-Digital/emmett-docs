---
tags:
  - event-store
  - versioning
aliases:
  - Upcasting
  - Downcasting
  - Event Versioning
related:
  - "[[Reading Streams]]"
  - "[[Appending Events]]"
  - "[[Command Handling]]"
package: emmett
---

# Schema Versioning

As your application evolves, event schemas may change. Emmett supports ==upcasting== (transforming old stored formats to the current application format on read) and ==downcasting== (transforming the current application format to a storage format on write). There are no version numbers, no migration framework -- just plain transform functions.

```
┌──────────────────────┐
│   Stored Event       │    readStream / aggregateStream
│   (old schema)       │ ──────── upcast() ────────────> Application Event
└──────────────────────┘                                  (current schema)

┌──────────────────────┐
│   Application Event  │    appendToStream
│   (current schema)   │ ──────── downcast() ──────────> Stored Event
└──────────────────────┘                                  (storage schema)
```

## Upcasting (Read-Side)

Upcasting is applied during [[Reading Streams|readStream]] and [[Aggregating Streams|aggregateStream]]. Use it to migrate old event shapes to your current type definitions.

### Basic Upcasting

Define a function that accepts a stored event and returns the current event type:

```typescript
import type { Event } from '@event-driven-io/emmett';

// What's stored in the database (old schema -- prices are strings)
type ProductItemFromDB = {
  productId: string;
  quantity: string;
  price: string;
};
type ProductItemAddedFromDB = Event<
  'ProductItemAdded',
  { productItem: ProductItemFromDB }
>;

// What your application uses (current schema -- prices are numbers)
type PricedProductItem = {
  productId: string;
  quantity: number;
  price: number;
};
type ProductItemAdded = Event<
  'ProductItemAdded',
  { productItem: PricedProductItem }
>;

const upcast = (event: Event): ShoppingCartEvent => {
  switch (event.type) {
    case 'ProductItemAdded': {
      const e = event as ProductItemAddedFromDB;
      return {
        type: 'ProductItemAdded',
        data: {
          productItem: {
            productId: e.data.productItem.productId,
            quantity: Number(e.data.productItem.quantity),
            price: Number(e.data.productItem.price),
          },
        },
      };
    }
    default:
      return event as ShoppingCartEvent;
  }
};
```

> [!tip]
> Always include a `default` case that passes through events that don't need transformation. Your upcast function handles ==all event types== for a stream.

### With readStream

```typescript
const { events } = await eventStore.readStream<ShoppingCartEvent>(
  shoppingCartId,
  {
    schema: { versioning: { upcast } },
  },
);
// events are now typed as ReadEvent<ShoppingCartEvent>[]
```

### With aggregateStream

When using [[Aggregating Streams|aggregateStream]], the schema options are nested under a `read` property:

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

### With CommandHandler

The [[Command Handling|`CommandHandler`]] accepts `schema.versioning` in its options and automatically applies it to both read and write paths:

```typescript
import { CommandHandler } from '@event-driven-io/emmett';

const handleCommand = CommandHandler<ShoppingCart, ShoppingCartEvent>({
  evolve,
  initialState,
  schema: { versioning: { upcast } },
});
```

### Advanced: RecordedMessage Overload

The upcast function can operate at two levels:

1. **Plain message level** (most common): `(event: StoredEvent) => StreamEvent`
2. **RecordedMessage level**: `(message: RecordedMessage<StoredEvent>) => RecordedMessage<StreamEvent>`

The RecordedMessage overload gives you access to event metadata and lets you transform metadata during upcasting. When using this form, Emmett performs a deep merge of metadata: original fields are preserved, and fields from the upcast result override or augment them.

## Downcasting (Write-Side)

Downcasting is applied during [[Appending Events|appendToStream]]. Use it to transform events to a storage format:

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

When both upcast and downcast are needed (e.g., in a [[Command Handling|CommandHandler]]), provide them together:

```typescript
const handleCommand = CommandHandler<
  ShoppingCart,
  ShoppingCartEvent,
  ShoppingCartEventStored
>({
  evolve,
  initialState,
  schema: {
    versioning: {
      upcast,    // stored -> application (on read)
      downcast,  // application -> stored (on write)
    },
  },
});
```

## Schema Options Reference

```typescript
// Read-side
type EventStoreReadSchemaOptions<
  StreamEvent extends Event = Event,
  StoredEvent extends Event = StreamEvent,
> = {
  versioning?: {
    upcast?: (event: StoredEvent) => StreamEvent;
  };
};

// Write-side
type EventStoreAppendSchemaOptions<
  StreamEvent extends Event = Event,
  StoredEvent extends Event = StreamEvent,
> = {
  versioning?: {
    downcast?: (event: StreamEvent) => StoredEvent;
  };
};

// Combined
type EventStoreSchemaOptions<
  StreamEvent extends Event = Event,
  StoredEvent extends Event = StreamEvent,
> = EventStoreReadSchemaOptions<StreamEvent, StoredEvent> &
  EventStoreAppendSchemaOptions<StreamEvent, StoredEvent>;
```
^schema-options-types

## Common Use Cases

> [!example]- Date and BigInt Deserialization
> JSON cannot natively represent `Date` or `bigint` values. Upcast them from their string representations:
> ```typescript
> const upcast = (event: Event): ShoppingCartOpened => {
>   if (event.type === 'ShoppingCartOpened') {
>     const e = event as ShoppingCartOpenedFromDB;
>     return {
>       type: 'ShoppingCartOpened',
>       data: {
>         clientId: e.data.clientId,
>         openedAt: new Date(e.data.openedAt),
>         loyaltyPoints: BigInt(e.data.loyaltyPoints),
>       },
>     };
>   }
>   return event as ShoppingCartOpened;
> };
> ```

> [!example]- Renamed Fields
> ```typescript
> const upcast = (event: Event): UserEvent => {
>   if (event.type === 'UserRegistered') {
>     const data = event.data as Record<string, unknown>;
>     return {
>       type: 'UserRegistered',
>       data: {
>         displayName: (data.displayName ?? data.userName) as string,
>       },
>     };
>   }
>   return event as UserEvent;
> };
> ```

> [!example]- Multi-Version Handling
> There is no built-in pipeline for chaining transforms (v1 -> v2 -> v3). Handle all versions in a single function:
> ```typescript
> const upcast = (event: Event): CurrentEvent => {
>   switch (event.type) {
>     case 'OrderPlaced': {
>       const data = event.data as Record<string, unknown>;
>       return {
>         type: 'OrderPlaced',
>         data: {
>           totalAmount: (data.totalAmount ?? data.amount) as number,
>           currency: (data.currency ?? 'USD') as string,
>         },
>       };
>     }
>     default:
>       return event as CurrentEvent;
>   }
> };
> ```
> If you prefer composable transforms, compose them yourself:
> ```typescript
> const upcast = (event: Event): CurrentEvent =>
>   upcastV2toV3(upcastV1toV2(event));
> ```

## Adapter Support

All Emmett event store adapters support both upcasting and downcasting with the same API:

| Adapter | Upcasting | Downcasting |
|---------|-----------|-------------|
| InMemory | Yes | Yes |
| PostgreSQL | Yes | Yes |
| MongoDB | Yes | Yes |
| SQLite | Yes | Yes |
| EventStoreDB | Yes | Yes |

## Gotchas

> [!warning]
> - **Metadata is deep-merged.** Upcast/downcast metadata fields override the original, but original fields not in the transform are preserved. You cannot delete a metadata field via upcasting.
> - **Downcast happens after metadata enrichment.** The downcast function receives the fully enriched `ReadEvent`, not the raw event.
> - **The upcast function receives the raw stored type.** Be defensive about data shapes since different events in the same stream may have been stored with different schemas.

## See Also

- [[Reading Streams]] -- Where upcasting is applied
- [[Appending Events]] -- Where downcasting is applied
- [[Command Handling]] -- Automatic versioning via `schema` option
- [[JSON Serialization]] -- BigInt handling in JSON (related concern)
