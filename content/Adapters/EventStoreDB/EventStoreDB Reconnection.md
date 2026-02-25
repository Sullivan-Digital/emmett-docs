---
tags:
  - adapter
  - eventstoredb
  - resilience
aliases:
  - ESDB Reconnection
related:
  - "[[EventStoreDB Consumer]]"
  - "[[Async Retry]]"
package: emmett-esdb
---

# EventStoreDB Reconnection

The [[EventStoreDB Consumer|EventStoreDB consumer]] automatically reconnects on subscription failures using ==exponential backoff== with configurable retry options. This resilience layer wraps the gRPC streaming subscription so that transient network issues don't require manual intervention.

## Default Configuration

```typescript
const EventStoreDBResubscribeDefaultOptions: AsyncRetryOptions = {
  forever: true,      // Retry indefinitely
  minTimeout: 100,    // Start at 100ms
  factor: 1.5,        // Multiply delay by 1.5 each retry
};
```

With these defaults, retry delays follow the pattern: 100ms, 150ms, 225ms, 337ms, ... growing without bound but slowing down over time.

## gRPC Code 14: Database Unavailable

The only error that ==permanently stops retries== is a gRPC "unavailable" error (code 14), which indicates the EventStoreDB server itself is down:

```typescript
const isDatabaseUnavailableError = (error: unknown) =>
  error instanceof Error &&
  'type' in error &&
  error.type === 'unavailable' &&
  'code' in error &&
  error.code === 14;
```

When this error occurs, the consumer stops attempting to reconnect. All other errors trigger a retry after the appropriate backoff delay.

> [!note]
> The retry loop also checks `shouldRetryResult: () => isRunning`, so stopping the consumer via `stop()` will also halt reconnection attempts.

## Custom Retry Configuration

You can override the default retry behavior when creating a consumer:

```typescript
const consumer = eventStoreDBEventStoreConsumer({
  connectionString: 'esdb://localhost:2113?tls=false',
  from: { stream: $all },
  resilience: {
    resubscribeOptions: {
      forever: false,
      retries: 10,          // Max 10 retry attempts
      minTimeout: 200,      // Start at 200ms
      factor: 2,            // Double the delay each time
    },
  },
});
```

The `resubscribeOptions` accepts the full `AsyncRetryOptions` type from Emmett's [[Async Retry|async retry]] utility, which wraps the `async-retry` library.

## How Reconnection Works

When a subscription drops:

1. The subscription pipeline errors or closes
2. The retry wrapper catches the error
3. If `isDatabaseUnavailableError(error)` returns `true`, retries stop permanently
4. Otherwise, the wrapper waits for the backoff delay
5. A new subscription is created from the ==last processed position==
6. Processing resumes from where it left off

> [!warning]
> Reconnection only handles ==connection-level== failures. If an `eachMessage` handler throws, the subscription stops entirely with no automatic retry. See [[EventStoreDB Consumer]] for details on handler error behavior.

## Comparison with Other Adapters

| Adapter | Reconnection Strategy |
|---|---|
| EventStoreDB | Exponential backoff with auto-retry |
| SQLite | N/A -- polling naturally retries on each poll cycle |
| PostgreSQL | N/A -- polling naturally retries on each poll cycle |
| MongoDB | Exponential backoff (similar pattern, for change stream reconnection) |

Polling-based adapters ([[SQLite Consumer|SQLite]], [[PostgreSQL Consumer|PostgreSQL]]) don't need explicit reconnection logic because each poll cycle is an independent database query. Connection failures are handled at the query level.

## See Also

- [[EventStoreDB Consumer]] -- The consumer that uses this reconnection logic
- [[Async Retry]] -- The underlying `asyncRetry()` utility
- [[Retry Logic]] -- Cross-cutting retry patterns in Emmett
