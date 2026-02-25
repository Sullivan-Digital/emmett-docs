---
tags:
  - reference
aliases:
  - Constants
  - Default Values
related:
  - "[[Utility Types]]"
  - "[[PostgreSQL Schema]]"
  - "[[Adapter Comparison]]"
  - "[[Concurrency Control]]"
package: emmett
---

# Constants Reference

All notable constants, default values, and magic strings used across Emmett.

---

## Naming Constants

These string constants are used throughout Emmett for naming streams, collections, and partitions.

| Constant | Value | Usage | Package |
|---|---|---|---|
| `emmettPrefix` | `'emt'` | Prefix for all Emmett-managed names | `emmett` |
| `globalTag` | `'global'` | Used for global-scope identifiers | `emmett` |
| `defaultTag` | `'emt:default'` | Default partition name | `emmett` |
| `unknownTag` | `'emt:unknown'` | Fallback stream type when name has no separator | `emmett` |

> [!info]
> These constants are defined in [[Utility Types]] and used by the [[PostgreSQL Schema]], [[SQLite Event Store]], and [[Stream Naming Conventions|stream name parsing]] logic.

---

## Default Stream Versions

The initial/default stream version varies by adapter:

| Adapter | Default Version | Constant Name |
|---|---|---|
| [[In-Memory Event Store\|In-Memory]] | `0n` | `InMemoryEventStoreDefaultStreamVersion` |
| [[PostgreSQL Event Store\|PostgreSQL]] | `0n` | `PostgreSQLEventStoreDefaultStreamVersion` |
| [[MongoDB Event Store\|MongoDB]] | `0n` | `MongoDBEventStoreDefaultStreamVersion` |
| [[SQLite Event Store\|SQLite]] | `0n` | `SQLiteEventStoreDefaultStreamVersion` |
| [[EventStoreDB Event Store\|EventStoreDB]] | ==-1n== | `EventStoreDBEventStoreDefaultStreamVersion` |

