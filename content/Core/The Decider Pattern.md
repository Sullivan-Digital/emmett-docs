---
tags:
  - core
  - pattern
  - decider
aliases:
  - Decider
  - decide/evolve/initialState
related:
  - "[[Command Handling]]"
  - "[[Evolve Function]]"
  - "[[Events]]"
  - "[[Commands]]"
  - "[[DeciderSpecification]]"
  - "[[Shopping Cart Example]]"
  - "[[Error Hierarchy]]"
package: emmett
---

# The Decider Pattern

The Decider pattern is ==Emmett's core abstraction for modeling aggregate business logic==. It separates concerns into three pure functions:

1. **`decide`** -- Business rules and validation. Takes a command and the current state, returns event(s).
2. **`evolve`** -- State transitions. Applies a single event to the current state, returns new state.
3. **`initialState`** -- Provides the starting state for a new aggregate.

This separation makes business logic easy to test, reason about, and compose independently of any event store or framework.

## Defining a Decider

```typescript
import type { Decider, Command, Event } from '@event-driven-io/emmett';

type Decider<State, CommandType extends Command, StreamEvent extends Event> = {
  decide: (command: CommandType, state: State) => StreamEvent | StreamEvent[];
  evolve: (currentState: State, event: StreamEvent) => State;
  initialState: () => State;
};
```

^decider-type-def

## The `decide` Function

`decide` contains your business logic. It receives a command and the current state, validates the command against the state, and returns the resulting event(s):

```typescript
const decide = (
  command: ShoppingCartCommand,
  state: ShoppingCart,
): ShoppingCartEvent | ShoppingCartEvent[] => {
  const { type } = command;

  switch (type) {
    case 'AddProductItemToShoppingCart':
      return addProductItem(command, state);
    case 'RemoveProductItemFromShoppingCart':
      return removeProductItem(command, state);
    case 'ConfirmShoppingCart':
      return confirm(command, state);
    case 'CancelShoppingCart':
      return cancel(command, state);
    default: {
      const _notExistingCommandType: never = type;
      throw new EmmettError('Unknown command type');
    }
  }
};
```

**Key behaviors:**

- Can return a **single event** or an **array of events**
- Can return an **empty array** `[]` to indicate the command was accepted but nothing happened (no-op)
- Can **throw errors** (typically `IllegalStateError` from the [[Error Hierarchy]]) to reject invalid commands

> [!tip] Exhaustiveness check
> Use the `const _: never = type` pattern in the default case to ensure all command types are handled at compile time. If you add a new command type to the union, TypeScript will immediately flag any `decide` function that does not handle it.

Individual command handlers validate state and produce events:

```typescript
import { IllegalStateError } from '@event-driven-io/emmett';

const addProductItem = (
  command: AddProductItemToShoppingCart,
  state: ShoppingCart,
): ProductItemAddedToShoppingCart => {
  if (state.status === 'Closed')
    throw new IllegalStateError('Shopping Cart already closed');

  const { data: { shoppingCartId, productItem }, metadata } = command;

  return {
    type: 'ProductItemAddedToShoppingCart',
    data: {
      shoppingCartId,
      productItem,
      addedAt: metadata.now,
    },
  };
};
```

## The `evolve` Function

`evolve` is a pure state reducer. It takes the current state and a single event, and returns the new state. See [[Evolve Function]] for full details on the three accepted signatures.

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

## The `initialState` Function

A factory function that returns the starting state for a new aggregate:

```typescript
const initialState = (): ShoppingCart => ({
  status: 'Empty',
});
```

> [!warning] No `getInitialState` helper exists
> There is no `getInitialState` helper function in Emmett. The convention is `initialState: () => State` as a property of the Decider. Call it directly: `decider.initialState()`.

## Complete Example: Shopping Cart

