---
tags:
  - core
  - type-system
aliases:
  - Message Type
related:
  - "[[Events]]"
  - "[[Commands]]"
  - "[[Recorded Messages]]"
  - "[[Message Handlers]]"
package: emmett
---

# Messages

`Message` is the ==union type of [[Commands|Command]] and [[Events|Event]]==. It provides a single type for contexts where code needs to handle both events and commands generically, such as [[Message Handlers|message handlers]], the [[Message Bus]], and [[Projectors|projections]].

## The Message Type

```typescript
type Message<
  Type extends string = string,
  Data extends DefaultRecord = DefaultRecord,
  MetaData extends DefaultRecord | undefined = undefined,
> = Command<Type, Data, MetaData> | Event<Type, Data, MetaData>;
```

^message-type-def

## Helper Types

Helper types mirror those on [[Events]] and [[Commands]]:

| Type | Description |
|---|---|
| `AnyMessage` | `AnyEvent \| AnyCommand` |
| `MessageKindOf<T>` | Extracts `kind` (`'Event'` or `'Command'`) |
| `MessageTypeOf<T>` | Extracts the `type` string literal |
| `MessageDataOf<T>` | Extracts the `data` type |
| `MessageMetaDataOf<T>` | Extracts metadata |

## CanHandle

```typescript
type CanHandle<T extends Message> = MessageTypeOf<T>[];
```

^canhandle-type

An array of message type strings, used by [[Projectors|projections]], [[Reactors|reactors]], and processors to declare which message types they handle:

```typescript
const canHandle: CanHandle<ShoppingCartEvent> = [
  'ProductItemAdded',
  'DiscountApplied',
];
```

> [!note] Resolves to string literal array
> `CanHandle<ShoppingCartEvent>` resolves to `('ProductItemAdded' | 'DiscountApplied')[]` -- an array of the specific string literal types from your event union. This enables type-safe filtering at the processor level.

## The `message()` Factory Function

Creates a message instance with an explicit `kind` parameter:

```typescript
import { message } from '@event-driven-io/emmett';

const msg = message<MyEvent>('Event', 'SomethingHappened', { value: 42 });
```

The first argument is the `kind` (`'Event'` or `'Command'`), followed by `type`, `data`, and optionally `metadata`. In practice, you will typically use the more specific `event()` or `command()` factories from [[Events]] and [[Commands]] respectively.

## See Also

- [[Events]] -- One half of the `Message` union
- [[Commands]] -- The other half
- [[Recorded Messages]] -- Messages after persistence, with system metadata
- [[Message Handlers]] -- The handler type taxonomy that operates on messages
- [[Message Bus]] -- In-memory message bus for commands and events
