---
tags:
  - web
  - testing
aliases:
  - ApiSpecification
  - ApiE2ESpecification
related:
  - "[[Express.js Integration]]"
  - "[[Hono Integration]]"
  - "[[Fastify Integration]]"
  - "[[DeciderSpecification]]"
  - "[[Assertions Library]]"
package: emmett-expressjs
---

# API Testing

Express and Hono provide two BDD-style test specification patterns that follow the same `given().when().then()` structure as [[DeciderSpecification]]:

- ==`ApiSpecification`== -- Unit-style tests with pre-seeded event streams
- ==`ApiE2ESpecification`== -- End-to-end tests with real HTTP requests

[[Fastify Integration|Fastify]] provides no test specification framework. Use Fastify's built-in `app.inject()` instead.

## ApiSpecification (Unit-Style)

`ApiSpecification` lets you pre-seed event streams and then assert on both the HTTP response and the events appended to the store. It wraps the event store internally (via `WrapEventStore()`) to capture appended events.

### Setup

```typescript
// Express
import {
  ApiSpecification,
  existingStream,
  expectNewEvents,
  expectResponse,
  expectError,
  getApplication,
} from '@event-driven-io/emmett-expressjs';
import { getInMemoryEventStore, type EventStore } from '@event-driven-io/emmett';

const given = ApiSpecification.for<ShoppingCartEvent>(
  () => getInMemoryEventStore(),
  (eventStore: EventStore) =>
    getApplication({
      apis: [shoppingCartApi(eventStore)],
    }),
);
```

```typescript
// Hono (same pattern, different imports)
import {
  ApiSpecification,
  existingStream,
  expectNewEvents,
  expectResponse,
  getApplication,
} from '@event-driven-io/emmett-honojs';

const given = ApiSpecification.for<ShoppingCartEvent>(
  () => getInMemoryEventStore(),
  (eventStore: EventStore) =>
    getApplication({
      apis: [shoppingCartApi(eventStore)],
    }),
);
```

### Testing Commands (Empty Stream)

```typescript
await given()
  .when((request) =>
    request
      .post('/clients/client-1/shopping-carts/current/product-items')
      .send({ productId: 'product-1', quantity: 2 }),
  )
  .then([
    expectResponse(204),
    expectNewEvents('shopping_cart:client-1:current', [
      {
        type: 'ProductItemAddedToShoppingCart',
        data: {
          shoppingCartId: 'shopping_cart:client-1:current',
          productItem: { productId: 'product-1', quantity: 2, unitPrice: 100 },
        },
      },
    ]),
  ]);
```

### Testing with Pre-Seeded Events

```typescript
await given(
  existingStream('shopping_cart:client-1:current', [
    {
      type: 'ProductItemAddedToShoppingCart',
      data: { shoppingCartId: 'shopping_cart:client-1:current', productItem },
    },
  ]),
)
  .when((request) =>
    request.post('/clients/client-1/shopping-carts/current/confirm'),
  )
  .then([
    expectResponse(204),
    expectNewEvents('shopping_cart:client-1:current', [
      { type: 'ShoppingCartConfirmed', data: { shoppingCartId: 'shopping_cart:client-1:current' } },
    ]),
  ]);
```

### Testing Error Responses

```typescript
await given(
  existingStream(shoppingCartId, [addedEvent, confirmedEvent]),
)
  .when((request) =>
    request.post(`/clients/${clientId}/shopping-carts/current/product-items`).send(productItem),
  )
  .then(
    expectError(403, {
      detail: 'Shopping Cart already closed',
      status: 403,
      title: 'Forbidden',
      type: 'about:blank',
    }),
  );
```

### Assertion Options

The `then()` method accepts multiple forms:

```typescript
// Assert only response
.then(expectResponse(200, { body: { status: 'Opened' } }));

// Assert only events
.then([expectNewEvents(streamId, [expectedEvent])]);

// Assert response AND events
.then([
  expectResponse(204),
  expectNewEvents(streamId, [expectedEvent]),
]);

// Custom assertion function
.then((response) => {
  assert.equal(response.statusCode, 200);
  assert.ok(response.body.items.length > 0);
});
```

### Type Signatures

```typescript
type TestRequest = (request: TestAgent<supertest.Test>) => Test;  // Express
type TestRequest = (request: HonoTestAgent) => HonoTestRequest | Promise<HonoResponse>;  // Hono

type ResponseAssert = (response: Response) => boolean | void;  // Express
type ResponseAssert = (response: HonoResponse) => boolean | void | Promise<boolean> | Promise<void>;  // Hono

type ApiSpecificationAssert<EventType extends Event = Event> =
  | TestEventStream<EventType>[]       // assert events only
  | ResponseAssert                      // assert response only
  | [ResponseAssert, ...TestEventStream<EventType>[]];  // assert both
```

> [!info] Hono supports async assertions
> Hono's `ResponseAssert` supports `Promise<boolean | void>` because `HonoResponse.body` may be `null` -- you may need to use the async `response.json()` method for reliable body access.

