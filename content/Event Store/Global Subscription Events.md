---
tags:
  - event-store
  - internal
aliases:
  - GlobalStreamCaughtUp
related:
  - "[[Consumer Architecture]]"
  - "[[Checkpointing]]"
package: emmett
---

# Global Subscription Events

Emmett defines internal events used by subscription and streaming infrastructure to coordinate processing. These events use the `__emt:` prefix to distinguish them from application events.

> [!note]
> These are internal implementation details. You typically do not need to work with these types directly unless you are building a custom consumer or subscription mechanism.

## GlobalStreamCaughtUp

The primary global subscription event signals that a consumer has caught up to a specific position in the global event stream:

```typescript
const GlobalStreamCaughtUpType = '__emt:GlobalStreamCaughtUp';

type GlobalStreamCaughtUp = Event<
  '__emt:GlobalStreamCaughtUp',
  { globalPosition: bigint },
  { globalPosition: bigint }
>;

type GlobalSubscriptionEvent = GlobalStreamCaughtUp;
```
^global-stream-caught-up-type

## Helper Functions

```typescript
import {
  isGlobalStreamCaughtUp,
  caughtUpEventFrom,
  globalStreamCaughtUp,
  isSubscriptionEvent,
  isNotInternalEvent,
} from '@event-driven-io/emmett';
```

| Function | Description |
|---|---|
| `isGlobalStreamCaughtUp(event)` | Type guard: checks if an event is a `GlobalStreamCaughtUp` |
| `globalStreamCaughtUp(data)` | Factory: creates a `GlobalStreamCaughtUp` event from `{ globalPosition }` |
| `caughtUpEventFrom(position)` | Returns a type guard function that checks if a `ReadEvent` is a caught-up event at a specific position |
| `isSubscriptionEvent(event)` | Type guard: checks if an event is any `GlobalSubscriptionEvent` |
| `isNotInternalEvent(event)` | Filter predicate: returns `true` for non-internal events (those without the `__emt:` prefix) |

## Usage in Consumers

The [[Consumer Architecture|consumer]] subsystem uses these events to signal processing milestones. For example, a consumer that has processed all available events up to a certain global position can emit a `GlobalStreamCaughtUp` event. Downstream processors can use `isNotInternalEvent` to filter these out when they only care about application events.

```typescript
// Filter out internal subscription events
const applicationEvents = events.filter(isNotInternalEvent);
```

## See Also

- [[Consumer Architecture]] -- Where these events are used for coordination
- [[Checkpointing]] -- Checkpoint tracking uses global position values
- [[Event Metadata]] -- The `globalPosition` field on read events
