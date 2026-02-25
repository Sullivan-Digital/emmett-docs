---
tags:
  - pattern
  - concurrency
aliases:
  - Optimistic Locking
related:
  - "[[Concurrency Control]]"
  - "[[ETag Utilities]]"
  - "[[Retry Logic]]"
  - "[[Error Hierarchy]]"
package: emmett
---

# Optimistic Concurrency

Optimistic concurrency is a pattern where operations proceed without locking, but a version check at write time detects conflicts. Emmett implements this at three levels: the event store, HTTP APIs, and automatic retry.

## Event Store Level

When appending events, you pass an ==expected stream version==. If the actual version has changed since you read it, the store throws a [[Error Hierarchy|ConcurrencyError]] (HTTP 412):

```typescript
// Read current state -- this also gives us the stream version
const { state, currentStreamVersion } = await eventStore.aggregateStream(
  streamName,
  { evolve, initialState },
);

// Append with expected version -- fails if someone else wrote first
await eventStore.appendToStream(streamName, events, {
  expectedStreamVersion: currentStreamVersion,
});
```

The [[Command Handling|CommandHandler]] does this automatically. It reads the stream, runs your handler, and appends with the version from the read.

> [!info] See [[Concurrency Control]] for the full set of version constants: `NO_CONCURRENCY_CHECK`, `STREAM_EXISTS`, `STREAM_DOES_NOT_EXIST`.

## HTTP API Level

Web frameworks use **ETags** (weak entity tags) to expose stream versions to HTTP clients. The stream version is encoded as a weak ETag in the response, and clients send it back in `If-Match` headers:

```
Response: ETag: W/"5"    (stream version 5)
Request:  If-Match: "5"  (client expects version 5)
```

The [[ETag Utilities]] provide helpers for this:

```typescript
import { toWeakETag, getWeakETagValue } from '@event-driven-io/emmett-expressjs';

// Set ETag on response
const etag = toWeakETag(result.nextExpectedStreamVersion);
res.set('ETag', etag);

// Read expected version from request
const expectedVersion = getWeakETagValue(req);
```

This gives HTTP clients the same optimistic concurrency guarantee that the event store provides internally.

## Automatic Retry

When a version conflict occurs, the [[Retry Logic|retry mechanism]] can automatically re-read the stream, re-run the handler, and re-attempt the append:

```typescript
const handle = CommandHandler<ShoppingCart, ShoppingCartEvent>({
  evolve,
  initialState,
  retry: { onVersionConflict: true }, // 3 retries, 100ms base, 1.5x backoff
});
```

The retry only triggers on `ExpectedVersionConflictError`. Other errors bail immediately without retrying.

> [!warning] Retry re-executes the entire operation (read + handler + append). Your handler function should be safe to call multiple times. Avoid side effects in handlers that are not idempotent.

## The Full Flow

1. Client sends command with `If-Match: "5"` header
2. Web framework extracts expected version from ETag
3. CommandHandler reads stream, runs handler, appends with expected version
4. If version conflict: retry up to N times with exponential backoff
5. If successful: return new ETag `W/"6"` to client
6. If conflict persists after retries: return HTTP 412 (Precondition Failed)

This three-layer approach means the event store guarantees consistency, the HTTP layer exposes it to clients, and the retry layer handles transient conflicts automatically.

## See Also

- [[Concurrency Control]] -- Event store version constants and per-adapter behavior
- [[ETag Utilities]] -- Weak ETag encoding/decoding utilities
- [[Retry Logic]] -- `asyncRetry` options and behavior
- [[Command Handling]] -- How `CommandHandler` integrates all three layers
