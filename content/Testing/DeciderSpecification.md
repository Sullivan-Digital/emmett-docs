---
tags:
  - testing
  - bdd
aliases:
  - DeciderSpecification
  - Given/When/Then
related:
  - "[[The Decider Pattern]]"
  - "[[Assertions Library]]"
  - "[[Shopping Cart Example]]"
package: emmett
---

# DeciderSpecification

`DeciderSpecification` provides a BDD-style **Given/When/Then** API for testing [[The Decider Pattern|Deciders]] in isolation, without needing an event store or any infrastructure.

## Basic Usage

```typescript
import { DeciderSpecification } from '@event-driven-io/emmett';

const given = DeciderSpecification.for({ decide, evolve, initialState });
```

Pass the three Decider functions to get a `given` function that starts the specification chain.

## Given/When/Then Pattern

**Given** existing events (the stream history), **When** a command is issued, **Then** expect certain events to be produced:

```typescript
import { describe, it } from 'node:test';

describe('ShoppingCart', () => {
  const given = DeciderSpecification.for({ decide, evolve, initialState });

  it('adds an item to an empty cart', () => {
    given([])
      .when({ type: 'AddItem', data: { item: 'Shoes' } })
      .then({ type: 'ItemAdded', data: { item: 'Shoes' } });
  });

  it('adds an item to a cart with existing items', () => {
    given([{ type: 'ItemAdded', data: { item: 'Hat' } }])
      .when({ type: 'AddItem', data: { item: 'Shoes' } })
      .then({ type: 'ItemAdded', data: { item: 'Shoes' } });
  });

  it('produces no events for a no-op command', () => {
    given([])
      .when(someNoOpCommand)
      .thenNothingHappened();
  });
});
```

You can pass a single event or an array to both `given()` and `then()`:

```typescript
// Single given event
given({ type: 'ItemAdded', data: { item: 'Hat' } })
  .when(command)
  .then({ type: 'ItemAdded', data: { item: 'Shoes' } });

// Multiple expected events
given([])
  .when(command)
  .then([
    { type: 'ItemAdded', data: { item: 'Shoes' } },
    { type: 'CartClosed', data: {} },
  ]);
```

## Testing Errors

Use `thenThrows` to verify that a command causes an error:

```typescript
it('rejects adding items to a closed cart', () => {
  given([
    { type: 'ItemAdded', data: { item: 'Hat' } },
    { type: 'CartClosed', data: {} },
  ])
    .when({ type: 'AddItem', data: { item: 'Shoes' } })
    .thenThrows();
});
```

`thenThrows` supports multiple overloads for specificity:

```typescript
// Any error
.thenThrows()

// Check error with a predicate
.thenThrows((error) => error.message === 'Cart is closed')

// Check error type (uses instanceof)
.thenThrows(IllegalStateError)

// Check error type AND a predicate
.thenThrows(IllegalStateError, (error) => error.message.includes('closed'))
```

## Async Deciders

If your `decide` function returns a `Promise`, `DeciderSpecification.for()` automatically returns an async specification where `then`, `thenNothingHappened`, and `thenThrows` return Promises:

```typescript
const asyncDecide = async (command: Command, state: State): Promise<Event[]> => {
  const result = await someAsyncValidation(command);
  return [{ type: 'SomethingHappened', data: result }];
};

const given = DeciderSpecification.for({
  decide: asyncDecide,
  evolve,
  initialState,
});

it('handles async decide', async () => {
  await given([])
    .when(command)
    .then(expectedEvent);
});

it('handles async errors', async () => {
  await given([])
    .when(invalidCommand)
    .thenThrows(ValidationError);
});
```

The factory uses function overloading to infer the correct return type -- no explicit type annotation needed.

## API Summary

| Method | Description |
|---|---|
| `given(events)` | Set up prior events (single event or array). Use `[]` for empty history. |
| `.when(command)` | Execute a command against the state built from given events |
| `.then(events)` | Assert the resulting events match |
| `.thenNothingHappened()` | Assert no events were produced |
| `.thenThrows()` | Assert the command throws an error |
| `.thenThrows(ErrorClass)` | Assert it throws a specific error type |
| `.thenThrows(check)` | Assert with a custom predicate |
| `.thenThrows(ErrorClass, check)` | Assert error type and a predicate |

> [!warning] Subset matching in `then()`
> The specification uses `containsOnlyElementsMatching()` from the [[Assertions Library]], which uses ==subset matching== via `isSubset`. Extra properties on actual events will NOT cause failures -- only the properties you specify in expected events are checked. If you need strict equality, use the assertions library directly.

## See Also

- [[The Decider Pattern]] -- The `decide`/`evolve`/`initialState` abstraction being tested
- [[Shopping Cart Example]] -- Complete test examples using `DeciderSpecification`
- [[WorkflowSpecification]] -- The same BDD pattern for Workflows
- [[Assertions Library]] -- The underlying assertion functions and `isSubset`
