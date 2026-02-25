---
tags:
  - errors
  - validation
aliases:
  - Validation
  - Type Guards
related:
  - "[[Error Hierarchy]]"
  - "[[Express.js Integration]]"
package: emmett
---

# Validation Helpers

Emmett provides type guards for runtime type checking, assertion validators that throw [[Error Hierarchy|ValidationError]], and date utilities for UTC date string handling.

## Type Guards

Simple type-checking functions that narrow types:

```typescript
import { isNumber, isBigint, isString } from '@event-driven-io/emmett';

isNumber(42);        // true
isNumber(NaN);       // false (NaN is excluded)
isNumber('42');      // false

isBigint(42n);       // true
isBigint(42);        // false

isString('hello');   // true
isString(42);        // false
```

> [!note] `isNumber` explicitly excludes `NaN` -- it checks both `typeof value === 'number'` and that the value is not `NaN`.

## Assertion Validators

Functions that validate a value and return it typed, or throw `ValidationError`:

```typescript
import {
  assertNotEmptyString,
  assertPositiveNumber,
  assertUnsignedBigInt,
} from '@event-driven-io/emmett';

// Returns the string if non-empty, throws ValidationError otherwise
const name: string = assertNotEmptyString(input.name);

// Returns the number if positive, throws ValidationError otherwise
const quantity: number = assertPositiveNumber(input.quantity);

// Parses and returns a non-negative bigint, throws ValidationError otherwise
const position: bigint = assertUnsignedBigInt('12345');
```

Each validator throws with a specific error code from the `ValidationErrors` enum:

| Validator | Error Code |
|---|---|
| `assertNotEmptyString` | `NOT_A_NONEMPTY_STRING` |
| `assertPositiveNumber` | `NOT_A_POSITIVE_NUMBER` |
| `assertUnsignedBigInt` | `NOT_AN_UNSIGNED_BIGINT` |

These are useful at system boundaries -- validating HTTP request parameters, parsing IDs, etc.

## Date Utilities

Utilities for working with UTC dates in `YYYY-MM-DD` format:

```typescript
import {
  formatDateToUtcYYYYMMDD,
  isValidYYYYMMDD,
  parseDateFromUtcYYYYMMDD,
} from '@event-driven-io/emmett';

// Format a Date to "YYYY-MM-DD" string in UTC
const dateStr = formatDateToUtcYYYYMMDD(new Date('2025-03-15T10:00:00Z'));
// "2025-03-15"

// Check if a string matches YYYY-MM-DD format
isValidYYYYMMDD('2025-03-15');  // true
isValidYYYYMMDD('15-03-2025');  // false
isValidYYYYMMDD('2025/03/15');  // false

// Parse a "YYYY-MM-DD" string to a UTC Date
const date = parseDateFromUtcYYYYMMDD('2025-03-15');
// Date representing 2025-03-15T00:00:00Z

// Throws ValidationError for invalid formats
parseDateFromUtcYYYYMMDD('not-a-date'); // throws ValidationError
```

## See Also

- [[Error Hierarchy]] -- `ValidationError` class and `EmmettError.Codes`
- [[Express.js Integration]] -- Using validators in route handlers
- [[Problem Details]] -- Automatic `ValidationError` to HTTP 400 mapping
