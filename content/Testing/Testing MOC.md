---
tags:
  - moc
  - testing
aliases:
  - Testing
related:
  - "[[Core MOC]]"
  - "[[Patterns MOC]]"
  - "[[Utilities MOC]]"
package: emmett
---

# Testing

Emmett provides a comprehensive, framework-agnostic testing toolkit. BDD-style specifications for deciders and workflows, an event store wrapper for integration tests, a full assertion library, and shared E2E test suites.

- [[DeciderSpecification]] -- Given/When/Then BDD testing for Deciders
- [[WorkflowSpecification]] -- Given/When/Then BDD testing for Workflows
- [[WrapEventStore]] -- Event store wrapper for tracking appended events in integration tests
- [[Assertions Library]] -- Framework-agnostic assertions: basic, deep comparison, arrays, mock verification
- [[E2E Feature Tests]] -- Shared reusable E2E test suites from `emmett-tests`

## See Also

- [[Testing Projections]] -- BDD-style projection testing across all adapters
- [[API Testing]] -- `ApiSpecification` and `ApiE2ESpecification` for HTTP endpoint tests
- [[The Decider Pattern]] -- The core abstraction tested by `DeciderSpecification`
- [[Workflow Pattern]] -- The orchestration pattern tested by `WorkflowSpecification`
