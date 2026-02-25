---
tags:
  - core
  - event-store
  - interface
aliases:
  - EventStore Interface
related:
  - "[[Reading Streams]]"
  - "[[Appending Events]]"
  - "[[Aggregating Streams]]"
  - "[[In-Memory Event Store]]"
  - "[[Adapters MOC]]"
package: emmett
---

# Event Store Interface

The ==`EventStore`== interface is the common abstraction that all Emmett database adapters implement. It defines four generic methods for interacting with event streams.

## Interface Definition

```typescript
interface EventStore<
  ReadEventMetadataType extends AnyReadEventMetadata = AnyReadEventMetadata,
> {
  aggregateStream<State, EventType extends Event, EventPayloadType extends Event = EventType>(
    streamName: string,
    options: AggregateStreamOptions<State, EventType, ReadEventMetadataType, EventPayloadType>,
  ): Promise<AggregateStreamResult<State>>;

  readStream<EventType extends Event, EventPayloadType extends Event = EventType>(
    streamName: string,
    options?: ReadStreamOptions<EventType, EventPayloadType>,
  ): Promise<ReadStreamResult<EventType, ReadEventMetadataType>>;

  appendToStream<EventType extends Event, EventPayloadType extends Event = EventType>(
    streamName: string,
    events: EventType[],
    options?: AppendToStreamOptions<EventType, EventPayloadType>,
  ): Promise<AppendToStreamResult>;

  streamExists(streamName: string): Promise<StreamExistsResult>;
}
```
^eventstore-interface-def

The interface is generic on `ReadEventMetadataType`, which lets each backend define what metadata is available on read events. For example, the [[In-Memory Event Store]] uses `ReadEventMetadataWithGlobalPosition`, which includes a `globalPosition` field. The `EventPayloadType` generic parameter supports [[Schema Versioning]], where the stored event type can differ from the application event type.

All methods are async and return Promises.

## The Four Methods

| Method | Purpose | Details |
|---|---|---|
| [[Reading Streams\|readStream]] | Read events from a specific stream | Supports filtering with `from`, `to`, `maxCount` |
| [[Appending Events\|appendToStream]] | Append one or more events to a stream | Atomic writes, auto-creates stream if needed |
| [[Aggregating Streams\|aggregateStream]] | Fold events into state via evolve/initialState | Primary way to rebuild entity state |
| `streamExists` | Check whether a stream has events | Returns `boolean`; empty streams are non-existent |

### streamExists

The simplest method -- checks whether a stream exists and has at least one event:

```typescript
const exists = await eventStore.streamExists('shoppingCart-123');
// true if the stream has at least one event, false otherwise
```

> [!warning]
> A stream with zero events is considered non-existent, even if the stream key was previously used.

## Creating an Event Store

The simplest way to get started is with the [[In-Memory Event Store]]:

```typescript
import { getInMemoryEventStore } from '@event-driven-io/emmett';

const eventStore = getInMemoryEventStore();
```

For production use, choose a database-backed implementation:

```typescript
// PostgreSQL
import { getPostgreSQLEventStore } from '@event-driven-io/emmett-postgresql';

// MongoDB
import { getMongoDBEventStore } from '@event-driven-io/emmett-mongodb';

// SQLite
import { getSQLiteEventStore } from '@event-driven-io/emmett-sqlite';

// EventStoreDB
import { getEventStoreDBEventStore } from '@event-driven-io/emmett-esdb';
```

All of these return an object implementing the `EventStore` interface, so the API usage is identical across backends. See [[Adapters MOC]] for adapter-specific details.

## Utility Types

Emmett exports several type helpers for extracting types from an event store instance:

```typescript
// Extract the read event metadata type from a store
type EventStoreReadEventMetadata<Store extends EventStore>;

// Extract the aggregate stream result type from a store
type AggregateStreamResultOfEventStore<Store extends EventStore>;

// Extract the append stream result type from a store
type AppendStreamResultOfEventStore<Store extends EventStore>;

// Aggregate result that includes global position (for stores that support it)
type AggregateStreamResultWithGlobalPosition<State>;

// Append result that includes global position
type AppendToStreamResultWithGlobalPosition;
```

## See Also

- [[Event Store MOC]] -- Overview of the event store subsystem
- [[Concurrency Control]] -- Optimistic concurrency via `expectedStreamVersion`
- [[Event Metadata]] -- System-assigned metadata on read events
- [[Schema Versioning]] -- Upcasting and downcasting event schemas
- [[Command Handling]] -- Wiring Deciders to the event store
