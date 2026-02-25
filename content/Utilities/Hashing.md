---
tags:
  - utility
aliases:
  - hashText
related:
  - "[[PostgreSQL Distributed Locking]]"
  - "[[Utilities MOC]]"
package: emmett
---

# Hashing

`hashText` computes a deterministic BigInt hash from a text string using SHA-256.

```typescript
import { hashText } from '@event-driven-io/emmett';

const hash: bigint = await hashText('some text');
```

Uses `crypto.subtle.digest('SHA-256', ...)` internally and returns the first 8 bytes interpreted as a ==signed 64-bit BigInt== via `BigInt64Array`.

> [!warning] The result can be **negative** because it uses a signed 64-bit interpretation. This is by design -- PostgreSQL advisory locks accept signed bigint values.

## BigInt Utilities

A related utility for normalizing bigint values to strings:

```typescript
import { bigInt } from '@event-driven-io/emmett';

bigInt.toNormalizedString(42n); // "0000000000000000042"
```

Pads to 19 characters with leading zeros, useful for lexicographic sorting of numeric values stored as strings.

## Usage in Emmett

`hashText` is used by [[PostgreSQL Distributed Locking]] to generate advisory lock keys from processor identifiers. The lock key is computed from `{partition}:{name}:{version}`, hashed to a bigint, and passed to `pg_try_advisory_xact_lock`.

## See Also

- [[PostgreSQL Distributed Locking]] -- Primary consumer of `hashText`
- [[Utilities MOC]] -- Other utility modules
