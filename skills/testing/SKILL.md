---
description: "Test Akka.NET actors with TestKit — classic and Hosting TestKit, TestProbe registration, state-only unit tests, async assertions, multi-step interaction flows"
---

# Akka.NET Actor Testing

## Three test styles

Pick the right one for what you're testing.

### 1. Actor integration tests (Classic TestKit)

Test message handling, inter-actor communication, and side effects.
Best for projects using marker interfaces and `Props.Create` in tests.

```csharp
using Akka.Actor;
using Akka.Hosting;
using Akka.TestKit.Xunit;

namespace <Project>.Tests;

public sealed class <Name>Tests : TestKit
{
    [Fact]
    public void Test_case_name()
    {
        // 1. Create probes and register
        // 2. Create actor
        // 3. Send message, assert
    }
}
```

### 2. Actor integration tests (Hosting TestKit)

When actors need DI services, persistence, or are registered via `WithResolvableActors`.
Provides full `IServiceProvider` and `AkkaConfigurationBuilder` access.

```csharp
using Akka.Actor;
using Akka.Hosting;
using Akka.Hosting.TestKit;

namespace <Project>.Tests;

public sealed class <Name>Spec : Akka.Hosting.TestKit.TestKit
{
    protected override void ConfigureAkka(AkkaConfigurationBuilder builder, IServiceProvider provider)
    {
        builder
            .AddTestPersistence()
            .WithActors((system, registry, resolver) =>
            {
                var actor = system.ActorOf(resolver.Props<MyActor>(), "my-actor");
                registry.Register<MyActor>(actor);
            });
    }

    protected override void ConfigureServices(IServiceCollection services)
    {
        services.AddSingleton(Options.Create(new MyOptions { MaxItems = 10 }));
        services.AddSingleton<IMyService, FakeMyService>();
    }

    [Fact(Timeout = 5000)]
    public async Task Should_process_command()
    {
        var actor = await ActorRegistry.GetAsync<MyActor>();
        actor.Tell(new ProcessCommand("test"));

        var result = await ExpectMsgAsync<ProcessCommandCompleted>();
        Assert.Equal("test", result.Id);
    }
}
```

Key differences from Classic TestKit:

| Feature | Classic (`Akka.TestKit.Xunit`) | Hosting (`Akka.Hosting.TestKit`) |
|---------|-------------------------------|----------------------------------|
| Base class | `Akka.TestKit.Xunit.TestKit` | `Akka.Hosting.TestKit.TestKit` |
| DI services | Manual (`Props.Create(() => new T(deps))`) | Full DI (`ConfigureServices`) |
| Actor registration | `ActorRegistry.For(Sys).Register<>()` | `builder.WithActors()` + `WithResolvableActors()` |
| Persistence | Manual HOCON config | `builder.AddTestPersistence()` |
| Actor resolution | `ActorRegistry.For(Sys)` | `ActorRegistry.GetAsync<T>()` |
| Assertions | Sync `ExpectMsg<T>()` | Async `ExpectMsgAsync<T>()` with CancellationToken |

### 3. State-only unit tests

Test state extension methods (Apply, ProcessCommand, GetSnapshot) without actors. Plain xUnit — no TestKit.

```csharp
public sealed class <Name>StateTests
{
    [Fact]
    public void Apply_event_updates_state()
    {
        var state = <Name>State.Empty;
        var newState = state.Apply(new SomeEvent(...));
        Assert.Equal(expected, newState.SomeField);
    }

    // Helper factory methods at bottom
    private static <Name>State MakeState(...) => <Name>State.Empty.Apply(...);
    private static SomeItem MakeItem(...) => new(...);
}
```

## Choosing a TestKit

| Scenario | TestKit |
|----------|---------|
| Pure actors with no DI, manual Props | **Classic** (`Akka.TestKit.Xunit.TestKit`) |
| Actors needing DI services (HTTP clients, options, EF Core) | **Hosting** (`Akka.Hosting.TestKit.TestKit`) |
| Actors registered via `WithResolvableActors` or `resolver.Props<T>()` | **Hosting** |
| Actors needing persistence (test journal/snapshot) | **Hosting** with `AddTestPersistence()` |
| State logic only (Apply, GetSnapshot, ProcessCommand) | **None** — plain xUnit |

