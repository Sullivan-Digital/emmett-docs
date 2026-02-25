---
tags:
  - adapter
  - postgresql
  - schema
aliases:
  - PostgreSQL Database Schema
  - emt_streams
related:
  - "[[PostgreSQL Event Store]]"
  - "[[PostgreSQL Distributed Locking]]"
  - "[[Stream Naming Conventions]]"
package: emmett-postgresql
---

# PostgreSQL Schema

The PostgreSQL adapter creates four tables and one sequence, all prefixed with `emt_`. All tables use PostgreSQL ==list partitioning== on the `partition` column for multi-tenancy support.

## Tables

### emt_streams

Tracks stream metadata and current version for [[Concurrency Control|optimistic concurrency]].

```sql
CREATE TABLE IF NOT EXISTS emt_streams(
    stream_id         TEXT       NOT NULL,
    stream_position   BIGINT     NOT NULL,
    partition         TEXT       NOT NULL DEFAULT 'emt:default',
    stream_type       TEXT       NOT NULL,
    stream_metadata   JSONB      NOT NULL,
    is_archived       BOOLEAN    NOT NULL DEFAULT FALSE,
    PRIMARY KEY (stream_id, partition, is_archived)
) PARTITION BY LIST (partition);
```

| Column | Type | Description |
|---|---|---|
| `stream_id` | `TEXT` | Stream identifier (e.g., `shopping_cart-abc123`) |
| `stream_position` | `BIGINT` | Current version of the stream |
| `partition` | `TEXT` | Partition key (default: `emt:default`) |
| `stream_type` | `TEXT` | Extracted from stream name (portion before first `-`) |
| `stream_metadata` | `JSONB` | Stream-level metadata |
| `is_archived` | `BOOLEAN` | Active/archived flag |

Each partition has `_active` and `_archived` sub-partitions.

### emt_messages

Stores all events (and commands) with global ordering.

```sql
CREATE TABLE IF NOT EXISTS emt_messages(
    stream_position        BIGINT       NOT NULL,
    global_position        BIGINT       DEFAULT nextval('emt_global_message_position'),
    transaction_id         XID8         NOT NULL,
    created                TIMESTAMPTZ  NOT NULL DEFAULT now(),
    is_archived            BOOLEAN      NOT NULL DEFAULT FALSE,
    message_kind           VARCHAR(1)   NOT NULL DEFAULT 'E',
    stream_id              TEXT         NOT NULL,
    partition              TEXT         NOT NULL DEFAULT 'emt:default',
    message_schema_version TEXT         NOT NULL,
    message_id             TEXT         NOT NULL,
    message_type           TEXT         NOT NULL,
    message_data           JSONB        NOT NULL,
    message_metadata       JSONB        NOT NULL,
    PRIMARY KEY (stream_id, stream_position, partition, is_archived)
) PARTITION BY LIST (partition);
```

| Column | Type | Description |
|---|---|---|
| `stream_position` | `BIGINT` | Position within the stream |
| `global_position` | `BIGINT` | Monotonically increasing global position (from sequence) |
| `transaction_id` | `XID8` | PostgreSQL 64-bit transaction ID for visibility checks |
| `created` | `TIMESTAMPTZ` | Timestamp of message creation |
| `is_archived` | `BOOLEAN` | Active/archived flag |
| `message_kind` | `VARCHAR(1)` | `'E'` for Event, `'C'` for Command |
| `stream_id` | `TEXT` | Stream this message belongs to |
| `partition` | `TEXT` | Partition key (default: `emt:default`) |
| `message_schema_version` | `TEXT` | Currently hardcoded to `'1'` |
| `message_id` | `TEXT` | Unique message identifier |
| `message_type` | `TEXT` | Event/command type name |
| `message_data` | `JSONB` | Event/command payload |
| `message_metadata` | `JSONB` | Message metadata |

> [!note] Primary Key Design
> The primary key is `(stream_id, stream_position, partition, is_archived)` -- optimized for stream-level reads. Global reads use `transaction_id` ordering, not `global_position`.

### emt_processors

Tracks [[Checkpointing|processor checkpoints]] and lock state for the [[PostgreSQL Consumer|consumer]] system.

