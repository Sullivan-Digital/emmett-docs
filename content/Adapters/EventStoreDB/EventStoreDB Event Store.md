---
tags:
  - adapter
  - eventstoredb
aliases:
  - getEventStoreDBEventStore
  - EventStoreDBEventStore
related:
  - "[[Event Store Interface]]"
  - "[[Concurrency Control]]"
  - "[[EventStoreDB Event Mapping]]"
package: emmett-esdb
---

# EventStoreDB Event Store

The EventStoreDB event store is created via ==`getEventStoreDBEventStore()`==, which takes an `EventStoreDBClient` instance and returns an [[Event Store Interface]] implementation. It is the simplest factory function among all adapters -- no driver selection, no schema management, no configuration options.

## Quick Start

```typescript
import { getEventStoreDBEventStore } from '@event-driven-io/emmett-esdb';
import { EventStoreDBClient } from '@eventstore/db-client';
import type { Event } from '@event-driven-io/emmett';

type GuestCheckedIn = Event<'GuestCheckedIn', { guestId: string }>;
type GuestCheckedOut = Event<'GuestCheckedOut', { guestId: string }>;
type GuestStayEvent = GuestCheckedIn | GuestCheckedOut;

// Connect to EventStoreDB
const client = EventStoreDBClient.connectionString(
  'esdb://localhost:2113?tls=false',
);

// Create the Emmett event store
const eventStore = getEventStoreDBEventStore(client);

// Append events
const result = await eventStore.appendToStream<GuestStayEvent>(
  'guestStay-guest42',
  [{ type: 'GuestCheckedIn', data: { guestId: 'guest42' } }],
);

// Read events
const { events, currentStreamVersion } =
  await eventStore.readStream('guestStay-guest42');
```

## Event Store Setup

```typescript
const client = EventStoreDBClient.connectionString(
  'esdb://admin:changeit@localhost:2113?tls=false',
);

const eventStore = getEventStoreDBEventStore(client);
```

> [!warning]
> There is ==no `close()` method== on the event store. You are responsible for managing the `EventStoreDBClient` lifecycle. There is also no schema management -- EventStoreDB handles storage internally.

## Core Operations

### Appending Events

```typescript
const result = await eventStore.appendToStream<GuestStayEvent>(
  'guestStay-guest42',
  [
    { type: 'GuestCheckedIn', data: { guestId: 'guest42' } },
    { type: 'GuestCheckedOut', data: { guestId: 'guest42' } },
  ],
);
// result: { nextExpectedStreamVersion, lastEventGlobalPosition, createdNewStream }
```

With optimistic concurrency:

```typescript
const result2 = await eventStore.appendToStream<GuestStayEvent>(
  'guestStay-guest42',
  [{ type: 'GuestCheckedOut', data: { guestId: 'guest42' } }],
  { expectedStreamVersion: result.nextExpectedStreamVersion },
);
```

Internally, `appendToStream`:
1. Downcasts messages if `schema.versioning` is provided
2. Converts to ESDB format via `jsonEvent()`
3. Maps expected version to ESDB's `AppendExpectedRevision`
4. Calls `client.appendToStream()`
5. Maps `WrongExpectedVersionError` to `ExpectedVersionConflictError`

> [!warning]
> The adapter uses `appendResult.position!.commit` with a ==non-null assertion==. In rare edge cases where EventStoreDB doesn't return position information, this could throw.

### Reading a Stream

```typescript
const { events, currentStreamVersion, streamExists } =
  await eventStore.readStream('guestStay-guest42');

// With options
const { events: subset } = await eventStore.readStream('guestStay-guest42', {
  from: 2n,      // start from revision 2
  maxCount: 10,
});
```

If the stream does not exist, the result has `events: []` and `streamExists: false`. Internally, `StreamNotFoundError` is caught and mapped to this empty result.

### Aggregating a Stream

```typescript
const { state, currentStreamVersion, streamExists } =
  await eventStore.aggregateStream<GuestStayState, GuestStayEvent>(
    'guestStay-guest42',
    {
      evolve: (state, event) => {
        switch (event.type) {
          case 'GuestCheckedIn':
            return { guestId: event.data.guestId, status: 'checked-in' };
          case 'GuestCheckedOut':
            return { ...state, status: 'checked-out' };
        }
      },
      initialState: () => ({ guestId: '', status: 'checked-in' }),
    },
  );
```

