---
tags:
  - adapter
  - postgresql
  - testing
aliases:
  - PostgreSQLProjectionSpec
related:
  - "[[Testing Projections]]"
  - "[[PostgreSQL Projections]]"
  - "[[Assertions Library]]"
package: emmett-postgresql
---

# PostgreSQL Testing

The PostgreSQL adapter provides ==`PostgreSQLProjectionSpec`==, a BDD-style test helper for validating inline projections, along with assertion helpers for both SQL and Pongo document projections.

## PostgreSQLProjectionSpec

The spec follows a given/when/then pattern for testing [[PostgreSQL Projections|inline projections]]:

```typescript
import {
  documentExists,
  PostgreSQLProjectionSpec,
} from '@event-driven-io/emmett-postgresql';
import { PostgreSqlContainer } from '@testcontainers/postgresql';

// Setup -- start a real PostgreSQL container
const postgres = await new PostgreSqlContainer().start();
const connectionString = postgres.getConnectionUri();

const given = PostgreSQLProjectionSpec.for({
  projection: shoppingCartDetailsProjection,
  connectionString,
});

// Test
const shoppingCartId = 'shopping_cart-client1:current';

await given([])           // given: existing events (empty = fresh start)
  .when([                 // when: new events arrive
    {
      type: 'ProductItemAddedToShoppingCart',
      data: {
        shoppingCartId,
        clientId: 'client1',
        productItem: { unitPrice: 100, productId: 'shoes', quantity: 2 },
        addedAt: new Date(),
      },
      metadata: { streamName: shoppingCartId, clientId: 'client1' },
    },
  ])
  .then(                  // then: assert projection state
    documentExists<ShoppingCartDetails>(
      {
        status: 'Opened',
        clientId: 'client1',
        totalAmount: 200,
        productItems: [{ quantity: 2, productId: 'shoes', unitPrice: 100 }],
        productItemsCount: 2,
      },
      {
        inCollection: 'shoppingCartDetails',
        withId: shoppingCartId,
      },
    ),
  );
```

## Event Helper Functions

Use `eventInStream` and `eventsInStream` to set stream context on events:

```typescript
import { eventInStream, eventsInStream } from '@event-driven-io/emmett-postgresql';

// Single event in a stream
given(eventsInStream('my-stream-1', [event1, event2]))
  .when(eventInStream('my-stream-1', event3))
  .then(...);
```

## Additional Test Capabilities

### Repeating Events

Test idempotency by repeating events multiple times:

```typescript
given([]).when(events, { numberOfTimes: 3 }).then(...);
```

### Asserting Errors

```typescript
given([]).when(invalidEvents).thenThrows(SomeErrorType);
```

### Raw SQL Assertions

For [[Raw SQL Projections|raw SQL projections]], assert query results directly:

```typescript
given([]).when(events).then(
  assertSQLQueryResultMatches(
    sql`SELECT * FROM order_totals`,
    [{ order_id: 'order-1', total: 100 }],
  ),
);
```

### Fluent SQL Assertions

```typescript
given([]).when(events).then(
  expectSQL
    .query(sql`SELECT count(*) FROM order_totals`)
    .resultRows.toBeTheSame([{ count: 1 }]),
);
```

## Pongo Assertion Helpers

For [[Pongo Document Projections|Pongo-based projections]], use these assertion helpers:

```typescript
import {
  documentExists,
  documentDoesNotExist,
  documentsAreTheSame,
  documentsMatchingHaveCount,
  documentMatchingExists,
} from '@event-driven-io/emmett-postgresql';
```

### documentExists

Check that a specific document exists with expected values:

```typescript
documentExists<MyDoc>(
  expectedDoc,
  { inCollection: 'myCollection', withId: 'doc-1' },
);
```

### documentDoesNotExist

Check that a document does not exist:

```typescript
documentDoesNotExist({ inCollection: 'myCollection', withId: 'doc-1' });
```

### documentsAreTheSame

Check all documents in a collection match expected values:

```typescript
documentsAreTheSame<MyDoc>(
  expectedDocs,
  { inCollection: 'myCollection' },
);
```

### documentsMatchingHaveCount

Count matching documents:

```typescript
documentsMatchingHaveCount(5, { inCollection: 'myCollection' });
```

### documentMatchingExists

Check that a document matching a filter exists:

```typescript
documentMatchingExists({
  inCollection: 'myCollection',
  matchingFilter: { status: 'active' },
});
```

### Fluent Pongo API

```typescript
import { expectPongoDocuments } from '@event-driven-io/emmett-postgresql';

expectPongoDocuments
  .fromCollection('myCollection')
  .withId('doc-1')
  .toBeEqual(expectedDoc);
```

## Testcontainers Integration

The PostgreSQL tests typically use `@testcontainers/postgresql` to spin up a real PostgreSQL instance:

```typescript
import { PostgreSqlContainer } from '@testcontainers/postgresql';

let postgres: StartedPostgreSqlContainer;

beforeAll(async () => {
  postgres = await new PostgreSqlContainer().start();
});

afterAll(async () => {
  await postgres.stop();
});
```

> [!tip]
> Using a real PostgreSQL container ensures your projection tests exercise the actual SQL, advisory locks, and transaction behavior. This is more reliable than mocking the database layer.

## See Also

- [[Testing Projections]] -- BDD-style testing across all adapters
- [[PostgreSQL Projections]] -- The projection types these tests validate
- [[Assertions Library]] -- The core assertion utilities
- [[E2E Feature Tests]] -- Shared E2E test suites in `emmett-tests`
