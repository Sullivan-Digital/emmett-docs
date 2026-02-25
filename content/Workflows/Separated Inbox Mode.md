---
tags:
  - workflow
  - durability
aliases:
  - Separated Inbox
  - Two-Phase Workflow
related:
  - "[[Workflow Processor]]"
  - "[[Workflow Idempotency]]"
package: emmett
---

# Separated Inbox Mode

> [!abstract]
> Separated inbox mode splits workflow processing into two phases: **store** and **process**. This provides durability -- the input is persisted before processing begins, so if the process crashes between phases, the input will be picked up again on restart.

## Enabling Separated Inbox

```typescript
consumer.workflowProcessor({
  workflow: GroupCheckoutWorkflow,
  getWorkflowId: (input) => input.data.groupCheckoutId ?? null,
  separateInputInboxFromProcessing: true,
  inputs: {
    commands: ['InitiateGroupCheckout'],
    events: ['GuestCheckedOut'],
  },
  outputs: {
    commands: ['CheckOut'],
    events: ['GroupCheckoutCompleted'],
  },
});
```

## Two-Phase Processing

### Phase 1: Store

When an ==unprefixed input== arrives (e.g., `GuestCheckedOut`):

1. Store the input in the workflow stream with a [[Workflow Pattern#Event Type Prefixing|prefixed type]] (e.g., `GroupCheckoutWorkflow:GuestCheckedOut`)
2. **Stop** -- no `decide` call happens. The input is persisted but not yet processed.

### Phase 2: Process

On the consumer's **next poll/subscription batch**, the prefixed event appears:

1. The workflow processor recognizes it as a prefixed input (e.g., `GroupCheckoutWorkflow:GuestCheckedOut`)
2. Read the workflow stream and rebuild state
3. Run `decide(input, state)` with the unprefixed input
4. Append outputs to the workflow stream
5. The prefixed input is ==not re-stored== (it is already in the stream from Phase 1)

```
Phase 1 (store):
  GuestCheckedOut -> Store as GroupCheckoutWorkflow:GuestCheckedOut -> Done

Phase 2 (process):
  GroupCheckoutWorkflow:GuestCheckedOut -> decide(input, state) -> Append outputs
```

## Why Use Separated Inbox?

In **regular mode** (default), the input is stored and processed in a single operation. If the process crashes after `decide` but before the stream append completes, the input may be lost.

In **separated inbox mode**, the input is ==persisted first== as a separate step. If a crash occurs:

- **Between Phase 1 and Phase 2**: The prefixed event is already in the stream. On restart, the consumer will deliver it again and Phase 2 will run.
- **During Phase 2**: The prefixed event will be redelivered. [[Workflow Idempotency|Idempotency]] prevents duplicate processing.

> [!tip]
> Use separated inbox mode when you need strong durability guarantees and can tolerate the slight latency of two polling cycles instead of one.

## `canHandle` Expansion

When separated inbox mode is enabled, the processor's `canHandle` list includes ==both original and prefixed type names==. For example, with 4 input types:

| Original Type | Prefixed Type |
|---|---|
| `InitiateGroupCheckout` | `GroupCheckoutWorkflow:InitiateGroupCheckout` |
| `GuestCheckedOut` | `GroupCheckoutWorkflow:GuestCheckedOut` |
| `GuestCheckoutFailed` | `GroupCheckoutWorkflow:GuestCheckoutFailed` |
| `TimeoutGroupCheckout` | `GroupCheckoutWorkflow:TimeoutGroupCheckout` |

This results in 8 entries in `canHandle` (4 original + 4 prefixed).

## Regular Mode Comparison

| Aspect | Regular Mode | Separated Inbox |
|---|---|---|
| **Steps** | 1 (store + process together) | 2 (store, then process on next poll) |
| **Latency** | Lower (single pass) | Higher (two polling cycles) |
| **Durability** | Input can be lost on crash | Input persisted before processing |
| **`canHandle` entries** | Original types only | Original + prefixed types |

## See Also

- [[Workflow Processor]] -- Full processor configuration and execution flow
- [[Workflow Idempotency]] -- Duplicate detection that complements separated inbox
- [[Workflow Pattern]] -- Event type prefixing convention
