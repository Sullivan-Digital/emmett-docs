---
tags:
  - home
  - emmett
aliases:
  - Home
  - Index
  - Start Here
package: emmett
---

# Emmett

Emmett is a Node.js/TypeScript event sourcing library built around the [[The Decider Pattern|Decider pattern]]. It provides a common [[Event Store Interface]] with pluggable database adapters, inline and async [[Projections MOC|projections]], a [[Consumer Architecture|consumer/processor]] model for reactive event processing, and [[Web Frameworks MOC|web framework integrations]] for building HTTP APIs with optimistic concurrency.

> [!abstract] What Emmett gives you
> - A type-safe, four-method [[Event Store Interface]] that all adapters implement
> - The [[The Decider Pattern|Decider pattern]] (`decide` / `evolve` / `initialState`) as the core abstraction for aggregate business logic
> - [[Inline Projections Overview|Inline]] and [[Async Projections|async]] projections for building read models
> - [[Workflow Pattern|Workflow orchestration]] for multi-step processes
> - [[Response Helpers|Response helpers]], [[ETag Utilities|ETag utilities]], and [[Problem Details|RFC 7807 problem details]] for web APIs

---

## Getting Started

- [[Event Sourcing Primer]] -- What is event sourcing?
- [[The Decider Pattern]] -- The core abstraction: `decide`, `evolve`, `initialState`
- [[Shopping Cart Example]] -- Complete working example from domain to API
- [[Adapter Comparison]] -- Which database adapter to choose?

---

## Core Library

- [[Core MOC]] -- Types, Decider, command handling
- [[Event Store MOC]] -- The EventStore interface and operations
- [[Projections MOC]] -- Read models from event streams

---

## Database Adapters

- [[PostgreSQL MOC]] -- Full-featured: global position, distributed locks, schema migrations, Pongo projections
- [[MongoDB MOC]] -- Single-document-per-stream, change stream consumers, embedded projections
- [[SQLite MOC]] -- Multi-driver (native sqlite3, Cloudflare D1), lightweight
- [[EventStoreDB MOC]] -- Thin wrapper over `@eventstore/db-client`

> [!tip] Choosing an adapter
> See [[Adapter Comparison]] for a comprehensive feature matrix comparing all adapters.

---

## Processing

- [[Consumers MOC]] -- Async event processing with reactors and projectors
- [[Workflows MOC]] -- Multi-step orchestration via the Workflow pattern

---

## Web Frameworks

- [[Express.js Integration]] -- Full response helpers, ETag support, supertest-based testing
- [[Hono Integration]] -- Lightweight, Context-based handlers
- [[Fastify Integration]] -- Plugin-based, intentionally minimal Emmett integration

---

## Testing

- [[Testing MOC]] -- BDD specs, assertions, E2E feature tests
- [[DeciderSpecification]] -- Given/When/Then for Deciders
- [[Testing Projections]] -- BDD projection testing across all adapters

---

## Errors & Validation

- [[Errors MOC]] -- Error hierarchy and validation helpers
- [[Error Hierarchy]] -- `EmmettError` base class with HTTP-aligned error codes

---

## Utilities

- [[Utilities MOC]] -- JSON serialization, retry, message bus, task processing, hashing, and more

---

## Reference

- [[Adapter Comparison]] -- Feature matrix across all adapters
- [[Gotchas]] -- Caveats and pitfalls to watch for
- [[Stream Naming Conventions]] -- How stream names are formed and parsed
- [[Constants Reference]] -- Default values, prefixes, and named constants
