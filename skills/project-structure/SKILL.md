---
description: "Akka.NET solution structure — layered multi-project layout, domain isolation, and naming conventions"
---

# Akka.NET Project Structure

Layered multi-project solution pattern for Akka.NET services. Enforces domain isolation,
reference direction, and naming conventions.

## When to use

- Setting up a new Akka.NET project
- Adding a new domain to an existing project
- Reviewing project structure and dependencies

## Solution layout

All .NET source under `src/`:

```
src/
  <Project>.slnx
  Directory.Build.props               # Shared properties (TargetFramework, nullable, etc.)
  Directory.Packages.props            # Central Package Management (all NuGet versions)
  global.json                         # SDK version pin

  <Project>/                          # Host: Program.cs, DI, Configuration
  <Project>.Core/                     # Common types, marker interfaces, framework refs
  <Project>.Messages/                 # Commands, queries, responses (all domains) — OR nested in actors
  <Project>.Persistence/              # DTOs, events (extend-only, versioned)

  <Project>.<Domain1>/                # Domain project (e.g. Search, Routing, Processing)
  <Project>.<Domain2>/                # Domain project (e.g. Download, Import, Metadata)
  ...

  <Project>.Api/                      # Minimal API endpoints, Models/, Extensions/

  <Project>.<Domain1>.Tests/          # Per-domain test project
  <Project>.<Domain2>.Tests/
  <Project>.Tests.Shared/             # Shared test infrastructure (fixtures, fakes, helpers)
```

Variations:
- **Messages** may be a dedicated project (Pattern A) or nested records in actors (Pattern B) — see `akka-messages` skill
- Projects can be grouped into subdirectories (e.g. `/Foundation/`, `/Domain/`, `/Adapters/`, `/Tests/`)

## Reference direction

```
Host → Api → Domain projects → Core → Messages / Persistence
```

- **Never upward** — domains never reference Api or Host
- **Never cross-domain** — domain projects never reference each other
- **Core bundles framework refs** — Core references Messages + Persistence and declares Akka NuGet packages. Domain projects reference only Core.
- **Messages/Persistence have no Akka dependency** — pure records, no framework coupling

Enforce these rules with architecture tests in your test suite.

## Domain isolation

Each domain is a separate project. Domains communicate only through Messages (via actor Tell/Ask).

When adding a new domain:
1. Create `<Project>.<NewDomain>/` with its own `.csproj`
2. Reference only `<Project>.Core`
3. Add `<Project>.<NewDomain>.Tests/` test project
4. Add architecture tests enforcing isolation from other domains
5. Add marker interface to Core if actors need runtime resolution

## Actor naming

| Suffix | Role | Lifecycle |
|--------|------|-----------|
| `*Manager` | Orchestrator / Cluster Singleton | Long-lived, one per cluster |
| `*Worker` | Sharded Entity / child worker | Per-entity or per-task |
| `*Bridge` | External service proxy | Wraps HTTP/gRPC/MQTT clients |
| `*Actor` | Everything else | Local, child, transient |

## Core project

The Core project is the hub that all domain projects reference:

```csharp
// <Project>.Core/ActorKeys.cs — marker interfaces for actor resolution
public interface IOrderManager;
public interface IPaymentRegion;
public interface ICatalogBridge;
```

Core also contains:
- Shared enums, value objects, and constants used across domains
- Framework NuGet references (Akka.Hosting, etc.)

## Persistence project

```csharp
// <Project>.Persistence/Events/<Domain>/<EventName>.cs
namespace <Project>.Persistence.Events.<Domain>;

public sealed record ItemCreated(Guid Id, string Name, DateTimeOffset Timestamp);
public sealed record Persisted<Name>State(ItemCreated[] Items);
```

Rules:
- **Extend-only** — never remove or rename fields, only add
- New properties must be nullable or have a default
- No `Event` or `Dto` suffix on record names
- No Akka dependency
- No custom serializer — records must be serializable by the configured serializer

