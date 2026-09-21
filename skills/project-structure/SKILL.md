---
description: "Akka.NET solution structure — layered multi-project layout, domain isolation, architecture tests, and naming conventions"
---

# Akka.NET Project Structure

Layered multi-project solution pattern for Akka.NET services. Enforces domain isolation,
reference direction, and naming conventions via architecture tests.

## When to use

- Setting up a new Akka.NET project
- Adding a new domain to an existing project
- Reviewing project structure and dependencies
- Creating architecture tests

## Solution layout

All .NET source under `src/`:

```
src/
  <Project>.slnx
  Directory.Build.props               # Shared properties (TargetFramework, nullable, etc.)
  Directory.Packages.props            # Central Package Management (all NuGet versions)
  global.json                         # SDK version pin

  <Project>/                          # Host: Program.cs, DI, Configuration
  <Project>.Core/                     # Common types, marker interfaces, framework refs (Akka/Servus)
  <Project>.Messages/                 # Commands, queries, responses (all domains) — OR nested in actors
  <Project>.Persistence/              # DTOs, events (extend-only, versioned)

  <Project>.<Domain1>/                # Domain project (e.g. Search, Routing, Processing)
  <Project>.<Domain2>/                # Domain project (e.g. Download, Import, Metadata)
  ...

  <Project>.Api/                      # Minimal API endpoints, Models/, Extensions/

  <Project>.<Domain1>.Tests/          # Per-domain test project
  <Project>.<Domain2>.Tests/
  <Project>.Architecture.Tests/       # ArchUnitNET convention enforcement
  <Project>.Tests.Shared/             # Shared test infrastructure (fixtures, fakes, helpers)
```

Variations across projects:
- **syn** groups projects into subdirectories: `/Foundation/`, `/Domain/`, `/Adapters/`, `/Tests/`
- **Messages** may be a dedicated project (Pattern A) or nested records in actors (Pattern B) — see `akka-messages` skill

## Reference direction

```
Host → Api → Domain projects → Core → Messages / Persistence
```

- **Never upward** — domains never reference Api or Host
- **Never cross-domain** — domain projects never reference each other
- **Core bundles framework refs** — Core references Messages + Persistence and declares Akka/Servus NuGet packages. Domain projects reference only Core.
- **Messages/Persistence have no Akka dependency** — pure records, no framework coupling

## Domain isolation

Each domain is a separate project. Domains communicate only through Messages (via actor Tell/Ask).

When adding a new domain:
1. Create `<Project>.<NewDomain>/` with its own `.csproj`
2. Reference only `<Project>.Core`
3. Add `<Project>.<NewDomain>.Tests/` test project
4. Add isolation test in Architecture.Tests (see below)
5. Add `AssemblyMarker` class for ArchUnit discovery

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
public interface ISearchManager;
public interface IDownloadManager;
public interface IRuleSetResolver;
```

Core also contains:
- Shared enums, value objects, and constants used across domains
- `AssemblyMarker` class
- Framework NuGet references (Akka.Hosting, Servus.Akka, etc.)

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

## Infrastructure files

| File | Purpose |
|------|---------|
| `<Project>.slnx` | Modern XML solution format |
| `Directory.Build.props` | Shared MSBuild properties (TargetFramework, nullable, implicit usings) |
| `Directory.Packages.props` | Central Package Management — all NuGet versions here, never in .csproj |
| `global.json` | SDK version pin |
| `.editorconfig` | Code style rules (enforced by `dotnet format`) |

Every project contains an `AssemblyMarker` class for ArchUnitNET discovery:

```csharp
namespace <Project>.<Domain>;

public sealed class AssemblyMarker;
```

## Architecture tests

Use ArchUnitNET (xUnit v3) to enforce conventions. Test project: `<Project>.Architecture.Tests/`.

### Template

```csharp
using ArchUnitNET.Domain;
using ArchUnitNET.Loader;
using ArchUnitNET.xUnitV3;
using static ArchUnitNET.Fluent.ArchRuleDefinition;
using Assembly = System.Reflection.Assembly;

