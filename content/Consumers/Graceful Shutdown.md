---
tags:
  - consumer
  - lifecycle
aliases:
  - onShutdown
  - ShutdownHandler
related:
  - "[[Consumer Architecture]]"
  - "[[Utilities MOC]]"
package: emmett
---

# Graceful Shutdown

> [!abstract]
> Processors automatically register for graceful shutdown when started. On `SIGTERM` or `SIGINT`, each processor's `close()` method is called, the `onClose` hook fires, and the processor marks itself as inactive.

## `onShutdown()`

The `onShutdown` function registers a handler for process termination signals:

```typescript
import { onShutdown } from '@event-driven-io/emmett';

const cleanup = onShutdown(async () => {
  console.log('Shutting down...');
  await database.close();
  await messageBus.drain();
});

// Later, if you want to unregister the handler:
cleanup();
```

The return value is a cleanup function that unregisters the signal handlers when called.

## Runtime Support

| Runtime | Mechanism |
|---|---|
| **Node.js** / **Bun** | `process.on(signal)` / `process.off(signal)` |
| **Deno** | `Deno.addSignalListener` / `Deno.removeSignalListener` |
| **Browser** / **Cloudflare Workers** | No-op (returns empty cleanup function) |

## Automatic Registration

Processors auto-register for shutdown when started. No manual setup is needed:

```typescript
// Processors auto-register for shutdown. No manual setup needed.
const consumer = postgreSQLEventStoreConsumer({ connectionString });
consumer.reactor({ processorId: 'my-reactor', eachMessage: handler });
await consumer.start();
// On SIGTERM: processor.close() is called automatically
```

Internally, the `reactor()` function registers a shutdown handler on `start()`:

```typescript
closeSignal = onShutdown(() => close(startOptions));
```

## Shutdown Sequence

When a signal is received:

1. The shutdown handler calls the processor's `close()` method
2. The `onClose` [[Reactors#Lifecycle Hooks|lifecycle hook]] fires
3. The processor marks itself as inactive (`isActive = false`)

## Double-Close Prevention

If you manually call `close()` before a signal arrives, the shutdown handler is cleaned up to prevent double-close:

> [!warning]
> There is a known edge case where `close()` can be called multiple times, each triggering the `onClose` hook. Design your `onClose` handlers to be idempotent.

## See Also

- [[Consumer Architecture]] -- The consumer lifecycle that shutdown integrates with
- [[Reactors]] -- Lifecycle hooks (`onInit`, `onStart`, `onClose`)
- [[Utilities MOC]] -- Other utility modules in the core package
