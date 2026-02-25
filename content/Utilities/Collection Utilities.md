---
tags:
  - utility
aliases:
  - arrayUtils
related:
  - "[[Utilities MOC]]"
package: emmett
---

# Collection Utilities

Array manipulation helpers bundled under `arrayUtils`.

## merge

Finds and replaces an element in an array, or appends a new one:

```typescript
import { arrayUtils } from '@event-driven-io/emmett';

const items = [
  { id: 1, name: 'Alice' },
  { id: 2, name: 'Bob' },
];

// Update existing element
const updated = arrayUtils.merge(
  items,
  { id: 2, name: 'Bobby' },
  (current) => current.id === 2,             // find predicate
  (current) => ({ ...current, name: 'Bobby' }), // transform if found
  () => undefined,                            // if not found (undefined = don't add)
);
// [{ id: 1, name: 'Alice' }, { id: 2, name: 'Bobby' }]

// Add new element if not found
const withNew = arrayUtils.merge(
  items,
  { id: 3, name: 'Charlie' },
  (current) => current.id === 3,
  (current) => current,
  () => ({ id: 3, name: 'Charlie' }),  // return value to append
);
// [{ id: 1, name: 'Alice' }, { id: 2, name: 'Bob' }, { id: 3, name: 'Charlie' }]
```

The five parameters:
1. `array` -- The source array
2. `item` -- The item to merge
3. `where` -- Predicate to find the existing element
4. `onExisting` -- Transform function if found
5. `onNotFound` -- Factory if not found (return `undefined` to skip adding)

## hasDuplicates / getDuplicates

```typescript
const items = [
  { id: 1, name: 'Alice' },
  { id: 1, name: 'Alice Copy' },
  { id: 2, name: 'Bob' },
];

arrayUtils.hasDuplicates(items, (item) => item.id); // true

arrayUtils.getDuplicates(items, (item) => item.id);
// [{ id: 1, name: 'Alice' }, { id: 1, name: 'Alice Copy' }]
```

Both accept a predicate that maps items to a key for duplicate detection.

## See Also

- [[Utilities MOC]] -- Other utility modules