## TestProbe registration

### Classic TestKit — marker interfaces

Every actor dependency gets a `TestProbe` registered via `ActorRegistry`:

```csharp
var fooProbe = CreateTestProbe();
var barProbe = CreateTestProbe();

var registry = ActorRegistry.For(Sys);
registry.Register<IFooActor>(fooProbe);
registry.Register<IBarActor>(barProbe);
```

Register **all** marker interfaces the actor resolves via `Context.GetActor<T>()`, even if the test doesn't exercise that path — the actor will fail on startup otherwise.

### Hosting TestKit — concrete types or fakes

```csharp
protected override void ConfigureAkka(AkkaConfigurationBuilder builder, IServiceProvider provider)
{
    builder.WithActors((system, registry, resolver) =>
    {
        // Register a TestProbe as a dependency
        var pipelineProbe = new TestProbe(system);
        registry.Register<PipelineActor>(pipelineProbe);

        // Register the actor under test
        var actor = system.ActorOf(resolver.Props<SchedulerActor>(), "scheduler");
        registry.Register<SchedulerActor>(actor);
    });
}
```

### Record helper for many probes

```csharp
private record TestProbes(
    Akka.TestKit.TestProbe Foo,
    Akka.TestKit.TestProbe Bar,
    Akka.TestKit.TestProbe Baz);

private TestProbes RegisterProbes()
{
    var probes = new TestProbes(
        CreateTestProbe(), CreateTestProbe(), CreateTestProbe());

    var registry = ActorRegistry.For(Sys);
    registry.Register<IFooActor>(probes.Foo);
    registry.Register<IBarActor>(probes.Bar);
    registry.Register<IBazActor>(probes.Baz);

    return probes;
}
```

## Actor creation

### Classic TestKit

