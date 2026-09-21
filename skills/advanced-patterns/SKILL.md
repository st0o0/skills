---
description: "Akka.NET advanced actor patterns — IWithTimers, ReceiveAsync, PipeTo, DeathWatch, Passivation, PersistAll, and PreStart initialization"
---

# Akka.NET Advanced Actor Patterns

Patterns that go beyond basic Receive/Persist: timers, async handlers, task bridging,
actor watching, entity passivation, batch persistence, and startup initialization.

## IWithTimers — scheduled messages

Use `IWithTimers` for periodic or delayed self-messages. Preferred over raw
`Context.System.Scheduler` because timers are automatically cancelled when the
actor stops.

```csharp
using Akka.Actor;

public sealed class SensorMonitor : ReceiveActor, IWithTimers
{
    private static class TimerKeys
    {
        public static readonly object ExpiryCheck = new();
        public static readonly object Heartbeat = new();
    }

    public ITimerScheduler Timers { get; set; } = null!;

    public SensorMonitor(TimeSpan expiryInterval)
    {
        // Periodic timer — repeats until cancelled or actor stops
        Timers.StartPeriodicTimer(TimerKeys.ExpiryCheck,
            new CheckExpiry(), expiryInterval);

        // Single-shot timer — fires once
        Timers.StartSingleTimer(TimerKeys.Heartbeat,
            new SendHeartbeat(), TimeSpan.FromSeconds(30));

        Receive<CheckExpiry>(Handle);
        Receive<SendHeartbeat>(Handle);
        Receive<UpdateReading>(Handle);
    }

    private void Handle(CheckExpiry msg) { /* expire stale readings */ }
    private void Handle(SendHeartbeat msg) { /* send heartbeat, reschedule */ }
    private void Handle(UpdateReading msg) { /* process sensor data */ }
}
```

Rules:
- Timer keys are `object` constants — use a nested static class for organization
- `StartPeriodicTimer` — repeating interval, first fire after one interval
- `StartSingleTimer` — fires once, useful for timeouts and delayed actions
- `Timers.Cancel(key)` to stop a specific timer
- Timers are **automatically cancelled** when the actor is stopped — no manual cleanup

### Timeout pattern

```csharp
Timers.StartSingleTimer(TimerKeys.Timeout,
    new OperationTimedOut(operationId), TimeSpan.FromSeconds(10));

Receive<OperationCompleted>(msg =>
{
    Timers.Cancel(TimerKeys.Timeout);
    // process result
});
Receive<OperationTimedOut>(msg =>
{
    _log.Warning("Operation {Id} timed out", msg.Id);
    Sender.Tell(new OperationFailed(new TimeoutException()));
});
```

## ReceiveAsync — async message handlers

For messages that require async work (HTTP calls, database queries) **where the actor
must process them one at a time** (no concurrent handling).

```csharp
public sealed class EnrichmentActor : ReceiveActor
{
    private readonly IHttpClientFactory _httpClientFactory;

    public EnrichmentActor(IHttpClientFactory httpClientFactory)
    {
        _httpClientFactory = httpClientFactory;
        ReceiveAsync<EnrichItems>(HandleEnrich);
    }

    private async Task HandleEnrich(EnrichItems msg)
    {
        var client = _httpClientFactory.CreateClient("enrichment-api");
        var response = await client.GetAsync($"/api/items/{msg.Id}");
        var data = await response.Content.ReadFromJsonAsync<EnrichmentData>();

        Sender.Tell(new EnrichItemsCompleted(data!));
    }
}
```

Rules:
- `ReceiveAsync<T>` suspends the actor's mailbox during the async operation — **no other messages are processed**
- Safe to access `Sender`, `Self`, `_state` inside the async handler — the actor context is preserved
- Use for simple request-response patterns with external services
- **Avoid for long-running operations** — the actor is blocked; use `PipeTo` instead

### When to use ReceiveAsync vs PipeTo

