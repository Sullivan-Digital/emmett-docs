---
tags:
  - adapter
  - eventstoredb
aliases:
  - mapFromESDBEvent
related:
  - "[[EventStoreDB Event Store]]"
  - "[[Recorded Messages]]"
  - "[[Checkpointing]]"
package: emmett-esdb
---

# EventStoreDB Event Mapping

The ==`mapFromESDBEvent()`== function converts EventStoreDB's `ResolvedEvent` into Emmett's `RecordedMessage` format. This mapping runs automatically during reads and subscription processing, translating between ESDB's gRPC data model and Emmett's internal types.

## Field Mapping

| EventStoreDB (`ResolvedEvent`) | Emmett (`RecordedMessage`) |
|---|---|
| `event.type` | `type` |
| `event.data` | `data` |
| `event.id` | `metadata.eventId` |
| `event.streamId` | `metadata.streamName` |
| `event.revision` | `metadata.streamPosition` |
| `event.position.commit` | `metadata.globalPosition` |

## Function Signature

```typescript
const mapFromESDBEvent = <MessageType>(
  resolvedEvent: ResolvedEvent<MessageType>,
  from?: EventStoreDBEventStoreConsumerType,
): RecordedMessage<MessageType, EventStoreDBReadEventMetadata>;
```

The optional `from` parameter determines how the `checkpoint` field is computed, based on the subscription type.

## Checkpoint Semantics

The checkpoint value is critical for correct consumer resume behavior and ==differs by subscription type==:

### `$all` Subscriptions

Checkpoint uses the ==global commit position==:

```
link.position.commit ?? event.position.commit
```

This is a global position across all streams in EventStoreDB.

### Named Stream Subscriptions

Checkpoint uses the ==stream revision==:

```
link.revision ?? event.revision
```

This is the position within the specific stream being subscribed to.

> [!warning]
> Mixing subscription types for the same `processorId` will cause ==incorrect resume positions==. An `$all` checkpoint is a global position, while a stream checkpoint is a revision number -- they are not interchangeable.

### Link vs Event Position

When EventStoreDB returns a `ResolvedEvent`, it may include both a `link` (the pointer event in a projection like `$ce-`) and the actual `event`. The mapper prefers the link's position when available, falling back to the event's position:

- **Category projections** (`$ce-`): The link carries the `$all` position of the link event
- **Direct streams**: No link, so the event's own position is used

## Start Position Mapping

When a consumer starts (or restarts), the checkpoint is mapped back to ESDB's subscription position:

**For `$all`:**
- `'BEGINNING'` maps to `START`
- `'END'` maps to `END`
- Checkpoint value maps to `{ prepare: bigint, commit: bigint }`

**For named streams:**
- `'BEGINNING'` maps to `START`
- `'END'` maps to `END`
- Checkpoint value maps to a `bigint` (stream revision)

## See Also

- [[EventStoreDB Consumer]] -- Where the mapping is used in subscription processing
- [[EventStoreDB Event Store]] -- Where the mapping is used during `readStream`
- [[Recorded Messages]] -- The target format: `RecordedMessage` and metadata types
- [[Checkpointing]] -- How checkpoints drive consumer resumption