```sql
CREATE TABLE IF NOT EXISTS emt_processors(
    last_processed_transaction_id XID8         NOT NULL,
    version                       INT          NOT NULL DEFAULT 1,
    processor_id                  TEXT         NOT NULL,
    partition                     TEXT         NOT NULL DEFAULT 'emt:default',
    status                        TEXT         NOT NULL DEFAULT 'stopped',
    last_processed_checkpoint     TEXT         NOT NULL,
    processor_instance_id         TEXT         DEFAULT 'emt:unknown',
    created_at                    TIMESTAMPTZ  NOT NULL DEFAULT now(),
    last_updated                  TIMESTAMPTZ  NOT NULL DEFAULT now(),
    PRIMARY KEY (processor_id, partition, version)
) PARTITION BY LIST (partition);
```

| Column | Type | Description |
|---|---|---|
| `processor_id` | `TEXT` | Processor identifier (e.g., `emt:projector:shoppingCartDetails`) |
| `version` | `INT` | Processor version (default: 1) |
| `partition` | `TEXT` | Partition key |
| `status` | `TEXT` | `'running'` or `'stopped'` |
| `last_processed_checkpoint` | `TEXT` | Zero-padded (19 chars) global position string |
| `last_processed_transaction_id` | `XID8` | Transaction ID of last processed message |
| `processor_instance_id` | `TEXT` | UUID v7 identifying the running instance |
| `last_updated` | `TIMESTAMPTZ` | Used for [[PostgreSQL Distributed Locking|stale lock detection]] |

> [!warning] Checkpoint Format
> Checkpoints are stored as ==19-character zero-padded strings== (e.g., `'0000000000000000042'`) to enable string comparison ordering. This is an internal format -- use `ProcessorCheckpoint` branded types in code.

### emt_projections

Tracks [[Projection Management|projection metadata]] and status.

```sql
CREATE TABLE IF NOT EXISTS emt_projections(
    version       INT          NOT NULL DEFAULT 1,
    type          VARCHAR(1)   NOT NULL,
    name          TEXT         NOT NULL,
    partition     TEXT         NOT NULL DEFAULT 'emt:default',
    kind          TEXT         NOT NULL,
    status        TEXT         NOT NULL,
    definition    JSONB        NOT NULL DEFAULT '{}'::jsonb,
    created_at    TIMESTAMPTZ  NOT NULL DEFAULT now(),
    last_updated  TIMESTAMPTZ  NOT NULL DEFAULT now(),
    PRIMARY KEY (name, partition, version)
) PARTITION BY LIST (partition);
```

| Column | Type | Description |
|---|---|---|
| `name` | `TEXT` | Projection name |
| `version` | `INT` | Projection version (default: 1) |
| `type` | `VARCHAR(1)` | `'i'` for inline, `'a'` for async |
| `kind` | `TEXT` | Projection kind (e.g., `'emt:projections:postgresql:pongo:single_stream'`) |
| `status` | `TEXT` | `'active'`, `'inactive'`, or `'async_processing'` |
| `definition` | `JSONB` | Serialized projection definition (for introspection) |

## Global Sequence

```sql
CREATE SEQUENCE IF NOT EXISTS emt_global_message_position;
```

This sequence provides monotonically increasing global positions across all streams. Each appended message gets a unique global position.

> [!warning] Global Position Gaps
> If an append transaction rolls back, the sequence value is consumed but no message exists at that position. The consumer handles this correctly (it reads by `transaction_id` ordering), but ==do not assume gap-free positions==.

## Key SQL Functions

### emt_append_to_stream

The core append function, called within a transaction. It handles stream creation, optimistic concurrency checks, and message insertion.

```sql
emt_append_to_stream(
    v_message_ids text[],
    v_messages_data jsonb[],
    v_messages_metadata jsonb[],
    v_message_schema_versions text[],
    v_message_types text[],
    v_message_kinds text[],
    v_stream_id text,
    v_stream_type text,
    v_expected_stream_position bigint DEFAULT NULL,
    v_partition text DEFAULT emt_sanitize_name('default_partition')
) RETURNS TABLE (success boolean, next_stream_position bigint,
                 global_positions bigint[], transaction_id xid8)
```

**Logic:**
1. Gets `pg_current_xact_id()` as the transaction ID
2. If `v_expected_stream_position` is `NULL`, reads the current position (or 0 if new stream)
3. Calculates `next_stream_position = expected + array_length(messages)`
4. New stream: `INSERT` into `emt_streams`; existing stream: `UPDATE WHERE stream_position = expected`
5. If `UPDATE` affected 0 rows: returns `success = FALSE` (concurrency conflict)
6. Inserts all messages into `emt_messages` with incrementing `stream_position` values
7. Returns array of `global_positions` from the inserted messages

