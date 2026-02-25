---
tags:
  - projections
  - operations
aliases:
  - rebuildPostgreSQLProjections
  - Projection Rebuild
related:
  - "[[Async Projections]]"
  - "[[Projection Management]]"
  - "[[PostgreSQL Distributed Locking]]"
package: emmett-postgresql
---

# Projection Rebuilding

> [!abstract]
> Rebuilding projections means truncating all existing projection data and replaying every event from the beginning. PostgreSQL provides a dedicated `rebuildPostgreSQLProjections()` function; other adapters use a manual consumer with `truncateOnStart: true`.

## PostgreSQL (Dedicated API)

The ==`rebuildPostgreSQLProjections`== function simplifies the rebuild process:

```typescript
import { rebuildPostgreSQLProjections } from '@event-driven-io/emmett-postgresql';

const consumer = rebuildPostgreSQLProjections({
  connectionString,
  projections: [shortInfoProjection, anotherProjection],
});

await consumer.start();
// Consumer automatically stops when all existing events are processed
```

### Key Behaviors

- Creates a consumer with `stopWhen: { noMessagesLeft: true }` -- it ==automatically stops== after processing all existing events
- Each projection gets `truncateOnStart: true` by default, clearing data before replaying
- Default lock policy: `{ type: 'retry', retries: 100, minTimeout: 100, maxTimeout: 5000 }` -- aggressive retry for acquiring [[PostgreSQL Distributed Locking|advisory locks]]
- Accepts either raw `PostgreSQLProjectionDefinition` objects or full `ProjectorOptions`

### Full Projector Options

For more control over the rebuild, pass full projector options:

```typescript
const consumer = rebuildPostgreSQLProjections({
  connectionString,
  projections: [
    {
      projection: shortInfoProjection,
      truncateOnStart: true,
      processorId: 'rebuild-short-info',
    },
  ],
});
```

> [!warning]
> `rebuildPostgreSQLProjections` is ==PostgreSQL-only==. Other adapters require manual setup.

## Other Adapters (Manual Rebuild)

For InMemory, MongoDB, SQLite, and EventStoreDB, rebuild projections by creating a consumer with `truncateOnStart: true`:

```typescript
const consumer = createConsumer({ /* adapter options */ });

consumer.projector({
  projection: myProjection,
  truncateOnStart: true,
});

await consumer.start();
```

Unlike the PostgreSQL dedicated API, the consumer does not automatically stop. You need to manage the lifecycle yourself (e.g., stopping after catching up to the current position).

## When to Rebuild

Common reasons to rebuild projections:

- **Schema change**: The read model structure has changed and existing data is incompatible
- **Bug fix**: A bug in the `evolve` function produced incorrect data
- **New projection**: Adding a new projection that needs to process historical events
- **Data corruption**: Projection data has become inconsistent

> [!tip]
> For PostgreSQL, you can [[Projection Management|deactivate]] a projection before rebuilding to prevent the inline projection from processing new events during the rebuild.

## See Also

- [[Async Projections]] -- The consumer/projector pattern that powers rebuilds
- [[Projection Management]] -- Activate/deactivate projections during rebuild
- [[PostgreSQL Distributed Locking]] -- Advisory locks used during rebuild
- [[PostgreSQL Rebuilding Projections]] -- PostgreSQL adapter-specific details
