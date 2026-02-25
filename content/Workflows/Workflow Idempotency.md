---
tags:
  - workflow
  - idempotency
aliases:
  - Workflow Deduplication
related:
  - "[[Workflow Processor]]"
  - "[[Concurrency Control]]"
  - "[[Retry Logic]]"
package: emmett
---

# Workflow Idempotency

> [!abstract]
> Workflow processing is ==idempotent by default==. Each input message's `metadata.messageId` is tracked in the workflow's internal state. If the same message is delivered again (e.g., after a crash), it is detected during stream aggregation and skipped.

## How Deduplication Works

The workflow handler maintains an internal `processedInputIds` Set as part of the workflow state:

1. **During stream aggregation** (evolve): Input events that have `metadata.input === true` contribute their `originalMessageId` to the `processedInputIds` Set
2. **Before processing**: The handler checks if the incoming message's `messageId` is already in `processedInputIds`
3. **If duplicate**: Returns an empty `newMessages` array with the current stream version -- no `decide` call, no new events stored

```
Input arrives -> Rebuild state from stream
  -> messageId in processedInputIds?
     Yes -> Return empty result (skip)
     No  -> Run decide(), append events
```

> [!note]
> The deduplication state is rebuilt from the event stream on every invocation. There is no separate deduplication table -- the workflow stream itself is the source of truth for which messages have been processed.

## Retry on Version Conflict

When multiple processors or instances attempt to write to the same workflow stream concurrently, an `ExpectedVersionConflictError` can occur. The `retry` option handles this:

### Simple Form (defaults)

```typescript
// Default: 3 retries, 100ms min timeout, factor 1.5
consumer.workflowProcessor({
  // ...
  retry: { onVersionConflict: true },
});
```

### Custom Retry Count

```typescript
consumer.workflowProcessor({
  // ...
  retry: { onVersionConflict: 5 },
});
```

### Full Control

```typescript
consumer.workflowProcessor({
  // ...
  retry: {
    onVersionConflict: {
      retries: 10,
      minTimeout: 200,
      maxTimeout: 5000,
      factor: 2,
    },
  },
});
```

> [!warning]
> Only `ExpectedVersionConflictError` triggers a retry. Other errors propagate immediately without retrying.

### How Retry Works

On a version conflict:

1. The entire workflow handler re-executes from the beginning
2. The stream is re-read and state is rebuilt (picking up the concurrent write)
3. The idempotency check runs again -- if the conflicting write was the same input, it will be detected as a duplicate and skipped
4. Otherwise, `decide` runs against the updated state

This means retries are safe: either the message was already processed (idempotency catches it) or the state has changed and `decide` produces a correct result based on the latest state.

### Default Retry Configuration

| Parameter | Default Value |
|---|---|
| `retries` | 3 |
| `minTimeout` | 100ms |
| `maxTimeout` | (unbounded) |
| `factor` | 1.5 |

## Interaction with Separated Inbox

When [[Separated Inbox Mode]] is enabled, idempotency provides an additional safety net:

- If Phase 1 (store) succeeds but the consumer crashes before acknowledging, the unprefixed input may be redelivered
- Phase 1 runs again, but the idempotency check in Phase 2 will detect the duplicate input since it was already stored with a prefixed type in the workflow stream

## See Also

- [[Workflow Processor]] -- The full execution flow including idempotency checks
- [[Separated Inbox Mode]] -- Two-phase processing that complements idempotency
- [[Concurrency Control]] -- `ExpectedVersionConflictError` and optimistic concurrency
- [[Retry Logic]] -- The `asyncRetry` utility and `AsyncRetryOptions` used internally
