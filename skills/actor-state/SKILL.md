---
description: "Akka.NET actor state pattern — immutable state records, Apply/GetSnapshot, persistence layer, and core actor rules"
---

# Akka.NET Actor State Pattern

Architectural pattern for Akka.NET actors: separate immutable state from actor plumbing.
Actors are thin shells (message routing, persistence, lifecycle). All domain logic lives
in state records and extension methods.

## When to use

- Creating a new actor (persistent or non-persistent)
- Adding state to an existing actor
- Making a non-persistent actor persistent
- Adding new events/commands to an existing actor's state

## File layout

Every actor with state gets two files in the same project:

```
<Project>.<Domain>/
  <Name>.cs           # Actor class — thin plumbing only
  <Name>State.cs      # State record + extensions
```

State is NEVER a nested class. Always a separate file.

## Actor resolution (DI)

Every actor that other actors resolve at runtime needs a marker interface:

```csharp
public interface I<Name>;
```

Register via Akka.Hosting: `ActorRegistry.For(system).Register<IMyActor>(actorRef)`
Resolve at runtime: `Context.GetActor<IMyActor>()`
NEVER pass `IActorRef` as a constructor param.

For actors needing DI services: `resolver.Props<T>()`.
For pure actors (no DI): `Props.Create<T>()` or `Props.Create(() => new T(args))`.

## Non-persistent actor

### State file (`<Name>State.cs`)

```csharp
namespace <Project>.<Domain>;

// Snapshot record — what callers see (may differ from internal state)
public sealed record <Name>Snapshot(...);

// Internal state — never sent to callers directly
public sealed record <Name>State(...)
{
    public static readonly <Name>State Empty = new(...defaults...);

    public static <Name>State FromSnapshot(<Name>Snapshot snapshot) =>
        new(...map snapshot fields...);
}

public static class <Name>StateExtensions
{
    // One Apply overload per event/change type — returns new state (immutable)
    public static <Name>State Apply(this <Name>State state, SomeEvent evt) =>
        state with { ... };

    // Returns snapshot for callers (queries, API responses)
    public static <Name>Snapshot GetSnapshot(this <Name>State state) =>
        new(...map state fields...);

    // Domain query helpers as needed
    public static SomeResult ToSomeResult(this <Name>State state, ...) => ...;
}
```

Key rules:
- `Empty` is a static readonly field, not a method
- `FromSnapshot` is a static method on the state record
- `GetSnapshot` is an extension method returning a separate snapshot record
- `Apply` returns a new state (immutable), one overload per event type
- Domain helpers (queries, aggregations) are extension methods on state

### Actor file (`<Name>.cs`)

```csharp
namespace <Project>.<Domain>;

public sealed class <Name> : ReceiveActor
{
    private <Name>State _state = <Name>State.Empty;

    public <Name>(/* DI deps */)
    {
        Receive<SomeCommand>(Handle);
        Receive<SomeQuery>(_ => Sender.Tell(_state.ToSomeResult()));
    }

    private void Handle(SomeCommand cmd)
    {
        _state = _state.Apply(someEvent);
        Sender.Tell(new SomeCommandCompleted(...));
    }
}
```

## Persistent actor

### Persistence events

Events and persisted state snapshots live in a dedicated persistence project (or namespace).
No `Event` or `Dto` suffix on record names.

```csharp
namespace <Project>.Persistence.Events.<Domain>;

public sealed record SomethingHappened(Guid Id, string Data, DateTimeOffset Timestamp);
public sealed record Persisted<Name>State(SomethingHappened[] Entries);
```

### State file (`<Name>State.cs`)

Extends the non-persistent pattern with persistence methods:

