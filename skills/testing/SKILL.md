---
description: "Test Akka.NET actors with TestKit — TestProbe registration, state-only unit tests, multi-step interaction flows"
---

# Akka.NET Actor Testing

## Two test styles

Pick the right one for what you're testing.

### Actor integration tests (TestKit)

Test message handling, inter-actor communication, and side effects.

```csharp
using Akka.Actor;
using Akka.Hosting;
using Akka.TestKit.Xunit;

namespace <Project>.Tests;

public sealed class <Name>Tests : TestKit
{
    // Optional: temp directory for actors that need filesystem access
    private readonly string _tempDir;

    public <Name>Tests()
    {
        _tempDir = Path.Combine(Path.GetTempPath(), $"test-{Guid.NewGuid():N}");
        Directory.CreateDirectory(_tempDir);
    }

    [Fact]
    public void Test_case_name()
    {
        // 1. Create probes and register
        // 2. Create actor
        // 3. Send message, assert
        Cleanup();
    }

    private void Cleanup()
    {
        try { Directory.Delete(_tempDir, recursive: true); }
        catch { /* ignored */ }
    }
}
```

### State-only unit tests

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

## TestProbe registration

Every actor dependency gets a `TestProbe` registered via `ActorRegistry`:

```csharp
var fooProbe = CreateTestProbe();
var barProbe = CreateTestProbe();

var registry = ActorRegistry.For(Sys);
registry.Register<IFooActor>(fooProbe);
registry.Register<IBarActor>(barProbe);
```

Register **all** marker interfaces the actor resolves via `Context.GetActor<T>()`, even if the test doesn't exercise that path — the actor will fail on startup otherwise.

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

Always `Props.Create` in tests — never `resolver.Props<T>()` (that's for production DI).

```csharp
// With DI deps (pass manually)
var actor = Sys.ActorOf(Props.Create(() => new MyActor(opts, service)));

// Without DI deps
var actor = Sys.ActorOf(Props.Create<MyActor>());
```

## IOptionsMonitor in tests

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
private static IOptionsMonitor<MyOptions> TestOpts(int poolSize = 1) =>
    new TestOptionsMonitor<MyOptions>(new MyOptions { PoolSize = poolSize });

var actor = Sys.ActorOf(Props.Create(() => new MyActor(TestOpts())));
```

## Assertion patterns

**Expect reply on TestKit** (actor replies to sender = TestActor):

```csharp
actor.Tell(new QuerySomething("id"));
var result = ExpectMsg<SomethingResult>();
Assert.Equal("id", result.Id);
```

**Expect message on probe** (actor sends to a dependency):

```csharp
var query = probe.ExpectMsg<DoSomething>();
Assert.Equal("value", query.Field);
```

**Reply from probe** (simulate dependency response):

```csharp
probe.ExpectMsg<DoSomething>();
probe.Reply(new DoSomethingCompleted(data));
```

**Expect no message** (verify actor does NOT send):

```csharp
probe.ExpectNoMsg(TimeSpan.FromMilliseconds(200));
```

**Wait for async condition**:

```csharp
AwaitCondition(() => someCondition);
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

## Running tests

xUnit v3 on Microsoft.Testing.Platform — use `dotnet run`, **not** `dotnet test`:

```powershell
dotnet run --project src/<Project>.Tests/<Project>.Tests.csproj
```