| Pattern | Actor blocked? | Use when |
|---------|---------------|----------|
| `ReceiveAsync` | Yes — mailbox paused | Simple, sequential async work. One message at a time. |
| `PipeTo` | No — actor keeps processing | Need concurrency. Actor should handle other messages while waiting. |

## PipeTo — bridging Tasks into actor messages

Convert a `Task<T>` result into an actor message delivered to any actor's mailbox.
The actor remains responsive during the async operation.

### Basic PipeTo

```csharp
Receive<FetchData>(msg =>
{
    var sender = Sender;  // capture before async — Sender changes between messages

    _httpClient.GetAsync($"/api/{msg.Id}")
        .ContinueWith(t => t.Result.Content.ReadFromJsonAsync<DataResult>()).Unwrap()
        .PipeTo(Self, sender,
            success: result => new FetchDataCompleted(result!),
            failure: ex => new FetchDataFailed(ex));
});

Receive<FetchDataCompleted>(msg => { /* process */ });
Receive<FetchDataFailed>(msg => _log.Warning(msg.Cause, "Fetch failed"));
```

### Task.Run for CPU-bound work

```csharp
Receive<ProcessItems>(msg =>
{
    var items = msg.Items;

    Task.Run(() =>
    {
        var result = ExpensiveComputation(items);
        return new ProcessItemsCompleted(result) as object;
    })
    .PipeTo(Self, Sender,
        success: r => r,
        failure: ex => new ProcessItemsFailed(ex));
});
```

### Fan-out with Task.WhenAll

```csharp
Receive<EnrichAll>(msg =>
{
    var tasks = msg.Items.Select(item =>
        _enrichmentService.EnrichAsync(item));

    Task.WhenAll(tasks)
        .PipeTo(Self, Sender,
            success: results => new EnrichAllCompleted(results),
            failure: ex => new EnrichAllFailed(ex));
});
```

### Critical PipeTo rules

1. **Always provide a `failure` handler** — without it, exceptions are silently lost
2. **Capture `Sender` before the async gap** — `Sender` refers to the current message's sender,
   which changes on every message
3. **Never access actor state inside the Task** — only reference captured local variables
4. **PipeTo target is usually `Self`** — the result comes back as a normal message

## DeathWatch — monitoring actor lifecycle

Watch another actor and receive `Terminated` when it stops (gracefully or by crash).

```csharp
public sealed class SupervisorActor : ReceiveActor
{
    private readonly ILoggingAdapter _log = Context.GetLogger();

    public SupervisorActor()
    {
        Receive<WatchChild>(msg =>
        {
            var child = Context.ActorOf(Props.Create<Worker>(), msg.Name);
            Context.Watch(child);
        });

        Receive<Terminated>(msg =>
        {
            _log.Warning("Watched actor terminated: {Path}", msg.ActorRef.Path);
            // restart, notify, or clean up
        });
    }
}
```

Rules:
- `Context.Watch(actorRef)` — start watching; receive `Terminated` when it stops
- `Context.Unwatch(actorRef)` — stop watching (optional, auto-cleared when watcher stops)
- `Terminated` is a **system message** — delivered even if the watched actor crashes
- Check `msg.ActorRef` to identify which watched actor terminated
- Common use: parent watches dynamically created children for cleanup

### Watching resolved actors

```csharp
protected override void PreStart()
{
    base.PreStart();
    var dependency = Context.GetActor<IDependencyActor>();
    Context.Watch(dependency);
}

// In constructor:
Receive<Terminated>(msg =>
{
    _log.Error("Critical dependency {Path} terminated", msg.ActorRef.Path);
    throw new DependencyFailedException(msg.ActorRef.Path.ToString());
});
```

## Passivation — idle entity shutdown

For sharded entities: stop after an idle period to free resources.
Uses `ReceiveTimeout` + `Passivate` to cooperate with Cluster Sharding.

