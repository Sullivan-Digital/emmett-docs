---
tags:
  - workflow
  - processor
aliases:
  - workflowProcessor
  - consumer.workflowProcessor
related:
  - "[[Workflow Pattern]]"
  - "[[Consumer Architecture]]"
  - "[[PostgreSQL Consumer]]"
  - "[[SQLite Consumer]]"
  - "[[Retry Logic]]"
package: emmett
---

# Workflow Processor

> [!abstract]
> A workflow processor implements stateful, multi-step orchestration by registering a [[Workflow Pattern|Workflow]] on a consumer. It maintains a dedicated event stream in the event store and coordinates inputs and outputs through that stream.

## Creating a Workflow Processor

Register a workflow processor on a consumer with `consumer.workflowProcessor()`:

```typescript
const consumer = postgreSQLEventStoreConsumer({
  connectionString: 'postgresql://localhost:5432/mydb',
});

consumer.workflowProcessor<
  GroupCheckoutInput,
  GroupCheckout,
  GroupCheckoutOutput
>({
  workflow: GroupCheckoutWorkflow,
  getWorkflowId: (input) =>
    (input.data as { groupCheckoutId?: string }).groupCheckoutId ?? null,
  inputs: {
    commands: ['InitiateGroupCheckout', 'TimeoutGroupCheckout'],
    events: ['GuestCheckedOut', 'GuestCheckoutFailed'],
  },
  outputs: {
    commands: ['CheckOut'],
    events: [
      'GroupCheckoutInitiated',
      'GroupCheckoutCompleted',
      'GroupCheckoutFailed',
      'GroupCheckoutTimedOut',
    ],
  },
  retry: { onVersionConflict: true },
});

await consumer.start();
```

## Configuration Options

| Option | Required | Description |
|---|---|---|
| `workflow` | Yes | The [[Workflow Pattern\|Workflow]] definition (decide/evolve/initialState) |
| `getWorkflowId` | Yes | Extracts the workflow instance ID from each input message |
| `inputs` | Yes | `{ commands, events }` -- message types that trigger the workflow |
| `outputs` | Yes | `{ commands, events }` -- message types the workflow produces |
| `retry` | No | Retry configuration for version conflicts (see [[Workflow Idempotency]]) |
| `separateInputInboxFromProcessing` | No | Enable [[Separated Inbox Mode]] for durability |
| `mapWorkflowId` | No | Custom stream name mapping function |
| `processorId` | No | Defaults to `emt:processor:workflow:${workflowName}` |

> [!note]
> The `getWorkflowId` function receives each incoming message and must return a string ID or `null`. If it returns `null`, the message is silently skipped -- no workflow stream is created or read.

## Execution Flow

When a workflow processor receives a message:

1. **Extract `workflowId`** via `getWorkflowId(message)`. If `null`, skip.
2. **Read the workflow stream** (`emt:workflow:${workflowName}:${workflowId}`) and rebuild state via `evolve`.
3. **Check [[Workflow Idempotency|idempotency]]**: if this message's `messageId` was already processed, return early.
4. **Run `decide(input, state)`** to produce output messages.
5. **Store** the input (with a [[Workflow Pattern#Event Type Prefixing|prefixed type]]) and all outputs to the workflow stream.
6. **Return** the output messages.

```
Input message
  -> getWorkflowId() -> null? skip
  -> Read stream, rebuild state
  -> Already processed? return early
  -> decide(input, state)
  -> Append input + outputs to stream
  -> Return outputs
```

## `inputs` and `outputs` Declarations

The `inputs` and `outputs` declarations tell Emmett which message types trigger the workflow and which types it produces:

- **`inputs`** sets up `canHandle` filtering so the processor only receives relevant messages
- **`outputs`** are used to tag output messages with the correct [[Workflow Pattern#WorkflowMessageAction|action metadata]] (`'Sent'` for commands, `'Published'` for events)

```typescript
inputs: {
  commands: ['InitiateGroupCheckout', 'TimeoutGroupCheckout'],
  events: ['GuestCheckedOut', 'GuestCheckoutFailed'],
},
outputs: {
  commands: ['CheckOut'],
  events: ['GroupCheckoutCompleted', 'GroupCheckoutFailed'],
},
```

## Adapter Support

| Adapter | `workflowProcessor()` Support | Notes |
|---|---|---|
| **PostgreSQL** | Yes | Full support with transaction wrapping |
| **SQLite** | Yes | Auto-creates internal message store for workflow streams |
| **MongoDB** | No | Change stream consumer does not support workflows |
| **EventStoreDB** | No | In-memory consumer does not support workflows |

> [!warning]
> The workflow processor requires the handler context to include `{ connection: { messageStore: EventStore } }`. PostgreSQL provides this automatically. SQLite auto-creates an internal message store. Other adapters do not provide this context.

## Internal Implementation Details

> [!note]- Processor type classification
> Despite being built on top of `reactor()`, the workflow processor sets its internal type to `'projector'`. This is intentional for processor classification but may be surprising in logs or debugging.

The workflow processor uses the `WorkflowHandler` internally -- a curried function that encapsulates the full execution flow including stream reading, state aggregation, idempotency checks, and event appending.

## See Also

- [[Workflow Pattern]] -- The `Workflow` type and decide/evolve/initialState
- [[Workflow Idempotency]] -- Duplicate detection and retry on version conflict
- [[Separated Inbox Mode]] -- Two-phase processing for crash resilience
- [[Consumer Architecture]] -- The consumer-processor model
- [[PostgreSQL Consumer]] -- Primary adapter for workflow processing
- [[SQLite Consumer]] -- Alternative adapter with workflow support
