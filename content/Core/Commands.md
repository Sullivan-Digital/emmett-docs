---
tags:
  - core
  - type-system
aliases:
  - Command Type
  - Command Definition
related:
  - "[[Events]]"
  - "[[Messages]]"
  - "[[The Decider Pattern]]"
  - "[[Command Handling]]"
package: emmett
---

# Commands

Commands represent ==intentions to do something== -- requests that may be accepted or rejected. They are named in the imperative mood (e.g., `AddProductItem`, `ConfirmShoppingCart`). While [[Events]] are facts that have already happened, commands are proposals that the [[The Decider Pattern|Decider's]] `decide` function validates against the current state.

## Defining Commands

Use the `Command<Type, Data, MetaData>` generic, which mirrors the [[Events|Event]] type:

```typescript
import type { Command } from '@event-driven-io/emmett';

// Basic command (no custom metadata)
type AddProductItem = Command<
  'AddProductItem',
  { productItem: PricedProductItem }
>;

// Command with custom metadata
type AddProductItemToShoppingCart = Command<
  'AddProductItemToShoppingCart',
  { shoppingCartId: string; productItem: PricedProductItem },
  { clientId: string; now: Date }
>;
```

^command-type-def

**Generic parameters:**

| Parameter | Constraint | Default | Description |
|---|---|---|---|
| `CommandType` | `extends string` | `string` | String literal for the command type |
| `CommandData` | `extends DefaultRecord` | `DefaultRecord` | The command payload |
| `CommandMetaData` | `extends DefaultRecord \| undefined` | `undefined` | Optional metadata |

Commands carry an optional `kind?: 'Command'` discriminant, mirroring [[Events|events]].

Like events, commands can also be defined as plain object types:

```typescript
type OpenShoppingCart = {
  type: 'OpenShoppingCart';
  data: { shoppingCartId: string; clientId: string; now: Date };
};
```

## Key Differences from Event

> [!warning] Double `Readonly` wrapping
> When `CommandMetaData` is `undefined` (no custom metadata), the `data` property gets an extra `Readonly<>` wrapper -- it becomes `Readonly<CommandData>` inside an already `Readonly<{...}>` object. This is a subtle type-level difference from [[Events]], where `data` is only wrapped once.

When no custom metadata is provided, the command type also gains an optional `metadata?: DefaultCommandMetadata` property (see below).

## The `command()` Factory Function

The ==`command()`== factory creates command instances and sets `kind: 'Command'` at runtime:

```typescript
import { command, type Command } from '@event-driven-io/emmett';

type AddProductItem = Command<
  'AddProductItem',
  { productItem: PricedProductItem }
>;

const cmd = command<AddProductItem>(
  'AddProductItem',
  { productItem: { productId: 'p1', quantity: 1, price: 5.0 } }
);
// Result: { type: 'AddProductItem', data: { ... }, kind: 'Command' }
```

The factory uses the same variadic conditional tuple pattern as `event()` -- if the command type has custom metadata, the third argument is required; otherwise, it is optional and typed as `DefaultCommandMetadata`.

## DefaultCommandMetadata

When a command has no custom metadata (`CommandMetaData = undefined`), it still accepts an optional metadata property:

```typescript
type DefaultCommandMetadata = { now: Date };
```

^default-command-metadata

This means any command without explicit custom metadata can optionally carry a `now` timestamp:

```typescript
const cmd = command<AddProductItem>(
  'AddProductItem',
  { productItem: myItem },
  { now: new Date() }  // optional -- DefaultCommandMetadata
);
```

> [!tip] Standardized timestamp
> `DefaultCommandMetadata` provides a convenient, standardized way to pass the current time into your [[The Decider Pattern|Decider's]] `decide` function without defining custom metadata for every command type.

## Command Helper Types

| Type | Description |
|---|---|
| `AnyCommand` | `Command<any, any, any>` -- loosened type for generic contexts |
| `CommandTypeOf<T>` | Extracts the `type` string literal from a command type |
| `CommandDataOf<T>` | Extracts the `data` type |
| `CommandMetaDataOf<T>` | Extracts metadata type; returns `undefined` if no metadata |
| `CreateCommandType<Type, Data, MetaData>` | Alias identical to `Command<>` |

## See Also

- [[Events]] -- The fact counterpart to commands
- [[Messages]] -- The union type encompassing both events and commands
- [[The Decider Pattern]] -- How `decide` validates commands against state
- [[Command Handling]] -- Wiring command processing to the event store
