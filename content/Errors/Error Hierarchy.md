---
tags:
  - errors
  - core
aliases:
  - EmmettError
  - Error Classes
related:
  - "[[Problem Details]]"
  - "[[Concurrency Control]]"
  - "[[Validation Helpers]]"
package: emmett
---

# Error Hierarchy

All Emmett errors extend a common `EmmettError` base class that carries a numeric ==`errorCode`== aligned with HTTP status codes. This makes domain errors directly mappable to HTTP responses via [[Problem Details]] middleware.

## EmmettError (Base Class)

```typescript
import { EmmettError } from '@event-driven-io/emmett';

// String message (defaults to error code 500)
throw new EmmettError('Something went wrong');

// Numeric error code (uses a default message)
throw new EmmettError(404);

// Object with explicit code and message
throw new EmmettError({ errorCode: 400, message: 'Invalid input' });
```

The constructor accepts three forms:
- `string` -- Sets the message, defaults `errorCode` to 500
- `number` -- Sets the `errorCode`, uses a default message
- `{ errorCode: number; message?: string }` -- Explicit code and optional message

### Error Codes

Error codes map directly to HTTP status codes: ^error-codes

```typescript
EmmettError.Codes = {
  ValidationError: 400,      // Bad Request
  IllegalStateError: 403,    // Forbidden
  NotFoundError: 404,        // Not Found
  ConcurrencyError: 412,     // Precondition Failed
  InternalServerError: 500,  // Internal Server Error
};
```

## Concrete Error Classes

### ConcurrencyError (412)

Thrown when an [[Concurrency Control|optimistic concurrency]] check fails:

```typescript
import { ConcurrencyError } from '@event-driven-io/emmett';

throw new ConcurrencyError(
  '5',         // current version (or undefined)
  '3',         // expected version
  'Optional custom message',
);
// Default message: "Expected version 3 does not match current 5"
```

There is also a `ConcurrencyInMemoryDatabaseError` variant used internally by the [[In-Memory Event Store]] for document-level conflicts.

### ValidationError (400)

Thrown when input data fails validation. Also thrown by the [[Validation Helpers|assertion validators]]:

```typescript
import { ValidationError } from '@event-driven-io/emmett';

throw new ValidationError('Email address is required');
// Default message: "Validation Error ocurred during Emmett processing"
```

### IllegalStateError (403)

Thrown when an operation is attempted in an invalid state. This is the error you throw in [[The Decider Pattern|`decide`]] functions:

```typescript
import { IllegalStateError } from '@event-driven-io/emmett';

const decide = (command: AddItem, state: ShoppingCart) => {
  if (state.status === 'closed') {
    throw new IllegalStateError('Cannot add items to a closed cart');
  }
  return { type: 'ItemAdded', data: { item: command.data.item } };
};
```

### NotFoundError (404)

Thrown when a requested entity does not exist:

```typescript
import { NotFoundError } from '@event-driven-io/emmett';

// With type and ID context
throw new NotFoundError({ type: 'ShoppingCart', id: 'cart-123' });
// Message: "ShoppingCart with cart-123 was not found during Emmett processing"

// No arguments (generic message)
throw new NotFoundError();
// Message: "State was not found during Emmett processing"
```

## Static Methods

### EmmettError.mapFrom(error)

Converts any error into an `EmmettError`, preserving the `errorCode` if present:

```typescript
try {
  await someOperation();
} catch (error) {
  const emmettError = EmmettError.mapFrom(error);
  // If error already had an errorCode, it's preserved
  // Otherwise defaults to 500 (InternalServerError)
}
```

### EmmettError.isInstanceOf(error, errorCode?)

Type guard using ==duck typing== (checks for a numeric `errorCode` property). Does **not** use `instanceof`:

```typescript
if (EmmettError.isInstanceOf(error)) {
  // error has a numeric errorCode
  console.log(error.errorCode);
}

if (EmmettError.isInstanceOf(error, EmmettError.Codes.NotFoundError)) {
  // error has errorCode === 404
}
```

> [!tip] Using duck typing instead of `instanceof` makes this work across module boundaries and different package versions. Always prefer `EmmettError.isInstanceOf()` over `instanceof EmmettError`.

## Integration with Web Frameworks

The web framework integrations ([[Express.js Integration]], [[Hono Integration]]) provide [[Problem Details]] middleware that catches `EmmettError` instances and returns RFC 7807 responses:

```typescript
// In an Express route handler, just throw Emmett errors:
const handler = async (req, res) => {
  const result = await handleCommand(eventStore, id, (state) => {
    if (!state) throw new NotFoundError({ type: 'Cart', id });
    if (!state.isOpen) throw new IllegalStateError('Cart is closed');
    if (!req.body.item) throw new ValidationError('Item is required');
    return { type: 'ItemAdded', data: { item: req.body.item } };
  });
  res.status(200).json(result);
};
// The problem details middleware converts these to appropriate HTTP responses
```

## See Also

- [[Validation Helpers]] -- Type guards and assertion validators that throw `ValidationError`
- [[Problem Details]] -- RFC 7807 middleware for automatic error-to-HTTP mapping
- [[Concurrency Control]] -- Where `ConcurrencyError` is thrown
- [[Message Bus]] -- Where `EmmettError` is thrown for missing command handlers
