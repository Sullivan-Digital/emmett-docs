---
tags:
  - web
  - hono
aliases:
  - Hono
  - emmett-honojs
related:
  - "[[Response Helpers]]"
  - "[[ETag Utilities]]"
  - "[[Problem Details]]"
  - "[[API Testing]]"
package: emmett-honojs
---

# Hono Integration

The `@event-driven-io/emmett-honojs` package provides a lightweight web framework integration with the same API surface as [[Express.js Integration|Express]]: [[Response Helpers]], [[ETag Utilities]], [[Problem Details|RFC 7807 problem details]], and BDD-style [[API Testing|test specifications]]. The key difference is that Hono route handlers return native `Response` objects directly, and every response helper requires a ==`context`== parameter.

## Application Setup

The ==`getApplication()`== factory is synchronous (like Express, unlike [[Fastify Integration|Fastify]]):

```typescript
import { getApplication, startAPI } from '@event-driven-io/emmett-honojs';

const app = getApplication({
  apis: [
    shoppingCartApi(eventStore),
    orderApi(eventStore),
  ],
});

startAPI(app); // Listens on port 3000 by default
```

`startAPI` uses `@hono/node-server`'s `serve()` function under the hood.

### ApplicationOptions

```typescript
type ApplicationOptions = {
  apis: WebApiSetup[];                      // Array of route registration functions
  mapError?: ErrorToProblemDetailsMapping;   // Custom error-to-problem-details mapping
  disableProblemDetailsMiddleware?: boolean; // Default: false (problem details enabled)
};
```

Hono has fewer configuration options than Express because JSON parsing and URL encoding are handled natively by Hono.

### Initialization Sequence

`getApplication()` performs the following:

1. Creates a `Hono` app and a `Hono` router
2. Adds the `etag()` middleware from `hono/etag`
3. Passes the router to each API setup function in the `apis` array
4. Mounts the router at `/`
5. Sets `app.onError()` for [[Problem Details|problem details]] (unless disabled)

## Route Registration

Hono uses the ==`WebApiSetup`== type, same concept as Express but typed for `Hono`:

```typescript
type WebApiSetup = (router: Hono) => void;
```

Route handlers work with `Context` directly -- no `on()` wrapper is needed:

```typescript
import { NoContent, OK, NotFound, type WebApiSetup } from '@event-driven-io/emmett-honojs';
import type { Hono, Context } from 'hono';

const shoppingCartApi = (eventStore: EventStore): WebApiSetup =>
  (router: Hono) => {
    router.post('/carts/:id/items', async (context: Context) => {
      // ... business logic ...
      return NoContent({ context });
    });

    router.get('/carts/:id', async (context: Context) => {
      const result = await getCart(context.req.param('id'));
      if (!result) return NotFound({ context, problemDetails: 'Cart not found' });
      return OK({ context, body: result });
    });
  };
```

> [!warning] Context is always required
> Every Hono response helper requires `{ context, ...options }`. Express helpers can be called with no arguments (e.g., `NotFound()`), but in Hono you must always pass context: `NotFound({ context })`.

## Context Utility Types

Hono provides typed context variants for safe request data access:

```typescript
import type {
  ContextWithBody,
  ContextWithParams,
  ContextWithQuery,
} from '@event-driven-io/emmett-honojs';

// Typed request body
router.post('/carts', async (context: ContextWithBody<{ productId: string; quantity: number }>) => {
  const body = await context.req.json();
  // body is typed as { productId: string; quantity: number }
});

// Typed route params
router.get('/carts/:id', async (context: ContextWithParams<{ id: string }>) => {
  const id = context.req.param('id');
  // id is typed as string
});
```

These types narrow the `Context` type for better TypeScript inference in route handlers.

## Module Exports

The package re-exports from:
- `./application` -- `getApplication()`, `startAPI()`, `ApplicationOptions`, `WebApiSetup`
- `./etag` -- [[ETag Utilities]]
- `./handler` -- [[Response Helpers]], context utility types
- `./middlewares` -- [[Problem Details|problem details]] mapping (unlike Express, this ==is== directly re-exported)
- `./responses` -- Response option types
- `./testing` -- [[API Testing|ApiSpecification]], [[API Testing|ApiE2ESpecification]]

## Key Differences from Express

| Aspect | Express | Hono |
|---|---|---|
| Response helpers return | `HttpResponse` closures | Native `Response` objects |
| Route handler wrapper | `on()` required | Not needed |
| Context/request | Access via `request: Request` | Access via `context: Context` |
| `set()` in tests | `.set('key', 'value')` (two args) | `.set({ key: value })` (object) |
| `ErrorToProblemDetailsMapping` | `(error, request) => ProblemDocument` | `(error) => ProblemDocument` |
| `onError` handling | Express error middleware (stackable) | ==`app.onError()` singleton== |

> [!warning] `app.onError()` is a singleton
> A Hono app can only have **one** `onError` handler. If you enable problem details (the default) and also set a custom `onError` handler, only the last one registered takes effect. Use the `mapError` option in `ApplicationOptions` rather than setting `onError` directly.

## See Also

- [[Express.js Integration]] -- Similar API surface with curried closure pattern
- [[Fastify Integration]] -- Minimal alternative without response helpers
- [[Response Helpers]] -- Detailed comparison of response helper patterns
- [[API Testing]] -- `HonoTestAgent` and its differences from supertest
