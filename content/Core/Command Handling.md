---
tags:
  - core
  - pattern
  - command-handling
aliases:
  - CommandHandler
  - DeciderCommandHandler
related:
  - "[[The Decider Pattern]]"
  - "[[Concurrency Control]]"
  - "[[Retry Logic]]"
  - "[[Schema Versioning]]"
  - "[[Middleware Pattern]]"
  - "[[Event Store Interface]]"
package: emmett
---

# Command Handling

Command handling in Emmett wires the [[The Decider Pattern|Decider pattern]] to an [[Event Store Interface|event store]], handling stream reading, state reconstruction, event appending, and optional retry logic. There are two main entry points:

- ==`CommandHandler`== -- Low-level, flexible handler where you provide a function that takes state and returns events
- ==`DeciderCommandHandler`== -- Higher-level wrapper that integrates directly with the Decider pattern

## CommandHandler

`CommandHandler` is a curried factory function. You first configure it with options (evolve, initialState, etc.), then call the returned function with a store, stream ID, handler function(s), and optional per-call options.

### Basic Usage

```typescript
import { CommandHandler, IllegalStateError } from '@event-driven-io/emmett';

// Create the handler
const handleCommand = CommandHandler<ShoppingCart, ShoppingCartEvent>({
  evolve,
  initialState,
});

// Use it with an event store
const result = await handleCommand(
  eventStore,
  'shopping-cart-123',
  (state) => {
    if (state.status === 'closed') {
      throw new IllegalStateError('Cart is already closed');
    }
    return { type: 'ItemAdded', data: { item: 'Shoes' } };
  },
);
```

### Options

```typescript
type CommandHandlerOptions<State, StreamEvent, StoredEvent = StreamEvent> = {
  evolve: (state: State, event: StreamEvent) => State;
  initialState: () => State;
  mapToStreamId?: (id: string) => string;
  retry?: CommandHandlerRetryOptions;
  schema?: {
    versioning?: {
      upcast?: (event: StoredEvent) => StreamEvent;
      downcast?: (event: StreamEvent) => StoredEvent;
    };
  };
};
```

**`mapToStreamId`** transforms the raw entity ID into a stream name:

```typescript
const handleCommand = CommandHandler<ShoppingCart, ShoppingCartEvent>({
  evolve,
  initialState,
  mapToStreamId: (id) => `shopping_cart-${id}`,
});

// Now pass just the ID -- internally reads/appends to "shopping_cart-123"
await handleCommand(eventStore, '123', handler);
```

### Result

The handler returns a `CommandHandlerResult`:

```typescript
type CommandHandlerResult<State, StreamEvent, Store> = {
  newState: State;                    // Aggregate state after all events
  newEvents: StreamEvent[];           // Events produced by the handler(s)
  nextExpectedStreamVersion: bigint;  // For optimistic concurrency on next call
  createdNewStream: boolean;          // True if stream didn't exist before
};
```

^command-handler-result

When the handler produces no events (no-op), `newEvents` is `[]`, `createdNewStream` is `false`, and the state is returned unchanged.

### Multiple Handlers (Atomic Composition)

You can pass an **array of handler functions**. Each handler receives the state updated by previous handlers' events. All produced events are appended atomically in a single operation:

```typescript
const result = await handleCommand(
  eventStore,
  'shopping-cart-123',
  [
    (state) => ({ type: 'ItemAdded', data: { item: 'Shoes' } }),
    (state) => {
      // This handler sees the state WITH 'Shoes' already added
      if (state.items.length >= 1) {
        return { type: 'CartClosed', data: {} };
      }
      return [];
    },
  ],
);
// result.newEvents contains both ItemAdded and CartClosed
```

> [!tip] Use atomic composition for multi-step operations
> This pattern is useful when later decisions depend on earlier ones, and all events should be appended in a single stream write.

## DeciderCommandHandler

`DeciderCommandHandler` wraps `CommandHandler` for use with the [[The Decider Pattern|Decider pattern]]. Instead of passing raw handler functions, you pass command(s) and the Decider's `decide` function is used automatically:

