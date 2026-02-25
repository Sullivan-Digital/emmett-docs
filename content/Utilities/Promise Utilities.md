---
tags:
  - utility
aliases:
  - delay
  - asyncAwaiter
related:
  - "[[Utilities MOC]]"
package: emmett
---

# Promise Utilities

Two promise helpers for async control flow.

## delay

A simple promise-based timer:

```typescript
import { delay } from '@event-driven-io/emmett';

await delay(1000); // Wait 1 second
```

## asyncAwaiter

Creates a manually resolvable/rejectable promise with a ==reset== capability:

```typescript
import { asyncAwaiter } from '@event-driven-io/emmett';

const awaiter = asyncAwaiter<string>();

// In one part of your code, wait for the value
const result = await awaiter.wait; // blocks until resolved

// In another part, provide the value
awaiter.resolve('done');

// Reset to create a fresh promise for reuse
awaiter.reset();
```

This is similar to `Promise.withResolvers()` (available in newer runtimes) with the addition of a `reset()` method for reusable await points.

### Type

```typescript
type AsyncAwaiter<T = void> = {
  wait: Promise<T>;
  resolve: (value: T | PromiseLike<T>) => void;
  reject: (reason?: any) => void;
  reset: () => void;
};
```

## See Also

- [[Utilities MOC]] -- Other utility modules
