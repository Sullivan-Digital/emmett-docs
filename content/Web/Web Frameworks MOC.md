---
tags:
  - moc
  - web
aliases:
  - Web Frameworks
related:
  - "[[Express.js Integration]]"
  - "[[Fastify Integration]]"
  - "[[Hono Integration]]"
  - "[[Response Helpers]]"
  - "[[ETag Utilities]]"
  - "[[Problem Details]]"
  - "[[API Testing]]"
---

# Web Frameworks

Emmett provides integration packages for three Node.js web frameworks: **Express.js**, **Hono**, and **Fastify**. Each package provides application bootstrapping via `getApplication()` and `startAPI()`. Express and Hono additionally offer [[Response Helpers]], [[ETag Utilities]], [[Problem Details|RFC 7807 problem details]], and BDD-style [[API Testing|test specifications]].

## Quick Comparison

| Feature | Express | Fastify | Hono |
|---|---|---|---|
| Package | `@event-driven-io/emmett-expressjs` | `@event-driven-io/emmett-fastify` | `@event-driven-io/emmett-honojs` |
| `getApplication()` | Synchronous | ==Async== | Synchronous |
| Return type | `Application` | `FastifyInstance` | `Hono` |
| Default port | 3000 | ==5000== | 3000 |
| Response helpers | Yes (curried closures) | ==No== | Yes (direct returns) |
| `on()` handler wrapper | Yes | No | No |
| ETag utilities | Yes | ==No== (uses @fastify/etag) | Yes |
| Problem details | Express error middleware | ==No== | `app.onError()` |
| Test specifications | supertest-based | ==No== (use `app.inject()`) | Custom `HonoTestAgent` |
| Graceful shutdown | Not included | Built-in (close-with-grace) | Not included |
| Compression | Not included | @fastify/compress built-in | Not included |

Express and Hono share the most API surface -- they both provide response helpers, ETag utilities, problem details support, and test specifications with an identical `given().when().then()` BDD pattern. Fastify provides only application bootstrapping with sensible plugin defaults.

## Notes in This Folder

- [[Express.js Integration]] -- Full-featured integration with `on()` wrapper, 7-step initialization, `WebApiSetup` type
- [[Fastify Integration]] -- Minimal integration with async setup, plugin system, and graceful shutdown
- [[Hono Integration]] -- Lightweight integration with direct Context handlers and context utility types
- [[Response Helpers]] -- `OK()`, `Created()`, `NotFound()`, and other HTTP response helpers for Express and Hono
- [[ETag Utilities]] -- Weak ETag encoding for optimistic concurrency control via `toWeakETag()` and `getETagValueFromIfMatch()`
- [[Problem Details]] -- RFC 7807 structured error responses with automatic Emmett error mapping
- [[API Testing]] -- BDD-style `ApiSpecification` and `ApiE2ESpecification` for unit and end-to-end testing

## Installation

```bash
# Express.js
npm install @event-driven-io/emmett-expressjs express express-async-errors http-problem-details supertest

# Hono
npm install @event-driven-io/emmett-honojs hono @hono/node-server http-problem-details

# Fastify
npm install @event-driven-io/emmett-fastify fastify @fastify/compress @fastify/etag @fastify/formbody close-with-grace
```

All three require `@event-driven-io/emmett` as a peer dependency.

## See Also

- [[Shopping Cart Example]] -- Complete end-to-end example using Express
- [[Error Hierarchy]] -- Emmett error types mapped to HTTP status codes by problem details
- [[Concurrency Control]] -- The event store side of optimistic concurrency (web side is in [[ETag Utilities]])
- [[Adapters MOC]] -- Database adapters that pair with these web frameworks
