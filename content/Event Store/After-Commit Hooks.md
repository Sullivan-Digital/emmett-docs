---
tags:
  - event-store
  - hooks
aliases:
  - onAfterCommit
related:
  - "[[Appending Events]]"
  - "[[Message Bus]]"
  - "[[Before-Commit Hooks]]"
package: emmett
---

# After-Commit Hooks

After-commit hooks run after events have been stored and [[Inline Projections|inline projections]] have completed. They are ==fire-and-forget==: if the hook fails, the append still succeeds and no error is thrown to the caller.

> [!abstract]
> Suitable for broadcasting events to a [[Message Bus|message bus]], lightweight notifications, and non-critical side effects. Not suitable for guaranteed delivery or operations that must be atomic with event storage.

## Registering a Hook

Pass the `onAfterCommit` hook when creating an event store via the `hooks` option:

```typescript
import { getInMemoryEventStore } from '@event-driven-io/emmett';

const eventStore = getInMemoryEventStore({
  hooks: {
    onAfterCommit: (events) => {
      console.log(`${events.length} event(s) committed:`);
      for (const event of events) {
        console.log(`  - ${event.type} at position ${event.metadata.streamPosition}`);
      }
    },
  },
});
```

The hook receives an array of `ReadEvent` objects -- the events that were just appended, enriched with [[Event Metadata|metadata]].

The same pattern works with MongoDB:

```typescript
import { getMongoDBEventStore } from '@event-driven-io/emmett-mongodb';

const eventStore = getMongoDBEventStore({
  client: mongoClient,
  hooks: {
    onAfterCommit: (events) => {
      // React to committed events
    },
  },
});
```

## Forwarding Events to a Message Bus

A common pattern is forwarding committed events to an in-memory [[Message Bus|message bus]]. Emmett provides the ==`forwardToMessageBus`== helper:

```typescript
import {
  getInMemoryEventStore,
  forwardToMessageBus,
  getInMemoryMessageBus,
} from '@event-driven-io/emmett';

const messageBus = getInMemoryMessageBus();

const eventStore = getInMemoryEventStore({
  hooks: {
    onAfterCommit: forwardToMessageBus(messageBus),
  },
});
```

`forwardToMessageBus` accepts any object implementing the `EventsPublisher` interface and publishes each event sequentially in order:

```typescript
interface EventsPublisher {
  publish<EventType extends Event>(event: EventType): Promise<void>;
}
```

> [!note]
> Events within a single batch are published one at a time via a `for...of` loop with `await`, not in parallel. This preserves ordering within a batch but may be slower for large batches.

## Error Handling and Reliability

```typescript
const eventStore = getInMemoryEventStore({
  hooks: {
    onAfterCommit: (events) => {
      throw new Error('Hook failed!');
    },
  },
});

// This still succeeds -- events are persisted
await eventStore.appendToStream(streamName, events);
```

When a hook throws:

1. The error is caught internally by `tryPublishMessagesAfterCommit`
2. The error is logged to `console.error`
3. No error is propagated to the caller of [[Appending Events|appendToStream]]
4. There is ==no retry== -- the hook will not be called again for those events

If the process crashes after events are committed but before the hook fires, the hook will not be retried on restart. For scenarios requiring guaranteed delivery, use the async consumer/projection system instead.

## tryPublishMessagesAfterCommit

If you are building a custom event store adapter or need to invoke the after-commit hook manually:

```typescript
import { tryPublishMessagesAfterCommit, forwardToMessageBus } from '@event-driven-io/emmett';

const success = await tryPublishMessagesAfterCommit(committedEvents, {
  onAfterCommit: forwardToMessageBus(messageBus),
});
// success is true if the hook ran without error, false otherwise
```

## Handler Type

```typescript
type AfterEventStoreCommitHandler<
  Store extends EventStore,
  HandlerContext extends DefaultRecord | undefined = undefined,
>
```
^after-commit-handler-type

This is a conditional type that resolves to:

- `(messages: ReadEvent[]) => Promise<void> | void` -- when `HandlerContext` is `undefined` (e.g., in-memory store)
- `(messages: ReadEvent[], context: HandlerContext) => Promise<void> | void` -- when `HandlerContext` is defined (e.g., database adapters with connection context)

## Adapter Support

| Adapter | `onAfterCommit` | Notes |
|---------|:---------------:|-------|
| InMemory | Yes | Fire-and-forget after append |
| MongoDB | Yes | Fire-and-forget after append |
| PostgreSQL | -- | Uses internal [[Before-Commit Hooks|before-commit hook]] for projections; no user-facing after-commit |
| SQLite | -- | Uses [[Before-Commit Hooks|onBeforeCommit]] instead |
| EventStoreDB | -- | No hooks |

> [!warning]
> After-commit hooks receive ==pre-downcast events==. If you use both [[Schema Versioning|downcasting]] and an after-commit hook, the hook receives events in their application-level types, not the stored format.

> [!warning]
> Under concurrent appends, there is ==no ordering guarantee== between hook invocations. A hook from a later append may complete before an earlier one. Within a single batch, order is preserved.

## See Also

- [[Before-Commit Hooks]] -- Transactional hooks that run inside the database commit
- [[Appending Events]] -- The execution order within `appendToStream`
- [[Message Bus]] -- Where forwarded events are published to
- [[Consumer Architecture]] -- For guaranteed delivery, use async consumers instead
