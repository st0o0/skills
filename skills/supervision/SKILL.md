---
description: "Akka.NET supervision strategies — BackoffSupervisor for persistent actors, custom SupervisorStrategy, and escalation patterns"
---

# Akka.NET Supervision

Actor supervision strategies: how parent actors handle child failures. Covers
BackoffSupervisor for persistent actors, custom strategies, and escalation.

## When to use

- Wrapping persistent actors to survive repeated crashes with exponential backoff
- Overriding default supervision for specific failure types
- Deciding between restart, resume, stop, and escalate

## Default supervision

Akka's default: **restart the child** on any unhandled exception. The child's
state is lost, but its mailbox is preserved. This works for most cases.

| Strategy | What happens | State | Mailbox | Use when |
|----------|-------------|-------|---------|----------|
| `Restart` | Child is restarted | Lost (PreStart runs) | Preserved | Default — transient failures |
| `Resume` | Child continues | Preserved | Preserved | Non-fatal, state is fine |
| `Stop` | Child is terminated | Lost | Lost | Unrecoverable failure |
| `Escalate` | Delegate to grandparent | — | — | Parent can't decide |

## BackoffSupervisor — restart with exponential backoff

Wraps an actor with a supervisor that restarts it after failures with increasing
delay. Essential for persistent actors that might fail repeatedly during recovery
(e.g. database unreachable).

```csharp
using Akka.Actor;
using Akka.Pattern;

public static IActorRef CreateWithBackoff<TActor>(
    ActorSystem system,
    IDependencyResolver resolver,
    string name) where TActor : ActorBase
{
    var childProps = resolver.Props<TActor>();

    var backoffProps = BackoffSupervisor.Props(
        Backoff.OnFailure(
            childProps,
            childName: name,
            minBackoff: TimeSpan.FromSeconds(1),
            maxBackoff: TimeSpan.FromSeconds(30),
            randomFactor: 0.2));

    return system.ActorOf(backoffProps, $"{name}-supervisor");
}
```

### BackoffSupervisor in Akka.Hosting setup

Register persistent actors wrapped in BackoffSupervisor:

```csharp
protected override void BuildSystem(AkkaConfigurationBuilder builder, IServiceProvider sp)
{
    builder.WithActors((system, registry, resolver) =>
    {
        RegisterWithBackoff<HistoryActor>(system, registry, resolver,
            "history", typeof(IHistoryActor));
        RegisterWithBackoff<DownloadManager>(system, registry, resolver,
            "download-manager", typeof(IDownloadManager));
    });
}

private static void RegisterWithBackoff<TActor>(
    ActorSystem system,
    IActorRegistry registry,
    IDependencyResolver resolver,
    string name,
    Type markerInterface) where TActor : ActorBase
{
    var childProps = resolver.Props<TActor>();

    var supervisorProps = BackoffSupervisor.Props(
        Backoff.OnFailure(
            childProps,
            childName: name,
            minBackoff: TimeSpan.FromSeconds(1),
            maxBackoff: TimeSpan.FromSeconds(30),
            randomFactor: 0.2));

    var supervisor = system.ActorOf(supervisorProps, $"{name}-supervisor");
    registry.Register(markerInterface, supervisor);
}
```

### Backoff.OnFailure vs Backoff.OnStop

| Variant | Triggers on | Use when |
|---------|------------|----------|
| `OnFailure` | Unhandled exception (child crashes) | Persistent actors, external service wrappers |
| `OnStop` | Child stops for any reason (including graceful) | Actors that should always be running |

### Backoff parameters

| Parameter | Purpose | Guidance |
|-----------|---------|----------|
| `minBackoff` | Initial delay after first failure | 1–3 seconds |
| `maxBackoff` | Maximum delay cap | 30–60 seconds |
| `randomFactor` | Jitter to prevent thundering herd | 0.1–0.3 |

The actual delay is: `min(maxBackoff, minBackoff * 2^restartCount * (1 + random * randomFactor))`

## Custom SupervisorStrategy

Override the default strategy when specific exceptions need different handling:

```csharp
public sealed class DownloadManager : ReceiveActor
{
    protected override SupervisorStrategy SupervisorStrategy()
    {
        return new OneForOneStrategy(
            maxNrOfRetries: 3,
            withinTimeRange: TimeSpan.FromMinutes(1),
            decider: Decider.From(ex => ex switch
            {
                HttpRequestException => Directive.Restart,
                TimeoutException => Directive.Restart,
                AuthenticationException => Directive.Stop,
                _ => Directive.Escalate,
            }));
    }
}
```

### OneForOne vs AllForOne

| Strategy | Scope | Use when |
|----------|-------|----------|
| `OneForOneStrategy` | Only the failed child is affected | Default — children are independent |
| `AllForOneStrategy` | All children are restarted/stopped | Children are interdependent (rare) |

### maxNrOfRetries and withinTimeRange

- `maxNrOfRetries: 3, withinTimeRange: 1 minute` — if the child fails 3 times within
  1 minute, the strategy **stops** it instead of restarting. Prevents infinite restart loops.
- `-1` for unlimited retries (use with BackoffSupervisor or when the actor is critical)

## Supervision and persistent actors

Persistent actors recover state from the journal on restart. Supervision restarts
preserve the PersistenceId, so recovery replays all events.

```
Failure → Supervision restarts actor → PreStart → Recovery (snapshot + events) → Ready
```

**The risk**: if the failure is caused by corrupt state in the journal, the actor
will crash again during recovery, creating a restart loop.

**The fix**: BackoffSupervisor with exponential backoff prevents rapid restart loops
and gives transient issues (database unreachable) time to resolve.

```csharp
// For every persistent actor, wrap in BackoffSupervisor:
BackoffSupervisor.Props(Backoff.OnFailure(
    resolver.Props<MyPersistentActor>(),
    childName: "my-actor",
    minBackoff: TimeSpan.FromSeconds(1),
    maxBackoff: TimeSpan.FromSeconds(30),
    randomFactor: 0.2))
```

## Escalation pattern

When a supervisor doesn't know how to handle a failure, escalate to its parent:

```csharp
protected override SupervisorStrategy SupervisorStrategy()
{
    return new OneForOneStrategy(ex => ex switch
    {
        KnownTransientException => Directive.Restart,
        _ => Directive.Escalate,  // let parent decide
    });
}
```

Escalation bubbles up until a supervisor handles it. If it reaches the root
guardian, the actor system terminates.

## Checklist

- [ ] Every persistent actor wrapped in `BackoffSupervisor` (OnFailure)
- [ ] Backoff parameters: `minBackoff` 1–3s, `maxBackoff` 30–60s, `randomFactor` 0.2
- [ ] Custom `SupervisorStrategy` only when default restart-all isn't appropriate
- [ ] `OneForOneStrategy` unless children are truly interdependent
- [ ] `maxNrOfRetries` + `withinTimeRange` to prevent infinite restart loops
- [ ] Escalate unknown exceptions — don't silently swallow
- [ ] External service wrappers: restart on transient, stop on auth/config errors
