---
tags:
  - adapter
  - mongodb
  - testing
aliases:
  - MongoDBInlineProjectionSpec
related:
  - "[[Testing Projections]]"
  - "[[MongoDB Inline Projections]]"
  - "[[Assertions Library]]"
package: emmett-mongodb
---

# MongoDB Testing

The `@event-driven-io/emmett-mongodb` package provides BDD-style projection testing and dummy MongoDB stubs for unit tests.

## BDD Projection Specs

`MongoDBInlineProjectionSpec` provides a given/when/then API for testing [[MongoDB Inline Projections|inline projections]] against a real MongoDB instance:

```typescript
import {
  MongoDBInlineProjectionSpec,
  eventInStream,
  eventsInStream,
  expectInlineReadModel,
} from '@event-driven-io/emmett-mongodb';

const given = MongoDBInlineProjectionSpec.for<
  ShoppingCartId,
  ShoppingCartEvent
>({
  projection: summaryProjection,
  client: mongoClient,
});
```

### Given / When / Then

The `given` function accepts prior events, `.when()` accepts new events, and `.then()` runs assertions:

```typescript
await given(
  eventsInStream('shopping_cart:abc-123', [
    {
      type: 'ProductItemAdded',
      data: {
        productItem: { productId: 'shoes-1', quantity: 2, price: 100 },
      },
    },
  ]),
)
  .when([
    {
      type: 'ProductItemAdded',
      data: {
        productItem: { productId: 'hat-1', quantity: 1, price: 50 },
      },
    },
  ])
  .then(
    expectInlineReadModel.toHave({
      productItemsCount: 3,
      totalAmount: 250,
    }),
  );
```

### Event Helpers

| Helper | Description |
|---|---|
| `eventInStream(streamName, event)` | Wraps a single event with its stream name |
| `eventsInStream(streamName, events)` | Wraps multiple events with their stream name |

### Assertions

| Assertion | Description |
|---|---|
| `expectInlineReadModel.toHave(expected)` | Partial match -- checks a subset of fields |
| `expectInlineReadModel.toDeepEquals(expected)` | Exact match -- all fields must match |
| `expectInlineReadModel.toMatch(fn)` | Custom predicate function |
| `expectInlineReadModel.toExist()` | Projection exists (is not null) |
| `expectInlineReadModel.notToExist()` | Projection is null (soft-deleted or never created) |

### Named Projection Assertions

For [[MongoDB Inline Projections#Named Projections|named projections]], chain `.withName()` before the assertion:

```typescript
expectInlineReadModel
  .withName('shoppingCartShortInfo')
  .toHave({
    productItemsCount: 3,
  });
```

### Full Test Example

> [!example]- Complete BDD Test
> ```typescript
> import { describe, it, beforeEach, afterEach } from 'vitest';
> import { MongoClient } from 'mongodb';
> import {
>   MongoDBInlineProjectionSpec,
>   eventsInStream,
>   expectInlineReadModel,
>   mongoDBInlineProjection,
> } from '@event-driven-io/emmett-mongodb';
>
> describe('Shopping Cart Summary Projection', () => {
>   let client: MongoClient;
>   let given: ReturnType<typeof MongoDBInlineProjectionSpec.for>;
>
>   beforeEach(async () => {
>     client = new MongoClient('mongodb://localhost:27017/?replicaSet=rs0');
>     await client.connect();
>
>     given = MongoDBInlineProjectionSpec.for({
>       projection: summaryProjection,
>       client,
>     });
>   });
>
>   afterEach(async () => {
>     await client.close();
>   });
>
>   it('calculates totals from added items', async () => {
>     await given(
>       eventsInStream('shopping_cart:test-1', [
>         {
>           type: 'ProductItemAdded',
>           data: {
>             productItem: { productId: 'p1', quantity: 2, price: 50 },
>           },
>         },
>       ]),
>     )
>       .when([
>         {
>           type: 'ProductItemAdded',
>           data: {
>             productItem: { productId: 'p2', quantity: 1, price: 30 },
>           },
>         },
>       ])
>       .then(
>         expectInlineReadModel.toHave({
>           productItemsCount: 3,
>           totalAmount: 130,
>         }),
>       );
>   });
>
>   it('soft deletes on cancellation', async () => {
>     await given(
>       eventsInStream('shopping_cart:test-2', [
>         {
>           type: 'ProductItemAdded',
>           data: {
>             productItem: { productId: 'p1', quantity: 1, price: 100 },
>           },
>         },
>       ]),
>     )
>       .when([{ type: 'ShoppingCartCancelled', data: {} }])
>       .then(expectInlineReadModel.notToExist());
>   });
> });
> ```

## Dummy MongoDB Objects

For unit tests that need MongoDB type shapes without a real connection:

```typescript
import {
  getDummyClient,
  getDummyDb,
  getDummyCollection,
} from '@event-driven-io/emmett-mongodb';

const client = getDummyClient({ defaultDBName: 'test-db' });
const db = getDummyDb('test-db');
const collection = getDummyCollection<MyDocType>('my-collection');
```

These create stub objects with the minimal interface needed for testing [[MongoDB Storage Strategies|storage resolution]] logic. They do not perform real MongoDB operations.

| Function | Returns | Purpose |
|---|---|---|
| `getDummyClient(options?)` | `MongoClient` | Stub client with optional `defaultDBName` |
| `getDummyDb(dbName?, options?)` | `Db` | Stub database |
| `getDummyCollection<TSchema>(name, options?)` | `Collection<TSchema>` | Stub collection |

> [!info] See Also
> - [[Testing Projections]] -- BDD projection testing across all adapters, including `InMemoryProjectionSpec`, `PostgreSQLProjectionSpec`, and `SQLiteProjectionSpec`
> - [[Assertions Library]] -- Framework-agnostic assertion utilities used by all test specs
