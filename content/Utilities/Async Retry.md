---
tags:
  - utility
  - resilience
aliases:
  - asyncRetry function
related:
  - "[[Retry Logic]]"
  - "[[Utilities MOC]]"
package: emmett
---

# Async Retry

The `asyncRetry` utility wraps the `async-retry` library with additional filtering capabilities. For the pattern-level view of how retry is used across Emmett, see [[Retry Logic]].

## Usage

```typescript
import { asyncRetry, NoRetries } from '@event-driven-io/emmett';

// Basic retry
const result = await asyncRetry(
  async () => {
    return await fetchData();
  },
  { retries: 3, minTimeout: 100, factor: 2 },
);

// Retry only on specific errors
const result = await asyncRetry(
  async () => fetchData(),
  {
    retries: 5,
    shouldRetryError: (error) => error instanceof TransientError,
  },
);

// Retry based on the result value
const result = await asyncRetry(
  async () => pollForStatus(),
  {
    retries: 10,
    minTimeout: 500,
    shouldRetryResult: (result) => result.status === 'pending',
  },
);

// Explicitly disable retries
const result = await asyncRetry(fn, NoRetries);
```

## Options

```typescript
type AsyncRetryOptions<T = unknown> = {
  retries: number;
  minTimeout?: number;
  maxTimeout?: number;
  factor?: number;
  shouldRetryResult?: (result: T) => boolean;
  shouldRetryError?: (error?: unknown) => boolean;
};
```

## Behavior

- If `opts` is `undefined` or `retries` is `0`, the function is called once with no retry wrapper (short-circuit for performance)
- **`shouldRetryError`**: Errors that don't match this predicate cause an immediate bail. Errors that match proceed with normal retry logic.
- **`shouldRetryResult`**: When the predicate returns `true` for a result, an error is thrown internally to trigger a retry

## See Also

- [[Retry Logic]] -- How retry is used by CommandHandler, WorkflowProcessor, and adapters
- [[Utilities MOC]] -- Other utility modules
