---
tags:
  - event-store
  - pattern
  - concurrency
aliases:
  - Expected Version
  - Optimistic Concurrency Control
related:
  - "[[Appending Events]]"
  - "[[Error Hierarchy]]"
  - "[[Type Branding]]"
  - "[[Retry Logic]]"
  - "[[ETag Utilities]]"
  - "[[Optimistic Concurrency]]"
package: emmett
---

# Concurrency Control

Optimistic concurrency control prevents conflicting writes from silently overwriting each other. Both [[Reading Streams|readStream]] and [[Appending Events|appendToStream]] accept an optional `expectedStreamVersion` that is checked before the operation proceeds.

## Expected Version Constants

Emmett provides three sentinel constants:

```typescript
import {
  NO_CONCURRENCY_CHECK,
  STREAM_EXISTS,
  STREAM_DOES_NOT_EXIST,
} from '@event-driven-io/emmett';
```

| Constant | Matches when |
|---|---|
| ==`NO_CONCURRENCY_CHECK`== | Always (no check performed) |
| ==`STREAM_DOES_NOT_EXIST`== | The stream is empty or does not exist (version equals the default, typically `0n`) |
| ==`STREAM_EXISTS`== | The stream has at least one event (version differs from the default) |

When `expectedStreamVersion` is not provided, it defaults to `NO_CONCURRENCY_CHECK`.

> [!note]
> The version type uses [[Type Branding|`Flavour`]] for nominal typing:
> ```typescript
> type ExpectedStreamVersion = ExpectedStreamVersionWithValue | ExpectedStreamVersionGeneral;
> type ExpectedStreamVersionWithValue = Flavour<StreamPosition, 'StreamVersion'>;
> type ExpectedStreamVersionGeneral = Flavour<
>   'STREAM_EXISTS' | 'STREAM_DOES_NOT_EXIST' | 'NO_CONCURRENCY_CHECK',
>   'StreamVersion'
> >;
> ```

## Using Sentinels

**Creating a new stream:**

```typescript
await eventStore.appendToStream<ShoppingCartEvent>(
  'shoppingCart-123',
  [{ type: 'ProductItemAdded', data: { productItem } }],
  { expectedStreamVersion: STREAM_DOES_NOT_EXIST },
);
```

**Appending to an existing stream:**

```typescript
await eventStore.appendToStream<ShoppingCartEvent>(
  'shoppingCart-123',
  [{ type: 'DiscountApplied', data: { percent: 10, couponId: 'SAVE10' } }],
  { expectedStreamVersion: STREAM_EXISTS },
);
```

## Exact Version Matching

Pass a `bigint` value to require an exact version match. This is the most common pattern -- read the current version, make decisions, then write with the version you read:

```typescript
// Read current state and version
const { state, currentStreamVersion } = await eventStore.aggregateStream<
  ShoppingCart,
  ShoppingCartEvent
>(
  'shoppingCart-123',
  { evolve, initialState },
);

// Decide what events to produce based on state
const newEvents = decide(command, state);

// Append with the exact version we read
await eventStore.appendToStream<ShoppingCartEvent>(
  'shoppingCart-123',
  newEvents,
  { expectedStreamVersion: currentStreamVersion },
);
```

If another process appended events between the read and the write, the version won't match and the operation fails with an `ExpectedVersionConflictError`.

## Matching Logic

| Expected Value | Matches When |
|---|---|
| `NO_CONCURRENCY_CHECK` | Always matches (returns `true`) |
| `STREAM_DOES_NOT_EXIST` | `current === defaultVersion` |
| `STREAM_EXISTS` | `current !== defaultVersion` |
| A `bigint` value | `current === expected` (exact match) |

The matching is performed by `matchesExpectedVersion()` and `assertExpectedVersionMatchesCurrent()`, both exported from `@event-driven-io/emmett`.

## Handling Concurrency Errors

```typescript
import {
  ExpectedVersionConflictError,
  isExpectedVersionConflictError,
} from '@event-driven-io/emmett';

try {
  await eventStore.appendToStream(streamName, events, {
    expectedStreamVersion: expectedVersion,
  });
} catch (error) {
  if (isExpectedVersionConflictError(error)) {
    // The stream version didn't match -- another write happened first.
    // Re-read the stream, re-apply business logic, and retry.
    console.log('Concurrency conflict, retrying...');
  }
  throw error;
}
```

`ExpectedVersionConflictError` extends `ConcurrencyError`, which extends `EmmettError` with `errorCode = 412`. See [[Error Hierarchy]] for the full error class tree.

> [!tip]
> The `isExpectedVersionConflictError` function uses both `instanceof` checks and duck-typing (`errorCode === 412`), making it resilient across module boundaries where `instanceof` might fail.

## See Also

- [[Appending Events]] -- Where `expectedStreamVersion` is used most commonly
- [[Reading Streams]] -- Also supports `expectedStreamVersion` for read-time checks
- [[Error Hierarchy]] -- `ExpectedVersionConflictError` extends `ConcurrencyError` extends `EmmettError`
- [[Retry Logic]] -- Automatic retry on version conflicts via `CommandHandler`
- [[ETag Utilities]] -- Weak ETag encoding of stream versions for HTTP concurrency
- [[Optimistic Concurrency]] -- Pattern-level view across the full stack
