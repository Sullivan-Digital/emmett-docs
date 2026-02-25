---
tags:
  - consumer
  - projector
aliases:
  - Projector
  - consumer.projector
related:
  - "[[Reactors]]"
  - "[[Async Projections]]"
  - "[[Projection Rebuilding]]"
  - "[[Projection Concepts]]"
package: emmett
---

# Projectors

> [!abstract]
> A projector is a specialized [[Reactors|reactor]] designed for building and maintaining read models. It wraps a `ProjectionDefinition` and adds features like `truncateOnStart` for rebuilds.

## Creating a Projector

```typescript
consumer.projector({
  projection: {
    name: 'shopping-cart-summary',
    canHandle: ['ProductItemAdded', 'ProductItemRemoved', 'ShoppingCartConfirmed'],
    handle: async (events, context) => {
      for (const event of events) {
        switch (event.type) {
          case 'ProductItemAdded':
            await context.execute(
              sql('INSERT INTO cart_items ... VALUES ...')
            );
            break;
          case 'ProductItemRemoved':
            await context.execute(
              sql('DELETE FROM cart_items WHERE ...')
            );
            break;
        }
      }
    },
  },
});
```

### How It Differs from a Reactor

| Feature | Reactor | Projector |
|---|---|---|
| Handler | `eachMessage` callback | `ProjectionDefinition` with `handle` |
| `processorId` | Must be provided | Auto-generated: `emt:processor:projector:${name}` |
| `canHandle` | Manually specified | Taken from `projection.canHandle` |
| `truncateOnStart` | Not available | Clears read model on startup |
| Internal type | `'reactor'` | `'projector'` |

> [!tip]
> The `processorId` defaults to `emt:processor:projector:${projectionName}`. You can override it if needed, but the default is usually sufficient.

## Truncate on Start

Set `truncateOnStart: true` to clear the read model before processing. This calls the projection's `truncate()` function during startup:

```typescript
consumer.projector({
  projection: {
    name: 'my-projection',
    canHandle: ['SomeEvent'],
    handle: async (events, context) => { /* ... */ },
    truncate: async (context) => {
      await context.execute(sql('TRUNCATE TABLE my_read_model'));
    },
  },
  truncateOnStart: true,
});
```

> [!note]
> The `truncate()` call happens ==before== the `onStart` lifecycle hook. This ensures the read model is clean before processing begins.

## Rebuilding Projections

The PostgreSQL adapter provides `rebuildPostgreSQLProjections`, a convenience function that creates a consumer configured specifically for projection rebuilds:

```typescript
import { rebuildPostgreSQLProjections } from '@event-driven-io/emmett-postgresql';

const consumer = rebuildPostgreSQLProjections({
  connectionString: 'postgresql://localhost:5432/mydb',
  projections: [cartSummaryProjection, orderHistoryProjection],
});

// Runs until all existing messages are processed, then stops
await consumer.start();
await consumer.close();
```

This function:
- Sets `stopWhen: { noMessagesLeft: true }` so the consumer stops after processing all existing events
- Sets `truncateOnStart: true` on every projection
- Uses a retry [[PostgreSQL Distributed Locking|lock acquisition policy]] (100 retries, 100-5000ms backoff) to coordinate with running consumers

> [!info]
> For other adapters, manually create a consumer with `truncateOnStart: true` and `startFrom: 'BEGINNING'` on your projectors. See [[Projection Rebuilding]] for the full pattern.

## See Also

- [[Reactors]] -- The base processor type that projectors build on
- [[Projection Concepts]] -- The `ProjectionDefinition` interface and evolve patterns
- [[Async Projections]] -- How projections connect to the consumer/projector model
- [[Projection Rebuilding]] -- Strategies for rebuilding projection data
- [[PostgreSQL Rebuilding Projections]] -- PostgreSQL-specific rebuild convenience function
