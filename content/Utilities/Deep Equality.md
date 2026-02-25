---
tags:
  - utility
aliases:
  - deepEquals
  - Equatable
related:
  - "[[Assertions Library]]"
  - "[[Utilities MOC]]"
package: emmett
---

# Deep Equality

`deepEquals` provides comprehensive deep comparison for any JavaScript value type.

```typescript
import { deepEquals } from '@event-driven-io/emmett';

deepEquals({ a: 1, b: [2, 3] }, { a: 1, b: [2, 3] }); // true
deepEquals(new Date('2025-01-01'), new Date('2025-01-01')); // true
deepEquals(new Map([['a', 1]]), new Map([['a', 1]])); // true
deepEquals(new Set([1, 2]), new Set([1, 2])); // true
```

## Supported Types

Primitives, arrays, Dates, RegExps, Errors, Maps, Sets, ArrayBuffers, typed arrays, boxed primitives, and plain objects.

## Equatable Protocol

Objects can implement the `Equatable<T>` interface for custom equality logic:

```typescript
import { type Equatable } from '@event-driven-io/emmett';

class Money implements Equatable<Money> {
  constructor(public amount: number, public currency: string) {}

  equals(other: Money): boolean {
    return this.amount === other.amount && this.currency === other.currency;
  }
}

deepEquals(new Money(100, 'USD'), new Money(100, 'USD')); // true (uses .equals())
```

If either object implements `Equatable` (has an `equals` method), that method is used instead of the default comparison.

## Notable Behavior

- **Functions in objects** -- When comparing object properties, if both values are functions they are treated as ==equal== (skipped). Functions compared directly use reference equality (`===`).
- **DataView, WeakMap, WeakSet** -- Comparisons always return `false` (these types cannot be meaningfully compared).
- **Circular references** -- Not explicitly handled. Deeply recursive structures may cause stack overflow.

## Types

```typescript
type Equatable<T> = { equals: (right: T) => boolean } & T;

const isEquatable: <T>(left: T) => left is Equatable<T>;
```

## See Also

- [[Assertions Library]] -- `assertDeepEqual` uses `deepEquals` internally
- [[Utilities MOC]] -- Other utility modules
