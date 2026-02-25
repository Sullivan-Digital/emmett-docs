---
tags:
  - utility
  - locking
aliases:
  - InProcessLock
related:
  - "[[Task Processing]]"
  - "[[Utilities MOC]]"
package: emmett
---

# In-Process Locking

`InProcessLock` provides mutual exclusion for async operations within a single process, built on top of [[Task Processing|TaskProcessor]].

```typescript
import { InProcessLock } from '@event-driven-io/emmett';

const lock = InProcessLock();
```

## acquire / release

```typescript
await lock.acquire({ lockId: 'resource-123' });
try {
  // Exclusive access to the resource
  await doWork();
} finally {
  await lock.release({ lockId: 'resource-123' });
}
```

Internally, `acquire()` enqueues a task for the lock's group ID, and the task's `ack` function is stored. The task slot stays occupied until `release()` calls the stored `ack()`.

## withAcquire

A convenience method that acquires, runs the handler, and releases in a `finally` block:

```typescript
const result = await lock.withAcquire(
  async () => {
    // Exclusive access here
    return await processResource();
  },
  { lockId: 'resource-123' },
);
```

This is the recommended way to use locks -- it guarantees release even if the handler throws.

## tryAcquire

Attempts to acquire the lock without blocking. Returns `false` immediately if the lock is already held:

```typescript
const acquired = await lock.tryAcquire({ lockId: 'resource-123' });
if (acquired) {
  try {
    await doWork();
  } finally {
    await lock.release({ lockId: 'resource-123' });
  }
} else {
  console.log('Resource is busy');
}
```

> [!warning] `tryAcquire` currently only checks the held locks map. It does ==not check the pending queue==, so it may return a false positive if a lock is queued but not yet acquired.

## Types

```typescript
type Lock = {
  acquire(options: AcquireLockOptions): Promise<void>;
  tryAcquire(options: AcquireLockOptions): Promise<boolean>;
  release(options: ReleaseLockOptions): Promise<boolean>;
  withAcquire: <Result = unknown>(
    handle: () => Promise<Result>,
    options: AcquireLockOptions,
  ) => Promise<Result>;
};

type AcquireLockOptions = { lockId: string };
type ReleaseLockOptions = { lockId: string };
```

## See Also

- [[Task Processing]] -- The `TaskProcessor` that powers this locking mechanism
- [[PostgreSQL Distributed Locking]] -- Distributed locks for multi-process environments
- [[Utilities MOC]] -- Other utility modules
