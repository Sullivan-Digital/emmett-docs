---
tags:
  - core
  - pattern
aliases:
  - evolve
  - State Reducer
related:
  - "[[The Decider Pattern]]"
  - "[[Aggregating Streams]]"
  - "[[Projection Concepts]]"
  - "[[Events]]"
  - "[[Recorded Messages]]"
package: emmett
---

# Evolve Function

The `evolve` function is a ==pure state reducer== -- it takes the current state and a single event, and returns the new state. It is one of the three components of the [[The Decider Pattern|Decider pattern]] and is also used independently by [[Aggregating Streams|`aggregateStream`]] and [[Projection Concepts|projections]].

## Basic Signature

```typescript
const evolve = (
  state: ShoppingCart,
  event: ShoppingCartEvent,
): ShoppingCart => {
  switch (event.type) {
    case 'ProductItemAddedToShoppingCart': {
      const productItems = state.status === 'Opened'
        ? state.productItems
        : new Map();
      const { productId, quantity } = event.data.productItem;
      const updated = new Map(productItems);
      updated.set(productId, (updated.get(productId) ?? 0) + quantity);
      return { status: 'Opened', productItems: updated };
    }
    case 'ShoppingCartConfirmed':
    case 'ShoppingCartCancelled':
      return { status: 'Closed' };
  }
};
```

## Three Accepted Signatures

The `Evolve` type in the [[Event Store Interface|event store]] accepts three signature variants, giving you flexibility in how much metadata your evolve function consumes:

```typescript
// 1. Raw event (most common for Deciders)
(state: State, event: EventType) => State

// 2. With full recorded metadata
(state: State, event: ReadEvent<EventType, ReadEventMetadataType>) => State

// 3. With default recorded metadata
(state: State, event: ReadEvent<EventType>) => State
```

^evolve-three-signatures

> [!tip] When to use which signature
> - Use the **raw event** signature (1) for most Deciders -- it keeps your business logic independent of persistence concerns.
> - Use the **ReadEvent** signatures (2, 3) when your evolve function needs access to `messageId`, `streamPosition`, `streamName`, or other system metadata from [[Recorded Messages]].

## Where Evolve Is Used

The `evolve` function appears in several contexts across Emmett:

| Context | How it is used |
|---|---|
| [[The Decider Pattern\|Decider]] | Part of `Decider<State, CommandType, StreamEvent>` -- applies events to aggregate state |
| [[Aggregating Streams\|aggregateStream]] | Folds all events in a stream into state using `evolve` and `initialState` |
| [[Command Handling\|CommandHandler]] | Rebuilds state before handler execution, and updates state between multiple handlers |
| [[Projection Concepts\|Projections]] | Inline and async projections use an evolve-like function to update read models |

## Evolve in Projections

Projections use a related but slightly different evolve pattern. The key difference is that projection evolve functions typically accept **nullable state** (the document may not exist yet):

```typescript
// Decider evolve: state is always present (initialized by initialState)
(state: ShoppingCart, event: ShoppingCartEvent) => ShoppingCart

// Projection evolve: state may be null (first event for this document)
(document: ShoppingCartSummary | null, event: ShoppingCartEvent) => ShoppingCartSummary | null
```

See [[Projection Concepts]] for full details on projection evolve patterns.

> [!warning] Evolve must be pure
> The `evolve` function should be a **pure function** with no side effects. It should not make API calls, write to databases, or throw errors. Given the same inputs, it must always produce the same output. All validation and side-effect logic belongs in the [[The Decider Pattern|Decider's]] `decide` function.

## See Also

- [[The Decider Pattern]] -- The overarching pattern that `evolve` is part of
- [[Aggregating Streams]] -- How `aggregateStream` uses `evolve` to rebuild state
- [[Projection Concepts]] -- Evolve patterns for projections (nullable state, deletion via null)
- [[Events]] -- The event types that `evolve` processes
- [[Recorded Messages]] -- `ReadEvent` type for metadata-aware evolve signatures
