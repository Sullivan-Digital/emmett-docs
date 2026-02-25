---
tags:
  - reference
  - comparison
aliases:
  - Feature Matrix
  - Adapter Feature Comparison
related:
  - "[[Adapters MOC]]"
  - "[[PostgreSQL MOC]]"
  - "[[MongoDB MOC]]"
  - "[[SQLite MOC]]"
  - "[[EventStoreDB MOC]]"
  - "[[In-Memory Event Store]]"
package: emmett
---

# Adapter Comparison

Emmett's [[Event Store Interface]] is implemented by five adapters: [[PostgreSQL Event Store|PostgreSQL]], [[MongoDB Event Store|MongoDB]], [[SQLite Event Store|SQLite]], [[EventStoreDB Event Store|EventStoreDB]], and the [[In-Memory Event Store]]. Each adapter makes different trade-offs around features, consistency, and operational complexity.

---

## Feature Matrix

| Feature | PostgreSQL | MongoDB | SQLite | EventStoreDB | In-Memory |
|---|:---:|:---:|:---:|:---:|:---:|
| **Default stream version** | `0n` | `0n` | `0n` | ==-1n== | `0n` |
| **Global position** | Yes (sequence) | ==No== | Yes (auto-increment) | Yes (native) | Yes (computed) |
| **Schema management** | Auto-migration | Auto (indexes) | Auto-migration | None needed | N/A |
| **Consumer mechanism** | Polling | Change streams | Polling | Native subscription | N/A |
| **Reactor support** | Yes | Yes | Yes | Yes (in-memory) | N/A |
| **Projector support** | Yes | Yes | Yes | Yes (in-memory) | N/A |
| **Workflow processor** | Yes | ==No== | Yes | ==No== | N/A |
| **Durable checkpointing** | `emt_processors` table | `emt:processors` collection | `emt_processors` table | ==In-memory only== | N/A |
| **Inline projections** | Pongo, Raw SQL, Generic | Embedded in document | Pongo, Raw SQL | None | In-memory collections |
| **Distributed locking** | Advisory locks | N/A | N/A | N/A | N/A |
| **Projection rebuilding** | `rebuildPostgreSQLProjections()` | Manual (truncate + replay) | Manual (truncate + replay) | N/A | N/A |
| **Projection management** | `emt_projections` table | N/A | N/A | N/A | N/A |
| **Event versioning (upcast)** | Yes | Yes | Yes | Yes | Yes |
| **Event versioning (downcast)** | Yes | Yes | Yes | Yes | Yes |
| **After-commit hooks** | ==Internal only== | Yes | No (has `onBeforeCommit`) | No | Yes |
| **Before-commit hooks** | Internal (inline projections) | N/A | Yes (user-configurable) | N/A | N/A |
| **Sessions** | Yes (`withSession`) | N/A | N/A | N/A | N/A |
| **Connection management** | Pool or client | Client or connection string | Driver-based | Client (caller-managed) | N/A |
| **Reconnection** | Adaptive backoff | Exponential backoff | Adaptive backoff | Exponential backoff | N/A |
| **Multi-tenancy** | Table partitioning | N/A | N/A | N/A | N/A |
| **`close()` method** | Yes | Yes (if owns client) | Yes | ==No== | N/A |

---

## Stream Naming

| Adapter | Format | Example |
|---|---|---|
| PostgreSQL | `{streamType}-{id}` | `shopping_cart-abc123` |
| SQLite | `{streamType}-{id}` | `shopping_cart-abc123` |
| EventStoreDB | `{streamType}-{id}` | `shopping_cart-abc123` |
| MongoDB | `{streamType}:{streamId}` | `shopping_cart:abc123` |

> [!info]
> See [[Stream Naming Conventions]] for full details on stream type extraction and special prefixes.

---

## Inline Projection Types

| Projection Type | PostgreSQL | MongoDB | SQLite | In-Memory |
|---|:---:|:---:|:---:|:---:|
| Single-stream document (Pongo) | Yes | N/A | Yes | Yes |
| Multi-stream document (Pongo) | Yes | N/A | Yes | Yes |
| Raw SQL (per-event) | Yes | N/A | Yes | N/A |
| Raw SQL (batch) | Yes | N/A | Yes | N/A |
| Generic (full control) | Yes | N/A | Yes | Yes |
| Embedded in stream document | N/A | ==Yes== | N/A | N/A |

> [!note]
> MongoDB's inline projections are architecturally different from other adapters. Read models are stored inside the stream document itself under a `projections` field, rather than in separate tables or collections. See [[MongoDB Inline Projections]].

---

## Consumer Comparison

| Aspect | PostgreSQL | MongoDB | SQLite | EventStoreDB |
|---|---|---|---|---|
| **Delivery model** | Pull (polling) | Push (change stream) | Pull (polling) | Push (subscription) |
| **Default batch size** | 100 | N/A | 100 | 100 |
| **Default poll frequency** | 50ms | N/A | 50ms | 50ms |
| **Backoff strategy** | Adaptive (100ms-1000ms) | Exponential | Adaptive (100ms-1000ms) | Exponential |
| **Transaction wrapping** | Per-batch (PostgreSQL transaction) | None | Per-batch (SQLite transaction) | None |
| **Processor handler context** | `connection`, `execute`, `messageStore` | `client` | `connection`, `execute` | In-memory only |
| **Requirements** | PostgreSQL database | ==Replica set, MongoDB 5+== | sqlite3 or D1 | EventStoreDB instance |
| **Processor lock support** | Advisory locks | N/A | N/A | N/A |

---

## When to Choose Each Adapter

> [!tip] PostgreSQL
> Best for production systems needing the full feature set: distributed locking, projection management, workflow processors, multi-tenancy via partitioning, and dedicated rebuild tooling. The most battle-tested adapter.

> [!tip] MongoDB
> Good fit if your existing infrastructure uses MongoDB and you need embedded read models with strong read-after-write consistency. Be aware of the ==16MB document size limit== per stream and the requirement for a replica set for async processing.

> [!tip] SQLite
> Ideal for lightweight deployments, serverless (Cloudflare D1), prototyping, or embedded applications. Supports workflow processors and most projection types. No distributed locking.

> [!tip] EventStoreDB
> Choose this if you are already running EventStoreDB. Emmett provides a thin adapter over the native client. No inline projections, no workflow processors, and checkpoints are in-memory only (lost on restart).

> [!tip] In-Memory
> Use for testing and prototyping only. Provides the full [[Event Store Interface]] with no persistence. The `database` property gives direct access to projected data for assertions.

---

## See Also

- [[Adapters MOC]] -- Links to all adapter-specific notes
- [[Event Store Interface]] -- The common interface all adapters implement
- [[Gotchas]] -- Adapter-specific caveats
- [[Constants Reference]] -- Default values per adapter
