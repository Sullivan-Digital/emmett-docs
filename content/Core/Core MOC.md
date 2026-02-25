---
tags:
  - moc
  - core
aliases:
  - Core
related:
  - "[[Event Store MOC]]"
  - "[[Projections MOC]]"
  - "[[Testing MOC]]"
  - "[[Patterns MOC]]"
package: emmett
---

# Core

The `emmett` core package provides the type system, patterns, and utilities that all other Emmett packages build on. Everything here is framework- and database-agnostic -- pure TypeScript types, the [[The Decider Pattern|Decider pattern]], and [[Command Handling|command handling]] infrastructure.

## Type System

- [[Events]] -- Defining and creating events with `Event<Type, Data, MetaData>`, the `event()` factory, and helper types
- [[Commands]] -- Defining commands with `Command<>`, the `command()` factory, and `DefaultCommandMetadata`
- [[Messages]] -- The `Message` union type, `CanHandle`, and the `message()` factory
- [[Recorded Messages]] -- Types for persisted messages: `RecordedMessage`, `ReadEvent`, metadata merging

## The Decider Pattern

- [[The Decider Pattern]] -- Core abstraction: `decide`, `evolve`, `initialState`
- [[Evolve Function]] -- The pure state reducer, its three accepted signatures, and where it is used
- [[Command Handling]] -- Wiring Deciders to event stores via `CommandHandler` and `DeciderCommandHandler`

## Utility Types

- [[Type Branding]] -- Nominal typing with `Brand<K, T>` and `Flavour<K, T>`
- [[Deep Readonly and Mutable]] -- Recursive type transformations
- [[Message Handlers]] -- Handler type taxonomy: single/batch, raw/recorded, with/without context
- [[Utility Types]] -- `DefaultRecord`, `AnyRecord`, `StreamPosition`, `GlobalPosition`, and string constants

## See Also

- [[Event Store MOC]] -- The `EventStore` interface that consumes these types
- [[DeciderSpecification]] -- BDD testing for Deciders
- [[Shopping Cart Example]] -- Complete working example using all core types
- [[Error Hierarchy]] -- `EmmettError`, `IllegalStateError`, and other errors thrown by Deciders