```csharp
namespace <Project>.<Domain>;

public sealed record <Name>State(...)
{
    public static readonly <Name>State Empty = new(...defaults...);

    // Reconstruct from persisted snapshot
    public static <Name>State FromPersistence(Persisted<Name>State persisted) =>
        new(...map persisted fields to internal state...);
}

public static class <Name>StateExtensions
{
    // Command processing — returns (newState, event) tuple
    public static (<Name>State State, SomethingHappened Event) ProcessCommand(
        this <Name>State state, SomeCommand cmd)
    {
        var evt = new SomethingHappened(cmd.Id, cmd.Data, DateTimeOffset.UtcNow);
        return (state.Apply(evt), evt);
    }

    // Apply persisted event to state
    public static <Name>State Apply(this <Name>State state, SomethingHappened evt) =>
        new(...apply event to state...);

    // Serialize to persistence format
    public static Persisted<Name>State GetPersistenceState(this <Name>State state) =>
        new(state.Items.Select(i => new SomethingHappened(...)).ToArray());

    // GetSnapshot may delegate to GetPersistenceState or return a separate type
    public static Persisted<Name>State GetSnapshot(this <Name>State state) =>
        state.GetPersistenceState();
}
```

### Actor file (`<Name>.cs`)

```csharp
namespace <Project>.<Domain>;

public sealed class <Name> : ReceivePersistentActor
{
    public override string PersistenceId => "<domain>-<name>";
    // Or for sharded: public override string PersistenceId { get; }
    //   set in constructor: PersistenceId = $"<domain>-{entityId}";

    private <Name>State _state = <Name>State.Empty;

    public <Name>(/* DI deps */)
    {
        // Recovery
        Recover<SnapshotOffer>(offer =>
        {
            if (offer.Snapshot is Persisted<Name>State persisted)
            {
                _state = <Name>State.FromPersistence(persisted);
            }
        });
        Recover<SomethingHappened>(evt => _state = _state.Apply(evt));

        // Commands
        Command<SomeCommand>(cmd =>
        {
            var (newState, evt) = _state.ProcessCommand(cmd);
            Persist(evt, _ =>
            {
                _state = newState;
                if (LastSequenceNr % snapshotInterval == 0)
                {
                    SaveSnapshot(_state.GetPersistenceState());
                }
            });
        });

        // Queries — read from state, never from persistence directly
        Command<SomeQuery>(_ => Sender.Tell(_state.ToSomeResult()));

        // Snapshot lifecycle
        Command<SaveSnapshotSuccess>(_ => { });
        Command<SaveSnapshotFailure>(f => _log.Warning(f.Cause,
            "Snapshot save failed at sequence {SequenceNr}", f.Metadata.SequenceNr));
    }
}
```

## Three-tier state model

```
┌─────────────────────┐
│  Internal State      │  <Name>State record — full detail, mutable via Apply()
│  (never leaves actor)│  Lives in: <Domain> project
├─────────────────────┤
│  Snapshot            │  <Name>Snapshot or query result records — what callers see
│  (for API/queries)   │  Produced by: GetSnapshot() / ToSomeResult()
├─────────────────────┤
│  Persisted State     │  Persisted<Name>State — extend-only, versioned
│  (for Akka journal)  │  Lives in: Persistence project
└─────────────────────┘
```

- Internal → Snapshot: `GetSnapshot()` / domain query extensions
- Internal → Persisted: `GetPersistenceState()`
- Persisted → Internal: `FromPersistence()`
- Snapshot → Internal: `FromSnapshot()` (for non-persistent reconstruction)

See `akka-persistence` for deep-dive on the persistence tier (extend-only rules, versioning, recovery).

## Critical rules

1. **State never sent directly** — always use `GetSnapshot()` or a query helper returning a response record
2. **`SaveSnapshot(_state.GetPersistenceState())`** — NEVER `SaveSnapshot(_state)`
3. **No `Event`/`Dto` suffix** on persistence records
4. **No custom serializer** — persistence records must be serializable by the configured serializer
5. **No `Status.Failure`** — use project-owned `XxxFailed(Exception Cause)` records
6. **`PipeTo` must have failure handler** — `failure: ex => new XxxFailed(ex)`
7. **Actors are thin plumbing** — message routing, persistence lifecycle, scheduling only. All logic in state extensions.
8. **Immutable state** — `Apply` always returns a new state via `with` or `new()`
9. **No `EventStream`** — use explicit actor references via registry
10. **No `IActorRef` constructor params** — resolve via `Context.GetActor<IMarker>()`
