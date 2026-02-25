---
tags:
  - utility
  - messaging
aliases:
  - MessageBus
  - getInMemoryMessageBus
  - CommandBus
  - EventBus
related:
  - "[[After-Commit Hooks]]"
  - "[[Events]]"
  - "[[Commands]]"
  - "[[Utilities MOC]]"
package: emmett
---

# Message Bus

Emmett provides an in-memory message bus for dispatching commands and publishing events. It distinguishes between commands (single-handler dispatch) and events (multi-handler broadcast), mirroring CQRS semantics.

## Core Interfaces

```typescript
interface CommandSender {
  send<CommandType extends Command>(command: CommandType): Promise<void>;
}

interface EventsPublisher {
  publish<EventType extends Event>(event: EventType): Promise<void>;
}

type ScheduleOptions = { afterInMs: number } | { at: Date };

interface MessageScheduler<CommandOrEvent extends Command | Event> {
  schedule<MessageType extends CommandOrEvent>(
    message: MessageType,
    when?: ScheduleOptions,
  ): void;
}

interface CommandBus extends CommandSender, MessageScheduler<Command> {}
interface EventBus extends EventsPublisher, MessageScheduler<Event> {}
interface MessageBus extends CommandBus, EventBus {}
```

## Creating the Bus

```typescript
import { getInMemoryMessageBus } from '@event-driven-io/emmett';

const messageBus = getInMemoryMessageBus();
```

Returns `MessageBus & EventSubscription & CommandProcessor & ScheduledMessageProcessor`.

## Sending Commands

```typescript
// Register a command handler
messageBus.handle(
  async (command) => {
    console.log('Adding item:', command.data.item);
  },
  'AddItem',
);

// Send a command
await messageBus.send({
  type: 'AddItem',
  data: { item: 'Shoes' },
});
```

A handler can respond to multiple command types:

```typescript
messageBus.handle(
  async (command) => {
    switch (command.type) {
      case 'AddItem': /* ... */ break;
      case 'RemoveItem': /* ... */ break;
    }
  },
  'AddItem',
  'RemoveItem',
);
```

## Publishing Events

```typescript
messageBus.subscribe(
  async (event) => {
    await updateReadModel(event);
  },
  'ItemAdded',
);

messageBus.subscribe(
  async (event) => {
    await sendNotification(event);
  },
  'ItemAdded', // Multiple subscribers for the same event type
);

// Publish -- both subscribers are called
await messageBus.publish({
  type: 'ItemAdded',
  data: { item: 'Shoes' },
});
```

## Commands vs Events

| Aspect | Commands (`send`) | Events (`publish`) |
|---|---|---|
| Handler registration | `handle()` | `subscribe()` |
| Handlers per type | ==Exactly one== | Zero or more |
| Missing handler | Throws `EmmettError` | Silent no-op |
| Duplicate registration | Throws `EmmettError` | Accumulates handlers |
| Semantics | Intent (do something) | Fact (something happened) |

## Scheduling Messages

```typescript
// Schedule with a delay
messageBus.schedule(
  { type: 'SendReminder', data: { cartId: '123' } },
  { afterInMs: 60000 },
);

// Schedule at a specific time
messageBus.schedule(
  { type: 'CartExpired', data: { cartId: '123' } },
  { at: new Date('2025-01-01T00:00:00Z') },
);

// Retrieve and clear all pending scheduled messages
const pending: ScheduledMessage[] = messageBus.dequeue();
```

> [!warning] There is no automatic timer-based dispatch. Your application must call `dequeue()` and process messages at the appropriate time.

## Connecting to the Event Store

The message bus is commonly used with [[After-Commit Hooks]] to publish events after they are persisted:

```typescript
const messageBus = getInMemoryMessageBus();

messageBus.subscribe(
  async (event) => {
    await updateShoppingCartReadModel(event);
  },
  'ItemAdded',
  'CartClosed',
);

// Publish events after a command is handled
const result = await handleCommand(eventStore, id, handler);
for (const event of result.newEvents) {
  await messageBus.publish(event);
}
```

## Command Routing

```typescript
messageBus.handle(
  async (command) => {
    await handleShoppingCartCommand(eventStore, command);
  },
  'AddItem',
  'RemoveItem',
  'CloseCart',
);

messageBus.handle(
  async (command) => {
    await handleOrderCommand(eventStore, command);
  },
  'PlaceOrder',
  'CancelOrder',
);

// Route through the bus
await messageBus.send({ type: 'AddItem', data: { item: 'Shoes' } });
```

## See Also

- [[After-Commit Hooks]] -- `forwardToMessageBus` helper for automatic event publishing
- [[Events]] -- Event type definitions
- [[Commands]] -- Command type definitions
- [[Error Hierarchy]] -- `EmmettError` thrown for missing/duplicate command handlers
