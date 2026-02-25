---
tags:
  - reference
  - gotchas
aliases:
  - Caveats
  - Pitfalls
  - Known Issues
related:
  - "[[Adapter Comparison]]"
  - "[[Event Store Interface]]"
  - "[[Concurrency Control]]"
package: emmett
---

# Gotchas

A consolidated list of caveats, surprising behaviors, and common pitfalls across Emmett. Each entry links to the relevant detail note.

---

## Event Store Interface

> [!warning] Stream positions are 1-based, but `from`/`to` are 0-based slice indices
> When calling [[Reading Streams|readStream]], the `from` and `to` options use 0-based indexing (like array slicing), while the stream positions stored in [[Event Metadata]] start at 1. These are two different coordinate systems.

> [!warning] `aggregateStream` returns filtered count, not actual stream version
> When using `from`/`to` filters, the `currentStreamVersion` in the result reflects the count of filtered events, not the true stream version. See [[Aggregating Streams]].

> [!warning] `onAfterCommit` errors are silently swallowed
> [[After-Commit Hooks]] are fire-and-forget. If the hook throws, the error is caught and logged to `console.error`, but the event append has already succeeded. No retry is attempted.

> [!warning] Inline projection failures prevent after-commit hooks
> [[Inline Projections Overview|Inline projections]] run before [[After-Commit Hooks|after-commit hooks]]. If a projection throws, the after-commit hook never executes.

> [!warning] User metadata keys are overwritten by system metadata
> When custom metadata and system metadata have the same key, the system value wins. System metadata includes `messageId`, `streamPosition`, `streamName`, `globalPosition`, and `checkpoint`. See [[Event Metadata]].

> [!warning] `streamExists` treats empty streams as non-existent
> The [[Event Store Interface|streamExists]] method returns `false` for streams that have no events, even if the stream was previously created. This is consistent across all adapters.

> [!warning] The in-memory event store has no built-in locking
> [[In-Memory Event Store]] does not provide any concurrency protection beyond the event loop. Concurrent async operations on the same stream can cause race conditions.

> [!warning] Downcast receives `ReadEvent` (with full metadata), not raw event
> The [[Schema Versioning|downcast]] function receives the event with all metadata already assigned (messageId, streamPosition, etc.), not the raw event as passed by the user.

---

## Concurrency & Versioning

> [!warning] SQLite does not enforce `STREAM_DOES_NOT_EXIST` or `STREAM_EXISTS` sentinels
> On the [[SQLite Event Store]], passing `STREAM_DOES_NOT_EXIST` or `STREAM_EXISTS` as the expected version has ==no effect==. Only exact bigint version matching and `NO_CONCURRENCY_CHECK` work. See [[Concurrency Control]].

> [!warning] EventStoreDB default stream version is `-1n`, not `0n`
> Unlike all other adapters which default to `0n`, the [[EventStoreDB Event Store]] uses ==-1n== as the default stream version. This matches EventStoreDB's native convention.

> [!warning] No built-in upcasting pipeline
> [[Schema Versioning]] uses a single transform function, not a chain. If you need to upcast through multiple versions (V1 -> V2 -> V3), you must compose the transforms manually with a switch/case or function composition.

---

## PostgreSQL

> [!warning] Stream type extraction is simplistic
> Stream types are extracted by splitting on the ==first `-`==. A stream named `my-aggregate-123` yields type `my`, not `my-aggregate`. Use underscores in type names. See [[Stream Naming Conventions]].

> [!warning] Schema migration is lazy and memoized
> [[PostgreSQL Event Store]] runs schema migration on the first operation (not at construction time). The result is memoized, so subsequent calls do not re-run migration. If migration fails on the first call, it will fail on all subsequent calls.

> [!warning] Global position sequence can have gaps
> The `emt_global_message_position` sequence may produce gaps if a transaction rolls back. The sequence increment is not rolled back, leaving unused position numbers. See [[PostgreSQL Schema]].

> [!warning] PostgreSQL has no user-facing after-commit hook
> Unlike MongoDB and In-Memory, the [[PostgreSQL Event Store]] does not expose `onAfterCommit`. It uses an internal `beforeCommitHook` for inline projections instead.

> [!warning] Inline projections pause during async rebuilds
> When a [[Projection Rebuilding|projection rebuild]] is running, inline projections for that projection are skipped. The async processor sets an `async_processing` status, and inline projections check this status via a shared advisory lock. See [[PostgreSQL Distributed Locking]].

> [!warning] Consumer and event store should use separate pools
> The [[PostgreSQL Consumer]] creates its own connection pool internally. Sharing a pool between the event store and consumer can lead to connection exhaustion under load.

