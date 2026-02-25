---
tags:
  - event-store
  - metadata
aliases:
  - ReadEvent Metadata
related:
  - "[[Recorded Messages]]"
  - "[[Checkpointing]]"
  - "[[Reading Streams]]"
package: emmett
---

# Event Metadata

When events are read back from the store, they include system-assigned metadata alongside any user-provided metadata. The exact metadata shape depends on the store implementation, but all stores provide a common set of fields.

## Metadata Fields

For the [[In-Memory Event Store]], metadata includes:

```typescript
{
  messageId: string;           // UUID v4, auto-generated on append
  streamName: string;          // the stream the event was appended to
  streamPosition: bigint;      // 1-based position within the stream
  globalPosition: bigint;      // 1-based position across ALL streams
  checkpoint: string | null;   // branded string derived from globalPosition, used by processors
}
```
^inmemory-metadata-fields

> [!warning]
> Stream positions are ==1-based==: the first event in a stream has `streamPosition: 1n`. The default (empty) stream version is `0n`.

## ReadEvent Structure

A `ReadEvent` is the original event (`type`, `data`) with `kind` set to `'Event'` and `metadata` extended with the store's system fields:

```typescript
// ReadEvent is a RecordedMessage:
// CombineMetadata<MessageType, MessageMetaDataType> & { kind: NonNullable<MessageKindOf<Message>> }
```

See [[Recorded Messages]] for the full type hierarchy.

## Metadata Merging

User-provided metadata on the original event is preserved, but ==system fields take precedence== if there is a naming conflict:

```typescript
// If you append an event with metadata.streamName = 'custom-name'
const event = {
  type: 'ProductItemAdded',
  data: { productItem },
  metadata: { streamName: 'custom-name', customField: 'preserved' },
};

await eventStore.appendToStream('shoppingCart-123', [event]);

// When read back:
// event.metadata.streamName === 'shoppingCart-123' (system overwrote user value)
// event.metadata.customField === 'preserved' (user value kept)
```

The merge order is: user metadata is spread first, then system metadata (`streamName`, `messageId`, `streamPosition`, `globalPosition`, `checkpoint`) is spread on top.

## Accessing Metadata in Evolve

```typescript
import type { ReadEvent } from '@event-driven-io/emmett';

const evolveWithMetadata = (
  state: ShoppingCart,
  event: ReadEvent<ShoppingCartEvent>,
): ShoppingCart => {
  console.log(event.metadata.streamPosition); // 1n, 2n, 3n, ...
  console.log(event.metadata.streamName);     // 'shoppingCart-123'

  switch (event.type) {
    case 'ProductItemAdded':
      return { /* ... */ };
    case 'DiscountApplied':
      return { /* ... */ };
  }
};
```

The [[Aggregating Streams|aggregateStream]] method supports [[Aggregating Streams#^evolve-three-signatures|three evolve signatures]], including one that receives the full `ReadEvent` with metadata.

## Global Position

The `globalPosition` is a 1-based counter across all streams in the store. In the [[In-Memory Event Store]], it is computed on-the-fly by summing all event counts across all streams at the time of append.

The `checkpoint` field is a branded `ProcessorCheckpoint` string derived from `globalPosition` via `bigIntProcessorCheckpoint()`. It is used by async consumers and processors to track their position in the global event stream. See [[Checkpointing]] for details.

## See Also

- [[Recorded Messages]] -- The `RecordedMessage` and `ReadEvent` type hierarchy
- [[Appending Events]] -- When and how metadata is assigned
- [[Checkpointing]] -- How `checkpoint` is used by processors
- [[Reading Streams]] -- Where metadata appears on returned events
