---
tags:
  - projections
  - inline
aliases:
  - Inline Projection Lifecycle
related:
  - "[[Projection Concepts]]"
  - "[[InMemory Projections]]"
  - "[[Pongo Document Projections]]"
  - "[[MongoDB Inline Projections]]"
  - "[[Before-Commit Hooks]]"
package: emmett
---

# Inline Projections Overview

> [!abstract]
> Inline projections execute synchronously within the same transaction as `appendToStream`, guaranteeing ==read-after-write consistency==. The read model is always up-to-date immediately after writing events.

## Registration

Inline projections are registered when creating the event store using `projections.inline()`:

```typescript
import { projections } from '@event-driven-io/emmett';

const eventStore = getEventStore({
  projections: projections.inline([
    myProjectionA,
    myProjectionB,
  ]),
});
```

This wraps each `ProjectionDefinition` in a `ProjectionRegistration` with `type: 'inline'` and passes them to the event store factory.

## Lifecycle

The inline projection lifecycle follows five steps:

### 1. Registration

Projections are passed to the event store factory via the `projections` option. The factory calls `filterProjections('inline', ...)` to extract inline projections from the registration set.

### 2. Filtering

`filterProjections` filters by type and throws an `EmmettError` if duplicate `name` values are detected within the filtered set.

### 3. Schema Initialization

During `schema.migrate()`, each projection's `init()` function is called with the appropriate context. This is where tables or collections are created.

> [!warning]
> `init()` behavior varies by adapter:
> - **PostgreSQL**: Called during `schema.migrate()` within a transaction; also registers the projection in the `emt_projections` table
> - **SQLite**: Called during migration but does not register in a management table
> - **InMemory**: Not automatically called
> - **MongoDB**: Does not support `init()`

### 4. beforeCommitHook

After events are written to the stream (but ==before the transaction commits==), a hook calls the projection handler with the newly appended events. This ensures atomicity -- if the projection fails, the entire transaction rolls back.

### 5. Handler Execution

For each projection whose `canHandle` includes at least one event type from the batch, `projection.handle(events, context)` is called with the full batch of events and an adapter-specific context object.

## Per-Adapter Wiring

Each adapter wires inline projections differently, providing adapter-specific context to the handlers:

### InMemory

```typescript
// After appendToStream:
await handleInMemoryProjections({
  projections: inlineProjections,
  events: newEvents,
  database: eventStore.database,
  eventStore,
});
```

The context provides access to the `InMemoryDatabase` and `InMemoryEventStore`. No transactions are involved -- updates happen immediately in memory.

### PostgreSQL

The PostgreSQL adapter uses a `beforeCommitHook` that runs inside the database transaction:

```typescript
const beforeCommitHook = async (events, { transaction }) =>
  handleProjections({
    projections: inlineProjections,
    events,
    ...transactionToPostgreSQLProjectionHandlerContext(connectionString, pool, transaction),
  });
```

Before processing each projection, PostgreSQL also:
- Acquires a shared advisory lock (`pg_try_advisory_xact_lock_shared`) scoped to the projection name/partition/version
- Checks the projection's `is_active` status in the [[Projection Management|management table]]

> [!note]
> If the advisory lock cannot be acquired, the projection is ==silently skipped== for that transaction -- no error is thrown.

### MongoDB

MongoDB inline projections are unique: read models are stored ==inside the event stream document== under a `projections` field.

```typescript
await handleInlineProjections({
  readModels: stream?.projections ?? {},
  streamId,
  events: eventsToAppend,
  projections: inlineProjections,
  collection,
  updates,
  client: {},
});
```

See [[MongoDB Inline Projections]] for details on this embedded architecture.

### SQLite

SQLite uses the same pattern as PostgreSQL with `handleProjections()` and a `beforeCommitHook`, but ==without advisory locks==.

## When to Use Inline Projections

Inline projections are ideal when:
- You need ==read-after-write consistency== (the read model must reflect the latest write immediately)
- The projection logic is fast and lightweight
- You want transactional guarantees (projection update succeeds or fails with the write)

For decoupled, eventually-consistent processing, use [[Async Projections]] instead.

## See Also

- [[Projection Concepts]] -- Core concepts: evolve, single-stream vs multi-stream, deletion
- [[Async Projections]] -- The alternative eventually-consistent approach
- [[Before-Commit Hooks]] -- The mechanism that powers inline projections in PostgreSQL/SQLite
- [[Appending Events]] -- Where inline projections are triggered
