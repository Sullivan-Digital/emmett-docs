---
tags:
  - web
  - api
aliases:
  - HTTP Response Helpers
related:
  - "[[Express.js Integration]]"
  - "[[Hono Integration]]"
  - "[[Problem Details]]"
  - "[[Error Hierarchy]]"
package: emmett-expressjs
---

# Response Helpers

Express and Hono provide a matching set of response helpers for common HTTP status codes. These helpers standardize response formatting and integrate with [[ETag Utilities]] and [[Problem Details]]. [[Fastify Integration|Fastify]] does not provide response helpers -- use the native `reply` API.

## Success Helpers

| Helper | Status | Notes |
|---|---|---|
| `OK(options?)` | 200 | Returns body if provided |
| `Created(options)` | 201 | Auto-generates `Location` header from `createdId` or `url` |
| `Accepted(options)` | 202 | Requires `location` |
| `NoContent(options?)` | 204 | No body |
| `HttpResponse(statusCode, options?)` | Custom | For any status code |

## Error Helpers

| Helper | Status | Notes |
|---|---|---|
| `BadRequest(options?)` | 400 | |
| `Forbidden(options?)` | 403 | |
| `NotFound(options?)` | 404 | Can be called with no arguments (Express) |
| `Conflict(options?)` | 409 | |
| `PreconditionFailed(options)` | 412 | For optimistic concurrency failures |
| `HttpProblem(statusCode, options?)` | Custom | For any error status code |

## Express Pattern: Curried Closures

In [[Express.js Integration|Express]], response helpers return ==closures== -- functions that accept an Express `Response` and apply the status code, headers, and body. The `on()` wrapper bridges these closures into Express middleware.

```typescript
import { on, OK, Created, NoContent, NotFound, Conflict } from '@event-driven-io/emmett-expressjs';

type HttpResponse = (response: Response) => void;
```

```typescript
// OK with body
router.get('/carts/:id', on(async (request: Request) => {
  return OK({ body: { items: cart.items } });
}));

// Created with auto-generated Location header
// Sets Location to "${request.url}/abc-123" and body to { id: "abc-123" }
router.post('/carts', on(async (request: Request) => {
  return Created({ createdId: 'abc-123' });
}));

// Created with explicit URL and ETag
router.post('/carts', on(async (request: Request) => {
  return Created({ url: '/carts/abc-123', body: { id: 'abc-123' }, eTag: toWeakETag(1n) });
}));

// NoContent (common for command handlers)
router.post('/carts/:id/confirm', on(async (request: Request) => {
  return NoContent();
}));
```

Error helpers in Express:

```typescript
// Simple 404
return NotFound();

// 404 with detail message
return NotFound({ problemDetails: 'Shopping cart not found' });

// 409 with a full ProblemDocument
return Conflict({
  problem: new ProblemDocument({
    type: 'https://example.com/cart-already-confirmed',
    title: 'Cart Already Confirmed',
    detail: 'Cannot modify a confirmed shopping cart',
    status: 409,
  }),
});
```

## Hono Pattern: Direct Response Returns

In [[Hono Integration|Hono]], response helpers return native `Response` objects directly and ==always require a `context` parameter==:

```typescript
import { OK, Created, NoContent, NotFound, Conflict } from '@event-driven-io/emmett-honojs';

router.post('/carts/:id/items', async (context: Context) => {
  return Created({ context, createdId: 'abc-123' });
});

router.get('/carts/:id', async (context: Context) => {
  return OK({ context, body: { items: cart.items } });
});

router.post('/carts/:id/confirm', async (context: Context) => {
  return NoContent({ context });
});

router.get('/carts/:id', async (context: Context) => {
  return NotFound({ context, problemDetails: 'Shopping cart not found' });
});
```

> [!warning] Context is always required in Hono
> Express helpers can be called with no arguments (e.g., `NotFound()`). In Hono, you must always pass context: `NotFound({ context })` is the minimum.

## Option Types

Both Express and Hono share the same option type definitions:

```typescript
type HttpResponseOptions = {
  body?: unknown;
  location?: string;
  eTag?: ETag;
};

type CreatedHttpResponseOptions = (
  | { createdId: string }
  | { createdId?: string; url: string }
) & HttpResponseOptions;

type AcceptedHttpResponseOptions = {
  location: string;
} & HttpResponseOptions;

type NoContentHttpResponseOptions = Omit<HttpResponseOptions, 'body'>;

type HttpProblemResponseOptions = {
  location?: string;
  eTag?: ETag;
} & (
  | { problem: ProblemDocument }     // Full ProblemDocument
  | { problemDetails: string }       // Just a detail message (auto-wraps)
);
```

> [!tip] `Created` auto-generates Location
> When using `Created({ createdId: 'abc-123' })`, the helper auto-generates a `Location` header as `${request.url}/abc-123` and sets the body to `{ id: 'abc-123' }`. Use the `url` option for a custom Location path.

## Low-Level Send Functions

Both packages expose low-level send functions used internally by the helpers:

```typescript
// Express
const send = (response: Response, statusCode: number, options?: HttpResponseOptions): void;
const sendCreated = (response: Response, options: CreatedHttpResponseOptions): void;
const sendAccepted = (response: Response, options: AcceptedHttpResponseOptions): void;
const sendProblem = (response: Response, statusCode: number, options?: HttpProblemResponseOptions): void;

// Hono (same names, takes Context instead of Response, returns Response)
const send = (context: Context, statusCode: StatusCode, options?: HttpResponseOptions): Response;
const sendCreated = (context: Context, options: CreatedHttpResponseOptions): Response;
const sendAccepted = (context: Context, options: AcceptedHttpResponseOptions): Response;
const sendProblem = (context: Context, statusCode: StatusCode, options?: HttpProblemResponseOptions): Response;
```

In Hono, `send()` uses `context.json(body)` for bodies and `context.body(null)` for bodyless responses. `sendProblem` sets `Content-Type: application/problem+json`.

## Fastify: No Response Helpers

[[Fastify Integration|Fastify]] has no response helpers. Use the native `reply` API:

```typescript
app.post('/carts', async (request, reply) => {
  const cartId = createCart(request.body);
  return reply.code(201).send({ id: cartId });
});

app.get('/carts/:id', async (request, reply) => {
  const cart = await getCart(request.params.id);
  if (!cart) return reply.code(404).send();
  return reply.code(200).send(cart);
});
```

> [!bug] Default error text has a typo
> When calling error helpers without specifying problem details (e.g., `NotFound()` in Express or `NotFound({ context })` in Hono), the default detail message is =="Error occured!"== (note the typo). You may want to always provide explicit problem details for clearer error messages.

## See Also

- [[Problem Details]] -- How error helpers integrate with RFC 7807
- [[ETag Utilities]] -- Using the `eTag` option in response helpers
- [[Error Hierarchy]] -- Emmett error types that map to HTTP status codes
