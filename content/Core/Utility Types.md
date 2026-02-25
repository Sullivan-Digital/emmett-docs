---
tags:
  - core
  - type-system
aliases:
  - DefaultRecord
  - AnyRecord
related:
  - "[[Type Branding]]"
  - "[[Constants Reference]]"
package: emmett
---

# Utility Types

The core `emmett` package exports several foundational types and constants used throughout the type system. These are defined in `src/packages/emmett/src/typing/index.ts`.

## DefaultRecord and AnyRecord

```typescript
type DefaultRecord = Record<string, unknown>;  // Strict -- used as default for event/command data
type AnyRecord = Record<string, any>;           // Loose -- suppresses some TS checks
```

^record-types

`DefaultRecord` is the default constraint for the `Data` and `MetaData` generic parameters on [[Events|Event]], [[Commands|Command]], and [[Messages|Message]] types. `AnyRecord` is the looser variant used in contexts where `unknown` would create excessive type narrowing.

## Stream Position Types

```typescript
type StreamPosition = bigint;   // Position within a single stream
type GlobalPosition = bigint;   // Global ordering across all streams
```

^position-types

`StreamPosition` represents an event's ordinal position within its stream. `GlobalPosition` represents the global ordering across all streams -- supported by PostgreSQL and SQLite adapters but not all backends.

See [[Recorded Messages]] for how these types appear in `RecordedMessageMetadata`.

## NonNullable\<T\>

```typescript
type NonNullable<T> = T extends null | undefined ? never : T;
```

> [!warning] Shadows TypeScript built-in
> Emmett exports its own `NonNullable<T>` that **shadows** TypeScript's built-in `NonNullable`. The behavior is identical, but be aware that importing from `@event-driven-io/emmett` will give you Emmett's version.

## String Constants

```typescript
const emmettPrefix = 'emt';
const globalTag = 'global';
const defaultTag = 'emt:default';   // `${emmettPrefix}:default`
const unknownTag = 'emt:unknown';   // `${emmettPrefix}:unknown`
```

^string-constants

These constants are used internally for tagging and categorizing streams, processors, and partitions. You will see them in stream names, processor IDs, and the internal infrastructure of [[Consumer Architecture|consumers]].

| Constant | Value | Usage |
|---|---|---|
| `emmettPrefix` | `'emt'` | Prefix for internal stream names and identifiers |
| `globalTag` | `'global'` | Tag for global stream subscriptions |
| `defaultTag` | `'emt:default'` | Default partition tag |
| `unknownTag` | `'emt:unknown'` | Fallback tag when stream type cannot be determined |

See [[Constants Reference]] for a comprehensive list of all constants across Emmett packages.

## See Also

- [[Type Branding]] -- `Brand` and `Flavour` for nominal typing
- [[Deep Readonly and Mutable]] -- Recursive type transformations
- [[Events]] -- Uses `DefaultRecord` as the default data constraint
- [[Recorded Messages]] -- Uses `StreamPosition` and `GlobalPosition`
- [[Constants Reference]] -- Full list of constants across all packages
