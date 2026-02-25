---
tags:
  - workflow
  - pattern
aliases:
  - Workflow Type
  - WorkflowDefinition
related:
  - "[[The Decider Pattern]]"
  - "[[Workflow Processor]]"
  - "[[Events]]"
  - "[[Commands]]"
package: emmett
---

# Workflow Pattern

> [!abstract]
> A `Workflow` follows the same ==decide/evolve/initialState== pattern as a [[The Decider Pattern|Decider]], but for orchestration rather than aggregate state management. It receives input events and commands, makes decisions based on accumulated state, and produces output events and commands.

## The `Workflow` Type

```typescript
import type { Workflow } from '@event-driven-io/emmett';

type Workflow<
  Input extends AnyEvent | AnyCommand,
  State,
  Output extends AnyEvent | AnyCommand,
  Name extends string = string,
> = {
  name: Name;
  decide: (command: Input, state: State) => WorkflowOutput<Output>;
  evolve: (currentState: State, event: WorkflowEvent<Input | Output>) => State;
  initialState: () => State;
};
```

| Property | Purpose |
|---|---|
| `name` | Identifies the workflow; used in [[#Stream Naming|stream names]] and event type prefixing |
| `decide` | Produces output messages based on input and current state |
| `evolve` | Updates workflow state from both input and output events |
| `initialState` | Returns the starting state for a new workflow instance |

The `WorkflowOutput` type allows returning a single message or an array:

```typescript
type WorkflowOutput<Output extends AnyEvent | AnyCommand | EmmettError> =
  Output | Output[];
```

## Defining a Workflow

> [!example]- Full Group Checkout Workflow
> ```typescript
> import type { Workflow, Event, Command } from '@event-driven-io/emmett';
>
> type GroupCheckoutInput =
>   | Event<'InitiateGroupCheckout', {
>       groupCheckoutId: string; clerkId: string;
>       guestStayAccountIds: string[]; now: Date;
>     }>
>   | Event<'GuestCheckedOut', {
>       guestStayAccountId: string; checkedOutAt: Date;
>       groupCheckoutId: string;
>     }>
>   | Event<'GuestCheckoutFailed', {
>       guestStayAccountId: string; groupCheckoutId: string;
>       reason: string;
>     }>
>   | Command<'TimeoutGroupCheckout', { groupCheckoutId: string }>;
>
> type GroupCheckoutOutput =
>   | Event<'GroupCheckoutInitiated', {
>       groupCheckoutId: string; guestStayAccountIds: string[];
>     }>
>   | Event<'GroupCheckoutCompleted', { groupCheckoutId: string }>
>   | Event<'GroupCheckoutFailed', {
>       groupCheckoutId: string; reason: string;
>     }>
>   | Event<'GroupCheckoutTimedOut', { groupCheckoutId: string }>
>   | Command<'CheckOut', { guestStayAccountId: string }>;
>
> type GroupCheckout = {
>   status: 'NotExisting' | 'Pending' | 'Completed' | 'Failed' | 'TimedOut';
>   guestStayAccountIds: string[];
>   completedCheckouts: Set<string>;
>   failedCheckouts: Set<string>;
> };
>
> const GroupCheckoutWorkflow: Workflow<
>   GroupCheckoutInput,
>   GroupCheckout,
>   GroupCheckoutOutput
> > = {
>   name: 'GroupCheckoutWorkflow',
>   decide: (input, state) => {
>     // Return output events/commands based on input and current state
>     // ...
>   },
>   evolve: (state, event) => {
>     // Update workflow state based on events
>     // ...
>   },
>   initialState: () => ({
>     status: 'NotExisting',
>     guestStayAccountIds: [],
>     completedCheckouts: new Set(),
>     failedCheckouts: new Set(),
>   }),
> };
> ```

Key differences from a Decider:

| Aspect | Decider | Workflow |
|---|---|---|
| **Input** | Commands only | Events **and** commands |
| **Output** | Events only | Events **and** commands |
| **Stream** | Aggregate stream | Dedicated workflow stream (`emt:workflow:...`) |
| **Scope** | Single aggregate | Cross-aggregate orchestration |

## `WorkflowMessageAction`

Messages stored in the workflow stream are tagged with an action indicating their role:

```typescript
type WorkflowMessageAction =
  'InitiatedBy' | 'Received' | 'Sent' | 'Published' | 'Scheduled';
```

| Action | Applied To | When |
|---|---|---|
| `InitiatedBy` | Input event/command | First input that creates the workflow stream |
| `Received` | Input event/command | Subsequent inputs after the stream exists |
| `Sent` | Output command | Commands produced by `decide` |
| `Published` | Output event | Events produced by `decide` |
| `Scheduled` | (reserved) | Defined but not used in current code |

## Stream Naming

Workflow events are stored in a dedicated stream with the naming convention:

```
emt:workflow:${workflowName}:${workflowId}
```

For example: `emt:workflow:GroupCheckoutWorkflow:abc-123`

You can customize the stream name with `mapWorkflowId` on the [[Workflow Processor]]:

```typescript
consumer.workflowProcessor({
  // ...
  mapWorkflowId: (workflowId) => `custom-stream-${workflowId}`,
});
```

## Event Type Prefixing

Input events stored in the workflow stream have their type prefixed with the workflow name:

```
${workflowName}:${originalType}
```

For example: `GroupCheckoutWorkflow:InitiateGroupCheckout`

> [!note]
> This prefixing is ==transparent== to your `evolve` function. During state rebuild, the prefix is stripped before passing events to `evolve`, so your evolve logic works with the original unprefixed type names. Output events retain their original type names.

## See Also

- [[The Decider Pattern]] -- The foundational decide/evolve/initialState pattern
- [[Workflow Processor]] -- Registering and running a workflow on a consumer
- [[Workflow Idempotency]] -- How duplicate inputs are detected and skipped
- [[Separated Inbox Mode]] -- Two-phase processing for crash resilience
- [[WorkflowSpecification]] -- BDD testing for workflow definitions
- [[Stream Naming Conventions]] -- Full stream naming reference