> [!example]- Full Shopping Cart Decider
>
> ```typescript
> import {
>   type Command,
>   type Decider,
>   type Event,
>   IllegalStateError,
> } from '@event-driven-io/emmett';
>
> // -- State --
>
> type EmptyShoppingCart = { status: 'Empty' };
> type OpenedShoppingCart = { status: 'Opened'; productItems: Map<string, number> };
> type ClosedShoppingCart = { status: 'Closed' };
> type ShoppingCart = EmptyShoppingCart | OpenedShoppingCart | ClosedShoppingCart;
>
> // -- Events --
>
> type ShoppingCartEvent =
>   | Event<'ProductItemAddedToShoppingCart', {
>       shoppingCartId: string;
>       productItem: PricedProductItem;
>       addedAt: Date;
>     }>
>   | Event<'ShoppingCartConfirmed', {
>       shoppingCartId: string;
>       confirmedAt: Date;
>     }>
>   | Event<'ShoppingCartCancelled', {
>       shoppingCartId: string;
>       cancelledAt: Date;
>     }>;
>
> // -- Commands --
>
> type ShoppingCartCommand =
>   | Command<'AddProductItemToShoppingCart', {
>       shoppingCartId: string;
>       productItem: PricedProductItem;
>     }, { now: Date }>
>   | Command<'ConfirmShoppingCart', {
>       shoppingCartId: string;
>     }, { now: Date }>
>   | Command<'CancelShoppingCart', {
>       shoppingCartId: string;
>     }, { now: Date }>;
>
> // -- Decider --
>
> export const decider: Decider<
>   ShoppingCart,
>   ShoppingCartCommand,
>   ShoppingCartEvent
> > = {
>   decide: (command, state) => {
>     switch (command.type) {
>       case 'AddProductItemToShoppingCart': {
>         if (state.status === 'Closed')
>           throw new IllegalStateError('Shopping Cart already closed');
>         return {
>           type: 'ProductItemAddedToShoppingCart',
>           data: {
>             shoppingCartId: command.data.shoppingCartId,
>             productItem: command.data.productItem,
>             addedAt: command.metadata.now,
>           },
>         };
>       }
>       case 'ConfirmShoppingCart': {
>         if (state.status !== 'Opened')
>           throw new IllegalStateError('Shopping Cart is not opened');
>         return {
>           type: 'ShoppingCartConfirmed',
>           data: {
>             shoppingCartId: command.data.shoppingCartId,
>             confirmedAt: command.metadata.now,
>           },
>         };
>       }
>       case 'CancelShoppingCart': {
>         if (state.status !== 'Opened')
>           throw new IllegalStateError('Shopping Cart is not opened');
>         return {
>           type: 'ShoppingCartCancelled',
>           data: {
>             shoppingCartId: command.data.shoppingCartId,
>             cancelledAt: command.metadata.now,
>           },
>         };
>       }
>     }
>   },
>   evolve: (state, event) => {
>     switch (event.type) {
>       case 'ProductItemAddedToShoppingCart': {
>         const productItems = state.status === 'Opened'
>           ? state.productItems
>           : new Map();
>         const { productId, quantity } = event.data.productItem;
>         const updated = new Map(productItems);
>         updated.set(productId, (updated.get(productId) ?? 0) + quantity);
>         return { status: 'Opened', productItems: updated };
>       }
>       case 'ShoppingCartConfirmed':
>       case 'ShoppingCartCancelled':
>         return { status: 'Closed' };
>     }
>   },
>   initialState: () => ({ status: 'Empty' }),
> };
> ```

^shopping-cart-decider

## Wiring to an Event Store

Use [[Command Handling|DeciderCommandHandler]] to wire a Decider to an event store. It handles stream reading, state reconstruction, and event appending:

```typescript
import { DeciderCommandHandler } from '@event-driven-io/emmett';

const handle = DeciderCommandHandler({ ...decider });

await handle(eventStore, shoppingCartId, command);
```

## Testing with DeciderSpecification

Use [[DeciderSpecification]] for BDD-style **given/when/then** testing:

```typescript
import { DeciderSpecification } from '@event-driven-io/emmett';

const given = DeciderSpecification.for({ decide, evolve, initialState });

given([])
  .when({
    type: 'AddProductItemToShoppingCart',
    data: { shoppingCartId, productItem },
    metadata: { now },
  })
  .then([{
    type: 'ProductItemAddedToShoppingCart',
    data: { shoppingCartId, productItem, addedAt: now },
  }]);
```

## See Also

- [[Evolve Function]] -- Deep dive into the evolve function and its three signatures
- [[Command Handling]] -- `CommandHandler` and `DeciderCommandHandler` internals
- [[DeciderSpecification]] -- Full BDD testing API
- [[Shopping Cart Example]] -- Complete end-to-end example with API routes and projections
- [[Error Hierarchy]] -- `IllegalStateError` and other errors for rejecting commands