See `akka-persistence` for the full persistence pattern.

## Infrastructure files

| File | Purpose |
|------|---------|
| `<Project>.slnx` | Modern XML solution format |
| `Directory.Build.props` | Shared MSBuild properties (TargetFramework, nullable, implicit usings) |
| `Directory.Packages.props` | Central Package Management — all NuGet versions here, never in .csproj |
| `global.json` | SDK version pin |
| `.editorconfig` | Code style rules (enforced by `dotnet format`) |

## Build & test

All commands run from repo root:

```powershell
dotnet build src/<Project>.slnx
dotnet format src/<Project>.slnx --verify-no-changes
```

Tests use xUnit v3 on Microsoft.Testing.Platform — `dotnet run`, **not** `dotnet test`:

```powershell
dotnet run --project src/<Project>.<Domain>.Tests/<Project>.<Domain>.Tests.csproj
```

## Architecture tests (ArchUnitNET)

Enforce reference direction and domain isolation with automated architecture tests
in a dedicated `<Project>.Architecture.Tests` project.

```csharp
using ArchUnitNET.Domain;
using ArchUnitNET.Loader;
using ArchUnitNET.Fluent;
using static ArchUnitNET.Fluent.ArchRuleDefinition;

namespace <Project>.Architecture.Tests;

public sealed class ArchitectureSpec
{
    private static readonly System.Reflection.Assembly[] Assemblies =
    [
        typeof(<Project>.Core.SomeType).Assembly,
        typeof(<Project>.Messages.SomeMessage).Assembly,
        typeof(<Project>.Persistence.Events.SomeEvent).Assembly,
        typeof(<Project>.Search.SearchManager).Assembly,
        typeof(<Project>.Download.DownloadManager).Assembly,
        // ... all domain assemblies
    ];

    private static readonly Architecture Arch = new ArchLoader()
        .LoadAssemblies(Assemblies)
        .Build();

    // Messages project has no dependencies on other projects
    [Fact]
    public void Messages_should_not_depend_on_any_project()
    {
        Types().That().ResideInNamespace("<Project>.Messages", useRegularExpressions: false)
            .Should().NotDependOnAny(
                Types().That().ResideInNamespace("<Project>.Core"))
            .Check(Arch);
    }

    // Persistence has no Akka dependency
    [Fact]
    public void Persistence_should_not_depend_on_akka()
    {
        Types().That().ResideInNamespace("<Project>.Persistence", useRegularExpressions: false)
            .Should().NotDependOnAny(
                Types().That().ResideInNamespace("Akka"))
            .Check(Arch);
    }

    // Domains cannot reference each other
    [Fact]
    public void Search_should_not_depend_on_download()
    {
        Types().That().ResideInNamespace("<Project>.Search", useRegularExpressions: false)
            .Should().NotDependOnAny(
                Types().That().ResideInNamespace("<Project>.Download"))
            .Check(Arch);
    }

    // All non-abstract types must be sealed
    [Fact]
    public void All_types_should_be_sealed()
    {
        Classes().That().AreNotAbstract()
            .Should().BeSealed()
            .Check(Arch);
    }
}
```

Rules to enforce:
- Messages and Persistence have no Akka dependency
- Each domain cannot depend on other domains (one test per pair)
- Domains cannot depend on adapters (Api, ArrApi)
- All non-abstract types must be sealed

## Checklist — adding a new domain

- [ ] Create `<Project>.<NewDomain>/` with `.csproj` referencing only Core
- [ ] Create `<Project>.<NewDomain>.Tests/` test project
- [ ] Add architecture tests enforcing domain isolation
- [ ] Add marker interface to Core if actors need runtime resolution
- [ ] Register actors in actor system setup (see `akka-cluster-hosting`)
- [ ] Add setup container if domain has DI/endpoints (see `akka-setup-container`)
