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

- Marker interface in Core (e.g. `IOrderManager`, `ICatalogManager`)
- Actor name is the Akka path name — lowercase, hyphenated
- `resolver.Props<T>()` for actors needing DI services
- Resolve at runtime: `Context.GetActor<IOrderManager>()`

### Example — multiple singletons

```csharp
builder
    .WithSingleton<IOrderManager>("order-manager",
        (_, _, resolver) => resolver.Props<OrderManager>())
    .WithSingleton<ICatalogManager>("catalog-manager",
        (_, _, resolver) => resolver.Props<CatalogManager>())
    .WithSingleton<INotificationManager>("notification-manager",
        (_, _, resolver) => resolver.Props<NotificationManager>());
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
    .WithShardRegion<IOrderRegion>("order-worker",
        (_, _, resolver) => _ => resolver.Props<OrderWorker>(),
        new ShardMessageExtractor(),
        new ShardOptions { PassivateIdleEntityAfter = TimeSpan.FromSeconds(30) })
    .WithShardRegion<IPaymentRegion>("payment-worker",
        (_, _, resolver) => _ => resolver.Props<PaymentWorker>(),
        new ShardMessageExtractor(),
        new ShardOptions())
    .WithShardRegion<ILedgerRegion>("ledger",
        (_, _, resolver) => entityId => resolver.Props<LedgerWorker>(entityId),
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
        IWithOrderId m => m.OrderId.ToString(),
        IWithPaymentId m => m.PaymentId.ToString(),
        IWithLedgerId m => m.LedgerId,
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

public interface IWithOrderId { Guid OrderId { get; } }
public interface IWithPaymentId { Guid PaymentId { get; } }
public interface IWithLedgerId { string LedgerId { get; } }
```

Messages implement them for routing — C# records auto-implement the getter:

```csharp
public sealed record PlaceOrder(Guid OrderId, string Item, ...) : IWithOrderId;
public sealed record CancelOrder(Guid OrderId) : IWithOrderId;
```

All messages for the same entity type share the same marker interface.

## Sending to sharded entities

Resolve the shard region by its marker interface, then Tell/Ask. The message
extractor routes to the correct entity automatically.

```csharp
var shardRegion = Context.GetActor<IOrderRegion>();
shardRegion.Tell(new PlaceOrder(orderId, item, quantity));
```

No need to know which node the entity lives on — Akka Cluster Sharding handles
routing, creation, and passivation transparently.

## Marker interfaces — where they live

| Interface type | Location | Example |
|---------------|----------|---------|
| Actor markers (for `GetActor<T>`) | `<Project>.Core` | `IOrderManager`, `IOrderRegion` |
| Message markers (for extractor) | `<Project>.Messages` | `IWithOrderId`, `IWithPaymentId` |

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
