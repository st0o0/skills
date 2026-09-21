---
description: "Servus AppBuilder setup containers — composable DI, actor system, and app pipeline configuration per domain"
---

# Servus Setup Container Pattern

Compose application startup into domain-scoped containers using Servus AppBuilder.
Each container encapsulates DI registration, actor system setup, or app pipeline
configuration for one domain.

## When to use

- Setting up a new Akka.NET + ASP.NET Core project with Servus
- Adding a new domain that needs its own DI registrations and/or endpoints
- Reviewing or refactoring startup composition

## Container types

Three container interfaces from Servus, each with a single responsibility:

### 1. `IServiceSetupContainer` — DI registration

Register services, options, HTTP clients. No app pipeline access.

```csharp
using Servus.Core.Application.Startup;

namespace <Project>.Configuration;

public sealed class <Domain>SetupContainer : IServiceSetupContainer
{
    public void SetupServices(IServiceCollection services, IConfiguration configuration)
    {
        services
            .AddOptions<<Domain>Options>()
            .Bind(configuration.GetSection(<Domain>Options.SectionName))
            .ValidateOnStart();

        services.AddSingleton<I<Service>, <Service>>();

        services.AddHttpClient(HttpClientNames.<ExternalApi>, client =>
        {
            client.BaseAddress = new Uri("https://api.example.com/");
        })
        .AddStandardResilienceHandler();
    }
}
```

### 2. `ActorSystemSetupContainer` — Akka actor system

One per project. Configures the entire actor system: persistence, clustering,
singletons, shard regions.

```csharp
using Akka.Cluster.Hosting;
using Akka.Hosting;
using Akka.Persistence.Sql.Hosting;
using Akka.Remote.Hosting;
using Servus.Akka.Startup;

namespace <Project>.Configuration;

public sealed class AkkaSetupContainer : ActorSystemSetupContainer
{
    protected override string GetActorSystemName() => "<project-name>";

    protected override void BuildSystem(AkkaConfigurationBuilder builder, IServiceProvider sp)
    {
        builder
            .ConfigureLoggers(loggers =>
            {
                loggers.ClearLoggers();
                loggers.AddLoggerFactory();
            })
            .WithSqlPersistence(connectionString, providerName, autoInitialize: true,
                journalBuilder: journal => journal.WithHealthCheck(),
                snapshotBuilder: snapshot => snapshot.WithHealthCheck())
            .WithActorSystemLivenessCheck()
            .WithRemoting(new RemoteOptions { HostName = "localhost", Port = 2552 })
            .WithClustering(new ClusterOptions
            {
                SeedNodes = [$"akka.tcp://<project-name>@localhost:2552"],
            })
            // Singletons — long-lived orchestrators
            .WithSingleton<I<Manager>>("manager-name",
                (_, _, resolver) => resolver.Props<<Manager>>())
            // Shard regions — per-entity actors
            .WithShardRegion<I<Region>>("region-name",
                (_, _, resolver) => _ => resolver.Props<<Worker>>(),
                new ShardMessageExtractor(),
                new ShardOptions { PassivateIdleEntityAfter = TimeSpan.FromSeconds(30) });
    }
}
```

### 3. `ApplicationSetupContainer<TApp>` — App pipeline

Endpoint registration, middleware, health checks.

```csharp
using Servus.Core.Application.Startup;

namespace <Project>.Configuration;

public sealed class ApplicationSetupContainer : ApplicationSetupContainer<WebApplication>
{
    protected override void SetupApplication(WebApplication app)
    {
        app.MapOpenApi();
        app.UseStaticFiles();
        app.UseOutputCache();
        app.MapHealthChecks("/healthz");
        app.MapSystemApi();
        app.MapFallbackToFile("index.html");
    }
}
```

## Dual containers

A container can implement **both** `IServiceSetupContainer` and extend
`ApplicationSetupContainer<TApp>`, grouping DI + endpoints by domain:

```csharp
public sealed class <Domain>SetupContainer : ApplicationSetupContainer<WebApplication>, IServiceSetupContainer
{
    public void SetupServices(IServiceCollection services, IConfiguration configuration)
    {
        services
            .AddOptions<<Domain>Options>()
            .Bind(configuration.GetSection(<Domain>Options.SectionName))
            .ValidateOnStart();

        services.Add<Domain>Services();
    }

    protected override void SetupApplication(WebApplication app)
    {
        app.Map<Domain>Api();
    }
}
```

Use dual containers when a domain has both services and endpoints — keeps
registration and route mapping together so adding/removing a domain is one file.

## Program.cs composition

