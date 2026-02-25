---
tags:
  - adapter
  - postgresql
  - projections
aliases:
  - rebuildPostgreSQLProjections
related:
  - "[[Projection Rebuilding]]"
  - "[[PostgreSQL Consumer]]"
  - "[[PostgreSQL Distributed Locking]]"
package: emmett-postgresql
---

# PostgreSQL Rebuilding Projections

Rebuild projections by replaying all events from the beginning using ==`rebuildPostgreSQLProjections()`==. This is useful when a projection's logic changes and you need to recompute the read model from scratch.

## Usage

```typescript
import { rebuildPostgreSQLProjections } from '@event-driven-io/emmett-postgresql';

const consumer = rebuildPostgreSQLProjections({
  connectionString,
  projections: [
    shoppingCartDetailsProjection,
    shoppingCartShortInfoProjection,
  ],
  lock: {
    // Default: aggressive retry for rebuilds
    acquisitionPolicy: {
      type: 'retry',
      retries: 100,
      minTimeout: 100,
      maxTimeout: 5000,
    },
  },
  pulling: {
    batchSize: 200,  // larger batches for rebuilds
  },
});

await consumer.start();  // processes all events, then stops automatically
await consumer.close();
```

## How It Works

The rebuild process follows these steps:

1. A [[PostgreSQL Consumer|consumer]] is created with `stopWhen: { noMessagesLeft: true }`
2. For each projection, a projector is registered with `truncateOnStart: true`
3. Each projector auto-generates a `processorId` from the projection name
4. On start, the projector acquires a [[PostgreSQL Distributed Locking|distributed lock]] (with aggressive retry policy)
5. The projection's `truncate()` method is called, ==clearing all existing projection data==
6. The consumer reads all events from the beginning and processes them through the projections
7. The consumer stops automatically when there are no more events
8. Locks are released and inline projections resume

## Options

```typescript
rebuildPostgreSQLProjections<EventType>({
  connectionString: string,
  pool?: Dumbo,
  projections: (ProjectorOptions | PostgreSQLProjectionDefinition)[],
  lock?: {
    acquisitionPolicy?: LockAcquisitionPolicy,
    timeoutSeconds?: number,
  },
  pulling?: {
    batchSize?: number;
    pullingFrequencyInMs?: number;
  },
});
```

**Default lock policy for rebuilds:**

```typescript
{ type: 'retry', retries: 100, minTimeout: 100, maxTimeout: 5000 }
```

This is more aggressive than the default `{ type: 'fail' }` policy used by normal processors, because rebuilds are typically a deliberate operation that should wait for existing processors to release their locks.

## Inline Projection Coordination

> [!warning]
> While the rebuild runs, ==inline projections for those same projection names are paused==. Events appended during the rebuild window will not update inline projections. Once the rebuild completes and the lock is released, inline projections resume for new appends.

This coordination works through the [[PostgreSQL Distributed Locking|projection lock mechanism]]:

1. The rebuild's async processor acquires an exclusive advisory lock
2. The projection status is set to `'async_processing'` in `emt_projections`
3. Inline projections check this status during `appendToStream` and skip if not `'active'`
4. On rebuild completion, status is reset to `'active'`

Since the rebuild replays all events (including those appended during the rebuild), no data is lost.

## Practical Considerations

> [!tip]
> Use larger `batchSize` values for rebuilds (e.g., 200-500) to improve throughput. The default of 100 is tuned for real-time processing latency, not batch throughput.

- The rebuild returns a consumer, so you can use `await consumer.start()` and `await consumer.close()` for lifecycle management
- Multiple projections can be rebuilt in a single pass -- they all process the same event stream
- The `stopWhen: { noMessagesLeft: true }` option means the consumer processes all events and then automatically stops

## See Also

- [[Projection Rebuilding]] -- General rebuilding concept across adapters
- [[PostgreSQL Consumer]] -- The consumer used internally by the rebuild
- [[PostgreSQL Distributed Locking]] -- Lock coordination during rebuilds
- [[PostgreSQL Projections]] -- The projection types that can be rebuilt
