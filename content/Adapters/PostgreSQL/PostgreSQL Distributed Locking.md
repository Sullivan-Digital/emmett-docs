---
tags:
  - adapter
  - postgresql
  - locking
aliases:
  - Advisory Locks
  - Lock Acquisition Policy
related:
  - "[[PostgreSQL Consumer]]"
  - "[[Projection Management]]"
  - "[[Projection Rebuilding]]"
package: emmett-postgresql
---

# PostgreSQL Distributed Locking

The PostgreSQL adapter uses ==PostgreSQL advisory locks== to coordinate processors across multiple application instances. There are two locking mechanisms: processor locks (exclusive) and projection locks (shared).

## Processor Locks

Processor locks ensure only one instance of a processor runs at a time across multiple application instances. This uses `pg_try_advisory_xact_lock` (non-blocking, transaction-scoped).

```typescript
consumer.projector({
  projection: myProjection,
  lock: {
    acquisitionPolicy: { type: 'fail' },  // throw on failure (default)
    timeoutSeconds: 300,                   // 5 minutes (default)
  },
});
```

### Lock Key Generation

The lock key is derived from the processor's identity and hashed to a bigint for the advisory lock:

- **Projectors:** `{partition}:{projectionName}:{version}` hashed to bigint
- **Reactors/Workflows:** `{partition}:{processorId}:{version}` hashed to bigint

> [!note]
> The hash function (`hashText`) uses SHA-256, taking the first 8 bytes as a signed 64-bit integer. This can produce negative values, which is valid for PostgreSQL advisory locks. See [[Hashing]] for details.

### Lock Acquisition Flow

When a processor starts, the `emt_try_acquire_processor_lock` SQL function:

1. Attempts `pg_try_advisory_xact_lock(key)` (non-blocking)
2. If acquired, upserts into `emt_processors` with status `'running'`
3. The upsert `ON CONFLICT` only succeeds if:
   - Same instance ID (re-acquiring own lock), OR
   - Instance is `'emt:unknown'` (previously released), OR
   - Status is `'stopped'`, OR
   - `last_updated` is older than `lock_timeout_seconds` (stale lock)
4. If a projection name is provided, also sets `emt_projections` status to `'async_processing'`
5. Returns the `last_processed_checkpoint` for the processor to resume from

When a processor stops, `emt_release_processor_lock`:

1. Sets projection status back to `'active'` (if applicable)
2. Sets processor status to `'stopped'` and instance ID to `'emt:unknown'`
3. Calls `pg_advisory_unlock(lock_key)` to release the advisory lock

## Projection Locks (Inline Coordination)

When an async processor is running (e.g., during a [[PostgreSQL Rebuilding Projections|rebuild]]), ==inline projections are automatically paused==. This coordination uses shared advisory locks:

1. The async processor acquires an **exclusive** lock and sets the projection status to `'async_processing'`
2. During `appendToStream`, inline projections attempt a **shared** lock via `pg_try_advisory_xact_lock_shared(key)` and check if the projection is `'active'`
3. If the projection is not active (because an async processor has it), the inline projection ==skips execution==

This means events appended during a rebuild will not update inline projections. The rebuild replays all events anyway, so no data is lost.

> [!warning] Events During Rebuild
> Inline projections pause during async rebuilds. Events appended during this window will not update inline projections. Once the rebuild completes and the lock is released, inline projections resume for new appends. The rebuild itself replays all historical events, so consistency is maintained.

## Lock Acquisition Policies

The `acquisitionPolicy` option controls what happens when a lock cannot be acquired:

```typescript
// Fail immediately (default) -- throws EmmettError
{ type: 'fail' }

// Silently skip if lock unavailable
{ type: 'skip' }

// Retry with exponential backoff
{ type: 'retry', retries: 10, minTimeout: 100, maxTimeout: 5000 }
```

The full type definition:

```typescript
type LockAcquisitionPolicy =
  | { type: 'fail' }
  | { type: 'skip' }
  | { type: 'retry'; retries: number; minTimeout?: number; maxTimeout?: number };
```

| Policy | Default For | Behavior |
|---|---|---|
| `fail` | Normal processors | Throws `EmmettError` if lock unavailable |
| `skip` | -- | Silently returns without processing |
| `retry` | [[PostgreSQL Rebuilding Projections\|Rebuilds]] | Retries with exponential backoff |

## Lock Timeout and Stale Recovery

If a process crashes without releasing its lock, the lock times out after `timeoutSeconds` (==default: 300 seconds / 5 minutes==). Another process instance can then take over.

Each processor instance is identified by a UUID v7 generated at startup (`processorInstanceId`). Stale lock detection works by checking the `last_updated` timestamp in the `emt_processors` table -- if it is older than the timeout, the lock is considered stale and can be acquired by a new instance.

> [!warning]
> Processor crash recovery takes up to 5 minutes by default. If a process crashes without releasing its advisory lock, another instance must wait for `timeoutSeconds` before it can take over the stale lock.

## Constants

| Constant | Value |
|---|---|
| `DefaultPostgreSQLProcessorLockPolicy` | `{ type: 'fail' }` |
| `PROCESSOR_LOCK_DEFAULT_TIMEOUT_SECONDS` | `300` (5 minutes) |
| Default rebuild lock policy | `{ type: 'retry', retries: 100, minTimeout: 100, maxTimeout: 5000 }` |

## See Also

- [[PostgreSQL Consumer]] -- Where processor locks are configured
- [[PostgreSQL Rebuilding Projections]] -- Uses aggressive retry lock policy
- [[Projection Management]] -- Projection status lifecycle (`active`, `inactive`, `async_processing`)
- [[PostgreSQL Schema]] -- The `emt_processors` and `emt_projections` tables that store lock state
- [[Hashing]] -- The `hashText` function used to generate advisory lock keys
