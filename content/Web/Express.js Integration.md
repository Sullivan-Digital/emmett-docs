---
tags:
  - web
  - express
aliases:
  - Express.js
  - emmett-expressjs
related:
  - "[[Response Helpers]]"
  - "[[ETag Utilities]]"
  - "[[Problem Details]]"
  - "[[API Testing]]"
  - "[[Shopping Cart Example]]"
package: emmett-expressjs
---

# Express.js Integration

The `@event-driven-io/emmett-expressjs` package provides the most full-featured web framework integration in Emmett. It includes application bootstrapping, a curried [[Response Helpers|response helper]] pattern with the `on()` wrapper, [[ETag Utilities]], [[Problem Details|RFC 7807 problem details]] middleware, and BDD-style [[API Testing|test specifications]] built on supertest.

## Application Setup

The ==`getApplication()`== factory function creates and configures an Express application, and ==`startAPI()`== starts listening:

```typescript
import { getApplication, startAPI } from '@event-driven-io/emmett-expressjs';

const app = getApplication({
  apis: [
    shoppingCartApi(eventStore),
    orderApi(eventStore),
  ],
});

startAPI(app); // Listens on port 3000 by default
```

### ApplicationOptions

```typescript
type ApplicationOptions = {
  apis: WebApiSetup[];                      // Array of route registration functions
  mapError?: ErrorToProblemDetailsMapping;   // Custom error-to-problem-details mapping
  enableDefaultExpressEtag?: boolean;        // Default: false (Emmett disables Express's built-in ETag)
  disableJsonMiddleware?: boolean;           // Default: false (JSON parsing enabled)
  disableUrlEncodingMiddleware?: boolean;    // Default: false (URL encoding enabled)
  disableProblemDetailsMiddleware?: boolean; // Default: false (problem details enabled)
};
```

### Initialization Sequence

`getApplication()` performs seven steps in order:

1. Creates an Express `Application`
2. Disables the default Express ETag (`app.set('etag', false)`) so Emmett can manage [[ETag Utilities|ETags]] for optimistic concurrency
3. Adds `express.json()` middleware (unless `disableJsonMiddleware` is true)
4. Adds `express.urlencoded({ extended: true })` (unless `disableUrlEncodingMiddleware` is true)
5. Creates a `Router` and passes it to each API setup function in the `apis` array
6. Mounts the router on the app
7. Adds the `problemDetailsMiddleware` error handler (unless `disableProblemDetailsMiddleware` is true)

### StartApiOptions

```typescript
type StartApiOptions = { port?: number };  // Default: 3000
const startAPI = (app: Application, options?: StartApiOptions): http.Server;
```

## Route Registration

Express uses the ==`WebApiSetup`== type for route registration. The pattern is a factory function that closes over dependencies and returns a route registration function:

```typescript
import { on, NoContent, OK, NotFound, type WebApiSetup } from '@event-driven-io/emmett-expressjs';
import type { Router, Request } from 'express';

const shoppingCartApi = (eventStore: EventStore): WebApiSetup =>
  (router: Router) => {
    router.post('/carts/:id/items', on(async (request: Request) => {
      // ... business logic ...
      return NoContent();
    }));

    router.get('/carts/:id', on(async (request: Request) => {
      const result = await getCart(request.params.id);
      if (!result) return NotFound();
      return OK({ body: result });
    }));
  };
```

The `WebApiSetup` type is defined as:

```typescript
type WebApiSetup = (router: Router) => void;
```

## The `on()` Wrapper

The ==`on()`== function is the bridge between Emmett's handler pattern and Express middleware. It calls your handler, receives back an `HttpResponse` closure, and invokes it with the Express response:

```typescript
type HttpResponse = (response: Response) => void;
type HttpHandler<RequestType extends Request> =
  (request: RequestType) => Promise<HttpResponse> | HttpResponse;

const on: <RequestType extends Request>(handle: HttpHandler<RequestType>) =>
  (request: RequestType, response: Response, next: NextFunction) => Promise<void>;
```

> [!warning] Express-only pattern
> The `on()` wrapper is unique to Express. [[Hono Integration|Hono]] returns `Response` objects directly from route handlers, and [[Fastify Integration|Fastify]] uses the native `reply` API. Do not attempt to use the `on()` pattern with other frameworks.

## Module Exports