The result also includes `lastEventGlobalPosition` when the stream exists.

### Checking Stream Existence

```typescript
const exists = await eventStore.streamExists('guestStay-guest42');
```

Internally, this attempts to read from the stream and returns `false` if a `StreamNotFoundError` occurs.

## Concurrency Control

EventStoreDB fully supports all of Emmett's expected version constants, unlike [[SQLite Event Store|SQLite]] where `STREAM_DOES_NOT_EXIST` and `STREAM_EXISTS` are not enforced:

| Emmett | EventStoreDB |
|---|---|
| `undefined` | `ANY` |
| `NO_CONCURRENCY_CHECK` | `ANY` |
| `STREAM_DOES_NOT_EXIST` | `NO_STREAM` |
| `STREAM_EXISTS` | `STREAM_EXISTS` |
| `bigint` value | Passed through directly |

```typescript
import {
  STREAM_DOES_NOT_EXIST,
  NO_CONCURRENCY_CHECK,
  STREAM_EXISTS,
  ExpectedVersionConflictError,
} from '@event-driven-io/emmett';

// Only succeed if the stream does not yet exist
await eventStore.appendToStream(streamName, events, {
  expectedStreamVersion: STREAM_DOES_NOT_EXIST,
});

// Only succeed if the stream already exists (any version)
await eventStore.appendToStream(streamName, events, {
  expectedStreamVersion: STREAM_EXISTS,
});

// Conflict results in ExpectedVersionConflictError
try {
  await eventStore.appendToStream(streamName, events, {
    expectedStreamVersion: 5n,
  });
} catch (error) {
  if (error instanceof ExpectedVersionConflictError) {
    // error.current -- the actual version
    // error.expected -- what you expected
  }
}
```

## Event Versioning

### Upcasting (Read-Time)

Transform events from their stored format to the current application schema when reading:

```typescript
const upcast = (event: Event): CurrentEventType => {
  if (event.type === 'OrderPlaced' && !('currency' in event.data)) {
    return {
      ...event,
      data: { ...event.data, currency: 'USD' },
    };
  }
  return event as CurrentEventType;
};

const { state } = await eventStore.aggregateStream(streamName, {
  evolve: myEvolve,
  initialState: () => myInitialState,
  read: { schema: { versioning: { upcast } } },
});
```

### Downcasting (Write-Time)

Transform events before writing. This is ==unique to the ESDB adapter== among Emmett adapters:

```typescript
const downcast = (event: CurrentEventType): StoredEventType => {
  // Convert to the format stored in EventStoreDB
  return transformedEvent;
};

await eventStore.appendToStream(streamName, events, {
  schema: { versioning: { downcast } },
});
```

> [!info]
> See [[Schema Versioning]] for a full discussion of upcasting and downcasting patterns.

## Default Stream Version

The EventStoreDB adapter uses ==`-1n`== for non-existing streams (`EventStoreDBEventStoreDefaultStreamVersion`). This is ESDB's native convention and differs from [[SQLite Event Store|SQLite]] and [[PostgreSQL Event Store|PostgreSQL]] which use `0n`.

## Key Types

```typescript
const EventStoreDBEventStoreDefaultStreamVersion = -1n;

type EventStoreDBReadEventMetadata = ReadEventMetadataWithGlobalPosition;
// Includes: messageId, streamName, streamPosition, globalPosition, checkpoint

type EventStoreDBReadEvent<EventType extends Event = Event> =
  ReadEvent<EventType, EventStoreDBReadEventMetadata>;
```

## Gotchas

> [!warning] Caveats
> - **Default stream version is `-1n`**, not `0n` like other adapters
> - **No `close()` on the event store** -- caller manages client lifecycle
> - **No inline projections** -- read models must be built via the [[EventStoreDB Consumer|consumer]]
> - **No workflow processor** -- use PostgreSQL or SQLite for orchestration
> - **Position non-null assertion** in `appendToStream` could throw in edge cases
>
> See [[Gotchas]] for the full list.
