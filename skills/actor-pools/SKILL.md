---
description: "Akka.NET actor pools — router-based pools, DI-aware pools, and stash-based capacity limiting"
---

# Akka.NET Actor Pools

Patterns for concurrent work distribution in Akka.NET: router-based pools for parallel
processing, DI-aware pools for injected workers, and stash-based limiting for external
API throttling.

## When to use which

| Pattern | Use case | DI needed? | Concurrency model |
|---------|----------|------------|-------------------|
| **Router pool** | CPU-bound or stateless work (scoring, parsing) | No | Fixed N children, auto-routed |
| **DI pool** | Workers needing injected services (HTTP clients, DB) | Yes | Fixed N children, auto-routed |
| **Stash limiter** | External API calls with rate limits | Either | Manual slot tracking, backpressure via stash |

## Pattern 1: Router-based pool

A manager creates a fixed-size pool of stateless workers via Akka's built-in router.
Best for CPU-bound or pure-logic work where children need no DI.

### Manager template

```csharp
using Akka.Actor;
using Akka.Routing;

public sealed class <Name>Manager : ReceiveActor
{
    private readonly IActorRef _pool;

    public <Name>Manager(IOptionsMonitor<<Name>Options> optionsMonitor)
    {
        var poolProps = Props.Create<<Name>Actor>()
            .WithRouter(new SmallestMailboxPool(optionsMonitor.CurrentValue.PoolSize));
        _pool = Context.ActorOf(poolProps, "<name>-pool");

        Receive<ProcessItem>(Handle);
    }

    private void Handle(ProcessItem msg)
    {
        // Forward to pool, preserving original Sender so the worker replies directly
        _pool.Tell(new Execute(msg.Data), Sender);
    }
}
```

### Router types

| Router | Routing strategy | When to use |
|--------|-----------------|-------------|
| `SmallestMailboxPool` | Fewest queued messages | Default — best for variable-duration work |
| `RoundRobinPool` | Cyclic distribution | Equal-duration tasks |
| `ConsistentHashingPool` | Hash-based affinity | When same key should hit same worker |

### Pool sizing

- **CPU-bound:** `Environment.ProcessorCount` or slightly below
- **I/O-bound:** 2–4× processor count (workers mostly wait)
- **External API:** Match the API's concurrency limit
- Make configurable via `IOptionsMonitor<T>` — never hardcode

## Pattern 2: DI-aware pool

Workers that need injected dependencies (HTTP clients, database, options) can't use
plain `Props.Create<T>()`. Use the DI resolver to create pool children.

### Manager template

```csharp
public sealed class <Name>Manager : ReceiveActor
{
    public <Name>Manager(ServiceA clientA, ServiceB clientB)
    {
        // Each pool gets its own router with DI-resolved children
        var poolA = Context.ResolveChildActor<WorkerA>("worker-a-pool",
            props => props.WithRouter(new SmallestMailboxPool(2)));

        var poolB = Context.ResolveChildActor<WorkerB>("worker-b-pool",
            props => props.WithRouter(new SmallestMailboxPool(2)));

        // Route by message type
        Receive<TaskTypeA>(msg => poolA.Forward(msg));
        Receive<TaskTypeB>(msg => poolB.Forward(msg));
    }
}
```

Notes:
- `Context.ResolveChildActor<T>()` is a Servus.Akka extension — it creates Props via the DI resolver
- `.Forward(msg)` preserves the original Sender (equivalent to `Tell(msg, Sender)`)
- Multiple pools in one manager is fine — each handles a different message type
- Pool size can be hardcoded for small fixed pools or driven by options

### Without Servus

Use `resolver.Props<T>()` directly:

```csharp
public <Name>Manager(IDependencyResolver resolver)
{
    var poolProps = resolver.Props<WorkerA>()
        .WithRouter(new SmallestMailboxPool(2));
    var pool = Context.ActorOf(poolProps, "worker-a-pool");

    Receive<TaskTypeA>(msg => pool.Forward(msg));
}
```

## Pattern 3: Stash-based capacity limiter

