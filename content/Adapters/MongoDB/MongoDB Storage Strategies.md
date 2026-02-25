---
tags:
  - adapter
  - mongodb
  - storage
aliases:
  - MongoDB Storage Options
related:
  - "[[MongoDB Event Store]]"
  - "[[MongoDB Inline Projections]]"
package: emmett-mongodb
---

# MongoDB Storage Strategies

The MongoDB adapter supports three strategies for organizing event stream documents across MongoDB collections. All strategies share the same [[#Document Structure|single-document-per-stream model]] -- they differ only in which collection a given stream type is stored in.

## COLLECTION_PER_STREAM_TYPE (Default)

Each stream type gets its own collection, named `emt:{streamType}`. This is the default when no `storage` option is provided.

```typescript
// All three are equivalent:
const eventStore = getMongoDBEventStore({ client });

const eventStore = getMongoDBEventStore({
  client,
  storage: 'COLLECTION_PER_STREAM_TYPE',
});

const eventStore = getMongoDBEventStore({
  client,
  storage: {
    type: 'COLLECTION_PER_STREAM_TYPE',
    databaseName: 'my-events-db', // optional override
  },
});
```

With this strategy, a stream named `shopping_cart:abc-123` is stored in collection `emt:shopping_cart`, and a stream named `order:xyz-789` goes into collection `emt:order`.

## SINGLE_COLLECTION

All stream types share one collection. The default collection name is `emt:streams`.

```typescript
// Default collection name (emt:streams):
const eventStore = getMongoDBEventStore({
  client,
  storage: 'SINGLE_COLLECTION',
});

// Custom collection name:
const eventStore = getMongoDBEventStore({
  client,
  storage: {
    type: 'SINGLE_COLLECTION',
    collectionName: 'all_events',
    databaseName: 'my-events-db', // optional
  },
});
```

> [!tip] When to Use Single Collection
> Single collection mode is useful when you want simpler operational management (fewer collections to monitor) or when the [[MongoDB Change Stream Consumer|change stream consumer]] watching `^emt:` collections would otherwise see too many collections.

## CUSTOM

Full control over which collection (and database) each stream type maps to.

```typescript
const eventStore = getMongoDBEventStore({
  client,
  storage: {
    type: 'CUSTOM',
    databaseName: 'fallback-db', // used when collectionFor returns a string
    collectionFor: (streamType: string) => {
      // Return a string (uses databaseName above):
      return `custom_${streamType}`;

      // Or return an object with a per-type database:
      // return {
      //   collectionName: `custom_${streamType}`,
      //   databaseName: 'specific-db',
      // };
    },
  },
});
```

The `collectionFor` callback can return:
- A `string` -- just the collection name (uses `databaseName` from parent options)
- A `MongoDBEventStoreCollectionResolution` object -- `{ collectionName: string; databaseName?: string }`

## Storage Types

```typescript
type MongoDBEventStoreStorageOptions =
  | 'COLLECTION_PER_STREAM_TYPE'   // string shorthand
  | 'SINGLE_COLLECTION'            // string shorthand
  | MongoDBEventStoreSingleCollectionStorageOptions
  | MongoDBEventStoreCollectionPerStreamTypeStorageOptions
  | MongoDBEventStoreCustomStorageOptions;
```

## Document Structure

Regardless of strategy, each stream is stored as a single MongoDB document:

```typescript
{
  streamName: "shopping_cart:abc-123",
  messages: [
    {
      type: "ProductItemAdded",
      data: { productItem: { productId: "shoes-1", quantity: 1, price: 100 } },
      metadata: {
        messageId: "550e8400-e29b-41d4-a716-446655440000",
        streamPosition: 1n,
        streamName: "shopping_cart:abc-123",
        // ...
      },
    },
    // ... more events
  ],
  metadata: {
    streamId: "abc-123",
    streamType: "shopping_cart",
    streamPosition: 1n,    // current version (bigint)
    createdAt: Date,
    updatedAt: Date,
  },
  projections: {
    // inline projection results stored here
  },
}
```

The `projections` field holds results from [[MongoDB Inline Projections|inline projections]], keyed by projection name (default `'_default'`).

## Indexing and Caching

A unique index is automatically created on `streamName` for each collection. Collections are cached internally using `Map` instances, so repeated access to the same stream type does not trigger redundant `createIndex` calls.

> [!warning] 16MB Document Size Limit
> All events for a stream are stored in a ==single MongoDB document==. MongoDB enforces a 16MB document size limit. Long-lived streams with many or large events can exceed this limit. There is no built-in snapshotting or stream splitting. Design your stream boundaries accordingly -- prefer shorter-lived streams or consider periodic stream archival for high-volume use cases.

## Constants

| Constant | Value | Description |
|---|---|---|
| `DefaultMongoDBEventStoreCollectionName` | `'emt:streams'` | Default collection name for `SINGLE_COLLECTION` |
| `MongoDBEventStoreDefaultStreamVersion` | `0n` | Default version for non-existent streams |
