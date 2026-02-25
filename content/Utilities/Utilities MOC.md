---
tags:
  - moc
  - utility
aliases:
  - Utilities
related:
  - "[[Core MOC]]"
  - "[[Patterns MOC]]"
  - "[[Errors MOC]]"
package: emmett
---

# Utilities

Utility modules in the core `emmett` package for infrastructure concerns: serialization, task processing, locking, messaging, retry, and more.

- [[JSON Serialization]] -- `JSONParser` with BigInt support and optional transforms
- [[Task Processing]] -- `TaskProcessor` bounded async task queue with group serialization
- [[In-Process Locking]] -- `InProcessLock` mutual exclusion built on `TaskProcessor`
- [[Deep Equality]] -- `deepEquals` comprehensive comparison with `Equatable` protocol
- [[Async Retry]] -- `asyncRetry` utility wrapping `async-retry` with filtering
- [[Message Bus]] -- In-memory command dispatch and event broadcasting
- [[Plugin System]] -- Declarative CLI plugin registration
- [[Promise Utilities]] -- `delay` and `asyncAwaiter` for manual promise control
- [[Collection Utilities]] -- Array merge, duplicate detection
- [[Hashing]] -- SHA-256 text hashing to BigInt

## See Also

- [[Retry Logic]] -- Pattern-level view of retry across the library
- [[Graceful Shutdown]] -- `onShutdown` for SIGTERM/SIGINT handling
- [[Error Hierarchy]] -- `EmmettError` thrown by `TaskProcessor` and `MessageBus`
