---
tags:
  - core
  - type-system
  - persistence
aliases:
  - ReadEvent
  - RecordedMessage
related:
  - "[[Events]]"
  - "[[Event Metadata]]"
  - "[[Checkpointing]]"
  - "[[Messages]]"
package: emmett
---

# Recorded Messages

When [[Messages|messages]] are persisted to an [[Event Store Interface|event store]], they gain additional system metadata. The type system tracks this through ==`RecordedMessage`== and its event-specific alias ==`ReadEvent`==.

## CommonRecordedMessageMetadata

Every recorded message gets these fields:

```typescript
type CommonRecordedMessageMetadata = Readonly<{
  messageId: string;                            // Unique identifier for this message
  streamPosition: StreamPosition;               // Position within the stream (bigint)
  streamName: string;                           // Name of the stream
  checkpoint?: ProcessorCheckpoint | null;      // For processor tracking
}>;
```

^common-recorded-metadata

## RecordedMessageMetadata and Global Position

Some stores (PostgreSQL, SQLite) track a ==global ordering across all streams==. The type system handles this conditionally:

```typescript
type RecordedMessageMetadata<HasGlobalPosition = undefined> =
  CommonRecordedMessageMetadata &
    (HasGlobalPosition extends undefined ? {} : WithGlobalPosition);

type WithGlobalPosition = Readonly<{ globalPosition: GlobalPosition }>;
```

When `HasGlobalPosition` is truthy, the metadata includes `globalPosition: bigint`. Convenience aliases:

| Type | Description |
|---|---|
| `RecordedMessageMetadataWithGlobalPosition` | Always has `globalPosition` |
| `RecordedMessageMetadataWithoutGlobalPosition` | Never has `globalPosition` |

## RecordedMessage

The full recorded message type combines the original message with recorded metadata:

```typescript
type RecordedMessage<
  MessageType extends Message = Message,
  MessageMetaDataType extends AnyRecordedMessageMetadata = AnyRecordedMessageMetadata,
> = CombineMetadata<MessageType, MessageMetaDataType> & {
  kind: NonNullable<MessageKindOf<Message>>;
};
```

^recorded-message-def

> [!warning] `kind` becomes required
> A `RecordedMessage` always has `kind` set as a **required** (non-optional) field. This differs from raw [[Events|events]] and [[Commands|commands]], where `kind` is optional (`kind?: 'Event'` / `kind?: 'Command'`).

## ReadEvent (Alias for RecordedMessage)

`ReadEvent` is simply an alias for `RecordedMessage` typed specifically for events:

```typescript
type ReadEvent<
  EventType extends Event = Event,
  EventMetaDataType extends AnyRecordedMessageMetadata = AnyRecordedMessageMetadata,
> = RecordedMessage<EventType, EventMetaDataType>;
```

^read-event-def

> [!note] Naming evolution
> The codebase evolved from event-centric to message-centric naming. `ReadEvent` remains as a convenience alias used widely in the [[Event Store Interface|event store]] API. Additional event-specific aliases also exist:
> - `AnyReadEvent<MetaData>` -- loosened read event type
> - `CommonReadEventMetadata` -- identical to `CommonRecordedMessageMetadata`
> - `ReadEventMetadata<HasGlobalPosition>` -- identical to `RecordedMessageMetadata<HasGlobalPosition>`
> - `ReadEventMetadataWithGlobalPosition` / `ReadEventMetadataWithoutGlobalPosition`

## Metadata Merging

When an event with custom metadata is read from the store, its metadata is **intersected** with `RecordedMessageMetadata`. The result contains both the custom fields and the system fields:

```typescript
// Original event metadata: { userId: string }
// After recording, metadata becomes:
// { userId: string } & { messageId: string; streamPosition: bigint; streamName: string; ... }
```

^metadata-merging

If the original event had no custom metadata, only the recorded metadata is present.

> [!tip] Plan for merged metadata in handlers
> When writing [[Message Handlers|handlers]] that consume recorded events, remember that the `metadata` object will contain both your custom fields and system fields like `messageId` and `streamPosition`. The [[Evolve Function]] can accept `ReadEvent` with metadata if your state reducer needs access to stream position or other system metadata.

## See Also

- [[Events]] -- Raw event types before persistence
- [[Event Metadata]] -- System-assigned metadata details (positions, IDs)
- [[Checkpointing]] -- How `ProcessorCheckpoint` in metadata enables processor resumption
- [[Messages]] -- The parent union type