```csharp
using <Project>.Configuration;
using Serilog;
using Servus.Core.Application.Startup;

Log.Logger = new LoggerConfiguration()
    .WriteTo.Console()
    .MinimumLevel.Debug()
    .CreateBootstrapLogger();

try
{
    var builder = WebApplication.CreateBuilder(args);
    builder.Logging.ClearProviders();

    var runner = AppBuilder.Create(builder, b => b.Build())
        .WithSetup<LoggingSetupContainer>()
        .WithSetup<ServiceSetupContainer>()
        .WithSetup<Domain1SetupContainer>()
        .WithSetup<Domain2SetupContainer>()
        .WithSetup<AkkaSetupContainer>()
        .WithSetup<ApplicationSetupContainer>()
        .Build();

    await runner.RunAsync();
}
catch (Exception ex)
{
    Log.Fatal(ex, "Application terminated unexpectedly");
}
finally
{
    await Log.CloseAndFlushAsync();
}
```

## Ordering rules

The `WithSetup` call order matters — containers execute in registration order:

```
1. LoggingSetupContainer        — bootstrap logger, Serilog config
2. ServiceSetupContainer        — core services (filesystem, paths, serialization)
3. Domain setup containers      — per-domain DI + endpoints (any order among them)
4. AkkaSetupContainer           — actor system (needs DI already registered)
5. ApplicationSetupContainer    — global middleware, health checks, fallback
```

Why this order:
- **Logging first** — captures startup errors from all subsequent containers
- **Core services before domains** — domains may depend on shared services
- **Domains before Akka** — actors resolve DI services at startup via `resolver.Props<T>()`
- **Akka before Application** — endpoints resolve actors via `IActorRegistry`
- **Application last** — global middleware wraps all registered endpoints

## Singleton vs ShardRegion registration

### Singletons — one instance per cluster

```csharp
.WithSingleton<IMarker>("actor-name",
    (_, _, resolver) => resolver.Props<MyManager>())
```

- Use for: orchestrators, managers, bridges to external services
- Actor naming convention: `*Manager`, `*Bridge`
- Marker interface in Core: `public interface IMyManager;`

### Shard regions — per-entity actors

```csharp
.WithShardRegion<IMarker>("region-name",
    (_, _, resolver) => _ => resolver.Props<MyWorker>(),
    new ShardMessageExtractor(),
    new ShardOptions { PassivateIdleEntityAfter = TimeSpan.FromSeconds(30) })
```

- Use for: per-entity state (downloads, searches, history entries)
- Actor naming convention: `*Worker`
- Marker interface in Core: `public interface IMyRegion;`
- Requires a `ShardMessageExtractor` (see below)

### ShardMessageExtractor

Route messages to the correct entity by extracting an entity ID:

```csharp
using Akka.Cluster.Sharding;
using <Project>.Messages;

namespace <Project>.Core;

public sealed class ShardMessageExtractor(int maxShards = 25) : HashCodeMessageExtractor(maxShards)
{
    public override string EntityId(object message) => message switch
    {
        IWithDownloadId m => m.DownloadId.ToString(),
        IWithSearchId m => m.SearchId.ToString(),
        IWithEntityId m => m.EntityId,
        _ => throw new ArgumentException(
            $"Unknown sharded message type: {message.GetType().Name}", nameof(message)),
    };
}
```

Messages routed to shard regions must implement a marker interface
(e.g. `IWithDownloadId`) declared in the Messages project.

### Shard regions with entity ID in Props

When the worker needs its entity ID at construction:

```csharp
.WithShardRegion<IMarker>("region-name",
    (_, _, resolver) => entityId => resolver.Props<MyWorker>(entityId),
    new ShardMessageExtractor(),
    new ShardOptions())
```

The `entityId` parameter flows through to the actor constructor.

## File layout

```
<Project>/Configuration/
    LoggingSetupContainer.cs          # Serilog + bootstrap
    ServiceSetupContainer.cs          # Core DI (filesystem, paths, serialization)
    <Domain1>SetupContainer.cs        # Domain DI + endpoints (dual)
    <Domain2>SetupContainer.cs        # Domain DI + endpoints (dual)
    AkkaSetupContainer.cs             # Actor system (one per project)
    ApplicationSetupContainer.cs      # Global middleware, health, fallback
```

## Conventions

- All containers are `sealed`
- Options: `.Bind(configuration.GetSection(T.SectionName)).ValidateOnStart()`
- HTTP clients: named via `HttpClientNames` constants, always `.AddStandardResilienceHandler()`
- One `AkkaSetupContainer` per project — never split actor registration across containers
- Domain containers group logically related services + endpoints
- Serilog bootstrap logger pattern: `CreateBootstrapLogger()` → `AddSerilog()` in container

## Checklist — adding a new domain container

- [ ] Create `<Domain>SetupContainer.cs` in `Configuration/`
- [ ] Implement `IServiceSetupContainer` and/or extend `ApplicationSetupContainer<WebApplication>`
- [ ] Register options with `.ValidateOnStart()`
- [ ] Add `app.Map<Domain>Api()` if domain has endpoints
- [ ] Add `.WithSetup<<Domain>SetupContainer>()` to Program.cs (before Akka, after core services)
- [ ] If domain has sharded actors: add marker interface to Messages, update `ShardMessageExtractor`
