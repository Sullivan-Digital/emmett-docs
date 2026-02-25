---
tags:
  - moc
  - errors
aliases:
  - Error Handling
related:
  - "[[Core MOC]]"
  - "[[Problem Details]]"
  - "[[Validation Helpers]]"
package: emmett
---

# Errors

Emmett provides a structured error hierarchy with HTTP-aligned error codes and validation utilities for type checking and input assertion.

- [[Error Hierarchy]] -- `EmmettError` base class and concrete error types (`ConcurrencyError`, `ValidationError`, `IllegalStateError`, `NotFoundError`)
- [[Validation Helpers]] -- Type guards, assertion validators, and date utilities

## See Also

- [[Problem Details]] -- RFC 7807 error response middleware in web frameworks
- [[Concurrency Control]] -- Where `ConcurrencyError` is thrown
- [[The Decider Pattern]] -- Where `IllegalStateError` is thrown in `decide` functions
- [[Message Bus]] -- Where `EmmettError` is thrown for missing command handlers
