---
tags:
  - core
  - type-system
aliases:
  - Brand
  - Flavour
  - Nominal Typing
related:
  - "[[Concurrency Control]]"
  - "[[ETag Utilities]]"
  - "[[Checkpointing]]"
  - "[[Utility Types]]"
package: emmett
---

# Type Branding

TypeScript uses **structural typing** -- two types with the same shape are interchangeable. This is usually convenient, but sometimes you need to prevent accidentally mixing values that have the same base type (e.g., a `UserId` and an `OrderId` are both strings but should never be confused). Emmett provides two nominal type branding utilities to solve this.

## Brand<K, T> -- Strict Branding

```typescript
type Brand<K, T> = K & { readonly __brand: T };
```

^brand-type-def

The `__brand` property is ==required==, which means branded types are **incompatible** with their base types:

```typescript
type UserId = Brand<string, 'UserId'>;
type OrderId = Brand<string, 'OrderId'>;

const userId: UserId = 'abc' as UserId;
const orderId: OrderId = 'abc' as OrderId;

// userId = orderId;         // Type error! Different brands
// const plain: string = userId;  // Type error! Brand is not assignable to string
```

**Used in Emmett for:**

| Branded Type | Base Type | Purpose |
|---|---|---|
| `ProcessorCheckpoint` | `string` | [[Checkpointing\|Checkpoint]] positions for processors |
| `ETag` | `string` | [[ETag Utilities\|HTTP ETag]] values |
| `WeakETag` | `` `W/${string}` `` | [[ETag Utilities\|HTTP Weak ETag]] values |

## Flavour<K, T> -- Soft Branding

```typescript
type Flavour<K, T> = K & { readonly __brand?: T };
```

^flavour-type-def

The `__brand` is ==optional== (`?`), which means base types **can** be assigned to flavoured types, but different flavours remain incompatible with each other:

```typescript
type ExpectedStreamVersionWithValue = Flavour<bigint, 'StreamVersion'>;

// This works -- base type is assignable to a flavour:
const version: ExpectedStreamVersionWithValue = 5n;

// But different flavours don't mix:
type OtherVersion = Flavour<bigint, 'OtherVersion'>;
// const v: OtherVersion = version;  // Type error!
```

> [!tip] When to choose Brand vs Flavour
> - Use **`Brand`** when you want strict isolation -- values must be explicitly cast (e.g., `'abc' as UserId`)
> - Use **`Flavour`** when you want soft protection -- base types can be assigned without casting, but different flavoured types remain distinct

## Flavour in Concurrency Control

Flavour is used for [[Concurrency Control|expected stream versions]] and their sentinel constants:

```typescript
type ExpectedStreamVersionWithValue = Flavour<bigint, 'StreamVersion'>;
type ExpectedStreamVersionGeneral = Flavour<
  'STREAM_EXISTS' | 'STREAM_DOES_NOT_EXIST' | 'NO_CONCURRENCY_CHECK',
  'StreamVersion'
>;
type ExpectedStreamVersion =
  | ExpectedStreamVersionWithValue
  | ExpectedStreamVersionGeneral;
```

^expected-version-types

The sentinel constants use `as` casts:

```typescript
const STREAM_EXISTS = 'STREAM_EXISTS' as ExpectedStreamVersionGeneral;
const STREAM_DOES_NOT_EXIST = 'STREAM_DOES_NOT_EXIST' as ExpectedStreamVersionGeneral;
const NO_CONCURRENCY_CHECK = 'NO_CONCURRENCY_CHECK' as ExpectedStreamVersionGeneral;
```

Because `Flavour` uses optional `__brand`, a plain `bigint` can be passed as an `ExpectedStreamVersionWithValue` without an explicit cast -- which is the desired ergonomic for passing stream version numbers.

## See Also

- [[Concurrency Control]] -- How expected stream versions use `Flavour`
- [[ETag Utilities]] -- How HTTP ETags use `Brand`
- [[Checkpointing]] -- How `ProcessorCheckpoint` uses `Brand`
- [[Utility Types]] -- Other foundational types in the core package
