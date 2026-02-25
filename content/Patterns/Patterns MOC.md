---
tags:
  - moc
  - pattern
aliases:
  - Patterns
related:
  - "[[Core MOC]]"
  - "[[Event Store MOC]]"
  - "[[Utilities MOC]]"
package: emmett
---

# Patterns

Cross-cutting architectural and design patterns used throughout Emmett. These notes focus on the "why" and "how" at a conceptual level, linking to the concrete implementation details in other folders.

- [[Event Sourcing Primer]] -- What is event sourcing and why does Emmett use it?
- [[Shopping Cart Example]] -- Complete end-to-end reference example used across the docs
- [[Optimistic Concurrency]] -- Version-based conflict detection from store to HTTP API
- [[Middleware Pattern]] -- Wrapping command handlers for cross-cutting concerns
- [[Retry Logic]] -- Automatic retry with exponential backoff and error filtering

## See Also

- [[The Decider Pattern]] -- The core business logic abstraction
- [[Command Handling]] -- Wiring deciders to event stores
- [[Error Hierarchy]] -- HTTP-aligned domain errors
- [[Concurrency Control]] -- Event store-level version checking