## ApiE2ESpecification (End-to-End)

`ApiE2ESpecification` uses real HTTP requests for both the "given" setup and the "when" action. Instead of pre-seeding event streams, you provide request functions that set up the state through the API itself.

```typescript
import {
  ApiE2ESpecification,
  expectResponse,
  getApplication,
  type TestRequest,
} from '@event-driven-io/emmett-expressjs';

const given = ApiE2ESpecification.for(
  () => eventStore,
  (eventStore: EventStore) =>
    getApplication({
      apis: [shoppingCartApi(eventStore, pool, messageBus, getUnitPrice, () => now)],
    }),
);

// Define setup requests
const addProduct: TestRequest = (request) =>
  request
    .post(`/clients/${clientId}/shopping-carts/current/product-items`)
    .send({ productId: 'product-1', quantity: 2 });

// Test: after adding a product, GET returns the cart
await given(addProduct)
  .when((request) =>
    request.get(`/clients/${clientId}/shopping-carts/current`).send(),
  )
  .then([
    expectResponse(200, {
      body: {
        clientId,
        id: shoppingCartId,
        productItems: [{ productId: 'product-1', quantity: 2, unitPrice: 100 }],
        status: 'Opened',
      },
    }),
  ]);

// Multiple setup requests are executed sequentially
const confirmCart: TestRequest = (request) =>
  request.post(`/clients/${clientId}/shopping-carts/current/confirm`);

await given(addProduct, confirmCart)
  .when((request) =>
    request.get(`/clients/${clientId}/shopping-carts/current`).send(),
  )
  .then([expectResponse(404)]);
```

## Hono Testing Differences

Hono uses a custom ==`HonoTestAgent`== instead of supertest. The main API differences:

### Header Setting

```typescript
// Express (supertest): two arguments
request.set('if-match', 'W/"1"');

// Hono (HonoTestAgent): object argument
request.set({ 'if-match': 'W/"1"' });
```

> [!warning] `set()` API is different
> Express/supertest uses `.set('key', 'value')` with two arguments. Hono's `HonoTestRequest` uses `.set({ key: value })` with an object. This is easy to miss when porting tests between frameworks.

### HonoTestAgent API

```typescript
class HonoTestAgent {
  get(path: string): HonoTestRequest;
  post(path: string): HonoTestRequest;
  put(path: string): HonoTestRequest;
  patch(path: string): HonoTestRequest;
  delete(path: string): HonoTestRequest;
}

class HonoTestRequest {
  send(body?: unknown): HonoTestRequest;
  set(headers: Record<string, string>): HonoTestRequest;
  expect(): Promise<HonoResponse>;
  execute(): Promise<HonoResponse>;
}

class HonoResponse {
  get statusCode(): number;
  get status(): number;
  get headers(): Record<string, string>;
  get body(): unknown;          // may be null!
  json(): Promise<unknown>;     // use this for reliable body access
  text(): Promise<string>;
}
```

> [!warning] `HonoResponse.body` may be null
> The `body` property on `HonoResponse` is synchronous and may return `null` if the response body has not been parsed yet. Use the async `response.json()` method for reliable access.

## E2E Testing Helpers

Both Express and Hono provide helpers for advanced E2E scenarios:

```typescript
// Verify the ETag in a response contains the next revision
const revision: bigint = expectNextRevisionInResponseEtag(response);

// Run a request twice to test idempotency/optimistic concurrency
await runTwice(() => request.post('/carts/cart-1/items').send(item))
  .expect(statuses(201, 412));  // first succeeds, second gets precondition failed
```

The `runTwice` pattern runs the same request twice and expects different status codes -- useful for testing optimistic concurrency where the first request succeeds and the second fails with 412.

## Fastify Testing

[[Fastify Integration|Fastify]] provides no test specification framework. Use Fastify's built-in `app.inject()`:

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

## Helper Functions Reference

| Function | Purpose |
|---|---|
| `existingStream(streamId, events)` | Pre-seeds events into a stream for unit-style tests |
| `expectNewEvents(streamId, events)` | Asserts events were appended to a stream |
| `expectResponse(statusCode, options?)` | Asserts HTTP status code and optionally body/headers |
| `expectError(errorCode, problemDetails?)` | Asserts an error response with problem details |
| `expectNextRevisionInResponseEtag(response)` | Extracts and validates the ETag revision |
| `runTwice(test)` | Runs a request twice for idempotency testing |
| `statuses(first, second)` | Creates assertion pair for `runTwice` |

## See Also

- [[DeciderSpecification]] -- Similar BDD pattern for testing [[The Decider Pattern|Deciders]] directly
- [[WrapEventStore]] -- The event store wrapper used internally by `ApiSpecification`
- [[Assertions Library]] -- Lower-level assertion utilities
- [[ETag Utilities]] -- Testing optimistic concurrency via `if-match` headers
