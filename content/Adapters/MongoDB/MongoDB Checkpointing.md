---
tags:
  - adapter
  - mongodb
  - checkpointing
aliases:
  - MongoDB Checkpoint
related:
  - "[[Checkpointing]]"
  - "[[MongoDB Change Stream Consumer]]"
package: emmett-mongodb
---

# MongoDB Checkpointing

The MongoDB adapter stores processor checkpoints in the `emt:processors` collection. Unlike [[PostgreSQL Event Store|PostgreSQL]] and [[SQLite Event Store|SQLite]] which use global position integers, MongoDB checkpoints encode ==change stream resume tokens== -- opaque binary tokens that allow the change stream to resume from exactly where it left off.

## Checkpoint Format

```
emt:chkpt:mongodb:{resumeTokenData}:{position}
```

For example:
```
emt:chkpt:mongodb:82687E948D000000...:1
```

The checkpoint is a branded string type:

```typescript
type MongoDBCheckpoint = `emt:chkpt:mongodb:${string}:${bigint}`;
```

The `resumeTokenData` portion is the hex-encoded `_data` field from MongoDB's change stream resume token. The `position` is a `bigint` index representing the event's position within the change batch.

## Storage

### Collection and Schema

Checkpoints are stored in the `emt:processors` collection (constant `DefaultProcessotCheckpointCollectionName`):

```typescript
{
  processorId: string,
  partitionId: string,        // default: "emt:default"
  lastProcessedCheckpoint: MongoDBCheckpoint,
  version: number,
}
```

The default partition is `'emt:default'` (constant `defaultTag`).

### Storing Checkpoints

`storeProcessorCheckpoint` uses optimistic concurrency to prevent conflicting updates:

```typescript
const result = await storeProcessorCheckpoint(client, {
  processorId: 'my-processor',
  lastStoredCheckpoint: previousCheckpoint,  // null for first store
  newCheckpoint: newCheckpoint,
  version: 1,
  partition: 'custom-partition',  // optional, defaults to 'emt:default'
  collectionName: 'custom-checkpoints',  // optional
  dbName: 'custom-db',  // optional
});
```

The result is one of three outcomes:

| Result | Meaning |
|---|---|
| `{ success: true, newCheckpoint }` | Stored successfully |
| `{ success: false, reason: 'MISMATCH' }` | Stored checkpoint does not match `lastStoredCheckpoint` (concurrent modification) |
| `{ success: false, reason: 'IGNORED' }` | `newCheckpoint` is same or earlier than what is already stored |

### Reading Checkpoints

```typescript
const result = await readProcessorCheckpoint(client, {
  processorId: 'my-processor',
  partition: 'custom-partition',  // optional
});
// result: { lastCheckpoint: MongoDBCheckpoint | null }
```

Returns `null` if the processor has no stored checkpoint.

## Checkpoint Utilities

The adapter exports several utility functions for working with checkpoints:

| Function | Description |
|---|---|
| `toMongoDBCheckpoint(resumeToken, position)` | Creates a checkpoint string from a resume token and position |
| `toMongoDBCheckpointValues(checkpoint)` | Parses a checkpoint back to `{ resumeToken, position }` |
| `toMongoDBResumeToken(checkpoint)` | Extracts the resume token object `{ _data: string }` |
| `isMongoDBCheckpoint(value)` | Type guard for `MongoDBCheckpoint` |
| `compareTwoMongoDBCheckpoints(a, b)` | Returns `-1`, `0`, or `1` for ordering |
| `compareTwoMongoDBTokens(a, b)` | Compares raw resume tokens via hex buffer comparison |

## Start Position Resolution

When the [[MongoDB Change Stream Consumer|consumer]] starts with multiple processors, `zipMongoDBMessageBatchPullerStartFrom()` determines the earliest checkpoint:

- If any processor's position is `undefined` or `'BEGINNING'`, the result is `'BEGINNING'` (starts from the earliest available oplog position, using a year-2000 timestamp)
- If all positions are `'END'`, the result is `'END'` (starts from now)
- Otherwise, it sorts by checkpoint comparison and returns the earliest

> [!note] `'BEGINNING'` Start Position
> The `'BEGINNING'` start position uses `startAtOperationTime` set to a year-2000 timestamp. In practice, this reads from the earliest point still available in the MongoDB oplog.

## How It Differs From Other Adapters

| Aspect | MongoDB | PostgreSQL / SQLite |
|---|---|---|
| Checkpoint format | Resume token + position | Global position (`bigint`) |
| Storage | `emt:processors` MongoDB collection | `emt_processors` SQL table |
| Ordering | Hex buffer comparison of resume tokens | Numeric comparison |
| Cross-stream ordering | No global ordering | Global sequence provides total ordering |

> [!info]
> See [[Checkpointing]] for the general checkpointing concepts shared across all adapters.
