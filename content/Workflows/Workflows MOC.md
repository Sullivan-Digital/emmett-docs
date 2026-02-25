---
tags:
  - moc
  - workflow
aliases:
  - Workflows
related:
  - "[[Workflow Pattern]]"
  - "[[Workflow Processor]]"
  - "[[Separated Inbox Mode]]"
  - "[[Workflow Idempotency]]"
  - "[[Consumers MOC]]"
package: emmett
---

# Workflows

Workflows implement ==stateful, multi-step orchestration== using the same decide/evolve/initialState pattern as the [[The Decider Pattern|Decider]]. Where a Decider manages a single aggregate's lifecycle, a workflow coordinates across multiple aggregates or external systems -- routing events, issuing commands, and tracking progress in a dedicated event stream.

## Notes in This Folder

- [[Workflow Pattern]] -- The `Workflow<Input, State, Output>` type, decide/evolve/initialState for orchestration, stream naming, and message action tags
- [[Workflow Processor]] -- `consumer.workflowProcessor()` registration, execution flow, `getWorkflowId`, `inputs`/`outputs`, and adapter support
- [[Separated Inbox Mode]] -- Two-phase durable processing with `separateInputInboxFromProcessing` for crash resilience
- [[Workflow Idempotency]] -- Built-in duplicate detection via `messageId` tracking and retry on version conflict

## See Also

- [[Consumers MOC]] -- The consumer-processor architecture that workflows build on
- [[The Decider Pattern]] -- The decide/evolve/initialState pattern that workflows share
- [[PostgreSQL Consumer]] -- Primary adapter for workflow processing
- [[SQLite Consumer]] -- Also supports `workflowProcessor()`
- [[WorkflowSpecification]] -- BDD testing for workflow definitions
