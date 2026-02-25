---
tags:
  - adapter
  - sqlite
  - testing
aliases:
  - SQLiteProjectionSpec
related:
  - "[[Testing Projections]]"
  - "[[SQLite Projections]]"
  - "[[Assertions Library]]"
package: emmett-sqlite
---

# SQLite Testing

The SQLite adapter provides BDD-style specification testing for projections via ==`SQLiteProjectionSpec`==, along with assertion helpers for both Pongo document projections and raw SQL projections.

## SQLiteProjectionSpec

Create a spec instance for your projection:

```typescript
import {
  SQLiteProjectionSpec,
  eventInStream,
  eventsInStream,
  newEventsInStream,
} from '@event-driven-io/emmett-sqlite';
import { sqlite3EventStoreDriver } from '@event-driven-io/emmett-sqlite/sqlite3';

const given = SQLiteProjectionSpec.for({
  projection: myProjection,
  driver: sqlite3EventStoreDriver,
  fileName: ':memory:',
});
```

### Given/When/Then Pattern

Test with no prior events:

```typescript
await given([])
  .when([
    eventInStream('cart-1', {
      type: 'ProductItemAdded',
      data: {
        productItem: { price: 100, productId: 'shoes', quantity: 2 },
      },
    }),
  ])
  .then(/* assertion */);
```

Test with existing events and new events:

```typescript
await given(
  eventsInStream('cart-1', [
    {
      type: 'ProductItemAdded',
      data: {
        productItem: { price: 100, productId: 'shoes', quantity: 2 },
      },
    },
  ]),
)
  .when(
    newEventsInStream('cart-1', [
      { type: 'DiscountApplied', data: { percent: 10, couponId: 'SAVE10' } },
    ]),
  )
  .then(/* assertion */);
```

### Idempotency Testing

Replay events multiple times to verify idempotency:

```typescript
await given(existingEvents)
  .when(newEvents, { numberOfTimes: 2 })
  .then(/* assertion should still hold */);
```

### Testing Failures

```typescript
await given([]).when(badEvents).thenThrows(MyErrorType);
```

### Test Helpers

- `eventInStream(streamName, event)` -- wraps a single event with stream metadata
- `eventsInStream(streamName, events)` -- wraps multiple events for the `given` clause
- `newEventsInStream(streamName, events)` -- wraps multiple events for the `when` clause

The spec auto-generates metadata (global positions, stream positions, message IDs) so you only need to provide event types and data.

## Pongo Assertion Helpers

For [[SQLite Projections|Pongo document projections]], use dedicated assertion helpers:

### documentExists

```typescript
import { documentExists } from '@event-driven-io/emmett-sqlite';

await given([])
  .when(events)
  .then(
    documentExists(
      { productItemsCount: 2, totalAmount: 200 },
      { inCollection: 'shoppingCartShortInfo', withId: 'cart-1' },
    ),
  );
```

### Fluent API

```typescript
import { expectPongoDocuments } from '@event-driven-io/emmett-sqlite';

await given([])
  .when(events)
  .then(
    expectPongoDocuments
      .fromCollection<ShoppingCartShortInfo>('shoppingCartShortInfo')
      .withId('cart-1')
      .toBeEqual({ productItemsCount: 2, totalAmount: 200 }),
  );
```

### Assert Non-Existence

```typescript
await given([])
  .when([])
  .then(
    expectPongoDocuments
      .fromCollection('shoppingCartShortInfo')
      .withId('non-existent')
      .notToExist(),
  );
```

### Other Pongo Assertions

```typescript
import {
  documentDoesNotExist,
  documentsAreTheSame,
  documentsMatchingHaveCount,
  documentMatchingExists,
} from '@event-driven-io/emmett-sqlite';

// Assert no document matches a filter
documentDoesNotExist({ inCollection: 'carts', withId: 'missing' });

// Assert exact document array match
documentsAreTheSame(expectedDocs, { inCollection: 'carts' });

// Assert count of matching documents
documentsMatchingHaveCount(3, {
  inCollection: 'carts',
  matchingFilter: { status: 'active' },
});

// Assert at least one document matches
documentMatchingExists({
  inCollection: 'carts',
  matchingFilter: { status: 'active' },
});
```

All accept `PongoAssertOptions`: `{ inCollection: string; inDatabase?: string }` with either `{ withId: string }` or `{ matchingFilter: PongoFilter }`.

## Raw SQL Assertions

For [[SQLite Projections|raw SQL projections]]:

```typescript
import { expectSQL } from '@event-driven-io/emmett-sqlite';
import { SQL } from '@event-driven-io/dumbo';

await given([])
  .when(events)
  .then(
    expectSQL
      .query(SQL`SELECT * FROM shoppingCartShortInfo WHERE id = ${'cart-1'}`)
      .resultRows.toBeTheSame([
        {
          id: 'cart-1',
          productItemsCount: 2,
          totalAmount: 200,
        },
      ]),
  );
```

The `assertSQLQueryResultMatches(sql, rows)` function is also available for a non-fluent style:

```typescript
import { assertSQLQueryResultMatches } from '@event-driven-io/emmett-sqlite';

await given([])
  .when(events)
  .then(assertSQLQueryResultMatches(
    SQL`SELECT * FROM shoppingCartShortInfo WHERE id = ${'cart-1'}`,
    [{ id: 'cart-1', productItemsCount: 2, totalAmount: 200 }],
  ));
```

## See Also

- [[Testing Projections]] -- BDD projection testing across all adapters
- [[SQLite Projections]] -- Defining the projections under test
- [[Assertions Library]] -- Core assertion utilities
- [[PostgreSQL Testing]] -- Same pattern for PostgreSQL projections
