---
tags:
  - projections
  - postgresql
aliases:
  - Projection Registration
  - activateProjection
related:
  - "[[PostgreSQL Distributed Locking]]"
  - "[[Projection Rebuilding]]"
  - "[[PostgreSQL Schema]]"
package: emmett-postgresql
---

# Projection Management

> [!abstract]
> PostgreSQL stores projection metadata in an ==`emt_projections`== table, enabling runtime activate/deactivate control and metadata queries. This is a PostgreSQL-only feature.

## The `emt_projections` Table

PostgreSQL automatically manages projection metadata when using `postgreSQLProjection()` or any Pongo/raw SQL projection factory. The table stores:

| Column | Type | Description |
|---|---|---|
| `name` | `text` | Projection name |
| `partition` | `text` | Partition identifier |
| `version` | `integer` | Projection version |
| `type` | `char(1)` | `'i'` (inline) or `'a'` (async) |
| `kind` | `text` | Projection kind tag |
| `status` | `text` | `'active'` or `'inactive'` |
| `definition` | `jsonb` | Projection definition metadata |
| `created_at` | `timestamp` | Creation timestamp |
| `last_updated` | `timestamp` | Last modification timestamp |

Projections are registered automatically during `schema.migrate()` when `postgreSQLProjection()` wraps the `init()` function to call `registerProjection()`.

## Management API

```typescript
import {
  activateProjection,
  deactivateProjection,
  readProjectionInfo,
} from '@event-driven-io/emmett-postgresql';
```

### Deactivate a Projection

Deactivating a projection causes it to be ==skipped during event processing==:

```typescript
await deactivateProjection(execute, {
  name: 'shoppingCartShortInfo',
  partition: 'default',
  version: 1,
});
```

When a projection is deactivated, PostgreSQL's `handleProjections` checks its status and skips it. This lets you pause a projection without removing it from the registration.

### Reactivate a Projection

```typescript
await activateProjection(execute, {
  name: 'shoppingCartShortInfo',
  partition: 'default',
  version: 1,
});
```

### Read Projection Metadata

```typescript
const info = await readProjectionInfo(execute, {
  name: 'shoppingCartShortInfo',
  partition: 'default',
  version: 1,
});
```

Returns the full projection record from `emt_projections`, including status, kind, definition, and timestamps.

## Advisory Locking

PostgreSQL inline projections use ==shared advisory locks== (`pg_try_advisory_xact_lock_shared`) for coordination. Before processing events, the system:

1. Generates a lock key by hashing `{partition}:{projectionName}:{version}` to a bigint
2. Attempts to acquire the shared advisory lock
3. If the lock cannot be acquired, the projection is ==silently skipped== for that transaction
4. The lock is automatically released when the transaction commits

> [!note]
> Advisory locks are transaction-scoped. They protect against concurrent inline projection processing but do not prevent multiple consumers from running async projections. Use [[PostgreSQL Distributed Locking|processor-level locking]] for async projection coordination.

> [!warning]
> Projections without a `name` skip locking entirely. Always provide a name for PostgreSQL projections.

## Adapter Availability

| Feature | PostgreSQL | SQLite | MongoDB | InMemory | EventStoreDB |
|---|---|---|---|---|---|
| Management table | Yes | No | No | No | No |
| Activate/deactivate | Yes | No | No | No | No |
| Advisory locking | Yes | No | No | No | No |
| `readProjectionInfo` | Yes | No | No | No | No |

## See Also

- [[Projection Rebuilding]] -- Rebuilding projections (often paired with deactivate/reactivate)
- [[PostgreSQL Distributed Locking]] -- The advisory lock mechanism
- [[PostgreSQL Schema]] -- Database schema including `emt_projections`
- [[Inline Projections Overview]] -- How inline projections use management data
