---
tags:
  - testing
  - e2e
aliases:
  - Shared E2E Tests
  - emmett-tests
related:
  - "[[Event Store Interface]]"
  - "[[Testing MOC]]"
package: emmett-tests
---

# E2E Feature Tests

The `emmett-tests` package provides shared, reusable E2E test suites that verify any [[Event Store Interface]] implementation behaves correctly. These tests ensure all adapters are functionally equivalent.

## Shared Test Suites

The test suites are parameterized -- you provide your event store instance and they exercise the full API:

- ==`testAggregateStream()`== -- Tests state reconstruction via `aggregateStream` with `evolve`/`initialState`, including version tracking and empty stream behavior
- ==`testCommandHandling()`== -- Tests the [[Command Handling|CommandHandler]] integration: reading state, executing handlers, appending events, and version conflict detection
- ==`testStreamExists()`== -- Tests the `streamExists` method for existing and non-existing streams

## Test Domain

The shared tests use a [[Shopping Cart Example|Shopping Cart]] domain defined in `emmett-tests/src/eventStore/shoppingCart.domain.ts`. This ensures all adapters are tested against the same business logic.

## Running Against an Adapter

Each adapter's test suite imports the shared tests and provides its own event store instance:

```typescript
import { testAggregateStream, testCommandHandling } from '@event-driven-io/emmett-tests';

describe('PostgreSQL Event Store', () => {
  let eventStore: PostgresEventStore;

  beforeEach(async () => {
    eventStore = getPostgreSQLEventStore(pool);
  });

  testAggregateStream(() => eventStore);
  testCommandHandling(() => eventStore);
  testStreamExists(() => eventStore);
});
```

This pattern guarantees that all adapters pass the same behavioral tests, while each adapter can add adapter-specific tests on top.

> [!info] The `emmett-testcontainers` package provides Testcontainer helpers for PostgreSQL, MongoDB, and EventStoreDB to simplify integration test setup.

## See Also

- [[Event Store Interface]] -- The interface these tests verify
- [[Testing MOC]] -- Overview of all testing utilities
- [[Adapter Comparison]] -- Feature differences between adapters
