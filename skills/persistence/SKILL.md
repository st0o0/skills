---
description: "Akka.NET persistence separation — three-tier state model, extend-only DTOs, SaveSnapshot rules, and recovery patterns"
---

# Akka.NET Persistence Separation

Deep-dive companion to `akka-actor-state`. Covers the persistence tier: how internal state
maps to what Akka stores, extend-only rules, snapshot lifecycle, and recovery.

## Three-tier state model

```
┌──────────────────────────────────────────────────────────────────┐
│ Tier 1 — Internal State                                         │
│ <Name>State record + extensions                                 │
│ Lives in: <Project>.<Domain>/                                   │
│ Full detail, immutable via Apply(), NEVER leaves the actor      │
│ May contain computed caches, runtime references, derived stats  │
├──────────────────────────────────────────────────────────────────┤
│ Tier 2 — Snapshot (for callers)                                 │
│ <Name>Snapshot or query result records                          │
│ Shaped for API / query consumers, may omit internal detail      │
│ Produced by: GetSnapshot() / ToXxxResult() extensions           │
├──────────────────────────────────────────────────────────────────┤
│ Tier 3 — Persisted State (for Akka journal)                     │
│ Persisted<Name>State + event records                            │
│ Lives in: <Project>.Persistence/Events/<Domain>/                │
│ Extend-only, versioned, no runtime data                         │
└──────────────────────────────────────────────────────────────────┘
```

### Conversion directions

```
Internal ──GetSnapshot()──────────→ Snapshot (callers)
Internal ──GetPersistenceState()──→ Persisted (Akka journal)
Persisted ──FromPersistence()─────→ Internal (recovery)
Snapshot  ──FromSnapshot()────────→ Internal (non-persistent reconstruction)
```

**NEVER** convert Persisted → Snapshot directly. Always go through Internal.

## Persistence project structure

```
<Project>.Persistence/
  Events/
    <Domain>/
      SomethingHappened.cs         # Event record — past-tense name
      SomethingElseOccurred.cs     # One record per file (or group related in one file)
      Persisted<Name>State.cs      # Snapshot record for SaveSnapshot
```

### Event records

```csharp
namespace <Project>.Persistence.Events.<Domain>;

public sealed record SomethingHappened(
    Guid Id,
    string Data,
    DateTimeOffset Timestamp);
```

Rules:
- **Past-tense naming** — `DownloadEnqueued`, `HistoryRecorded`, `ScoringRecorded`
- **No `Event` or `Dto` suffix** — `SomethingHappened`, not `SomethingHappenedEvent`
- **Sealed records** — always
- May reference types from the Messages project, never from domain projects

### Persisted state record

```csharp
namespace <Project>.Persistence.Events.<Domain>;

public sealed record Persisted<Name>State(SomethingHappened[] Entries);
```

The persisted state is a flat container holding the events/entries needed to reconstruct
internal state. It's what `SaveSnapshot` writes and `SnapshotOffer` delivers.

## Conversion methods

### GetPersistenceState — Internal → Persisted

Extension method on state. Converts internal representation to persistence format.

```csharp
public static Persisted<Name>State GetPersistenceState(this <Name>State state) =>
    new(state.Items
        .Select(i => new SomethingHappened(i.Id, i.Data, i.Timestamp))
        .ToArray());
```

**Why a separate method?** Internal state may contain:
- Computed/cached values (stats, aggregations) that are derivable
- Runtime-only data (IActorRef references, timer handles)
- Immutable collections (`ImmutableList<T>`) that need array conversion

None of these belong in the journal.

### FromPersistence — Persisted → Internal

Static method on the state record. Reconstructs full internal state from persisted data,
including re-deriving any computed values.

```csharp
public static <Name>State FromPersistence(Persisted<Name>State persisted)
{
    var items = persisted.Entries
        .Select(e => new InternalItem(e.Id, e.Data, e.Timestamp))
        .ToImmutableList();

    return new <Name>State(items, ComputeStats(items));
}
```

### GetSnapshot — Internal → Snapshot (for callers)

May delegate to `GetPersistenceState()` when the shapes align, or return a
different type shaped for API/query consumers.

```csharp
// When shapes align:
public static Persisted<Name>State GetSnapshot(this <Name>State state) =>
    state.GetPersistenceState();

// When callers need a different shape:
public static <Name>Snapshot GetSnapshot(this <Name>State state) =>
    new(state.Records.ToArray());
```

## SaveSnapshot rules

### The golden rule

```csharp
SaveSnapshot(_state.GetPersistenceState());  // ✅ ALWAYS
SaveSnapshot(_state);                         // ❌ NEVER
```

### Snapshot interval pattern

```csharp
Persist(evt, _ =>
{
    _state = newState;

    if (LastSequenceNr % snapshotInterval == 0)
    {
        SaveSnapshot(_state.GetPersistenceState());
    }
});
```

`snapshotInterval` comes from options (injectable, testable):

```csharp
var opts = _optionsMonitor.CurrentValue;
if (LastSequenceNr % opts.SnapshotInterval == 0)
{
    SaveSnapshot(_state.GetPersistenceState());
}
```

