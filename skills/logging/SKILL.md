---
description: "Akka.NET logging conventions — ILoggingAdapter in actors, ILogger in services, structured message templates, Serilog setup"
---

# Akka.NET Logging Conventions

## Two logging APIs — use the right one

| Context | Logger | Source |
|---------|--------|--------|
| **Inside actors** | `ILoggingAdapter` | `Context.GetLogger()` (from `Akka.Event`) |
| **In services / non-actor classes** | `ILogger<T>` | DI injection (Microsoft.Extensions.Logging) |

Never use `ILogger` inside actors — Akka's `ILoggingAdapter` integrates with Akka's
dispatcher and includes actor path context automatically.

Never use `ILoggingAdapter` outside actors — it requires an actor context.

## Actor logging setup

```csharp
using Akka.Event;

public sealed class MyActor : ReceiveActor
{
    private readonly ILoggingAdapter _log = Context.GetLogger();

    public MyActor()
    {
        Receive<SomeCommand>(cmd =>
        {
            _log.Info("Processing {CommandType} for {Id}", cmd.GetType().Name, cmd.Id);
        });
    }
}
```

Rules:
- Declare as `private readonly` field, initialized inline
- Name it `_log` (not `_logger`, not `Log`)
- Initialize with `Context.GetLogger()` — never inject via constructor

## Service / non-actor logging

```csharp
using Microsoft.Extensions.Logging;

public sealed class MyService
{
    private readonly ILogger<MyService> _logger;

    public MyService(ILogger<MyService> logger)
    {
        _logger = logger;
    }

    public async Task DoWork()
    {
        _logger.LogInformation("Starting work for {EntityId}", entityId);
    }
}
```

Rules:
- Inject `ILogger<T>` via constructor (typed logger)
- Name it `_logger` (not `_log`) to distinguish from actor logging

## Log levels

| Level | When to use | Example |
|-------|-------------|---------|
| `Debug` | Internal state transitions, flow tracing | `_log.Debug("At capacity ({Max}), stashing query", max)` |
| `Info` | Business events, lifecycle milestones | `_log.Info("Download {Id} enqueued: {Title}", id, title)` |
| `Warning` | Recoverable errors, unexpected but handled | `_log.Warning(ex, "Query failed, retrying")` |
| `Error` | Unrecoverable errors, data corruption | `_log.Error(ex, "Failed to extract rulesets")` |

## Structured message templates

Always use **message templates** with named placeholders — never string interpolation.

```csharp
// CORRECT — structured, searchable, no allocation when level is off
_log.Info("Search {SearchId} completed: {Count} results in {Duration}ms",
    searchId, results.Length, elapsed.TotalMilliseconds);

// WRONG — string interpolation, always allocates, not structured
_log.Info($"Search {searchId} completed: {results.Length} results");

// WRONG — positional {0} placeholders, not searchable in log aggregators
_log.Info("Search {0} completed: {1} results", searchId, results.Length);
```

Placeholder naming:
- PascalCase: `{SearchId}`, `{Count}`, `{Duration}`
- Descriptive: `{RuleSetId}` not `{Id}`, `{OldVersion}` / `{NewVersion}` not `{V1}` / `{V2}`
- Match parameter name when obvious: `_log.Info("Enqueuing {DownloadId}", downloadId)`

## Exception logging

Pass the exception as the **first argument**, message template second:

```csharp
// CORRECT — exception is first, structured message after
_log.Warning(ex, "Movie search {SearchId} failed", searchId);
_log.Error(ex, "Failed to extract community rulesets");

// WRONG — exception in the template
_log.Warning("Search failed: {Exception}", ex.Message);
```

## Actor-specific patterns

### SaveSnapshotFailure logging

```csharp
Command<SaveSnapshotFailure>(f => _log.Warning(f.Cause,
    "Snapshot save failed at sequence {SequenceNr}", f.Metadata.SequenceNr));
```

### PipeTo failure logging

```csharp
someTask.PipeTo(Self,
    success: result => new OperationCompleted(result),
    failure: ex => new OperationFailed(ex));
```

Log in the failure handler, not in the PipeTo — the actor handles the `OperationFailed`
message and decides what to log.

### Stash/capacity logging

```csharp
_log.Debug("At capacity ({MaxConcurrent}), stashing query", _maxConcurrent);
```

Use `Debug` for flow control decisions that happen frequently.

## Serilog setup

Configure Serilog in a setup container (when using Servus AppBuilder — see `akka-setup-container`)
or directly in `Program.cs`:

```csharp
using Serilog;
using Servus.Core.Application.Startup;

public sealed class LoggingSetupContainer : IServiceSetupContainer
{
    public void SetupServices(IServiceCollection services, IConfiguration configuration)
    {
        services.AddSerilog(config =>
        {
            config
                .ReadFrom.Configuration(configuration)
                .Enrich.WithMachineName()
                .Enrich.WithThreadId()
                .Enrich.FromLogContext()
                .Enrich.WithProperty("ApplicationVersion",
                    typeof(LoggingSetupContainer).Assembly.GetName().Version?.ToString() ?? "0.0.0")
                .WriteTo.Console(
                    outputTemplate: "[{Timestamp:HH:mm:ss} {Level:u3}] [{SourceContext}] {Message:lj}{NewLine}{Exception}");
        });
    }
}
```

Key enrichers:
- `WithMachineName()` — container/host identification
- `WithThreadId()` — correlate with Akka dispatchers
- `FromLogContext()` — pick up `LogContext.PushProperty` and Serilog scopes
- Custom `ApplicationVersion` property — version tracking

## Checklist

- [ ] Actors: `ILoggingAdapter _log = Context.GetLogger()` — never `ILogger`
- [ ] Services: `ILogger<T>` via DI — never `ILoggingAdapter`
- [ ] Message templates with `{PascalCase}` placeholders — never `$"interpolation"`
- [ ] Exceptions as first argument: `_log.Warning(ex, "message")`
- [ ] Appropriate level: Debug for flow, Info for events, Warning for recoverable, Error for unrecoverable
- [ ] SaveSnapshotFailure handler logs with Warning level
