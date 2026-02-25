---
tags:
  - testing
  - bdd
  - workflow
aliases:
  - WorkflowSpecification
related:
  - "[[Workflow Pattern]]"
  - "[[DeciderSpecification]]"
  - "[[Assertions Library]]"
package: emmett
---

# WorkflowSpecification

`WorkflowSpecification` provides the same Given/When/Then BDD pattern as [[DeciderSpecification]], adapted for testing [[Workflow Pattern|Workflows]]. The key difference is that workflow inputs and outputs can be either events or commands, and the given history consists of `WorkflowEvent` types.

## Basic Usage

```typescript
import { WorkflowSpecification } from '@event-driven-io/emmett';

const given = WorkflowSpecification.for({
  decide: workflowDecide,
  evolve: workflowEvolve,
  initialState: workflowInitialState,
});
```

## Testing Workflows

```typescript
it('sends a command in response to an event', () => {
  given([])
    .when({ type: 'OrderPlaced', data: { orderId: '123' } })
    .then({ type: 'InitiatePayment', data: { orderId: '123' } });
});

it('does nothing when already processed', () => {
  given([
    { type: 'OrderPlaced', data: { orderId: '123' } },
    { type: 'InitiatePayment', data: { orderId: '123' } },
  ])
    .when({ type: 'OrderPlaced', data: { orderId: '123' } })
    .thenNothingHappened();
});

it('rejects invalid input', () => {
  given([])
    .when(invalidInput)
    .thenThrows(ValidationError);
});
```

## API

The API is identical to [[DeciderSpecification]]:

| Method | Description |
|---|---|
| `given(events)` | Set up prior workflow events (single or array) |
| `.when(input)` | Provide the input event or command |
| `.then(outputs)` | Assert the resulting output events/commands |
| `.thenNothingHappened()` | Assert no output was produced |
| `.thenThrows()` | Assert the workflow throws an error |

The same subset matching behavior applies to `then()` -- see the warning in [[DeciderSpecification]].

## See Also

- [[Workflow Pattern]] -- The `Workflow` type and its decide/evolve/initialState
- [[DeciderSpecification]] -- The equivalent BDD spec for Deciders
- [[Assertions Library]] -- Underlying assertion functions