```typescript
import { DeciderCommandHandler } from '@event-driven-io/emmett';

const handleCommand = DeciderCommandHandler({
  decide,
  evolve,
  initialState,
  mapToStreamId: (id) => `shopping_cart-${id}`,
});

// Handle a single command
await handleCommand(eventStore, 'cart-123', {
  type: 'AddItem',
  data: { item: 'Hat' },
});

// Handle multiple commands atomically
await handleCommand(eventStore, 'cart-123', [
  { type: 'AddItem', data: { item: 'Hat' } },
  { type: 'CloseCart', data: {} },
]);
```

^decider-command-handler-usage

Internally, each command is mapped to `(state) => decide(command, state)` and executed sequentially, with intermediate state updated after each -- just like the array form of `CommandHandler`.

## Retry Logic

### Version Conflict Retries

In event sourcing, concurrent writes to the same stream can cause version conflicts. The command handler can automatically retry the entire operation (re-read state, re-run the handler, re-append) when this happens:

```typescript
const handleCommand = CommandHandler<ShoppingCart, ShoppingCartEvent>({
  evolve,
  initialState,
  retry: { onVersionConflict: true },
});
```

The `onVersionConflict` option has three forms:

```typescript
// Use defaults: 3 retries, 100ms min timeout, 1.5 exponential factor
retry: { onVersionConflict: true }

// Custom number of retries, other defaults unchanged
retry: { onVersionConflict: 5 }

// Full control over retry behavior
retry: {
  onVersionConflict: {
    retries: 3,
    minTimeout: 100,
    factor: 1.5,
  }
}
```

> [!note] Only version conflicts trigger retries by default
> By default, only `ExpectedVersionConflictError` triggers a retry. All other errors bail immediately. See [[Retry Logic]] for custom retry options and the underlying `asyncRetry` utility.

### Handle Options

Per-call options can override the configured defaults:

```typescript
await handleCommand(
  eventStore,
  'cart-123',
  (state) => ({ type: 'ItemAdded', data: { item: 'Shoes' } }),
  {
    expectedStreamVersion: 5n,       // Explicit expected version
    retry: { onVersionConflict: 10 }, // Per-call retry override
  },
);
```

`HandleOptions` also includes any store-specific append options (e.g., PostgreSQL-specific options), merged via intersection types.

## Schema Versioning

The command handler supports event schema evolution through upcasting and downcasting. See [[Schema Versioning]] for the full pattern:

```typescript
const handleCommand = CommandHandler<ShoppingCart, DomainEvent, StoredEvent>({
  evolve,
  initialState,
  schema: {
    versioning: {
      upcast: (stored) => ({ type: 'ItemAdded', data: { item: stored.data.productName } }),
      downcast: (domain) => ({ type: 'ItemAdded_v1', data: { productName: domain.data.item } }),
    },
  },
});
```

## How It Works Internally

The full flow of `CommandHandler` when called:

1. **Session** -- Opens an event store session if the store supports it, otherwise uses a no-op session wrapper
2. **Aggregate** -- Calls `eventStore.aggregateStream()` with [[Evolve Function|evolve]] and `initialState` to rebuild current state. If upcasting is configured, stored events are transformed first
3. **Execute handlers** -- Runs the handler function(s) sequentially, updating state after each by reducing new events through `evolve`
4. **Early return on no-op** -- If no events were produced, returns immediately without appending
5. **Determine expected version** -- Uses explicit option if provided, otherwise the current stream version from aggregation, or `STREAM_DOES_NOT_EXIST` for new streams
6. **Append** -- Appends all produced events to the stream atomically
7. **Retry** -- The entire operation (steps 1-6) is wrapped in `asyncRetry` for automatic retry on version conflicts

> [!info] Middleware pattern
> Since `CommandHandler` returns a plain async function, you can wrap it to add cross-cutting concerns like authorization, logging, or metrics. See [[Middleware Pattern]] for examples.

## See Also

- [[The Decider Pattern]] -- The `decide`/`evolve`/`initialState` structure that `DeciderCommandHandler` consumes
- [[Concurrency Control]] -- Expected version semantics and `ExpectedVersionConflictError`
- [[Retry Logic]] -- The underlying `asyncRetry` utility
- [[Schema Versioning]] -- Upcasting and downcasting for event schema evolution
- [[Middleware Pattern]] -- Wrapping command handlers for cross-cutting concerns
- [[Event Store Interface]] -- The `EventStore` that command handlers interact with
