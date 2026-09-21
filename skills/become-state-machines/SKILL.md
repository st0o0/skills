---
description: "Akka.NET Become/Unbecome state machines — multi-phase workflows, per-phase message handlers, stash-during-init, and mutable vs immutable state"
---

# Akka.NET Become State Machines

Patterns for actors that move through distinct phases, each handling a different set of
messages. Uses `Become()` to swap the active message handler set.

## When to use

- Multi-step workflows (fetch → process → enrich → complete)
- Actors that initialize asynchronously before accepting work
- Connection lifecycle (connecting → connected → disconnecting)
- Any actor where the valid message set changes over time

## Two state management styles

| Style | Use when | Example |
|-------|----------|---------|
| **Immutable record** | State accumulates across phases, phases are short | Config actor, aggregator |
| **Mutable class** | Each phase produces intermediate results consumed by the next | Search worker, pipeline worker |

### Immutable record state (standard pattern)

Same as `akka-actor-state` — `Become` just changes which messages are handled,
state still flows through `Apply()`:

```csharp
public sealed class ConnectionActor : ReceiveActor
{
    private ConnectionState _state = ConnectionState.Empty;

    public ConnectionActor()
    {
        Connecting();
    }

    private void Connecting()
    {
        Receive<Connected>(msg =>
        {
            _state = _state.Apply(msg);
            Become(Ready);
        });
        Receive<ConnectFailed>(msg =>
        {
            _state = _state.Apply(msg);
            // retry or escalate
        });
    }

    private void Ready()
    {
        Receive<SendData>(Handle);
        Receive<Disconnected>(_ => Become(Connecting));
    }

    private void Handle(SendData msg) { /* ... */ }
}
```

### Mutable class state (for multi-step workflows)

When each phase produces intermediate results that the next phase consumes.
The state class holds accumulated results across phases.

```csharp
public sealed class SearchWorkerState
{
    public SearchCommand? Command { get; set; }
    public MediathekItem[]? MediathekResults { get; set; }
    public RuleSet? ResolvedRuleSet { get; set; }
    public ScoredItem[]? ScoredItems { get; set; }
    public EnrichedItem[]? EnrichedItems { get; set; }

    public void Init(SearchCommand cmd) => Command = cmd;
    public void Apply(MediathekItem[] items) => MediathekResults = items;
    public void Apply(RuleSet ruleSet) => ResolvedRuleSet = ruleSet;
    public void Apply(ScoredItem[] items) => ScoredItems = items;
    public void Apply(EnrichedItem[] items) => EnrichedItems = items;
}
```

Use mutable state when:
- The workflow is linear (phase 1 → 2 → 3 → done)
- Each phase's result is consumed exactly once by the next
- The actor is short-lived (created per task, stops when done)
- Immutable `with` copies would be wasteful for large intermediate data

## Multi-step workflow pattern

A worker that progresses through phases, each waiting for a specific response:

```csharp
public sealed class SearchWorker : ReceiveActor
{
    private readonly ILoggingAdapter _log = Context.GetLogger();
    private readonly SearchWorkerState _state = new();

    public SearchWorker()
    {
        Receive<SearchCommand>(cmd =>
        {
            _state.Init(cmd);
            var mediathek = Context.GetActor<IMediathekManager>();
            mediathek.Tell(new QueryMediathek(cmd.Query));
            Become(Querying);
        });
    }

    private void Querying()
    {
        Receive<QueryMediathekCompleted>(msg =>
        {
            _state.Apply(msg.Items);
            var ruleSetResolver = Context.GetActor<IRuleSetResolver>();
            ruleSetResolver.Tell(new ResolveRuleSet(_state.Command!.RuleSetId));
            Become(ResolvingRuleSet);
        });
        Receive<QueryMediathekFailed>(msg =>
        {
            _log.Warning(msg.Cause, "Mediathek query failed for {Query}", _state.Command!.Query);
            Sender.Tell(new SearchFailed(msg.Cause));
            Context.Stop(Self);
        });
    }

    private void ResolvingRuleSet()
    {
        Receive<RuleSetResolved>(msg =>
        {
            _state.Apply(msg.RuleSet);
            var scorer = Context.GetActor<IScoringManager>();
            scorer.Tell(new ScoreItems(_state.MediathekResults!, msg.RuleSet));
            Become(Scoring);
        });
        Receive<RuleSetFailed>(msg =>
        {
            Sender.Tell(new SearchFailed(msg.Cause));
            Context.Stop(Self);
        });
    }

    private void Scoring()
    {
        Receive<ScoreCompleted>(msg =>
        {
            _state.Apply(msg.Items);
            Sender.Tell(new SearchCompleted(_state.ScoredItems!));
            Context.Stop(Self);
        });
        Receive<ScoringFailed>(msg =>
        {
            Sender.Tell(new SearchFailed(msg.Cause));
            Context.Stop(Self);
        });
    }
}
```

Key rules:
- Each phase is a **private void method** that sets up `Receive<>` handlers
- `Become(NextPhase)` replaces all handlers — previous phase messages are no longer handled
- Short-lived workers stop themselves when done: `Context.Stop(Self)`
- Failure at any phase reports back and stops

