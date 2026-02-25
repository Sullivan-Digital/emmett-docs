---
tags:
  - pattern
aliases:
  - Command Handler Middleware
related:
  - "[[Command Handling]]"
package: emmett
---

# Middleware Pattern

Since [[Command Handling|CommandHandler]] returns a plain async function, you can wrap it with additional logic for cross-cutting concerns like authorization, logging, or metrics. This is Emmett's approach to middleware -- simple function composition rather than a framework-level middleware chain.

## Authorization Example

```typescript
import { CommandHandler, type HandleOptions, type EventStore } from '@event-driven-io/emmett';

const rawCommandHandler = CommandHandler<ShoppingCart, ShoppingCartEvent>({
  evolve,
  initialState,
});

// Wrap with authorization middleware
const handleCommand = async <Store extends EventStore>(
  store: Store,
  id: string,
  decide: (state: ShoppingCart) => ShoppingCartEvent | ShoppingCartEvent[],
  options: HandleOptions<Store> & { requestHeaders: RequestHeaders },
) =>
  rawCommandHandler(
    store,
    id,
    async (state: ShoppingCart) => {
      await authorize(options.requestHeaders); // middleware logic
      return decide(state);
    },
    options,
  );
```

The wrapper intercepts the handler function to inject authorization before the business logic runs. If `authorize` throws, no events are produced.

## Composing Multiple Concerns

You can layer multiple wrappers:

```typescript
// Base handler
const baseHandler = CommandHandler<State, Event>({ evolve, initialState });

// Add logging
const withLogging = async (store, id, handler, options) => {
  console.log(`Handling command for ${id}`);
  const result = await baseHandler(store, id, handler, options);
  console.log(`Produced ${result.newEvents.length} events`);
  return result;
};

// Add metrics on top
const withMetrics = async (store, id, handler, options) => {
  const start = Date.now();
  const result = await withLogging(store, id, handler, options);
  metrics.record('command_duration', Date.now() - start);
  return result;
};
```

> [!tip] This pattern is simple and explicit. Each wrapper is a plain function with full type safety. There is no plugin system, middleware chain, or registration API -- just function composition.

## When to Use

- **Authorization** -- Check permissions before executing business logic
- **Logging** -- Record command execution for debugging
- **Metrics** -- Measure handler duration and event counts
- **Validation** -- Validate command data before reaching the decider
- **Tenancy** -- Route to the correct stream based on tenant context

## See Also

- [[Command Handling]] -- The `CommandHandler` and `DeciderCommandHandler` that this pattern wraps
- [[The Decider Pattern]] -- The business logic that middleware surrounds