> [!warning] Collection versioning adds a `_v{N}` suffix
> [[Pongo Document Projections]] with `version > 0` store data in collections named `{collectionName}_v{N}`. If you change the version, old data remains in the original collection and is not migrated.

---

## MongoDB

> [!warning] 16MB document size limit
> MongoDB stores all events for a stream in a ==single document==. The 16MB BSON document size limit caps how many events a single stream can hold. See [[MongoDB Storage Strategies]].

> [!warning] No global position
> Unlike PostgreSQL and SQLite, the [[MongoDB Event Store]] has ==no global position== tracking. Checkpointing uses resume tokens from change streams instead.

> [!warning] Replica set required for change streams
> The [[MongoDB Change Stream Consumer]] requires a MongoDB replica set (or sharded cluster). A standalone MongoDB instance cannot use change streams and therefore cannot run async consumers.

> [!warning] Consumer watches all `emt:` collections
> The [[MongoDB Change Stream Consumer]] watches all collections matching `^emt:` except `emt:processors`. It cannot be scoped to a specific stream type.

> [!warning] Default projection name is `'_default'`
> If you do not specify a projection name in [[MongoDB Inline Projections|mongoDBInlineProjection]], it defaults to `'_default'`. Registering two projections with the same name throws an `EmmettError`.

> [!warning] Soft deletes, not hard deletes
> Returning `null` from a MongoDB inline projection's `evolve` function sets the projection value to `null` rather than removing it from the document. Query helpers automatically filter these out.

> [!warning] Empty `streamNames` or `streamIds` array returns zero results
> When querying MongoDB inline projections with `find()`, passing an empty array for `streamNames` or `streamIds` returns no results, not all results.

---

## SQLite

> [!warning] Pongo projection kind strings use PostgreSQL naming
> SQLite's Pongo projections internally use kind strings like `'emt:projections:postgresql:pongo:single_stream'`. This is a copy-paste artifact from the PostgreSQL adapter and does not affect functionality.

> [!warning] D1 driver requires an extra read per append
> The [[SQLite Drivers|Cloudflare D1 driver]] performs an additional read query before each append to determine the current stream version, unlike the native sqlite3 driver which handles this in a single transaction.

> [!warning] Empty events array throws
> Calling `appendToStream` with an empty events array throws an error on SQLite. Always ensure you have at least one event to append.

> [!warning] `createdNewStream` may be inaccurate
> The `createdNewStream` flag in the SQLite append result can be incorrect in certain edge cases. Do not rely on it for critical logic.

> [!warning] Workflow processor creates internal event store
> The [[SQLite Consumer]]'s `workflowProcessor()` automatically creates a separate internal [[SQLite Event Store]] instance for storing workflow state. This is not configurable.

---

## EventStoreDB

> [!warning] In-memory processors only
> The [[EventStoreDB Consumer]] uses in-memory processors with ==no durable checkpointing==. Processor positions are lost on application restart.

> [!warning] No workflow processor support
> The EventStoreDB adapter does not support `consumer.workflowProcessor()`. Use PostgreSQL or SQLite for workflow processing.

> [!warning] Errors stop the subscription entirely
> An unhandled error in a processor handler stops the EventStoreDB subscription. The consumer will attempt to reconnect with exponential backoff, but any error in handler code is fatal for that subscription cycle.

> [!warning] Category projections require `resolveLinkTos: true`
> When subscribing to a category projection (`$ce-{category}`), you must set `resolveLinkTos: true` in the subscription options. Without this, you receive link events rather than the original events.

> [!warning] `close()` is the same as `stop()`
> On the [[EventStoreDB Consumer]], `close()` does not release any resources beyond what `stop()` does. The EventStoreDB client itself has no `close()` method.

> [!warning] Sequential processing despite `batchSize`
> The EventStoreDB consumer processes events one at a time through a Node.js Transform stream, regardless of the configured `batchSize`.

---

## Consumers & Processors

> [!warning] `consumer.start()` returns a long-running promise
> Calling `start()` on any consumer returns a promise that resolves only when the consumer stops (or never, for indefinite polling). Do not `await` it unless you want to block. See [[Consumer Architecture]].

> [!warning] `eachBatch` is effectively a no-op on reactors
> While the [[Reactors|reactor]] type accepts `eachBatch`, it is not used in practice. Always use `eachMessage` for reactor handlers.

> [!warning] Processor ID changes reset the checkpoint
> Changing the `processorId`, `version`, or `partition` of a processor effectively creates a new checkpoint entry, causing the processor to re-read all events from the configured `startFrom` position. See [[Checkpointing]].

> [!warning] Consumer must have processors before `start()`
> Calling `start()` on a consumer with no registered processors (reactors, projectors, or workflow processors) results in undefined behavior. Always register at least one processor first.