> [!warning]
> EventStoreDB uses `-1n` to match its native convention. All other adapters use `0n`. See [[Gotchas#EventStoreDB|EventStoreDB Gotchas]].

---

## Concurrency Sentinels

Three special values for the `expectedStreamVersion` parameter on [[Appending Events|appendToStream]]:

| Constant | Meaning | Package |
|---|---|---|
| `NO_CONCURRENCY_CHECK` | Skip version checking entirely | `emmett` |
| `STREAM_EXISTS` | Assert the stream already has at least one event | `emmett` |
| `STREAM_DOES_NOT_EXIST` | Assert this is a new stream (no existing events) | `emmett` |

> [!warning]
> `STREAM_EXISTS` and `STREAM_DOES_NOT_EXIST` are ==not enforced== by the SQLite adapter. See [[Concurrency Control]] and [[Gotchas]].

---

## PostgreSQL Defaults

| Constant | Value | Description |
|---|---|---|
| `DefaultPostgreSQLEventStoreProcessorBatchSize` | `100` | Events per polling batch |
| `DefaultPostgreSQLEventStoreProcessorPullingFrequencyInMs` | `50` | Polling interval (ms) when messages are present |
| `DefaultPostgreSQLProcessorLockPolicy` | `{ type: 'fail' }` | Default lock acquisition policy |
| `PROCESSOR_LOCK_DEFAULT_TIMEOUT_SECONDS` | `300` | Stale lock timeout (5 minutes) |
| Default rebuild lock policy | `{ type: 'retry', retries: 100, minTimeout: 100, maxTimeout: 5000 }` | Aggressive retry for [[Projection Rebuilding\|rebuilds]] |

### PostgreSQL Table Names

| Table | Purpose |
|---|---|
| `emt_streams` | Stream metadata (ID, position, type, partition) |
| `emt_messages` | Event storage (data, metadata, global position) |
| `emt_processors` | Processor checkpoint tracking |
| `emt_projections` | Projection registration and status |

### PostgreSQL Sequences

| Sequence | Purpose |
|---|---|
| `emt_global_message_position` | Monotonically increasing global position |

---

## SQLite Defaults

| Constant | Value | Description |
|---|---|---|
| `DefaultSQLiteEventStoreProcessorBatchSize` | `100` | Events per polling batch |
| `DefaultSQLiteEventStoreProcessorPullingFrequencyInMs` | `50` | Polling interval (ms) |

SQLite uses the same table names as PostgreSQL (`emt_streams`, `emt_messages`, `emt_processors`, `emt_projections`).

---

## EventStoreDB Defaults

| Constant | Value | Description |
|---|---|---|
| `DefaultEventStoreDBEventStoreProcessorBatchSize` | `100` | Events per batch |
| `DefaultEventStoreDBEventStoreProcessorPullingFrequencyInMs` | `50` | Polling interval (ms) |
| `EventStoreDBResubscribeDefaultOptions` | `{ forever: true, minTimeout: 100, factor: 1.5 }` | Default reconnection retry config |

---

## MongoDB Constants

| Constant | Value | Description |
|---|---|---|
| Default version | `0n` | `MongoDBEventStoreDefaultStreamVersion` |
| Checkpoint prefix | `'emt:chkpt:mongodb:'` | Prefix for checkpoint strings |
| Processors collection | `'emt:processors'` | Collection for checkpoint storage |
| Default projection name | `'_default'` | When no projection name is specified |

---

## Consumer & Processor Defaults

| Setting | Default | Applies To |
|---|---|---|
| Processor version | `1` | All adapters |
| Processor partition | `'emt:default'` | PostgreSQL, SQLite |
| Adaptive polling initial delay | `100ms` | PostgreSQL, SQLite |
| Adaptive polling max delay | `1000ms` | PostgreSQL, SQLite |

### Processor ID Patterns

| Processor Type | ID Pattern |
|---|---|
| Projector | `emt:processor:projector:{name}` |
| Workflow | `emt:processor:workflow:{name}` |
| Reactor | User-provided `processorId` |

---

## Retry Defaults

| Context | Retries | Min Timeout | Max Timeout | Factor |
|---|---|---|---|---|
| [[Command Handling]] (version conflict) | 3 | 100ms | -- | 1.5x |
| [[EventStoreDB Reconnection]] | Forever | 100ms | -- | 1.5x |
| [[MongoDB Change Stream Consumer\|MongoDB reconnection]] | Forever | 100ms | -- | 1.5x |
| [[Projection Rebuilding\|PostgreSQL rebuild]] lock | 100 | 100ms | 5000ms | -- |

The `NoRetries` constant disables retries entirely.

---

## Global Subscription Events

| Constant | Value | Description |
|---|---|---|
| `GlobalStreamCaughtUpType` | `'__emt:GlobalStreamCaughtUp'` | Internal event type for subscription coordination |

> [!note]
> Events prefixed with `__emt:` are internal subscription events. Use `isNotInternalEvent()` to filter them out in consumer handlers. See [[Global Subscription Events]].

---

## Checkpoint Format

| Adapter | Checkpoint Format | Example |
|---|---|---|
| PostgreSQL | Bigint as `ProcessorCheckpoint` branded string | `'0000000000000000042'` |
| SQLite | Zero-padded 19-digit string | `'0000000000000000042'` |
| MongoDB | `emt:chkpt:mongodb:{resumeTokenData}:{position}` | `emt:chkpt:mongodb:abc123:42` |
| EventStoreDB | In-memory bigint only | N/A |

The `bigIntProcessorCheckpoint()` function creates a `ProcessorCheckpoint` branded string from a bigint value.

---

## Migration Versions

### PostgreSQL

| Version | Changes |
|---|---|
| `0.38.7` | Initial schema |
| `0.42.0` | Renamed `emt_subscriptions` to `emt_processors` |
| `0.42.0-2` | Additional processor columns |
| `0.43.0` | Latest migration |

### SQLite

| Version | Changes |
|---|---|
| `0.41.0` | Initial schema |
| `0.42.0` | Renamed `emt_subscriptions` to `emt_processors`, checkpoint format conversion |

---

## See Also

- [[Utility Types]] -- Where naming constants are defined
- [[Adapter Comparison]] -- How defaults differ across adapters
- [[Concurrency Control]] -- How the concurrency sentinels are used
- [[PostgreSQL Schema]] -- Full database schema details
- [[Gotchas]] -- Surprising default behaviors
