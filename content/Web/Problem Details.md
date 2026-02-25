---
tags:
  - web
  - errors
aliases:
  - RFC 7807
  - ProblemDocument
related:
  - "[[Error Hierarchy]]"
  - "[[Express.js Integration]]"
  - "[[Hono Integration]]"
  - "[[Response Helpers]]"
package: emmett-expressjs
---

# Problem Details

Both Express and Hono automatically convert unhandled errors into [RFC 7807 Problem Details](https://datatracker.ietf.org/doc/html/rfc7807) responses with `Content-Type: application/problem+json`. This integrates with Emmett's [[Error Hierarchy|error hierarchy]] -- errors with an `errorCode` property are mapped to the corresponding HTTP status code.

[[Fastify Integration|Fastify]] does not provide problem details support. You must handle errors manually in your route handlers or register your own Fastify error handler.

## Default Error Mapping

If the error has a numeric `errorCode` property in the range [100, 600), it is used as the HTTP status code. Otherwise the status defaults to 500. This works with Emmett's built-in error types:

| Emmett Error | `errorCode` | HTTP Status |
|---|---|---|
| `IllegalStateError` | 403 | 403 Forbidden |
| `ValidationError` | 400 | 400 Bad Request |
| `NotFoundError` | 404 | 404 Not Found |
| `ConcurrencyError` | 412 | 412 Precondition Failed |

> [!tip] Error codes align with HTTP status codes
> Emmett's [[Error Hierarchy]] uses HTTP-aligned error codes by design. The `errorCode` on each `EmmettError` subclass directly corresponds to the HTTP status code, making the mapping transparent.

## Express.js Problem Details Middleware

Express uses a standard 4-argument error-handling middleware that is added as the ==last middleware== by `getApplication()`:

```typescript
// This is registered automatically by getApplication() unless disabled
app.use(problemDetailsMiddleware(mapError));
```

When an error is thrown (or passed to `next(error)`) in any route handler, the middleware:

1. Tries the custom `mapError` function if provided
2. Falls back to `defaultErrorToProblemDetailsMapping`
3. Sends the `ProblemDocument` as JSON with the appropriate status code

The `express-async-errors` package is imported as a side effect by the Express integration, which patches Express to automatically forward async errors to the error middleware. Without this, unhandled promise rejections in async route handlers would crash the process.

```typescript
const problemDetailsMiddleware = (mapError?: ErrorToProblemDetailsMapping) =>
  (error: Error, request: Request, response: Response, _next: NextFunction): void;

const defaultErrorToProblemDetailsMapping = (error: Error): ProblemDocument;
```

## Hono Problem Details Handler

Hono uses ==`app.onError()`== instead of middleware:

```typescript
// Registered automatically by getApplication() unless disabled
app.onError((error, context) => {
  const problemDetails = errorMapper(error);
  return context.json(problemDetails, problemDetails.status);
});
```

> [!warning] `app.onError()` is a singleton
> A Hono app can only have **one** `onError` handler. If you enable problem details (the default) and also set a custom `onError` handler after creating the application, it will **replace** Emmett's problem details handler. Use the `mapError` option in `ApplicationOptions` rather than setting `onError` directly.

## Custom Error Mapping

Both Express and Hono accept a ==`mapError`== function in `ApplicationOptions`:

```typescript
import { ProblemDocument } from 'http-problem-details';

const mapError = (error: Error): ProblemDocument | undefined => {
  if (error instanceof CartNotFoundError) {
    return new ProblemDocument({
      type: 'https://example.com/errors/cart-not-found',
      title: 'Cart Not Found',
      detail: error.message,
      status: 404,
    });
  }
  // Return undefined to fall back to default mapping
  return undefined;
};

const app = getApplication({
  apis: [shoppingCartApi(eventStore)],
  mapError,
});
```

If `mapError` returns `undefined`, the system falls back to `defaultErrorToProblemDetailsMapping`.

### Signature Difference

> [!warning] Express receives the request; Hono does not
> The Express `ErrorToProblemDetailsMapping` signature is:
> ```typescript
> type ErrorToProblemDetailsMapping = (error: Error, request: Request) => ProblemDocument | undefined;
> ```
> The Hono equivalent is:
> ```typescript
> type ErrorToProblemDetailsMapping = (error: Error) => ProblemDocument | undefined;
> ```
> Express gives you access to the request for context-aware error mapping. Hono does not.

## Disabling Problem Details

Both Express and Hono allow disabling problem details via `ApplicationOptions`:

```typescript
const app = getApplication({
  apis: [...],
  disableProblemDetailsMiddleware: true,
});
```

In Express, this skips registering the error middleware. In Hono, this skips setting the `app.onError()` handler, freeing it for your own use.

## See Also

- [[Error Hierarchy]] -- The `EmmettError` subclasses and their `errorCode` values
- [[Response Helpers]] -- Error helpers (`NotFound()`, `Conflict()`, etc.) that create problem detail responses manually
- [[Express.js Integration]] -- Express-specific application setup details
- [[Hono Integration]] -- Hono-specific `onError` limitations