Always `Props.Create` in tests — never `resolver.Props<T>()` (that's for production DI).

```csharp
// With DI deps (pass manually)
var actor = Sys.ActorOf(Props.Create(() => new MyActor(opts, service)));

// Without DI deps
var actor = Sys.ActorOf(Props.Create<MyActor>());
```

### Hosting TestKit

Use `resolver.Props<T>()` since DI is available:

```csharp
protected override void ConfigureAkka(AkkaConfigurationBuilder builder, IServiceProvider provider)
{
    builder.WithActors((system, registry, resolver) =>
    {
        var actor = system.ActorOf(resolver.Props<MyActor>(), "my-actor");
        registry.Register<MyActor>(actor);
    });
}

// In test method:
var actor = await ActorRegistry.GetAsync<MyActor>();
```

## Options in tests

### Classic TestKit — TestOptionsMonitor

Create a simple `TestOptionsMonitor<T>` in shared test infrastructure:

```csharp
public sealed class TestOptionsMonitor<T>(T value) : IOptionsMonitor<T>
{
    public T CurrentValue => value;
    public T Get(string? name) => value;
    public IDisposable? OnChange(Action<T, string?> listener) => null;
}
```

Use in tests:

```csharp
var actor = Sys.ActorOf(Props.Create(() => new MyActor(
    new TestOptionsMonitor<MyOptions>(new MyOptions { PoolSize = 1 }))));
```

### Hosting TestKit — Options.Create or DI

```csharp
protected override void ConfigureServices(IServiceCollection services)
{
    services.AddSingleton(Options.Create(new MyOptions { PoolSize = 1 }));
}
```

Or for `IOptionsMonitor<T>`:

```csharp
services.AddSingleton<IOptionsMonitor<MyOptions>>(
    new TestOptionsMonitor<MyOptions>(new MyOptions { PoolSize = 1 }));
```

## Assertion patterns

### Sync assertions (Classic TestKit)

```csharp
// Expect reply
actor.Tell(new QuerySomething("id"));
var result = ExpectMsg<SomethingResult>();
Assert.Equal("id", result.Id);

// Expect message on probe
var query = probe.ExpectMsg<DoSomething>();
Assert.Equal("value", query.Field);

// Reply from probe
probe.ExpectMsg<DoSomething>();
probe.Reply(new DoSomethingCompleted(data));

// Expect no message
probe.ExpectNoMsg(TimeSpan.FromMilliseconds(200));
```

### Async assertions (Hosting TestKit)

```csharp
[Fact(Timeout = 5000)]
public async Task Should_reply_with_result()
{
    var actor = await ActorRegistry.GetAsync<MyActor>();
    actor.Tell(new QuerySomething("id"));

    var result = await ExpectMsgAsync<SomethingResult>();
    Assert.Equal("id", result.Id);
}

// Probe assertions are also async in Hosting TestKit
[Fact(Timeout = 5000)]
public async Task Should_forward_to_dependency()
{
    var probe = CreateTestProbe();
    // ... register probe

    actor.Tell(new StartWork());

    var msg = await probe.ExpectMsgAsync<DoSomething>();
    Assert.Equal("expected", msg.Field);
    probe.Reply(new DoSomethingCompleted(data));

    await ExpectMsgAsync<StartWorkCompleted>();
}
```

### Wait for async condition

```csharp
// Classic
AwaitCondition(() => someCondition);

// Hosting
await AwaitConditionAsync(() => someCondition);
```

## Multi-step interaction tests

For actors with conversation flows (ask dependency → process → ask next):

```csharp
[Fact]
public void Full_workflow()
{
    var p = RegisterProbes();
    var worker = Sys.ActorOf(Props.Create(() => new MyWorker()));

    // 1. Send initial command
    worker.Tell(new StartWork(...), TestActor);

    // 2. First dependency interaction
    p.Foo.ExpectMsg<FetchData>();
    p.Foo.Reply(new FetchDataCompleted(data));

    // 3. Next dependency
    p.Bar.ExpectMsg<ProcessData>();
    p.Bar.Reply(new ProcessDataCompleted(result));

    // 4. Final result back to caller
    var result = ExpectMsg<StartWorkCompleted>();
    Assert.Equal(expected, result.Value);
}
```

## Persistence testing

### Hosting TestKit with test persistence

```csharp
public sealed class HistoryActorSpec : Akka.Hosting.TestKit.TestKit
{
    protected override void ConfigureAkka(AkkaConfigurationBuilder builder, IServiceProvider provider)
    {
        builder
            .AddTestPersistence()
            .WithActors((system, registry, resolver) =>
            {
                var actor = system.ActorOf(resolver.Props<HistoryActor>(), "history");
                registry.Register<HistoryActor>(actor);
            });
    }

    [Fact(Timeout = 5000)]
    public async Task Should_persist_and_recover()
    {
        var actor = await ActorRegistry.GetAsync<HistoryActor>();

        // Write
        actor.Tell(new RecordHistory("item-1", data));
        await ExpectMsgAsync<RecordHistoryCompleted>();

        // Verify state
        actor.Tell(new QueryHistory());
        var result = await ExpectMsgAsync<HistoryResult>();
        Assert.Single(result.Entries);
    }
}
```

### Snapshot recovery testing (PersistenceTestKit)

For testing that recovery from snapshots works correctly:

```csharp
public sealed class SnapshotRecoverySpec : Akka.Persistence.TestKit.PersistenceTestKit
{
    // Tests that exercise SaveSnapshot → restart → SnapshotOffer → state restored
}
```

## Running tests

xUnit v3 on Microsoft.Testing.Platform — use `dotnet run`, **not** `dotnet test`:

```powershell
dotnet run --project src/<Project>.Tests/<Project>.Tests.csproj
```

## Checklist

- [ ] Choose TestKit: Classic (no DI) or Hosting (full DI + persistence)
- [ ] Register all probes/dependencies the actor resolves — missing ones cause startup failures
- [ ] `[Fact(Timeout = 5000)]` for actor tests to catch deadlocks
- [ ] Hosting TestKit: `AddTestPersistence()` for persistent actors
- [ ] Async assertions (`ExpectMsgAsync`) in Hosting TestKit
- [ ] State-only tests: plain xUnit, no TestKit, test Apply/GetSnapshot/ProcessCommand
- [ ] Persistence roundtrip tests: state → persist → recover → compare
