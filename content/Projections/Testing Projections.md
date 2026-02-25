---
tags:
  - projections
  - testing
aliases:
  - Projection Testing
  - InMemoryProjectionSpec
related:
  - "[[InMemory Projections]]"
  - "[[PostgreSQL Testing]]"
  - "[[MongoDB Testing]]"
  - "[[SQLite Testing]]"
  - "[[Assertions Library]]"
package: emmett
---

# Testing Projections

> [!abstract]
> Emmett provides BDD-style ==given/when/then== test specifications for projections across every adapter: `InMemoryProjectionSpec`, `PostgreSQLProjectionSpec`, `PongoProjectionSpec` (assertions), and `MongoDBInlineProjectionSpec`.

## InMemoryProjectionSpec

The in-memory spec is the simplest way to test projection logic:

```typescript
import {
  InMemoryProjectionSpec,
  eventInStream,
  eventsInStream,
  expectInMemoryDocuments,
} from '@event-driven-io/emmett';

const given = InMemoryProjectionSpec.for({
  projection: shortInfoProjection,
});

void describe('ShoppingCartShortInfo projection', () => {
  void it('adds product items', () =>
    given([])
      .when(
        eventsInStream('shopping_cart-123', [
          {
            type: 'ProductItemAdded',
            data: { productItem: { productId: 'p1', quantity: 2, price: 10 } },
          },
        ]),
      )
      .then(
        expectInMemoryDocuments
          .fromCollection('shoppingCartShortInfo')
          .withId('shopping_cart-123')
          .toBeEqual({ productItemsCount: 2, totalAmount: 20 }),
      ));
});
```

### Pattern

```
given(existingEvents).when(newEvents, options?).then(assertion, message?)
given(existingEvents).when(newEvents, options?).thenThrows(errorType?, condition?)
```

### Test Helpers

- ==`eventInStream(streamName, event)`== -- Tag a single event with a stream name
- ==`eventsInStream(streamName, events)`== -- Tag multiple events with a stream name
- `newEventsInStream` -- Alias for `eventsInStream`

### Fluent Assertions

```typescript
expectInMemoryDocuments
  .fromCollection('shoppingCartShortInfo')
  .withId('shopping_cart-123')
  .toBeEqual({ productItemsCount: 2, totalAmount: 20 })
```

### Idempotency Testing

The `when` step accepts an optional `{ numberOfTimes: N }` parameter to repeat the events N times, verifying that the projection is idempotent:

```typescript
given([])
  .when(
    eventsInStream('cart-1', [productItemAddedEvent]),
    { numberOfTimes: 3 },
  )
  .then(/* assert same result regardless of replays */)
```

## PostgreSQLProjectionSpec

Tests raw SQL projections against a real PostgreSQL database:

```typescript
import {
  PostgreSQLProjectionSpec,
  eventsInStream,
  expectSQL,
} from '@event-driven-io/emmett-postgresql';

const given = PostgreSQLProjectionSpec.for({
  projection: rawSqlProjection,
  connectionString,
});

void it('creates the row', () =>
  given([])
    .when(eventsInStream('cart-1', [productItemAddedEvent]))
    .then(
      expectSQL
        .query(sql('SELECT * FROM shopping_cart_short_info WHERE id = %L', 'cart-1'))
        .resultRows.toBeTheSame([
          { id: 'cart-1', product_count: 2 },
        ]),
    ));
```

The spec auto-initializes the event store schema and calls `projection.init()` on first run.

### SQL Assertion Helpers

- ==`expectSQL.query(sql).resultRows.toBeTheSame(rows)`== -- Fluent SQL query assertion
- `assertSQLQueryResultMatches(sql, rows)` -- Functional alternative

## PongoProjectionSpec (Pongo Assertions)

For Pongo document projections, use the same `PostgreSQLProjectionSpec` with Pongo-specific assertions:

```typescript
import {
  PostgreSQLProjectionSpec,
  expectPongoDocuments,
} from '@event-driven-io/emmett-postgresql';

const given = PostgreSQLProjectionSpec.for({
  projection: pongoProjection,
  connectionString,
});

void it('creates the document', () =>
  given([])
    .when(eventsInStream('cart-1', [productItemAddedEvent]))
    .then(
      expectPongoDocuments
        .fromCollection('shoppingCartShortInfo')
        .withId('cart-1')
        .toBeEqual({ productItemsCount: 2, totalAmount: 20 }),
    ));
```

### Pongo Assertion Helpers

| Assertion | Description |
|---|---|
| `.withId(id).toBeEqual(doc)` | Assert document matches expected value |
| `.withId(id).toExist()` | Assert document exists |
| `.withId(id).notToExist()` | Assert document does not exist |
| `.matching(filter).toBeTheSame(docs)` | Assert filtered documents match |
| `.matching(filter).toHaveCount(n)` | Assert count of matching documents |

## MongoDBInlineProjectionSpec

Tests MongoDB inline projections with embedded read model assertions:

```typescript
import {
  MongoDBInlineProjectionSpec,
  expectInlineReadModel,
} from '@event-driven-io/emmett-mongodb';

const given = MongoDBInlineProjectionSpec.for({
  projection: shortInfoProjection,
  connectionString,
});

void it('projects product item added', () =>
  given({ streamName: 'shopping_cart-123', events: [] })
    .when([
      {
        type: 'ProductItemAdded',
        data: { productItem: { productId: 'p1', quantity: 2, price: 10 } },
      },
    ])
    .then(
      expectInlineReadModel.toHave<ShoppingCartShortInfo>({
        productItemsCount: 2,
        totalAmount: 20,
      }),
    ));
```

> [!note]
> The MongoDB spec has a different `given` signature -- it takes `{ streamName, events }` instead of a flat events array, because projections are scoped to a specific stream document.

### MongoDB Assertion Helpers

| Assertion | Description |
|---|---|
| `expectInlineReadModel.toHave<Doc>(expected)` | Partial match on the read model |
| `expectInlineReadModel.toDeepEquals<Doc>(expected)` | Exact match |
| `expectInlineReadModel.toMatch(fn)` | Custom matcher function |
| `expectInlineReadModel.notToExist()` | Assert no read model exists |
| `expectInlineReadModel.toExist()` | Assert read model exists |
| `expectInlineReadModel.withName(name).*` | Target a named projection |

## Summary: Spec Comparison

| Spec | Package | Database Required | Assertion Style |
|---|---|---|---|
| `InMemoryProjectionSpec` | `emmett` | No | In-memory document collection |
| `PostgreSQLProjectionSpec` | `emmett-postgresql` | Yes (PostgreSQL) | SQL queries or Pongo documents |
| `MongoDBInlineProjectionSpec` | `emmett-mongodb` | Yes (MongoDB) | Embedded read model |

## See Also

- [[InMemory Projections]] -- The projection type tested by `InMemoryProjectionSpec`
- [[Pongo Document Projections]] -- Pongo projections tested with `expectPongoDocuments`
- [[Raw SQL Projections]] -- SQL projections tested with `expectSQL`
- [[MongoDB Inline Projections]] -- MongoDB projections tested with `expectInlineReadModel`
- [[Assertions Library]] -- The underlying assertion utilities
- [[DeciderSpecification]] -- Similar BDD pattern for testing deciders
