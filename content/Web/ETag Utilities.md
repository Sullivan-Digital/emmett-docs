---
tags:
  - web
  - concurrency
aliases:
  - WeakETag
  - ETag
related:
  - "[[Concurrency Control]]"
  - "[[Type Branding]]"
  - "[[Optimistic Concurrency]]"
package: emmett-expressjs
---

# ETag Utilities

Express and Hono provide ETag utilities for implementing optimistic concurrency control in HTTP APIs. The pattern encodes the event stream version as a weak ETag (`W/"<version>"`), which clients send back in the `if-match` header when making updates. This connects the web layer to the [[Concurrency Control|expected version]] mechanism in the event store.

[[Fastify Integration|Fastify]] does not provide custom ETag utilities -- it relies on the `@fastify/etag` plugin for automatic ETag generation.

## Key Functions

```typescript
// Express
import { toWeakETag, getETagValueFromIfMatch, setETag } from '@event-driven-io/emmett-expressjs';

// Hono
import { toWeakETag, getETagValueFromIfMatch, setETag } from '@event-driven-io/emmett-honojs';
```

| Function | Purpose |
|---|---|
| ==`toWeakETag(value)`== | Creates `W/"<value>"` from a number, bigint, or string |
| ==`getETagValueFromIfMatch(request\|context)`== | Extracts the value from the `if-match` header; throws `ConcurrencyError` if missing or malformed |
| `getETagFromIfMatch(request\|context)` | Gets the raw ETag from the `if-match` header |
| `getWeakETagValue(etag)` | Extracts the inner value from a weak ETag |
| `isWeakETag(etag)` | Type guard for weak ETags |
| `setETag(response\|context, etag)` | Sets the `etag` header on the response |

## Branded Types

`ETag` and `WeakETag` are [[Type Branding|branded string types]], providing type safety so you cannot accidentally pass a plain string where an ETag is expected:

```typescript
type WeakETag = Brand<`W/${string}`, 'ETag'>;
type ETag = Brand<string, 'ETag'>;
```

## Constants and Errors

```typescript
const HeaderNames = {
  IF_MATCH: 'if-match',
  IF_NOT_MATCH: 'if-not-match',
  ETag: 'etag',
};

const WeakETagRegex = /W\/"(-?\d+.*)"/;

const enum ETagErrors {
  WRONG_WEAK_ETAG_FORMAT = 'WRONG_WEAK_ETAG_FORMAT',
  MISSING_IF_MATCH_HEADER = 'MISSING_IF_MATCH_HEADER',
  MISSING_IF_NOT_MATCH_HEADER = 'MISSING_IF_NOT_MATCH_HEADER',
}
```

## Optimistic Concurrency Flow

The typical pattern for optimistic concurrency in an API:

1. Client reads a resource -- response includes `ETag: W/"3"` (stream version)
2. Client sends an update with `If-Match: W/"3"`
3. Server extracts the version, passes it as `expectedStreamVersion`
4. If the stream has moved past version 3, the event store throws a [[Concurrency Control|ConcurrencyError]]

### Express Example

```typescript
import { on, OK, toWeakETag, getETagValueFromIfMatch } from '@event-driven-io/emmett-expressjs';
import { assertUnsignedBigInt } from '@event-driven-io/emmett';

router.put('/carts/:id/items', on(async (request: Request) => {
  const streamId = request.params.id;

  // 1. Extract version from if-match header
  const eTagValue = getETagValueFromIfMatch(request);
  const expectedVersion = assertUnsignedBigInt(eTagValue);

  // 2. Execute command with expected version
  const { newPosition } = await handle(
    eventStore,
    streamId,
    (state) => updateItem(command, state),
    { expectedStreamVersion: expectedVersion },
  );

  // 3. Return new version as ETag
  return OK({ eTag: toWeakETag(newPosition) });
}));
```

### Hono Example

The Hono equivalent uses `context` instead of `request`/`response`:

```typescript
import { OK, toWeakETag, getETagValueFromIfMatch } from '@event-driven-io/emmett-honojs';
import { assertUnsignedBigInt } from '@event-driven-io/emmett';

router.put('/carts/:id/items', async (context: Context) => {
  const streamId = context.req.param('id');

  const eTagValue = getETagValueFromIfMatch(context);  // takes Context, not Request
  const expectedVersion = assertUnsignedBigInt(eTagValue);

  const { newPosition } = await handle(
    eventStore,
    streamId,
    (state) => updateItem(command, state),
    { expectedStreamVersion: expectedVersion },
  );

  return OK({ context, eTag: toWeakETag(newPosition) });
});
```

## Framework-Specific ETag Handling

| Aspect | Express | Hono | Fastify |
|---|---|---|---|
| Default ETag | ==Disabled== by `getApplication()` | Uses `hono/etag` middleware | `@fastify/etag` plugin |
| Custom utilities | Yes (`toWeakETag`, `getETagValueFromIfMatch`, etc.) | Yes (same API, Context-based) | No |
| Header access | `request.headers['if-match']` | `context.req.header('if-match')` | Manual |
| Header setting | `response.setHeader('etag', value)` | `context.header('etag', value)` | Manual |

> [!info] Express disables its default ETag
> `getApplication()` sets `app.set('etag', false)` to disable Express's built-in ETag generation. Emmett uses its own ETag scheme based on stream versions for optimistic concurrency. If you need default Express ETags for non-event-sourced endpoints, pass `enableDefaultExpressEtag: true` in `ApplicationOptions`.

## See Also

- [[Concurrency Control]] -- Expected version mechanics in the event store
- [[Optimistic Concurrency]] -- Full pattern across the stack (event store + web layer)
- [[Response Helpers]] -- The `eTag` option in success response helpers
- [[Type Branding]] -- How `ETag` and `WeakETag` branded types work
