---
tags:
  - web
  - fastify
aliases:
  - Fastify
  - emmett-fastify
related:
  - "[[Web Frameworks MOC]]"
  - "[[Graceful Shutdown]]"
package: emmett-fastify
---

# Fastify Integration

The `@event-driven-io/emmett-fastify` package provides a ==minimal== integration focused on application bootstrapping with sensible defaults. Unlike [[Express.js Integration|Express]] and [[Hono Integration|Hono]], it does **not** provide [[Response Helpers]], [[ETag Utilities]], [[Problem Details]] middleware, or [[API Testing|test specifications]]. The entire implementation is a single file.

## Application Setup

The ==`getApplication()`== function is **async** (unlike Express and Hono) and returns a `FastifyInstance`:

```typescript
import { getApplication, startAPI } from '@event-driven-io/emmett-fastify';

const app = await getApplication({
  registerRoutes: (app) => {
    app.post('/carts', async (request, reply) => {
      // ... route handler ...
      return reply.code(201).send({ id: cartId });
    });
  },
});

await startAPI(app); // Listens on port 5000 by default
```

> [!warning] Default port is 5000
> Express and Hono default to port 3000. Fastify defaults to ==port 5000==. This is easy to overlook when switching between frameworks.

### ApplicationOptions

```typescript
interface ApplicationOptions {
  serverOptions?: { logger: boolean };      // Default: { logger: true }
  registerRoutes?: (app: FastifyInstance) => void;
  activeDefaultPlugins?: Plugin[];          // Default: [Etag, Compress, Form]
}
```

### Key Differences from Express and Hono

- `getApplication()` is **async** (returns `Promise<FastifyInstance>`)
- Uses a single `registerRoutes` callback instead of an `apis` array with the `WebApiSetup` type
- Default port is **5000** (not 3000)
- Includes built-in [[Graceful Shutdown|graceful shutdown]] via `close-with-grace` (500ms delay)
- Registers three default plugins automatically
- No custom ETag handling -- relies on `@fastify/etag` plugin

### Plugin Type

```typescript
type Plugin = {
  plugin: FastifyPluginAsync | FastifyPluginCallback;
  options: FastifyPluginOptions;
};
```

## Default Plugins

Fastify registers three plugins by default (configurable via `activeDefaultPlugins`):

| Plugin | Purpose |
|---|---|
| `@fastify/etag` | Automatic ETag generation |
| `@fastify/compress` | Response compression (global: false) |
| `@fastify/formbody` | Form body parsing |

## Route Registration

Routes are registered directly on the `FastifyInstance` using the native Fastify API:

```typescript
const app = await getApplication({
  registerRoutes: (app) => {
    app.post('/carts/:id/items', async (request, reply) => {
      // ... business logic ...
      return reply.code(204).send();
    });

    app.get('/carts/:id', async (request, reply) => {
      const result = await getCart(request.params.id);
      if (!result) return reply.code(404).send();
      return reply.code(200).send(result);
    });
  },
});
```

> [!tip] No `WebApiSetup` type
> Fastify does not use a `WebApiSetup` type. The `registerRoutes` option takes a single callback receiving the `FastifyInstance`. If you need to organize routes across multiple modules, compose them within this callback.

## Graceful Shutdown

Fastify includes built-in graceful shutdown via the `close-with-grace` package with a 500ms delay. An `onClose` hook is registered to clean up the grace listener. This is not included in the Express or Hono integrations.

## What Is Not Provided

The Fastify integration intentionally omits:

- **Response helpers** (`OK()`, `Created()`, `NotFound()`, etc.) -- use `reply.code(status).send(body)`
- **ETag utilities** for optimistic concurrency -- only automatic ETag generation via the plugin
- **Problem details middleware** -- handle errors manually in route handlers or register a Fastify error handler
- **Test specifications** (`ApiSpecification`, `ApiE2ESpecification`) -- use Fastify's built-in `app.inject()`

> [!info] Why so minimal?
> Fastify's built-in features (automatic serialization, error handling, lifecycle hooks) make many of these abstractions unnecessary, or the integration may simply be less mature. Check for updates in newer Emmett versions.

## Testing

Use Fastify's built-in ==`app.inject()`== for in-process request simulation:

```typescript
const app = await getApplication({
  registerRoutes: (app) => {
    app.post('/carts', async (request, reply) => {
      return reply.code(201).send({ id: 'cart-1' });
    });
  },
});

const response = await app.inject({
  method: 'POST',
  url: '/carts',
  payload: { productId: 'product-1', quantity: 2 },
});

assert.equal(response.statusCode, 201);
assert.deepEqual(response.json(), { id: 'cart-1' });
```

## See Also

- [[Express.js Integration]] -- Full-featured alternative with response helpers and testing framework
- [[Hono Integration]] -- Lightweight alternative with response helpers
- [[Web Frameworks MOC]] -- Comparison table across all three frameworks
