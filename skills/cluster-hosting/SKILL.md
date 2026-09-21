---
description: "Akka.NET cluster hosting — Singletons, ShardRegions, message extractors, and entity routing with Akka.Hosting"
---

# Akka.NET Cluster Hosting

Patterns for registering Cluster Singletons and Shard Regions using Akka.Hosting,
routing messages to sharded entities via marker interfaces and a central message extractor.

## When to use

- Registering a new Singleton or ShardRegion in the actor system setup
- Adding a new sharded entity type
- Creating or extending the message extractor
- Deciding between Singleton, ShardRegion, or plain actor

## Choosing the right pattern

| Pattern | Use case | Lifecycle | Naming |
|---------|----------|-----------|--------|
| **Singleton** | Orchestrators, managers, unique services | Long-lived, one per cluster, auto-failover | `*Manager` |
| **ShardRegion** | Per-entity state, parallel processing | Per-entity, passivatable | `*Worker` |
| **Plain actor** | Utility, child of singleton/shard | Created by parent, no cluster awareness | `*Actor`, `*Bridge` |

## Cluster Singleton

One instance per cluster, auto-failover on node failure.

### Registration

```csharp
builder.WithSingleton<IMarkerInterface>("actor-name",
    (_, _, resolver) => resolver.Props<MyManager>());
```

- Marker interface in Core (e.g. `ISearchManager`, `IDownloadManager`)
- Actor name is the Akka path name — lowercase, hyphenated
- `resolver.Props<T>()` for actors needing DI services
- Resolve at runtime: `Context.GetActor<ISearchManager>()`

### Example — multiple singletons

```csharp
builder
    .WithSingleton<ISearchManager>("search-manager",
        (_, _, resolver) => resolver.Props<SearchManager>())
    .WithSingleton<IDownloadManager>("download-manager",
        (_, _, resolver) => resolver.Props<DownloadManager>())
    .WithSingleton<IRuleSetManager>("ruleset-manager",
        (_, _, resolver) => resolver.Props<RuleSetManager>());
```

## Shard Region

Per-entity actors distributed across cluster nodes. Entities are created on demand
and optionally passivated after idle timeout.

### Registration

```csharp
builder.WithShardRegion<IRegionMarker>("shard-name",
    (_, _, resolver) => _ => resolver.Props<MyWorker>(),
    new ShardMessageExtractor(),
    new ShardOptions { PassivateIdleEntityAfter = TimeSpan.FromSeconds(30) });
```

**The double-lambda explained:**
- Outer: `(system, registry, resolver) =>` — runs once at startup, captures DI
- Inner: `entityId =>` — runs per entity creation, returns Props
- When entityId is NOT needed: `_ => resolver.Props<MyWorker>()`
- When entityId IS needed (e.g. for PersistenceId): `entityId => resolver.Props<MyWorker>(entityId)`

### Example — multiple shard regions

```csharp
builder
    .WithShardRegion<ITvSearchRegion>("tv-search",
        (_, _, resolver) => _ => resolver.Props<TvSearchWorker>(),
        new ShardMessageExtractor(),
        new ShardOptions { PassivateIdleEntityAfter = TimeSpan.FromSeconds(30) })
    .WithShardRegion<IDownloadRegion>("download-worker",
        (_, _, resolver) => _ => resolver.Props<DownloadWorker>(),
        new ShardMessageExtractor(),
        new ShardOptions())
    .WithShardRegion<IHistoryRegion>("history",
        (_, _, resolver) => entityId => resolver.Props<HistoryWorker>(entityId),
        new ShardMessageExtractor(),
        new ShardOptions());
```

### ShardOptions

| Option | Purpose | Guidance |
|--------|---------|----------|
| `PassivateIdleEntityAfter` | Auto-stop idle entities after timeout | Short-lived workers: 30s. Long-lived/persistent: omit or longer |
| `RememberEntities` | Re-create entities after rebalance | Use only when entity must survive shard moves |

## Shard Message Extractor

One central extractor per project, lives in Core. Routes messages to the correct
entity by extracting an entity ID via marker interfaces on messages.

### Extractor class

```csharp
using Akka.Cluster.Sharding;
using <Project>.Messages;

namespace <Project>.Core;

public sealed class ShardMessageExtractor(int maxShards = 25)
    : HashCodeMessageExtractor(maxShards)
{
    public override string EntityId(object message) => message switch
    {
        IWithDownloadId m => m.DownloadId.ToString(),
        IWithSearchId m => m.SearchId.ToString(),
        IWithRuleSetId m => m.RuleSetId,
        _ => throw new ArgumentException(
            $"Unknown sharded message type: {message.GetType().Name}", nameof(message)),
    };
}
```

Rules:
- Extend `HashCodeMessageExtractor` — it derives shard ID from `EntityId.GetHashCode()`
- Default 25 max shards (single-node cluster); increase for multi-node
- Pattern-match on marker interfaces, not concrete message types
- Throw on unknown types — fail fast, don't silently drop

### Adding a new entity type

1. Define marker interface in Messages:
   ```csharp
   public interface IWithOrderId { Guid OrderId { get; } }
   ```
2. Add case to `ShardMessageExtractor.EntityId`:
   ```csharp
   IWithOrderId m => m.OrderId.ToString(),
   ```
3. Implement interface on messages:
   ```csharp
   public sealed record ProcessOrder(Guid OrderId, ...) : IWithOrderId;
   ```

## Message Marker Interfaces

Live in the Messages project. Each sharded entity type gets one interface.

```csharp
namespace <Project>.Messages;

public interface IWithDownloadId { Guid DownloadId { get; } }
public interface IWithSearchId { Guid SearchId { get; } }
public interface IWithRuleSetId { string RuleSetId { get; } }
```

Messages implement them for routing — C# records auto-implement the getter:

```csharp
public sealed record InitDownload(Guid DownloadId, string Title, ...) : IWithDownloadId;
public sealed record CancelDownload(Guid DownloadId) : IWithDownloadId;
```

All messages for the same entity type share the same marker interface.

## Sending to sharded entities

Resolve the shard region by its marker interface, then Tell/Ask. The message
extractor routes to the correct entity automatically.

```csharp
var shardRegion = Context.GetActor<IDownloadRegion>();
shardRegion.Tell(new InitDownload(downloadId, title, url));
```

No need to know which node the entity lives on — Akka Cluster Sharding handles
routing, creation, and passivation transparently.

## Marker interfaces — where they live

| Interface type | Location | Example |
|---------------|----------|---------|
| Actor markers (for `GetActor<T>`) | `<Project>.Core` | `ISearchManager`, `IDownloadRegion` |
| Message markers (for extractor) | `<Project>.Messages` | `IWithDownloadId`, `IWithSearchId` |

## Checklist — adding a new sharded entity

- [ ] Create marker interface for the region in Core (e.g. `IOrderRegion`)
- [ ] Create message marker interface in Messages (e.g. `IWithOrderId`)
- [ ] Add case to `ShardMessageExtractor.EntityId`
- [ ] Implement marker on all messages for this entity
- [ ] Register `WithShardRegion<IOrderRegion>(...)` in actor system setup
- [ ] Create the Worker actor + State file (see `akka-actor-state` skill)

## Checklist — adding a new singleton

- [ ] Create marker interface in Core (e.g. `IOrderManager`)
- [ ] Register `WithSingleton<IOrderManager>(...)` in actor system setup
- [ ] Create the Manager actor + State file (see `akka-actor-state` skill)