namespace <Project>.Architecture.Tests;

public sealed class ArchitectureSpec
{
    // Load all assemblies via AssemblyMarker types
    private static readonly Assembly _messagesAssembly = typeof(Messages.AssemblyMarker).Assembly;
    private static readonly Assembly _persistenceAssembly = typeof(Persistence.AssemblyMarker).Assembly;
    private static readonly Assembly _coreAssembly = typeof(Core.AssemblyMarker).Assembly;
    private static readonly Assembly _domain1Assembly = typeof(<Domain1>.AssemblyMarker).Assembly;
    private static readonly Assembly _domain2Assembly = typeof(<Domain2>.AssemblyMarker).Assembly;
    private static readonly Assembly _apiAssembly = typeof(Api.AssemblyMarker).Assembly;

    private static readonly ArchUnitNET.Domain.Architecture _architecture =
        new ArchLoader()
            .LoadAssemblies(/* all assemblies */)
            .Build();

    private static IObjectProvider<IType> InAssembly(Assembly assembly)
        => Types().That().ResideInAssembly(assembly);

    // Define layers
    private static readonly IObjectProvider<IType> _messagesLayer = InAssembly(_messagesAssembly);
    // ... one per assembly

    // --- Enforced rules ---

    [Fact]
    public void Messages_should_not_depend_on_any_project()
    {
        Types().That().Are(_messagesLayer)
            .Should().NotDependOnAnyTypesThat().Are(_persistenceLayer)
            .AndShould().NotDependOnAnyTypesThat().Are(_coreLayer)
            // ... all other layers
            .Check(_architecture);
    }

    [Fact]
    public void Persistence_should_not_depend_on_layers_above_messages()
    {
        Types().That().Are(_persistenceLayer)
            .Should().NotDependOnAnyTypesThat().Are(_coreLayer)
            // ... all domain + adapter layers
            .Check(_architecture);
    }

    [Fact]
    public void Domain_should_not_depend_on_other_domains()
    {
        // One test per domain — each must not reference any other domain
        Types().That().Are(_domain1Layer)
            .Should().NotDependOnAnyTypesThat().Are(_domain2Layer)
            .Check(_architecture);
    }

    [Fact]
    public void Domains_should_not_depend_on_adapters()
    {
        // Domain projects must not reference Api or other adapter projects
        Types().That().Are(_domain1Layer)
            .Should().NotDependOnAnyTypesThat().Are(_apiLayer)
            .Check(_architecture);
    }

    [Fact]
    public void Messages_should_not_depend_on_akka()
    {
        Types().That().Are(_messagesLayer)
            .Should().NotDependOnAnyTypesThat().ResideInNamespace("Akka")
            .Check(_architecture);
    }

    [Fact]
    public void Persistence_should_not_depend_on_akka()
    {
        Types().That().Are(_persistenceLayer)
            .Should().NotDependOnAnyTypesThat().ResideInNamespace("Akka")
            .Check(_architecture);
    }

    [Fact]
    public void All_types_should_be_sealed()
    {
        // All non-abstract classes (except AssemblyMarker) must be sealed
        foreach (var layer in allDomainAndFoundationLayers)
        {
            Classes().That().Are(layer)
                .And().AreNotAbstract()
                .And().DoNotHaveName("AssemblyMarker")
                .Should().BeSealed()
                .Check(_architecture);
        }
    }
}
```

When adding a new domain, add its assembly + layer + isolation tests to this file.

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

## Checklist — adding a new domain

- [ ] Create `<Project>.<NewDomain>/` with `.csproj` referencing only Core
- [ ] Add `AssemblyMarker` class
- [ ] Create `<Project>.<NewDomain>.Tests/` test project
- [ ] Add assembly + layer to `ArchitectureSpec.cs`
- [ ] Add cross-domain isolation test (new domain must not depend on others)
- [ ] Add adapter isolation test (new domain must not depend on Api)
- [ ] Update existing domain tests to also exclude the new domain
- [ ] Add marker interface to Core if actors need runtime resolution
