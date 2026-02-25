---
tags:
  - utility
  - internal
aliases:
  - TaskProcessor
related:
  - "[[In-Process Locking]]"
  - "[[Utilities MOC]]"
package: emmett
---

# Task Processing

`TaskProcessor` is a bounded, group-aware async task queue for managing concurrent work. It is used internally by [[In-Process Locking]] and by consumers/processors.

## Creating a Processor

```typescript
import { TaskProcessor } from '@event-driven-io/emmett';

const processor = new TaskProcessor({
  maxActiveTasks: 10,     // Maximum concurrent tasks
  maxQueueSize: 100,      // Maximum pending queue size
  maxTaskIdleTime: 5000,  // Timeout for queued tasks (ms, optional)
});
```

## Enqueueing Tasks

Every task receives a `TaskContext` with an ==`ack`== callback. You **must** call `ack()` to release the active task slot:

```typescript
const result = await processor.enqueue(async ({ ack }) => {
  const data = await fetchSomething();
  ack(); // Signal that this task's slot can be released
  return data;
});
```

> [!note] The `ack()` callback controls when the task slot is released, NOT when the promise resolves. This separation is intentional -- it enables the [[In-Process Locking|locking mechanism]] where a lock holds the slot until explicitly released.

```typescript
// Task that holds its slot until explicitly released
await processor.enqueue(async ({ ack }) => {
  const resource = await acquireResource();
  // ack() is NOT called here -- slot stays occupied
  storeAckForLater(ack);
  return resource;
});
```

## Task Groups

Tasks with the same `taskGroupId` are serialized -- only one task per group runs at a time. Different groups run concurrently:

```typescript
// These run one at a time (same group)
processor.enqueue(task1, { taskGroupId: 'stream-abc' });
processor.enqueue(task2, { taskGroupId: 'stream-abc' });

// This runs concurrently with the above (different group)
processor.enqueue(task3, { taskGroupId: 'stream-xyz' });
```

## Queue Limits

When the queue is full, `enqueue` rejects immediately with an [[Error Hierarchy|EmmettError]]:

```typescript
try {
  await processor.enqueue(task);
} catch (error) {
  // EmmettError: "Too many pending connections. Please try again later."
}
```

## Waiting for Completion

```typescript
// Wait for all currently queued tasks to finish
await processor.waitForEndOfProcessing();
```

This enqueues a no-op task that immediately acks, effectively waiting for all preceding tasks to complete.

## Types

```typescript
type TaskProcessorOptions = {
  maxActiveTasks: number;
  maxQueueSize: number;
  maxTaskIdleTime?: number;
};

type Task<T> = (context: TaskContext) => Promise<T>;

type TaskContext = {
  ack: () => void;
};

type EnqueueTaskOptions = { taskGroupId?: string };
```

## See Also

- [[In-Process Locking]] -- Built on top of `TaskProcessor`
- [[Utilities MOC]] -- Other utility modules