### Snapshot lifecycle handlers

Always handle both — even if success is a no-op:

```csharp
Command<SaveSnapshotSuccess>(_ => { });
Command<SaveSnapshotFailure>(f => _log.Warning(f.Cause,
    "Snapshot save failed at sequence {SequenceNr}", f.Metadata.SequenceNr));
```

## Recovery pattern

Recovery happens in order: snapshot first (if available), then events replayed on top.

```csharp
// 1. Recover from snapshot — type-check against persisted state record
Recover<SnapshotOffer>(offer =>
{
    if (offer.Snapshot is Persisted<Name>State persisted)
    {
        _state = <Name>State.FromPersistence(persisted);
    }
});

// 2. Replay events on top of snapshot (or from empty if no snapshot)
Recover<SomethingHappened>(evt => _state = _state.Apply(evt));
Recover<SomethingElseOccurred>(evt => _state = _state.Apply(evt));
```

### Post-recovery hook

Use `OnReplaySuccess` for cleanup after all events are replayed (trim old entries, etc.):

```csharp
protected override void OnReplaySuccess()
{
    base.OnReplaySuccess();
    var opts = _optionsMonitor.CurrentValue;
    _state = _state.Trim(opts.MaxSnapshots, opts.MaxAgeDays);
}
```

## ProcessCommand pattern

Commands produce `(newState, event)` tuples. Logic lives in state extensions, not in the actor.

```csharp
public static (<Name>State State, SomethingHappened Event) ProcessCommand(
    this <Name>State state, SomeCommand cmd)
{
    var evt = new SomethingHappened(cmd.Id, cmd.Data, DateTimeOffset.UtcNow);
    return (state.Apply(evt), evt);
}
```

The actor persists the event, then updates state:

```csharp
Command<SomeCommand>(cmd =>
{
    var (newState, evt) = _state.ProcessCommand(cmd);
    Persist(evt, _ =>
    {
        _state = newState;
        // snapshot check, side effects...
    });
});
```

## Extend-only rules for persistence records

Persistence records are your contract with the journal. Breaking changes corrupt stored data.

| Rule | Why |
|------|-----|
| Never remove a field | Old snapshots/events in the journal still have it |
| Never rename a field | Serializer maps by name |
| Never rename `[JsonProperty]` strings | Stable wire names survive refactors |
| New properties must be nullable or have a default | Old records won't have the value |
| No custom serializer | Use the configured one (System.Text.Json / Newtonsoft) |
| Increment `Version` when semantics change | Recovery code must handle all versions ≥ 1 |

### Version field pattern (for semantic changes)

When the meaning of a field changes (not just additions):

```csharp
public sealed record Persisted<Name>State(
    int Version,    // increment when semantics change
    SomethingHappened[] Entries);
```

Recovery handles all versions:

```csharp
public static <Name>State FromPersistence(Persisted<Name>State persisted) =>
    persisted.Version switch
    {
        1 => MigrateV1(persisted),
        _ => MapCurrent(persisted),
    };
```

## When NOT to persist

Not every actor needs event sourcing. Skip persistence for:

- **Cache/computed state** — rebuildable from external sources on startup
- **Short-lived workers** — search workers, download workers that report results and stop
- **Projection read models** — rebuilt from the write-side journal
- **Managers with reconstructable state** — e.g. config pushed from another actor on startup

These actors use `ReceiveActor` (not `ReceivePersistentActor`) with the non-persistent
state pattern. See `akka-actor-state` for that template.

## Testing persistence state

Test `GetPersistenceState` / `FromPersistence` roundtrips in state-only tests (no TestKit):

```csharp
[Fact]
public void Roundtrip_persists_and_restores_state()
{
    var state = <Name>State.Empty.Apply(someEvent).Apply(anotherEvent);
    var persisted = state.GetPersistenceState();
    var restored = <Name>State.FromPersistence(persisted);

    Assert.Equal(state.Items.Count, restored.Items.Count);
}

[Fact]
public void Empty_state_roundtrips()
{
    var persisted = <Name>State.Empty.GetPersistenceState();
    var restored = <Name>State.FromPersistence(persisted);

    Assert.Equal(<Name>State.Empty, restored);
}
```

## Checklist

- [ ] Events in `<Project>.Persistence/Events/<Domain>/`, past-tense, no suffix
- [ ] `Persisted<Name>State` in same directory
- [ ] `GetPersistenceState()` extension on state — converts to persistence format
- [ ] `FromPersistence()` static on state record — reconstructs from persisted
- [ ] `SaveSnapshot(_state.GetPersistenceState())` — NEVER `SaveSnapshot(_state)`
- [ ] `Recover<SnapshotOffer>` type-checks for `Persisted<Name>State`
- [ ] `Recover<Event>` for each event type
- [ ] `SaveSnapshotSuccess` + `SaveSnapshotFailure` handlers
- [ ] All persistence records are extend-only
- [ ] Roundtrip test: state → GetPersistenceState → FromPersistence → compare
