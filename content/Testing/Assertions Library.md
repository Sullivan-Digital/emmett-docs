---
tags:
  - testing
aliases:
  - Assertions
  - assertThatArray
  - verifyThat
related:
  - "[[DeciderSpecification]]"
  - "[[Testing MOC]]"
  - "[[Deep Equality]]"
package: emmett
---

# Assertions Library

Emmett includes a comprehensive, framework-agnostic assertion library. All assertions throw `AssertionError` on failure and work with any test runner (Node.js built-in, Vitest, Jest, etc.).

> [!warning] The assertion error class is named ==`AssertionError`== (missing an "s") throughout the codebase. This is intentional and consistent -- do not try to import `AssertionError`.

## Basic Assertions

```typescript
import {
  assertTrue,
  assertFalse,
  assertOk,
  assertDefined,
  assertEqual,
  assertNotEqual,
  assertIsNotNull,
  assertIsNull,
} from '@event-driven-io/emmett';

assertTrue(condition);               // asserts condition is true
assertFalse(condition);              // asserts condition is false
assertOk(value);                     // asserts value is not null/undefined
assertDefined(value);                // asserts value is defined
assertEqual(expected, actual);       // strict equality (===)
assertNotEqual(obj, other);          // strict inequality (!==)
assertIsNotNull(result);             // asserts result is not null
assertIsNull(result);                // asserts result is null
```

## Deep Comparison

```typescript
import {
  assertDeepEqual,
  assertNotDeepEqual,
  assertMatches,
} from '@event-driven-io/emmett';

// Full deep equality (all properties must match)
assertDeepEqual(actual, expected);
assertNotDeepEqual(actual, other);

// Subset matching (actual can have extra properties)
assertMatches(
  { name: 'Alice', age: 30, extra: true },
  { name: 'Alice', age: 30 },
); // passes -- extra properties are ignored
```

`assertDeepEqual` uses [[Deep Equality|deepEquals]] internally. `assertMatches` uses `isSubset` for partial matching.

## Throwing Assertions

```typescript
import {
  assertThrows,
  assertThrowsAsync,
  assertDoesNotThrow,
  assertRejects,
  assertFails,
} from '@event-driven-io/emmett';

// Sync function throws
const error = assertThrows(() => {
  throw new ValidationError('bad input');
});

// Sync function throws with condition
assertThrows(
  () => { throw new Error('boom'); },
  (err) => err.message === 'boom',
);

// Async function throws
await assertThrowsAsync(
  async () => { throw new Error('async boom'); },
  (err) => err.message === 'async boom',
);

// Verify a function does NOT throw
assertDoesNotThrow(() => safeOperation());

// Promise rejects
await assertRejects(
  failingPromise(),
  (err) => err.message === 'failed',
);

// Unconditionally fail (useful for unreachable code paths)
assertFails('Should not reach here');
```

## Array Assertions (Fluent)

The `assertThatArray` builder provides a fluent API for array assertions: ^array-assertions

```typescript
import { assertThatArray } from '@event-driven-io/emmett';

const items = [1, 2, 3, 4, 5];

// Size checks
assertThatArray(items).isEmpty();
assertThatArray(items).isNotEmpty();
assertThatArray(items).hasSize(5);

// Containment checks
assertThatArray(items).contains(3);
assertThatArray(items).containsElements([2, 4]);
assertThatArray(items).containsAnyOf([7, 3, 9]);

// Exact matching
assertThatArray(items).containsExactly(1);                    // exactly one element
assertThatArray(items).containsExactlyElementsOf([1, 2, 3, 4, 5]); // exact order
assertThatArray(items).containsExactlyInAnyOrder([5, 4, 3, 2, 1]); // any order

// Subset matching (uses isSubset -- extra properties on actual items are OK)
assertThatArray(events).containsElementsMatching(expectedPartials);
assertThatArray(events).containsOnlyElementsMatching(expectedPartials);

// Uniqueness
assertThatArray(items).containsOnlyOnceElementsOf([1, 2]);

// Predicates
assertThatArray(items).allMatch((x) => x > 0);
assertThatArray(items).anyMatches((x) => x === 3);
await assertThatArray(items).allMatchAsync(async (x) => x > 0);
```

> [!info] `containsOnlyElementsMatching` is what [[DeciderSpecification]] uses for its `then()` assertion. It verifies that all expected elements are present (via subset matching) and that the array lengths match, but does not require strict deep equality.

## Mock Verification

Works with the standard `node:test` mock API:

```typescript
import { verifyThat, argValue, argMatches } from '@event-driven-io/emmett';
import { mock } from 'node:test';

const mockFn = mock.fn();

// Call count
verifyThat(mockFn).called();
verifyThat(mockFn).notCalled();
verifyThat(mockFn).calledTimes(3);

// Argument checks
verifyThat(mockFn).calledWith('arg1', 'arg2');
verifyThat(mockFn).calledOnceWith('arg1');

// Argument matchers
verifyThat(mockFn).calledWithArgumentMatching(
  argValue('exact-value'),
  argMatches<number>((n) => n > 10),
);
verifyThat(mockFn).notCalledWithArgumentMatching(
  argValue('unwanted'),
);
```

The `MockedFunction` type expects a `mock.calls` property on the function, which is the standard `node:test` mock shape.

## isSubset

The `isSubset(superObj, subObj)` function checks whether all properties of `subObj` exist in `superObj` with matching values. Nested objects are compared recursively. This is used by `assertMatches`, `containsElementsMatching`, and `containsOnlyElementsMatching`.

## See Also

- [[DeciderSpecification]] -- Uses `containsOnlyElementsMatching` for `then()` assertions
- [[Deep Equality]] -- The `deepEquals` function used by `assertDeepEqual`
- [[Testing MOC]] -- Overview of all testing utilities
