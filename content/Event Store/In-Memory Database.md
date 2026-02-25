---
tags:
  - event-store
  - database
aliases:
  - InMemoryDatabase
  - getInMemoryDatabase
related:
  - "[[In-Memory Event Store]]"
  - "[[InMemory Projections]]"
  - "[[Testing Projections]]"
package: emmett
---

# In-Memory Database

The ==`InMemoryDatabase`== is a simple document store used by the [[In-Memory Event Store]] for [[Inline Projections|inline projection]] results. It provides a collection-based API similar to MongoDB.

## Accessing the Database

The [[In-Memory Event Store]] exposes its database directly:

```typescript
const eventStore = getInMemoryEventStore({
  projections: inlineProjections([shoppingCartDetailsProjection]),
});

// After appending events, query the projected data:
const details = await eventStore.database
  .collection<ShoppingCartDetails>('shoppingCartDetails')
  .findOne((doc) => doc._id === 'shoppingCart-123');
```

## Creating a Standalone Database

You can create an in-memory database independently and share it:

```typescript
import { getInMemoryDatabase } from '@event-driven-io/emmett';

const database = getInMemoryDatabase();
const eventStore = getInMemoryEventStore({ database });

// Both the event store projections and your application code share the same database instance
```

## Interface

```typescript
interface InMemoryDatabase {
  collection: <T extends Document>(name: string) => InMemoryDocumentsCollection<T>;
}
```
^inmemory-database-interface

## Collection Methods

```typescript
interface InMemoryDocumentsCollection<T extends Document> {
  handle: (
    id: string,
    handle: DocumentHandler<T>,
    options?: DatabaseHandleOptions,
  ) => Promise<DatabaseHandleResult<T>>;

  findOne: (predicate?: Predicate<T>) => Promise<T | null>;
  find: (predicate?: Predicate<T>) => Promise<T[]>;

  insertOne: (document: OptionalUnlessRequiredIdAndVersion<T>) => Promise<InsertOneResult>;
  deleteOne: (predicate?: Predicate<T>) => Promise<DeleteResult>;
  replaceOne: (
    predicate: Predicate<T>,
    document: WithoutId<T>,
    options?: ReplaceOneOptions,
  ) => Promise<UpdateResult>;
}
```
^inmemory-collection-interface

### The `handle` Method

The `handle` method is the primary way [[Inline Projections|projections]] interact with documents. It handles insert/update/delete logic automatically based on whether the document exists and what the handler returns:

- If the document does not exist and the handler returns a value, the document is inserted
- If the document exists and the handler returns a value, the document is replaced
- If the handler returns `null`, the document is deleted

### Query Methods

- **`findOne(predicate?)`** -- Returns the first matching document, or `null`
- **`find(predicate?)`** -- Returns all matching documents as an array

### Mutation Methods

- **`insertOne(document)`** -- Inserts a new document
- **`deleteOne(predicate?)`** -- Deletes the first matching document
- **`replaceOne(predicate, document, options?)`** -- Replaces the first matching document

## See Also

- [[In-Memory Event Store]] -- The event store that uses this database
- [[InMemory Projections]] -- Projection types that write to this database
- [[Testing Projections]] -- BDD testing helpers that assert against this database
