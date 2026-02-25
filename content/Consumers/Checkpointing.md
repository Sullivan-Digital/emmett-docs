---
tags:
  - consumer
  - checkpointing
aliases:
  - ProcessorCheckpoint
  - Checkpoint
related:
  - "[[Consumer Architecture]]"
  - "[[Reactors]]"
  - "[[Type Branding]]"
  - "[[MongoDB Checkpointing]]"
package: emmett
---

# Checkpointing

> [!abstract]
> Checkpointing tracks each processor's position in the event stream so it can resume from where it left off after restarts. Checkpoints are stored per-adapter and keyed by `(processorId, partition, version)`.

## How Checkpointing Works

1. **On start**: The processor reads its last stored checkpoint
2. **On each message**: After handling a message, the processor stores the new checkpoint (the message's global position)
3. **On restart**: The processor reads the stored checkpoint and skips any messages at or before that position

```
Start -> Read checkpoint -> Process messages -> Store checkpoint -> ...
  |                                                                  |
  +--- On restart, resume from stored position <---------------------+
```

## `ProcessorCheckpoint` Type

A checkpoint is a ==branded string type== (see [[Type Branding]]):

```typescript
type ProcessorCheckpoint = Brand<string, 'ProcessorCheckpoint'>;
```

For adapters that use global position (PostgreSQL, SQLite), it represents a bigint value. For MongoDB, it encodes a change stream resume token.

Helper functions:

| Function | Purpose |
|---|---|
| `bigIntProcessorCheckpoint(value: bigint)` | Creates a checkpoint from a bigint (padded to 19 chars) |
| `parseBigIntProcessorCheckpoint(checkpoint)` | Parses a checkpoint back to bigint |
| `getCheckpoint(message)` | Extracts `message.metadata.checkpoint` |

## Deduplication with `wasMessageHandled()`

Messages are deduplicated using `wasMessageHandled()`, which compares the message's checkpoint against the stored checkpoint:

```typescript
function wasMessageHandled(message, checkpoint): boolean
```

Returns `true` if the message position is less than or equal to the stored position (already processed). This check runs before `canHandle` filtering and the handler call.

## Checkpoint Storage by Adapter

| Adapter | Storage | Table/Collection |
|---|---|---|
| **PostgreSQL** | SQL table | `emt_processors` |
| **SQLite** | SQL table | Same schema as PostgreSQL |
| **MongoDB** | MongoDB collection | `emt:processors` |
| **EventStoreDB** | In-memory only | `emt_processor_checkpoints` |
| **InMemory** | In-memory database | `emt_processor_checkpoints` |

> [!warning] EventStoreDB checkpoints are in-memory only
> ESDB processors lose their position on restart and reprocess all events from the beginning. Plan your handlers to be ==idempotent==.

## Checkpoint Key

Checkpoints are keyed by a composite of three values:

| Component | Default | Description |
|---|---|---|
| `processorId` | (required) | Stable identifier for the processor |
| `partition` | `'emt:default'` | Logical partition for multi-tenant scenarios |
| `version` | `1` | Processor schema version |

## Resetting Checkpoints

You can force a processor to reprocess from the beginning by changing any component of the checkpoint key:

- **Change `processorId`** -- e.g., rename `'cart-projector'` to `'cart-projector-v2'`
- **Change `version`** -- increment from `1` to `2`
- **Change `partition`** -- switch to a different partition value

Any of these changes means no checkpoint exists for the new key combination, so the processor starts fresh.

> [!tip]
> Changing `version` is the cleanest approach for intentional resets -- it preserves the processor's name for monitoring and logging.

## Checkpoint Flow During Message Processing

The full per-message flow inside a processor's `handle()`:

1. Check `wasMessageHandled(message, lastCheckpoint)` -- skip if already processed
2. Upcast message if schema versioning is configured
3. Filter via `canHandle` if set -- skip non-matching types (checkpoint still advances)
4. Call `eachMessage(message, context)` -- run the handler
5. Store new checkpoint via `checkpoints.store()`
6. Update local `lastCheckpoint` if store succeeded
7. Check handler result for `STOP` / `SKIP` signals

> [!note] Store checkpoint result
> The checkpoint store can fail with specific reasons: `IGNORED` (no change needed), `MISMATCH` (concurrent modification), or `CURRENT_AHEAD` (checkpoint already past this position). In all failure cases, the processor continues without updating its local state.

## See Also

- [[Consumer Architecture]] -- How consumers coordinate processor checkpoints
- [[Reactors]] -- Controlling start position with `startFrom`
- [[Type Branding]] -- The `Brand<K, T>` pattern used by `ProcessorCheckpoint`
- [[MongoDB Checkpointing]] -- MongoDB-specific resume token handling
- [[Constants Reference]] -- Default values for partition, version, and other constants
