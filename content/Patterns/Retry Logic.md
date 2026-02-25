---
tags:
  - pattern
  - resilience
aliases:
  - asyncRetry
  - AsyncRetryOptions
related:
  - "[[Command Handling]]"
  - "[[Workflow Processor]]"
  - "[[EventStoreDB Reconnection]]"
  - "[[Concurrency Control]]"
package: emmett
---

# Retry Logic

Emmett uses the `asyncRetry` utility throughout the library for resilient operations: command handler version conflict retries, workflow processor retries, and adapter reconnection.

## asyncRetry

The core utility wraps the `async-retry` library with additional filtering capabilities:

```typescript
import { asyncRetry, NoRetries } from '@event-driven-io/emmett';

const result = await asyncRetry(
  async () => {
    return await fetchData();
  },
  { retries: 3, minTimeout: 100, factor: 2 },
);
```

### AsyncRetryOptions

```typescript
type AsyncRetryOptions<T = unknown> = {
  retries: number;           // Number of retry attempts
  minTimeout?: number;       // Minimum delay between retries (ms)
  maxTimeout?: number;       // Maximum delay between retries (ms)
  factor?: number;           // Exponential backoff factor
  shouldRetryResult?: (result: T) => boolean;    // Retry if result matches
  shouldRetryError?: (error?: unknown) => boolean; // Only retry matching errors
};
```

**Behavior:**

- If `opts` is `undefined` or `retries` is `0`, the function is called once with no retry wrapper (short-circuit)
- ==`shouldRetryError`==: When provided, errors that don't match this predicate bail immediately. Errors that match proceed with normal retry.
- `shouldRetryResult`: When the predicate returns `true`, an error is thrown internally to trigger a retry

### NoRetries

```typescript
const NoRetries: AsyncRetryOptions = { retries: 0 };
```

Use `NoRetries` to explicitly disable retries:

```typescript
const result = await asyncRetry(fn, NoRetries);
```

## Command Handler Retries

The [[Command Handling|CommandHandler]] uses retry for optimistic concurrency conflict resolution. The `onVersionConflict` option has three forms:

```typescript
// Use defaults: 3 retries, 100ms min timeout, 1.5 exponential factor
retry: { onVersionConflict: true }

// Custom number of retries, other defaults unchanged
retry: { onVersionConflict: 5 }

// Full control over retry behavior
retry: {
  onVersionConflict: {
    retries: 3,
    minTimeout: 100,
    factor: 1.5,
  }
}
```

The default configuration:

```typescript
const CommandHandlerStreamVersionConflictRetryOptions: AsyncRetryOptions = {
  retries: 3,
  minTimeout: 100,
  factor: 1.5,
  shouldRetryError: isExpectedVersionConflictError,
};
```

> [!warning] By default, ==only `ExpectedVersionConflictError` triggers a retry==. All other errors bail immediately without retrying. The entire operation (read stream, run handler, append events) is re-executed on each retry.

For retrying on any error type, pass raw `AsyncRetryOptions`:

```typescript
retry: {
  retries: 5,
  minTimeout: 200,
  factor: 2,
  shouldRetryError: (error) => error instanceof SomeTransientError,
}
```

## Other Uses in Emmett

- **[[Workflow Processor]]** -- `retry.onVersionConflict` for workflow stream appends
- **[[EventStoreDB Reconnection]]** -- `forever: true` with exponential backoff for subscription recovery
- **[[PostgreSQL Rebuilding Projections]]** -- Aggressive retry (100 retries, 100-5000ms) for lock acquisition during projection rebuilds

## See Also

- [[Async Retry]] -- The lower-level utility function details
- [[Optimistic Concurrency]] -- The pattern that drives version conflict retries
- [[Concurrency Control]] -- Event store version checking
