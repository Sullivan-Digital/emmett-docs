---
tags:
  - core
  - type-system
aliases:
  - MessageHandler
  - MessageHandlerResult
related:
  - "[[Reactors]]"
  - "[[Projectors]]"
  - "[[Messages]]"
  - "[[Recorded Messages]]"
package: emmett
---

# Message Handlers

Emmett provides a comprehensive type system for message handler functions. These types are used internally by [[Projectors|projections]], [[Reactors|reactors]], the [[Message Bus]], and [[After-Commit Hooks|after-commit hooks]]. In practice, you rarely need to reference these types directly -- they are used to ensure type safety across the processing pipeline.

## MessageHandlerResult

All message handlers return one of three results:

```typescript
type MessageHandlerResult =
  | void                                                    // Success, continue processing
  | { type: 'SKIP'; reason?: string }                      // Skip this message, continue
  | { type: 'STOP'; reason?: string; error?: EmmettError }; // Stop processing
```

^message-handler-result

| Result | Meaning |
|---|---|
| `void` | The message was handled successfully. Continue processing the next message. |
| `SKIP` | Skip this message (e.g., it is irrelevant). Processing continues with the next message. |
| `STOP` | Stop the entire processing pipeline. Optionally include an error for diagnostics. |

## Handler Type Taxonomy

Handlers vary along three dimensions, forming a matrix of types:

| Dimension | Options | Description |
|---|---|---|
| **Cardinality** | Single / Batch | Handle one message or an array of messages |
| **Message form** | Raw / Recorded | Handle the original [[Messages\|Message]] or a [[Recorded Messages\|RecordedMessage]] (with system metadata) |
| **Context** | With / Without | Optionally receive a handler context (e.g., database connection, transaction) |

^handler-taxonomy

### Single Handlers

```typescript
// Raw message, no context
type SingleRawMessageHandlerWithoutContext<MessageType> =
  (message: MessageType) => Promise<MessageHandlerResult> | MessageHandlerResult;

// Recorded message, with context
type SingleRecordedMessageHandlerWithContext<MessageType, MetaData, Context> =
  (message: RecordedMessage<MessageType, MetaData>, context: Context) =>
    Promise<MessageHandlerResult> | MessageHandlerResult;
```

### Batch Handlers

```typescript
// Raw messages, no context
type BatchRawMessageHandlerWithoutContext<MessageType> =
  (messages: MessageType[]) => Promise<MessageHandlerResult> | MessageHandlerResult;

// Recorded messages, with context
type BatchRecordedMessageHandlerWithContext<MessageType, MetaData, Context> =
  (messages: RecordedMessage<MessageType, MetaData>[], context: Context) =>
    Promise<MessageHandlerResult> | MessageHandlerResult;
```

### Combined Types

The combined types union across these dimensions:

| Type | Description |
|---|---|
| `SingleMessageHandler<MessageType, MetaData, Context>` | Union of all single handler variants |
| `BatchMessageHandler<MessageType, MetaData, Context>` | Union of all batch handler variants |
| `MessageHandler<MessageType, MetaData, Context>` | Union of **all** handler types (single + batch) |

The full list of handler types:

- `SingleRawMessageHandlerWithoutContext<MessageType>`
- `SingleRecordedMessageHandlerWithoutContext<MessageType, MetaData>`
- `SingleRawMessageHandlerWithContext<MessageType, Context>`
- `SingleRecordedMessageHandlerWithContext<MessageType, MetaData, Context>`
- `BatchRawMessageHandlerWithoutContext<MessageType>`
- `BatchRecordedMessageHandlerWithoutContext<MessageType, MetaData>`
- `BatchRawMessageHandlerWithContext<MessageType, Context>`
- `BatchRecordedMessageHandlerWithContext<MessageType, MetaData, Context>`

> [!note] All handlers can be sync or async
> Every handler type accepts both synchronous (`MessageHandlerResult`) and asynchronous (`Promise<MessageHandlerResult>`) return values. You do not need separate types for async handlers.

## Where These Types Are Used

| Consumer | Handler Dimension Typically Used |
|---|---|
| [[Reactors]] (`eachMessage`) | Single, recorded, with context |
| [[Reactors]] (`eachBatch`) | Batch, recorded, with context |
| [[Projectors]] | Single, recorded, with context (via [[Projection Concepts\|ProjectionDefinition]]) |
| [[Message Bus]] | Single, raw, without context |
| [[After-Commit Hooks]] | Single, recorded, without context |

## See Also

- [[Reactors]] -- General-purpose message processors using `eachMessage` / `eachBatch`
- [[Projectors]] -- Specialized processors for building read models
- [[Messages]] -- The `Message` type that raw handlers receive
- [[Recorded Messages]] -- The `RecordedMessage` type that recorded handlers receive
