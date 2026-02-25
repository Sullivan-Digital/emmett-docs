---
tags:
  - core
  - type-system
aliases:
  - DeepReadonly
  - Mutable
related:
  - "[[Utility Types]]"
package: emmett
---

# Deep Readonly and Mutable

Emmett provides two recursive type transformation utilities for controlling mutability at the type level. These are defined in `src/packages/emmett/src/typing/deepReadonly.ts`.

## DeepReadonly\<T\>

Recursively makes ==all properties readonly==, handling nested objects, arrays, maps, sets, and promises:

```typescript
import type { DeepReadonly } from '@event-driven-io/emmett';

type MyState = { items: { name: string }[] };
type Frozen = DeepReadonly<MyState>;
// Frozen = { readonly items: ReadonlyArray<{ readonly name: string }> }
```

^deep-readonly-def

**Type mapping rules:**

| Input Type | Output Type |
|---|---|
| Primitives (`string`, `number`, `boolean`, etc.) | Returned as-is |
| `Date`, `RegExp` | Returned as-is |
| `Array<U>` | `ReadonlyArray<DeepReadonly<U>>` |
| `Map<K, V>` | `ReadonlyMap<DeepReadonly<K>, DeepReadonly<V>>` |
| `Set<M>` | `ReadonlySet<DeepReadonly<M>>` |
| `Promise<U>` | `Promise<DeepReadonly<U>>` |
| Objects | `{ readonly [P in keyof T]: DeepReadonly<T[P]> }` |

> [!note] Where DeepReadonly is used
> Emmett uses `DeepReadonly` internally in event and command types -- the `Readonly<...>` wrapper on [[Events|Event]] and [[Commands|Command]] types ensures that event data is immutable. `DeepReadonly` goes further by recursing into nested structures, which is useful when defining aggregate state types that should be fully frozen.

## Mutable\<T\>

The inverse of `DeepReadonly` -- recursively removes `readonly` from all properties:

```typescript
import type { Mutable } from '@event-driven-io/emmett';

type Thawed = Mutable<Frozen>;
// Thawed = { items: { name: string }[] }
```

^mutable-def

**Type mapping rules:**

| Input Type | Output Type |
|---|---|
| Primitives, Functions | Returned as-is |
| `ReadonlyArray<U>` | `Array<Mutable<U>>` |
| `ReadonlyMap<K, V>` | `Map<Mutable<K>, Mutable<V>>` |
| `ReadonlySet<M>` | `Set<Mutable<M>>` |
| Objects | `{ -readonly [P in keyof T]: Mutable<T[P]> }` |

> [!tip] When to use Mutable
> `Mutable` is exported for consumer use when you need to create mutable copies of event data for processing. It is not used internally in the core codebase, but can be handy when working with event data in contexts that require mutability (e.g., populating a form, building intermediate computation results).

## See Also

- [[Utility Types]] -- Other foundational types in the core package
- [[Events]] -- Event types use `Readonly` wrapping
- [[Commands]] -- Command types use `Readonly` (and sometimes double-`Readonly`) wrapping
