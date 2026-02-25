---
tags:
  - testing
aliases:
  - WrapEventStore
  - EventStoreWrapper
related:
  - "[[Event Store Interface]]"
  - "[[API Testing]]"
  - "[[Assertions Library]]"
package: emmett
---

# WrapEventStore

`WrapEventStore` wraps an event store instance to track which events have been appended, useful for integration tests where you need to verify side effects beyond the return value.

## Basic Usage

```typescript
import { WrapEventStore, getInMemoryEventStore } from '@event-driven-io/emmett';

const eventStore = getInMemoryEventStore();
const wrapped = WrapEventStore(eventStore);
```

## Setup / Given Phase

Use `setup()` to establish test preconditions. Events added via `setup()` go directly to the underlying store and are ==not tracked== in `appendedEvents`:

```typescript
await wrapped.setup('cart-123', [
  { type: 'ItemAdded', data: { item: 'Hat' } },
]);
```

## Execute / When Phase

Run the code under test using the wrapped store:

```typescript
await handleCommand(wrapped, 'cart-123', (state) => {
  return { type: 'CartClosed', data: {} };
});
```

## Verify / Then Phase

Check `appendedEvents` -- only events from `appendToStream` calls (not `setup`) appear here:

```typescript
const [streamName, events] = wrapped.appendedEvents.get('cart-123')!;
expect(events).toHaveLength(1);
expect(events[0].type).toBe('CartClosed');
```

The `appendedEvents` property is a `Map<string, TestEventStream>` where `TestEventStream` is `[string, Event[]]` -- a tuple of stream name and appended events.

## Key Behaviors

- ==`appendToStream()`== is intercepted to track events in `appendedEvents`
- ==`setup()`== appends events without tracking (for the "given" phase)
- `readStream()` and `aggregateStream()` delegate directly to the wrapped store
- The wrapper has the same type as the original store (`EventStoreWrapper<Store>` extends `Store`)

> [!tip] Use `WrapEventStore` when you need to verify what events were appended as a side effect of an operation. For testing Deciders in isolation (without an event store), use [[DeciderSpecification]] instead.

## See Also

- [[Event Store Interface]] -- The interface being wrapped
- [[DeciderSpecification]] -- Unit-level testing without an event store
- [[API Testing]] -- `ApiSpecification` which uses `WrapEventStore` internally
