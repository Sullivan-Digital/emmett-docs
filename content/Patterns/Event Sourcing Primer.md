---
tags:
  - pattern
  - concept
aliases:
  - Event Sourcing
  - ES Basics
related:
  - "[[Event Store Interface]]"
  - "[[The Decider Pattern]]"
  - "[[Projections MOC]]"
package: emmett
---

# Event Sourcing Primer

Event sourcing is a persistence strategy where ==state changes are stored as an immutable sequence of events== rather than overwriting the current state in place. Instead of a mutable row in a database, you have an append-only log of facts.

## Core Ideas

**Events are immutable facts.** Once recorded, events never change. An `ItemAdded` event is a permanent record that an item was added at a specific point in time.

**Streams are ordered sequences.** Events for a single entity (aggregate) are grouped into a stream. A shopping cart's history is its stream: `ItemAdded`, `ItemAdded`, `CartClosed`.

**State is reconstructed via fold.** To get the current state, replay all events through a reducer function ([[Evolve Function]]). This is what [[Aggregating Streams]] does internally:

```typescript
const state = events.reduce(evolve, initialState());
```

**Append-only storage.** New events are appended to the end of the stream. The [[Event Store Interface]] provides ==`appendToStream`== for this, with [[Concurrency Control]] to prevent conflicting writes.

## CQRS: Reads and Writes Separated

Event sourcing naturally leads to **Command Query Responsibility Segregation (CQRS)**:

- **Write side**: Commands go through the [[The Decider Pattern|Decider]], which validates business rules and produces events
- **Read side**: [[Projections MOC|Projections]] transform event streams into optimized read models (denormalized views)

The write model (event stream) and read models (projections) are updated independently, allowing each to be optimized for its purpose.

## Why Event Sourcing?

- **Complete audit trail** -- Every state change is recorded
- **Temporal queries** -- Reconstruct state at any point in time
- **Event-driven architecture** -- Events drive projections, workflows, and integrations
- **Debugging** -- Replay events to understand how state was reached

## Emmett's Approach

Emmett structures event sourcing around:

1. [[Events]] and [[Commands]] as typed messages
2. The [[The Decider Pattern|Decider pattern]] (`decide` / `evolve` / `initialState`) for business logic
3. A common [[Event Store Interface]] implemented by multiple [[Adapters MOC|database adapters]]
4. [[Projections MOC|Projections]] for read models (inline or async)
5. [[Consumers MOC|Consumers]] for async event processing
6. [[Workflows MOC|Workflows]] for multi-step orchestration

> [!tip] Start with the [[Shopping Cart Example]] for a complete working example that ties all these concepts together.