The package imports `express-async-errors` as a side effect and re-exports from:
- `./application` -- `getApplication()`, `startAPI()`, `ApplicationOptions`, `WebApiSetup`
- `./etag` -- [[ETag Utilities]]
- `./handler` -- `on()`, [[Response Helpers]]
- `./responses` -- Response option types
- `./testing` -- [[API Testing|ApiSpecification]], [[API Testing|ApiE2ESpecification]]

> [!info] Async error handling
> The `express-async-errors` import monkey-patches Express to automatically forward async errors to error middleware. Without this, unhandled promise rejections in async route handlers would crash the process. This is automatically active when importing from `@event-driven-io/emmett-expressjs`.

## Complete Example

> [!example]- Shopping Cart API with Express
> ```typescript
> // shoppingCarts/api.ts
> import {
>   CommandHandler,
>   assertNotEmptyString,
>   assertPositiveNumber,
>   type EventStore,
> } from '@event-driven-io/emmett';
> import {
>   NoContent,
>   NotFound,
>   OK,
>   on,
>   type WebApiSetup,
> } from '@event-driven-io/emmett-expressjs';
> import type { Request, Router } from 'express';
> import { addProductItem, confirm, cancel } from './businessLogic';
> import { evolve, initialState } from './shoppingCart';
>
> const handle = CommandHandler({ evolve, initialState });
>
> export const shoppingCartApi =
>   (eventStore: EventStore, getUnitPrice: (id: string) => Promise<number>): WebApiSetup =>
>   (router: Router) => {
>     // Add Product Item
>     router.post(
>       '/clients/:clientId/shopping-carts/current/product-items',
>       on(async (request: Request) => {
>         const clientId = assertNotEmptyString(request.params.clientId);
>         const shoppingCartId = `shopping_cart:${clientId}:current`;
>         const productId = assertNotEmptyString(request.body.productId);
>
>         const command = {
>           type: 'AddProductItemToShoppingCart' as const,
>           data: {
>             shoppingCartId,
>             clientId,
>             productItem: {
>               productId,
>               quantity: assertPositiveNumber(request.body.quantity),
>               unitPrice: await getUnitPrice(productId),
>             },
>           },
>         };
>
>         await handle(eventStore, shoppingCartId, (state) =>
>           addProductItem(command, state),
>         );
>
>         return NoContent();
>       }),
>     );
>
>     // Get Shopping Cart
>     router.get(
>       '/clients/:clientId/shopping-carts/current',
>       on(async (request: Request) => {
>         const clientId = assertNotEmptyString(request.params.clientId);
>         const shoppingCartId = `shopping_cart:${clientId}:current`;
>
>         const result = await getCartDetails(shoppingCartId);
>
>         if (result === null) return NotFound();
>
>         return OK({ body: result });
>       }),
>     );
>
>     // Confirm Shopping Cart
>     router.post(
>       '/clients/:clientId/shopping-carts/current/confirm',
>       on(async (request: Request) => {
>         const clientId = assertNotEmptyString(request.params.clientId);
>         const shoppingCartId = `shopping_cart:${clientId}:current`;
>
>         await handle(eventStore, shoppingCartId, (state) =>
>           confirm({ type: 'ConfirmShoppingCart', data: { shoppingCartId } }, state),
>         );
>
>         return NoContent();
>       }),
>     );
>   };
> ```
>
> ```typescript
> // index.ts
> import { getInMemoryMessageBus, projections } from '@event-driven-io/emmett';
> import { getApplication, startAPI } from '@event-driven-io/emmett-expressjs';
> import { getSQLiteEventStore, SQLiteConnectionPool } from '@event-driven-io/emmett-sqlite';
> import { shoppingCartApi } from './shoppingCarts/api';
>
> const pool = SQLiteConnectionPool({ fileName: './emmett_event_store.db' });
>
> const eventStore = getSQLiteEventStore({
>   fileName: './emmett_event_store.db',
>   projections: projections.inline(readModelProjections),
>   pool,
> });
> await eventStore.schema.migrate();
>
> const app = getApplication({
>   apis: [
>     shoppingCartApi(eventStore, getUnitPrice),
>   ],
> });
>
> startAPI(app);
> ```

## See Also

- [[Hono Integration]] -- Similar API surface with direct Response returns
- [[Fastify Integration]] -- Minimal alternative with async setup
- [[Command Handling]] -- The `CommandHandler` used in route handlers
- [[Validation Helpers]] -- `assertNotEmptyString`, `assertPositiveNumber` used for request validation
