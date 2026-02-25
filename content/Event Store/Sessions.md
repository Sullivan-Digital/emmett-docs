---
tags:
  - event-store
aliases:
  - EventStoreSession
  - withSession
related:
  - "[[Event Store Interface]]"
  - "[[Command Handling]]"
  - "[[PostgreSQL Event Store]]"
package: emmett
---

# Sessions

Some event store implementations (like PostgreSQL) support transactional sessions. Emmett provides a session abstraction that wraps an event store instance with a `close` callback.

## Type Definitions

```typescript
type EventStoreSession<EventStoreType extends EventStore> = {
  eventStore: EventStoreType;
  close: () => Promise<void>;
};

interface EventStoreSessionFactory<EventStoreType extends EventStore> {
  withSession<T = unknown>(
    callback: (session: EventStoreSession<EventStoreType>) => Promise<T>,
  ): Promise<T>;
}
```
^session-types

## Checking Session Support

Use ==`canCreateEventStoreSession()`== to check if a store supports sessions:

```typescript
import { canCreateEventStoreSession } from '@event-driven-io/emmett';

if (canCreateEventStoreSession(eventStore)) {
  // eventStore is an EventStoreSessionFactory
  await eventStore.withSession(async (session) => {
    await session.eventStore.appendToStream(streamName, events);
    // session.close() is called automatically when the callback completes
  });
}
```

> [!note]
> `canCreateEventStoreSession` uses duck-typing internally -- it checks for the presence of a `withSession` property. This allows stores to optionally implement session support without requiring all stores to implement the `EventStoreSessionFactory` interface.

## nulloSessionFactory

For stores that don't natively support sessions (like the [[In-Memory Event Store]]), ==`nulloSessionFactory`== wraps any store in a session-based API with a no-op `close()`:

```typescript
import { nulloSessionFactory } from '@event-driven-io/emmett';

const sessionFactory = nulloSessionFactory(eventStore);
await sessionFactory.withSession(async (session) => {
  // Works the same, but close() is a no-op
  await session.eventStore.appendToStream(streamName, events);
});
```

This is useful when code expects a session-based API but the underlying store doesn't support transactions.

## See Also

- [[Event Store Interface]] -- The base interface that sessions wrap
- [[Command Handling]] -- May use sessions for transactional command handling
- [[PostgreSQL Event Store]] -- Primary adapter that natively supports sessions