> [!warning] Adaptive polling starts at 100ms, not the configured frequency
> PostgreSQL and SQLite consumers start polling with a 100ms delay, doubling up to 1000ms when no messages are found, regardless of the `pullingFrequencyInMs` setting. The configured frequency only applies when messages are present.

---

## Projections

> [!warning] `asyncProjections` helper returns `type: 'inline'` instead of `type: 'async'`
> The `projections.async()` helper incorrectly returns projections with `type: 'inline'`. This is a known bug. See [[Async Projections]].

> [!warning] `init()` behavior differs between adapters
> In [[Raw SQL Projections]], the `init()` function runs only once (PostgreSQL uses projection management to track this). On SQLite, `init()` is called on every event store creation and should use `CREATE TABLE IF NOT EXISTS`.

> [!warning] Projection rebuilding is PostgreSQL-only (as a built-in)
> Only PostgreSQL has a dedicated [[Projection Rebuilding|rebuildPostgreSQLProjections()]] function. For other adapters, you must manually create a consumer with `truncateOnStart: true`.

> [!warning] SQLite has no projection locking
> Unlike PostgreSQL, SQLite has no advisory locks for coordinating between inline and async projection processing. Running both simultaneously on the same projection may cause conflicts.

---

## Web Frameworks

> [!warning] Express `on()` wrapper is Express-only
> The `on()` helper that bridges Emmett handlers to Express middleware is specific to `emmett-expressjs`. Hono and Fastify do not have this utility. See [[Express.js Integration]].

> [!warning] Hono `app.onError()` is a singleton
> Hono only allows one global error handler. If you set Emmett's [[Problem Details]] error handler, it replaces any existing `onError` handler (and vice versa).

> [!warning] Express disables its default ETag
> `emmett-expressjs` calls `app.set('etag', false)` to disable Express's built-in ETag generation, replacing it with Emmett's [[ETag Utilities|weak ETag]] based on stream version.

> [!warning] Error response helpers default message has a typo
> The default error message in response helpers is `"Error occured!"` (missing an `r` in "occurred"). This is present in the source code.

> [!warning] Hono response helpers always require `context`
> Unlike Express where response helpers are curried closures, Hono's [[Response Helpers]] require the Hono `Context` object as a parameter.

---

## Type System & Testing

> [!warning] `kind` field is optional in types but required in `RecordedMessage`
> The `kind` field (`'Event'` or `'Command'`) is optional when defining events and commands but is always present on [[Recorded Messages]]. The `event()` and `command()` factory functions set it automatically.

> [!warning] `decide` can return a single event, an array, or an empty array
> The [[The Decider Pattern|Decider's]] `decide` function is flexible in its return type. Returning an empty array means no events are produced (equivalent to `thenNothingHappened()` in tests).

> [!warning] `DeciderSpecification.then()` uses subset matching
> The `then()` assertion in [[DeciderSpecification]] uses ==subset matching==, not strict equality. Extra properties on events will not cause test failures.

> [!warning] `AssertionError` is intentionally misspelled
> The assertion error class in Emmett's [[Assertions Library]] is spelled `AssertionError` (missing an `s`). This is a known typo in the source code, not a documentation error.

> [!warning] `WrapEventStore.setup()` bypasses event tracking
> Events appended via the [[WrapEventStore|setup()]] method are not recorded in `appendedEvents`. Only events appended via the normal `appendToStream` path are tracked.

---

## Utilities

> [!warning] `JSONParser.stringify` converts BigInt to strings with no auto-restore
> [[JSON Serialization]] converts BigInt values to strings during `stringify`, but `parse` does not automatically convert them back. You must provide a custom `reviver` or `map` function. See [[Schema Versioning]] for handling BigInt in events.

> [!warning] `tryAcquire` has an incomplete pending queue check
> The [[In-Process Locking|InProcessLock.tryAcquire()]] method may report the lock as available even when other operations are waiting in the queue. Use `withAcquire` for reliable locking.

> [!warning] `hashText` can produce negative BigInt values
> The [[Hashing|hashText()]] function interprets the first 8 bytes of a SHA-256 hash as a signed 64-bit BigInt, which can be negative. This is used for [[PostgreSQL Distributed Locking|advisory lock]] key generation.

> [!warning] `deepEquals` treats functions in objects as equal
> The [[Deep Equality|deepEquals()]] function considers any two functions within compared objects as equal, regardless of their implementation. `DataView`, `WeakMap`, and `WeakSet` comparisons always return `false`.

---

## See Also

- [[Adapter Comparison]] -- Feature matrix for choosing between adapters
- [[Constants Reference]] -- Default values that may surprise you
- [[Stream Naming Conventions]] -- Stream type extraction caveats
