---
tags:
  - reference
  - convention
aliases:
  - Stream Names
  - Stream Name Format
related:
  - "[[Appending Events]]"
  - "[[Workflow Pattern]]"
  - "[[PostgreSQL Schema]]"
  - "[[MongoDB Storage Strategies]]"
package: emmett
---

# Stream Naming Conventions

Stream names identify the sequence of events for a given entity. The format varies slightly by adapter, but all follow a `{streamType}{separator}{id}` pattern.

---

## Format by Adapter

| Adapter | Separator | Format | Example |
|---|:---:|---|---|
| [[PostgreSQL Event Store\|PostgreSQL]] | `-` | `{streamType}-{id}` | `shopping_cart-abc123` |
| [[SQLite Event Store\|SQLite]] | `-` | `{streamType}-{id}` | `shopping_cart-abc123` |
| [[EventStoreDB Event Store\|EventStoreDB]] | `-` | `{streamType}-{id}` | `shopping_cart-abc123` |
| [[MongoDB Event Store\|MongoDB]] | `:` | `{streamType}:{streamId}` | `shopping_cart:abc123` |

---

## Stream Type Extraction

The stream type is extracted by splitting on the ==first occurrence== of the separator. This means:

| Stream Name | Extracted Type | Extracted ID |
|---|---|---|
| `shopping_cart-abc123` | `shopping_cart` | `abc123` |
| `user-profile-user-42` | `user` | `profile-user-42` |
| `order-2024-01-15-xyz` | `order` | `2024-01-15-xyz` |

> [!warning] Simplistic extraction
> The extraction splits on the first `-` (or `:` for MongoDB). If your stream type itself contains a `-`, the extracted type will be wrong. For example, `my-aggregate-123` extracts type `my` rather than `my-aggregate`. Use underscores in stream type names to avoid this.

---

## Default Stream Type

When the stream name does not contain the separator character, the stream type defaults to ==`emt:unknown`==.

```typescript
// Stream name "noSeparator" → type = "emt:unknown"
```

---

## Workflow Stream Naming

[[Workflow Pattern|Workflows]] use a specific naming convention with the `emt:workflow:` prefix:

```
emt:workflow:{workflowName}:{workflowId}
```

For example:
```
emt:workflow:group_checkout:checkout-42
```

The workflow processor constructs this name automatically from the `name` property of the [[Workflow Pattern|Workflow]] definition and the ID returned by `getWorkflowId`.

---

## Collection Naming (MongoDB)

MongoDB uses stream types to determine which collection stores the stream document:

| Storage Strategy | Collection Name |
|---|---|
| `COLLECTION_PER_STREAM_TYPE` (default) | `emt:{streamType}` |
| `SINGLE_COLLECTION` | `emt:streams` |
| `CUSTOM` | User-defined callback |

See [[MongoDB Storage Strategies]] for details.

---

## Partition Naming (PostgreSQL)

PostgreSQL tables are list-partitioned on a `partition` column. The default partition is ==`emt:default`==. Custom partitions can be added via the `emt_add_partition()` SQL function for multi-tenancy scenarios.

See [[PostgreSQL Schema]] for the full partitioning setup.

---

## Internal Prefixes

Emmett reserves stream names and collection names that start with `emt:` for internal use:

| Prefix | Usage |
|---|---|
| `emt:workflow:` | Workflow streams |
| `emt:processors` | Processor checkpoint storage (MongoDB) |
| `emt:streams` | Default single-collection name (MongoDB) |
| `emt:{streamType}` | Per-type collection naming (MongoDB) |

---

## Processor ID Naming

Processor IDs follow these conventions:

| Processor Type | ID Pattern |
|---|---|
| Projector | `emt:processor:projector:{name}` |
| Workflow | `emt:processor:workflow:{name}` |
| Reactor | User-specified `processorId` (required) |

Changing the `processorId`, `version`, or `partition` effectively resets the [[Checkpointing|checkpoint]], causing the processor to re-read events from the beginning.

---

## See Also

- [[PostgreSQL Schema]] -- Database tables and partitioning
- [[MongoDB Storage Strategies]] -- Collection organization in MongoDB
- [[Workflow Pattern]] -- Workflow stream naming in detail
- [[Checkpointing]] -- How processor IDs relate to checkpoint persistence
- [[Constants Reference]] -- The `emmettPrefix`, `defaultTag`, and `unknownTag` constants