> [!note]
> The `beforeCommitHook` (used by [[PostgreSQL Projections|inline projections]]) is called AFTER the SQL insert but BEFORE the transaction commits.

### store_processor_checkpoint

Stores processor progress with optimistic concurrency on the checkpoint value.

**Return codes:**
| Code | Meaning |
|---|---|
| `1` | Successfully updated/inserted |
| `0` | Idempotent (position already at this value) |
| `2` | Mismatch (`check_position` does not match current) |
| `3` | Current ahead (another process has progressed further) |

### Lock Functions

- `emt_try_acquire_processor_lock` -- Acquires an exclusive advisory lock for a processor
- `emt_release_processor_lock` -- Releases the lock and resets status
- `emt_try_acquire_projection_lock` -- Acquires a shared lock for inline projection coordination

See [[PostgreSQL Distributed Locking]] for detailed lock mechanics.

### Projection Management Functions

- `emt_register_projection` -- Registers a projection in `emt_projections`
- `emt_activate_projection` -- Sets status to `'active'`
- `emt_deactivate_projection` -- Sets status to `'inactive'`

### Utility Functions

- `emt_sanitize_name` -- Replaces non-alphanumeric characters with underscores
- `emt_add_table_partition` -- Creates a partition for a single table
- `emt_add_partition` -- Creates sub-partitions for all four tables

## Partitioning (Multi-Tenancy)

All four tables use PostgreSQL list partitioning on the `partition` column. The default partition is `emt:default`.

Custom partitions can be added via the `emt_add_partition()` SQL function, which creates sub-partitions for `emt_messages` and `emt_streams` (with active/archived sub-partitions) plus `emt_processors` and `emt_projections` partitions.

## Schema Migration

The adapter includes an ordered migration system that handles schema evolution:

```typescript
await eventStore.schema.migrate(options?);
```

```typescript
type CreateEventStoreSchemaOptions = {
  dryRun?: boolean;
  ignoreMigrationHashMismatch?: boolean;
  migrationTimeoutMs?: number;
};
```

**Schema creation flow:**
1. Lazy initialization on first operation (if `autoMigration !== 'None'`)
2. Memoized (runs once per event store instance)
3. Runs inside a transaction
4. Calls `onBeforeSchemaCreated` hook
5. Runs all SQL migrations via `@event-driven-io/dumbo`'s `runSQLMigrations`
6. Initializes inline projections (calls `projection.init()`)
7. Calls `onAfterSchemaCreated` hook

### Migration History

| Version | Migration | Changes |
|---|---|---|
| 0.38.7 | Events to Messages | Renamed `emt_events` to `emt_messages`, renamed columns, added `message_kind`, renamed sequence |
| 0.42.0 | Subscriptions to Processors | Created `emt_processors` and `emt_projections`, dual-write migration |
| 0.42.0-2 | Lock functions | Added `created_at`/`last_updated` columns, created lock SQL functions |
| 0.43.0 | Cleanup | Drops legacy `emt_subscriptions` table, removes dual-write logic |

Legacy tables can be cleaned up explicitly:

```typescript
import { cleanupLegacySubscriptionTables } from '@event-driven-io/emmett-postgresql';
await cleanupLegacySubscriptionTables(connectionString);
```

## Truncating Data

> [!danger]
> This operation deletes all event store data. Use only for testing or development.

```typescript
await eventStore.schema.dangerous.truncate({
  resetSequences: true,       // reset emt_global_message_position to 1
  truncateProjections: true,  // call truncate() on each registered projection
});
```

This truncates all four tables (`emt_streams`, `emt_messages`, `emt_processors`, `emt_projections`).

## Inspecting the Schema

```typescript
const sql = eventStore.schema.sql();   // returns the full schema SQL string
eventStore.schema.print();              // prints it to console
```

> [!bug] CLI Migration Stub
> The `emmett-postgresql` CLI plugin registers a `migrate` command but it currently does not perform actual migrations. Use `eventStore.schema.migrate()` programmatically.

## See Also

- [[PostgreSQL Event Store]] -- Factory function and connection options
- [[PostgreSQL Distributed Locking]] -- Advisory lock mechanics built on these tables
- [[Stream Naming Conventions]] -- Stream name format and type extraction rules
- [[Constants Reference]] -- Default values like `emt:default`, `emt:unknown`
