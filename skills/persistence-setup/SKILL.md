---
description: "Akka.NET persistence infrastructure — Akka.Persistence.Sql setup, provider selection, clustering, health checks, and NuGet packages"
---

# Akka.NET Persistence Setup

Infrastructure setup for Akka.NET persistence, clustering, and remoting using
`Akka.Hosting` and `Akka.Persistence.Sql`. Covers the builder chain in the
actor system setup container.

## When to use

- Setting up a new Akka.NET project with persistence
- Adding persistence to an existing project
- Configuring database provider selection (dev vs production)
- Adding health checks for the actor system

## NuGet packages

| Package | Purpose |
|---------|---------|
| `Akka.Hosting` | Builder API, `ConfigureLoggers`, `WithActorSystemLivenessCheck` |
| `Akka.Persistence.Sql.Hosting` | `WithSqlPersistence` (journal + snapshot store) |
| `Akka.Remote.Hosting` | `WithRemoting` (required for clustering) |
| `Akka.Cluster.Hosting` | `WithClustering`, `WithSingleton`, `WithShardRegion` |
| `LinqToDB` | Provider name constants (`ProviderName.PostgreSQL`, etc.) |
| `Npgsql` | PostgreSQL driver (production) |
| `Microsoft.Data.Sqlite` | SQLite driver (development) |

## Builder chain

Configure in this order inside the actor system setup:

```csharp
builder
    // 1. Logging — first, so everything after is logged
    .ConfigureLoggers(loggers =>
    {
        loggers.ClearLoggers();
        loggers.AddLoggerFactory();
    })

    // 2. Persistence — journal + snapshot store
    .WithSqlPersistence(connectionString, providerName, autoInitialize: true,
        journalBuilder: journal => journal.WithHealthCheck(),
        snapshotBuilder: snapshot => snapshot.WithHealthCheck())

    // 3. Health checks
    .WithActorSystemLivenessCheck()

    // 4. Remoting + Clustering (required for singletons and shard regions)
    .WithRemoting(new RemoteOptions
    {
        HostName = "localhost",
        Port = 2552
    })
    .WithClustering(new ClusterOptions
    {
        SeedNodes = ["akka.tcp://<system-name>@localhost:2552"],
    })

    // 5. Actor registration (see akka-cluster-hosting)
    .WithSingleton<IMyManager>(...)
    .WithShardRegion<IMyRegion>(...);
```

### Order matters

| Step | Why |
|------|-----|
| Loggers first | Persistence and clustering log during setup |
| Persistence before clustering | Journal must be ready before persistent actors start |
| Remoting before clustering | Cluster needs the transport layer |
| Clustering before actors | Singletons and shard regions need the cluster |

## Logger configuration

Bridge Akka's internal logging to `Microsoft.Extensions.Logging` (and from there to Serilog):

```csharp
.ConfigureLoggers(loggers =>
{
    loggers.ClearLoggers();       // Remove default stdout logger
    loggers.AddLoggerFactory();   // Use ILoggerFactory from DI
})
```

This makes Akka logs appear in the same pipeline as application logs, with
structured properties and Serilog sinks.

## WithSqlPersistence

```csharp
.WithSqlPersistence(
    connectionString,           // Database connection string
    providerName,               // LinqToDB provider constant
    autoInitialize: true,       // Create journal/snapshot tables on startup
    journalBuilder: journal => journal.WithHealthCheck(),
    snapshotBuilder: snapshot => snapshot.WithHealthCheck())
```

### Parameters

| Parameter | Purpose |
|-----------|---------|
| `connectionString` | Standard ADO.NET connection string for the database |
| `providerName` | `LinqToDB.ProviderName` constant — determines SQL dialect |
| `autoInitialize` | `true` creates tables on startup if missing. Safe for dev; review for production. |
| `journalBuilder` | Configure journal options — `.WithHealthCheck()` registers ASP.NET health check |
| `snapshotBuilder` | Configure snapshot options — `.WithHealthCheck()` registers ASP.NET health check |

### Supported providers

| Provider | `ProviderName` constant | Use case |
|----------|------------------------|----------|
| PostgreSQL | `ProviderName.PostgreSQL` | Production — recommended |
| SQLite | `ProviderName.SQLiteMS` | Development, single-instance, embedded |
| SQL Server | `ProviderName.SqlServer` | Enterprise environments |
| MySQL | `ProviderName.MySql` | Alternative production |

## Multi-provider pattern

Select the database provider based on configuration — PostgreSQL in production,
SQLite as fallback for local development:

```csharp
protected override void BuildSystem(AkkaConfigurationBuilder builder, IServiceProvider sp)
{
    var dbOptions = sp.GetRequiredService<IOptions<DatabaseOptions>>().Value;

    string connectionString;
    string providerName;

    if (!string.IsNullOrEmpty(dbOptions.Host))
    {
        connectionString = dbOptions.ToConnectionString();
        providerName = ProviderName.PostgreSQL;
    }
    else
    {
        connectionString = $"Data Source={dataPaths.Database}";
        providerName = ProviderName.SQLiteMS;
    }

    builder.WithSqlPersistence(connectionString, providerName, autoInitialize: true,
        journalBuilder: journal => journal.WithHealthCheck(),
        snapshotBuilder: snapshot => snapshot.WithHealthCheck());
    // ... rest of builder chain
}
```

This lets the same codebase run with a full PostgreSQL in Docker and a zero-config
SQLite for quick local development.

## Remoting + Clustering

Required when using Cluster Singletons or Shard Regions (see `akka-cluster-hosting`).

```csharp
.WithRemoting(new RemoteOptions
{
    HostName = "localhost",
    Port = 2552
})
.WithClustering(new ClusterOptions
{
    SeedNodes = ["akka.tcp://<system-name>@localhost:2552"],
})
```

For single-node deployments, the node seeds itself. Multi-node setups need
proper seed node discovery (Akka.Discovery, DNS, Kubernetes, etc.).

## Health checks

Three levels of health checking, all integrating with ASP.NET Core `IHealthCheck`:

| Check | What it verifies |
|-------|-----------------|
| `journal.WithHealthCheck()` | Journal database is reachable and writable |
| `snapshot.WithHealthCheck()` | Snapshot store is reachable and writable |
| `.WithActorSystemLivenessCheck()` | Actor system is responsive (not deadlocked) |

Map in the application setup:

```csharp
app.MapHealthChecks("/healthz");
```

## Checklist

- [ ] NuGet packages: `Akka.Hosting`, `Akka.Persistence.Sql.Hosting`, `Akka.Remote.Hosting`, `Akka.Cluster.Hosting`, `LinqToDB`
- [ ] Database driver package: `Npgsql` (PostgreSQL) or `Microsoft.Data.Sqlite` (SQLite)
- [ ] `ConfigureLoggers` with `AddLoggerFactory()` — first in chain
- [ ] `WithSqlPersistence` with `autoInitialize: true` and health checks
- [ ] `WithRemoting` + `WithClustering` if using singletons or shard regions
- [ ] `WithActorSystemLivenessCheck()` for actor system health
- [ ] Health check endpoint mapped in application setup
- [ ] Connection string sourced from options/configuration, not hardcoded