```csharp
public sealed class OrderWorker : ReceivePersistentActor
{
    public override string PersistenceId { get; }

    public OrderWorker(string entityId)
    {
        PersistenceId = $"order-{entityId}";

        // Passivate after 60 seconds of no messages
        SetReceiveTimeout(TimeSpan.FromSeconds(60));

        Command<ReceiveTimeout>(_ => Passivate());

        // ... other commands
    }

    private void Passivate()
    {
        Context.Parent.Tell(new Akka.Cluster.Sharding.Passivate(PoisonPill.Instance));
    }
}
```

Rules:
- `SetReceiveTimeout` sends `ReceiveTimeout` to Self after the interval with no messages
- **Never self-stop a sharded entity** — always go through `Passivate` so the shard
  can drain in-flight messages
- `Context.Parent.Tell(new Passivate(...))` — the shard region handles the shutdown
- The stop message (`PoisonPill.Instance`) is sent to the entity after draining
- Can also be configured via `ShardOptions.PassivateIdleEntityAfter` (simpler, no code needed)

## PersistAll — batch event persistence

Persist multiple events atomically. The callback fires once for **each** event.

```csharp
Command<ImportBatch>(cmd =>
{
    var events = cmd.Items
        .Select(item => new ItemImported(item.Id, item.Data, DateTimeOffset.UtcNow))
        .ToList();

    PersistAll(events, evt =>
    {
        _state = _state.Apply(evt);

        // Snapshot check only after the last event
        if (IsLastEvent(evt, events))
        {
            if (LastSequenceNr % snapshotInterval == 0)
            {
                SaveSnapshot(_state.GetPersistenceState());
            }
            Sender.Tell(new ImportBatchCompleted(events.Count));
        }
    });
});

private static bool IsLastEvent<T>(T current, IReadOnlyList<T> all) =>
    ReferenceEquals(current, all[^1]);
```

Rules:
- `PersistAll` guarantees all events are written before any callback fires
- The callback is invoked once **per event**, not once for the batch
- Side effects (replies, snapshots) should only happen in the **last** callback
- Use `IsLastEvent` or a counter to detect the final callback

## PreStart — actor initialization

Use `PreStart` for one-time setup that runs after the actor is fully constructed
but before it processes its first message.

```csharp
public sealed class StatsCollector : ReceiveActor
{
    protected override void PreStart()
    {
        base.PreStart();

        // Self-trigger to load initial data
        Self.Tell(new LoadInitialStats());
    }
}
```

Common uses:
- **Self-tell for initialization**: send yourself a command to trigger async setup
- **Start timers**: (though `IWithTimers` handles this in the constructor)
- **Watch dependencies**: `Context.Watch(Context.GetActor<IDep>())`
- **Log lifecycle**: `_log.Info("Actor started: {Path}", Self.Path)`

### PreStart vs constructor

| | Constructor | PreStart |
|---|---|---|
| Actor context available? | Yes | Yes |
| Self.Tell works? | Yes (queued) | Yes (queued) |
| Called on restart? | No (same instance) | Yes (re-initializes) |
| DI injection? | Yes | No |

Use `PreStart` when you need the action to repeat on actor restart (after supervision).
Use the constructor for one-time-only setup.

## Checklist

- [ ] `IWithTimers` for periodic/delayed self-messages — never raw Scheduler for self-targeted timers
- [ ] `ReceiveAsync` only for simple sequential async — `PipeTo` when actor must stay responsive
- [ ] `PipeTo` always has a `failure:` handler; `Sender` captured before the async gap
- [ ] `Context.Watch` for lifecycle monitoring; `Terminated` handler registered
- [ ] Sharded entities use `Passivate` via `Context.Parent.Tell`, never `Context.Stop(Self)`
- [ ] `PersistAll` callback fires per-event — side effects only in the last one
- [ ] `PreStart` for initialization that must repeat on restart
