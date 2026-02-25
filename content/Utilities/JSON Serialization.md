---
tags:
  - utility
aliases:
  - JSONParser
  - JSON Parsing
related:
  - "[[Schema Versioning]]"
  - "[[Utilities MOC]]"
package: emmett
---

# JSON Serialization

`JSONParser` provides `stringify` and `parse` with ==BigInt support== and optional transformations.

```typescript
import { JSONParser, ParseError } from '@event-driven-io/emmett';
```

## stringify

```typescript
// Basic usage (handles BigInt automatically)
const json = JSONParser.stringify({ id: 1n, name: 'Alice' });
// '{"id":"1","name":"Alice"}' -- BigInt values become strings

// With a map function to transform before stringifying
const json = JSONParser.stringify(domainObject, {
  map: (obj) => ({ ...obj, timestamp: obj.timestamp.toISOString() }),
});
```

BigInt values are automatically converted to strings via a replacer function.

## parse

```typescript
// Basic parse
const data = JSONParser.parse<MyType>('{"name":"Alice"}');

// With type checking (throws ParseError if check fails)
const data = JSONParser.parse<MyType>(json, {
  typeCheck: (value): value is MyType =>
    typeof value === 'object' && value !== null && 'name' in value,
});

// With a mapper to transform parsed data
const data = JSONParser.parse<RawData, DomainModel>(json, {
  map: (raw) => new DomainModel(raw.name, raw.age),
});

// With a custom reviver for JSON.parse
const data = JSONParser.parse<MyType>(json, {
  reviver: (key, value) => {
    if (key === 'date') return new Date(value as string);
    return value;
  },
});
```

### Types

```typescript
type ParseOptions<From, To = From> = {
  reviver?: (key: string, value: unknown) => unknown;
  map?: Mapper<From, To>;
  typeCheck?: <To>(value: unknown) => value is To;
};

type StringifyOptions<From, To = From> = {
  map?: Mapper<From, To>;
};
```

## BigInt Gotcha

> [!warning] BigInt values are converted to strings during `stringify`, but there is ==no built-in reviver to parse them back==. You must provide your own `reviver` or `map` function to restore BigInt values:
>
> ```typescript
> const data = JSONParser.parse<{ position: bigint }>(json, {
>   reviver: (key, value) => {
>     if (key === 'position' && typeof value === 'string') return BigInt(value);
>     return value;
>   },
> });
> ```

## See Also

- [[Schema Versioning]] -- Date/BigInt deserialization patterns in event upcasting
- [[Utilities MOC]] -- Other utility modules