## Stash-during-initialization pattern

When an actor needs async setup before accepting work. Messages arriving during
init are stashed and replayed when ready.

```csharp
public sealed class PipelineActor : ReceiveActor, IWithUnboundedStash
{
    public IStash Stash { get; set; } = null!;

    public PipelineActor(IServiceProvider sp)
    {
        // Phase 1: Initializing — resolve dependencies, stash work
        Initializing();
    }

    private void Initializing()
    {
        Receive<DependenciesResolved>(msg =>
        {
            // Setup complete — switch to ready and replay stashed messages
            Stash.UnstashAll();
            Become(Ready);
        });

        // Stash everything else until ready
        ReceiveAny(_ => Stash.Stash());
    }

    private void Ready()
    {
        Receive<ProcessData>(Handle);
        Receive<QueryStatus>(_ => Sender.Tell(_state.GetSnapshot()));
    }

    private void Handle(ProcessData msg) { /* ... */ }
}
```

Rules:
- `IWithUnboundedStash` — the `Stash` property is auto-injected by Akka
- `ReceiveAny(_ => Stash.Stash())` catches all messages during init
- `Stash.UnstashAll()` replays **all** stashed messages in order when transitioning
- Always `UnstashAll()` **before** `Become()` — ensures messages are delivered to the new handler

## Connection lifecycle state machine

For actors managing external connections (MQTT, gRPC, WebSocket):

```csharp
public sealed class ConnectionActor : ReceiveActor, IWithUnboundedStash
{
    private sealed record Connect;
    private sealed record Reconnect;

    private readonly ILoggingAdapter _log = Context.GetLogger();
    public IStash Stash { get; set; } = null!;

    public ConnectionActor(IConnectionFactory factory)
    {
        Disconnected();
        Self.Tell(new Connect());
    }

    private void Disconnected()
    {
        Receive<Connect>(_ =>
        {
            _log.Info("Connecting...");
            StartConnection();
            Become(Connecting);
        });
        ReceiveAny(_ => Stash.Stash());
    }

    private void Connecting()
    {
        Receive<Connected>(msg =>
        {
            _log.Info("Connected");
            Stash.UnstashAll();
            Become(Active);
        });
        Receive<ConnectionFailed>(msg =>
        {
            _log.Warning(msg.Cause, "Connection failed, scheduling retry");
            Context.System.Scheduler.ScheduleTellOnce(
                TimeSpan.FromSeconds(5), Self, new Reconnect(), ActorRefs.NoSender);
            Become(Disconnected);
        });
        ReceiveAny(_ => Stash.Stash());
    }

    private void Active()
    {
        Receive<PublishMessage>(Handle);
        Receive<Disconnected>(_ =>
        {
            _log.Warning("Disconnected, reconnecting...");
            Self.Tell(new Connect());
            Become(Disconnected);
        });
    }

    private void Handle(PublishMessage msg) { /* ... */ }
    private void StartConnection() { /* ... */ }
}
```

## Testing Become actors

Test each phase transition by sending the right messages in sequence:

```csharp
[Fact]
public void Full_search_workflow()
{
    var p = RegisterProbes();
    var worker = Sys.ActorOf(Props.Create(() => new SearchWorker()));

    // Phase 1: Initial command → Querying
    worker.Tell(new SearchCommand("test"), TestActor);

    // Phase 2: Mediathek responds → ResolvingRuleSet
    p.Mediathek.ExpectMsg<QueryMediathek>();
    p.Mediathek.Reply(new QueryMediathekCompleted(items));

    // Phase 3: RuleSet resolves → Scoring
    p.RuleSet.ExpectMsg<ResolveRuleSet>();
    p.RuleSet.Reply(new RuleSetResolved(ruleSet));

    // Phase 4: Scoring completes → done
    p.Scoring.ExpectMsg<ScoreItems>();
    p.Scoring.Reply(new ScoreCompleted(scoredItems));

    // Final result
    var result = ExpectMsg<SearchCompleted>();
    Assert.NotEmpty(result.Items);
}

[Fact]
public void Failure_in_querying_reports_and_stops()
{
    var p = RegisterProbes();
    var worker = Sys.ActorOf(Props.Create(() => new SearchWorker()));

    worker.Tell(new SearchCommand("test"), TestActor);

    p.Mediathek.ExpectMsg<QueryMediathek>();
    p.Mediathek.Reply(new QueryMediathekFailed(new TimeoutException()));

    ExpectMsg<SearchFailed>();
    Watch(worker);
    ExpectTerminated(worker);
}
```

## Checklist

- [ ] Each phase is a private void method with its own `Receive<>` handlers
- [ ] `Become(NextPhase)` on successful transition
- [ ] Failure at any phase reports back to the original sender
- [ ] Short-lived workers: `Context.Stop(Self)` when done or failed
- [ ] Stash-during-init: `IWithUnboundedStash` + `UnstashAll()` before `Become(Ready)`
- [ ] Mutable state only for linear workflows with intermediate results
- [ ] Tests cover the full happy path AND failure at each phase