When the actor itself does the work (e.g. HTTP calls via `Task.Run` + `PipeTo`) and you
need to limit concurrency without child actors. Uses Akka's Stash for backpressure.

### Template

```csharp
using Akka.Actor;
using Akka.Event;

public sealed class <Name>Manager : ReceiveActor, IWithUnboundedStash
{
    private sealed record RequestStarted;
    private sealed record RequestCompleted;
    private sealed record WorkCompleted(object Result);
    private sealed record WorkFailed(Exception Cause);

    private readonly ILoggingAdapter _log = Context.GetLogger();
    private readonly int _maxConcurrent;
    private <Name>ManagerState _state = <Name>ManagerState.Empty;

    public IStash Stash { get; set; } = null!;

    public <Name>Manager(int maxConcurrent = 3)
    {
        _maxConcurrent = maxConcurrent;

        Receive<DoWork>(HandleWork);
        Receive<WorkCompleted>(HandleCompleted);
        Receive<WorkFailed>(HandleFailed);
    }

    private void HandleWork(DoWork msg)
    {
        if (!_state.HasCapacity(_maxConcurrent))
        {
            _log.Debug("At capacity ({MaxConcurrent}), stashing request", _maxConcurrent);
            Stash.Stash();
            return;
        }

        _state = _state.Apply(new RequestStarted());
        ExecuteWork(msg);
    }

    private void ExecuteWork(DoWork msg)
    {
        var self = Self;
        var sender = Sender;

        Task.Run(async () =>
        {
            // ... async work (HTTP call, file I/O, etc.)
            return new WorkCompleted(result) as object;
        })
        .PipeTo(self, sender,
            success: r => r,
            failure: ex => new WorkFailed(ex));
    }

    private void HandleCompleted(WorkCompleted msg)
    {
        Sender.Tell(/* result */);
        SlotFreed();
    }

    private void HandleFailed(WorkFailed msg)
    {
        _log.Warning(msg.Cause, "Work failed");
        Sender.Tell(new WorkFailedResponse(msg.Cause));
        SlotFreed();
    }

    private void SlotFreed()
    {
        _state = _state.Apply(new RequestCompleted());
        Stash.Unstash();
    }
}
```

### State for capacity tracking

```csharp
public sealed record <Name>ManagerState(int ActiveCount)
{
    public static readonly <Name>ManagerState Empty = new(0);

    public bool HasCapacity(int max) => ActiveCount < max;
}

public static class <Name>ManagerStateExtensions
{
    public static <Name>ManagerState Apply(this <Name>ManagerState state, RequestStarted _) =>
        state with { ActiveCount = state.ActiveCount + 1 };

    public static <Name>ManagerState Apply(this <Name>ManagerState state, RequestCompleted _) =>
        state with { ActiveCount = state.ActiveCount - 1 };
}
```

### Key rules for stash pattern

- Implement `IWithUnboundedStash` — the `Stash` property is auto-injected
- `Stash.Stash()` queues the current message (including its Sender)
- `Stash.Unstash()` re-delivers the oldest stashed message to Self
- Call `Unstash()` in `SlotFreed()` — one unstash per completed slot
- Always track capacity in state via `Apply()`, not with a bare counter field
- Capture `Sender` before any async work — it changes between messages

## Sender preservation

| Method | Preserves Sender? | Use when |
|--------|-------------------|----------|
| `pool.Forward(msg)` | Yes (original caller) | DI pools, simple routing |
| `pool.Tell(msg, Sender)` | Yes (original caller) | Router pools with transformation |
| `pool.Tell(msg)` | No (becomes `Self`) | **Never** — breaks reply chain |

Always preserve the original Sender so the pool worker can reply directly to the caller
without the manager staying in the loop.

## Checklist

- [ ] Choose pattern: router pool (no DI), DI pool (injected workers), or stash limiter (self-work)
- [ ] Pool size configurable via `IOptionsMonitor<T>` or constructor param
- [ ] Sender preserved: `.Forward()` or `.Tell(msg, Sender)`
- [ ] For stash pattern: `IWithUnboundedStash`, capacity in state, `Unstash()` on slot freed
- [ ] For router pools: `SmallestMailboxPool` as default router
