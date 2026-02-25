---
tags:
  - event-store
  - hooks
aliases:
  - onBeforeCommit
  - beforeCommitHook
related:
  - "[[After-Commit Hooks]]"
  - "[[Inline Projections]]"
  - "[[Appending Events]]"
package: emmett
---

# Before-Commit Hooks

Before-commit hooks fire inside the database transaction, ==before the commit==. If a before-commit hook fails, the entire transaction -- including the event append -- is rolled back. This makes them fundamentally different from [[After-Commit Hooks|after-commit hooks]].

## Comparison with After-Commit Hooks

| Property | After-Commit | Before-Commit |
|----------|-------------|---------------|
| Timing | After successful append | Inside the transaction, before commit |
| On failure | Append still succeeds | Transaction rolls back |
| Atomicity | Not atomic with append | ==Atomic with append== |
| Use case | Notifications, message bus forwarding | Inline projections, atomic side effects |

## Handler Type

```typescript
type BeforeEventStoreCommitHandler<
  Store extends EventStore,
  HandlerContext extends DefaultRecord | undefined = undefined,
>
```

> [!note]
> `BeforeEventStoreCommitHandler` and `AfterEventStoreCommitHandler` have ==identical type signatures==. The naming difference is purely semantic to clarify when the hook runs.

## SQLite Before-Commit Hook

The SQLite adapter exposes `onBeforeCommit` to users. It receives the committed events plus a connection context:

```typescript
import { getSQLiteEventStore } from '@event-driven-io/emmett-sqlite';

const eventStore = getSQLiteEventStore(connection, {
  hooks: {
    onBeforeCommit: (events, { connection }) => {
      // This runs inside the SQLite transaction
      // If this throws, the entire append is rolled back
    },
  },
});
```

The context provides the SQLite connection so your hook can execute additional SQL statements within the same transaction.

> [!warning]
> If you also use [[Inline Projections|inline projections]] with SQLite, the adapter chains the internal projection handler ==before== your `onBeforeCommit` hook. Both run within the same transaction.

## PostgreSQL Internal Hook

The PostgreSQL adapter uses an internal `beforeCommitHook` for [[Inline Projections|inline projections]]. This hook is ==not exposed to users== as a configuration option -- it is managed internally by the adapter's projection system and runs within the PostgreSQL transaction.

```typescript
// PostgreSQL-specific internal type (not exported from emmett core)
type AppendToStreamBeforeCommitHook = (
  messages: RecordedMessage[],
  context: { transaction: PgTransaction },
) => Promise<void>;
```

## Adapter Support

| Adapter | `onBeforeCommit` | Notes |
|---------|:----------------:|-------|
| InMemory | -- | No transactions; uses [[After-Commit Hooks|onAfterCommit]] instead |
| MongoDB | -- | No transactions; uses [[After-Commit Hooks|onAfterCommit]] instead |
| PostgreSQL | Internal only | Used for inline projections; not user-configurable |
| SQLite | ==Yes== | User-configurable; runs in transaction |
| EventStoreDB | -- | No hooks |

## See Also

- [[After-Commit Hooks]] -- Fire-and-forget hooks for non-critical side effects
- [[Inline Projections]] -- The primary consumer of before-commit hooks
- [[Appending Events]] -- The execution order within `appendToStream`
- [[SQLite Event Store]] -- The adapter that exposes this hook to users
