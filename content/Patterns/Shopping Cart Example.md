---
tags:
  - pattern
  - example
aliases:
  - Shopping Cart
  - Shopping Cart Domain
related:
  - "[[The Decider Pattern]]"
  - "[[Command Handling]]"
  - "[[Inline Projections]]"
  - "[[Express.js Integration]]"
  - "[[DeciderSpecification]]"
package: emmett
---

# Shopping Cart Example

This note is the single reference point for the Shopping Cart domain used throughout the Emmett docs. Other notes embed sections from here via block references.

## State

The shopping cart has three possible states, modeled as a discriminated union: ^shopping-cart-state

```typescript
type PricedProductItem = {
  productId: string;
  quantity: number;
  price: number;
};

type EmptyShoppingCart = { status: 'Empty' };
type OpenedShoppingCart = {
  status: 'Opened';
  productItems: Map<string, number>;
};
type ClosedShoppingCart = { status: 'Closed' };

type ShoppingCart = EmptyShoppingCart | OpenedShoppingCart | ClosedShoppingCart;
```

## Events

Events represent facts that have happened to the cart: ^shopping-cart-events

```typescript
import type { Event } from '@event-driven-io/emmett';

type ShoppingCartEvent =
  | Event<'ProductItemAddedToShoppingCart', {
      shoppingCartId: string;
      productItem: PricedProductItem;
      addedAt: Date;
    }>
  | Event<'ShoppingCartConfirmed', {
      shoppingCartId: string;
      confirmedAt: Date;
    }>
  | Event<'ShoppingCartCancelled', {
      shoppingCartId: string;
      cancelledAt: Date;
    }>;
```

## Commands

Commands represent intentions to modify the cart: ^shopping-cart-commands

```typescript
import type { Command } from '@event-driven-io/emmett';

type ShoppingCartCommand =
  | Command<'AddProductItemToShoppingCart', {
      shoppingCartId: string;
      productItem: PricedProductItem;
    }, { now: Date }>
  | Command<'ConfirmShoppingCart', {
      shoppingCartId: string;
    }, { now: Date }>
  | Command<'CancelShoppingCart', {
      shoppingCartId: string;
    }, { now: Date }>;
```

## The Decider

The complete [[The Decider Pattern|Decider]] with `decide`, `evolve`, and `initialState`: ^shopping-cart-decider

```typescript
import {
  type Decider,
  IllegalStateError,
} from '@event-driven-io/emmett';

export const decider: Decider<
  ShoppingCart,
  ShoppingCartCommand,
  ShoppingCartEvent
> = {
  decide: (command, state) => {
    switch (command.type) {
      case 'AddProductItemToShoppingCart': {
        if (state.status === 'Closed')
          throw new IllegalStateError('Shopping Cart already closed');
        return {
          type: 'ProductItemAddedToShoppingCart',
          data: {
            shoppingCartId: command.data.shoppingCartId,
            productItem: command.data.productItem,
            addedAt: command.metadata.now,
          },
        };
      }
      case 'ConfirmShoppingCart': {
        if (state.status !== 'Opened')
          throw new IllegalStateError('Shopping Cart is not opened');
        return {
          type: 'ShoppingCartConfirmed',
          data: {
            shoppingCartId: command.data.shoppingCartId,
            confirmedAt: command.metadata.now,
          },
        };
      }
      case 'CancelShoppingCart': {
        if (state.status !== 'Opened')
          throw new IllegalStateError('Shopping Cart is not opened');
        return {
          type: 'ShoppingCartCancelled',
          data: {
            shoppingCartId: command.data.shoppingCartId,
            cancelledAt: command.metadata.now,
          },
        };
      }
    }
  },
  evolve: (state, event) => {
    switch (event.type) {
      case 'ProductItemAddedToShoppingCart': {
        const productItems =
          state.status === 'Opened' ? state.productItems : new Map();
        const { productId, quantity } = event.data.productItem;
        const updated = new Map(productItems);
        updated.set(productId, (updated.get(productId) ?? 0) + quantity);
        return { status: 'Opened', productItems: updated };
      }
      case 'ShoppingCartConfirmed':
      case 'ShoppingCartCancelled':
        return { status: 'Closed' };
    }
  },
  initialState: () => ({ status: 'Empty' }),
};
```

## Wiring to an Event Store

Using [[Command Handling|DeciderCommandHandler]] to connect the decider to an event store: ^shopping-cart-handler

```typescript
import { DeciderCommandHandler } from '@event-driven-io/emmett';

const handle = DeciderCommandHandler({
  ...decider,
  mapToStreamId: (id) => `shopping_cart-${id}`,
});

// Use in an API handler
const result = await handle(eventStore, shoppingCartId, command);
```

## BDD Testing

Testing with [[DeciderSpecification]]: ^shopping-cart-tests

```typescript
import { DeciderSpecification, IllegalStateError } from '@event-driven-io/emmett';
import { describe, it } from 'node:test';

const given = DeciderSpecification.for(decider);

describe('ShoppingCart', () => {
  it('adds a product item to an empty cart', () => {
    given([])
      .when({
        type: 'AddProductItemToShoppingCart',
        data: { shoppingCartId: 'cart-1', productItem },
        metadata: { now },
      })
      .then({
        type: 'ProductItemAddedToShoppingCart',
        data: { shoppingCartId: 'cart-1', productItem, addedAt: now },
      });
  });

  it('rejects adding to a closed cart', () => {
    given([
      {
        type: 'ProductItemAddedToShoppingCart',
        data: { shoppingCartId: 'cart-1', productItem, addedAt: oldTime },
      },
      {
        type: 'ShoppingCartConfirmed',
        data: { shoppingCartId: 'cart-1', confirmedAt: oldTime },
      },
    ])
      .when({
        type: 'AddProductItemToShoppingCart',
        data: { shoppingCartId: 'cart-1', productItem },
        metadata: { now },
      })
      .thenThrows(IllegalStateError);
  });
});
```

> [!info] The [[DeciderSpecification]] uses subset matching in `then()`, so extra properties on actual events will not cause failures. See [[Assertions Library]] for details on `isSubset`.

## Key Patterns Demonstrated

- **Discriminated union state** -- Three states: `Empty`, `Opened`, `Closed`
- **Business rule enforcement** -- `decide` throws [[Error Hierarchy|IllegalStateError]] for invalid operations
- **Immutable state transitions** -- `evolve` returns new state objects, never mutates
- **Exhaustive command handling** -- The `switch` in `decide` covers all command types
- **Timestamp via metadata** -- Commands carry `{ now: Date }` in metadata, mapped to event data
