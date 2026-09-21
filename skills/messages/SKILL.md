---
description: "Akka.NET message conventions — command/query/event/config naming, response hierarchies, and file organization"
---

# Akka.NET Message Conventions

Messages follow two organizational patterns depending on the project:

### Pattern A — Dedicated Messages project (default)

All messages live in `<Project>.Messages/<Domain>/`. Each command or query owns its full
response hierarchy in a single file. Use this when the project has many actors sharing messages
across domains. No domain-wide response interfaces.

### Pattern B — Nested in owning actor

Messages are `sealed record` types nested inside the actor class. Use this for smaller projects
or when messages are tightly coupled to a single actor. Same naming conventions apply.

Check the project's CLAUDE.md for which pattern it uses. The naming rules below apply to both.

## Command — VerbNoun

A command triggers an action. Name: `VerbNoun` (imperative). File: `VerbNoun.cs`.

```csharp
namespace <Project>.Messages.<Domain>;

public sealed record <VerbNoun>(
    <fields>);

public abstract record <VerbNoun>Response;

public sealed record <VerbNoun>Completed(
    <result fields>) : <VerbNoun>Response;

public sealed record <VerbNoun>Failed(
    Exception Cause) : <VerbNoun>Response;
```

**Rules:**
- `Failed` always has `Exception Cause` (may include additional context like `Guid RequestId`)
- `Completed` carries the result payload — name matches the command, not a generic "Result"
- Commands may extend a shared base but responses never share a base across domains
- Commands may implement marker interfaces (e.g. `IWithEntityId`) for routing

## Query — QueryNoun

A query retrieves data. Name: `QueryNoun` (prefix `Query`). File: `QueryNoun.cs`.
Response drops the `Query` prefix: `NounResponse` / `NounResult` / `NounFailed`.

```csharp
namespace <Project>.Messages.<Domain>;

public sealed record Query<Noun>(<fields>);

public abstract record <Noun>Response;

public sealed record <Noun>Result(
    <result fields>) : <Noun>Response;

public sealed record <Noun>Failed(
    Exception Cause) : <Noun>Response;
```

**Rules:**
- Complex results may nest helper records inside the `Result` record
- Supporting records can live in the same file outside the hierarchy
- Parameterless queries are valid: `public sealed record QuerySomething;`

### Simple query variant

Some queries return a plain result record without an abstract base (no failure path):

```csharp
public sealed record QueryItems(int Start = 0, int Limit = 0);
// Response is ItemsResult — no abstract base, no Failed variant
```

Use this only when the query cannot fail (direct state reads). Prefer the full
`abstract record` + `Result` / `Failed` pattern for anything involving actor Ask.

## Config — fire-and-forget

Config messages push configuration into an actor. Name: `Noun` (no verb prefix). No response.

```csharp
public sealed record SomeConfig(
    string Id,
    float Weight,
    SomeRule[] Rules);
```

Also used for simple commands that need no acknowledgement:

```csharp
public sealed record RemoveSomeConfig(string Id);
public sealed record CancelOperation(Guid OperationId);
```

## Events — past tense

Persistence events live in a dedicated Persistence project (`<Project>.Persistence/Events/<Domain>/`).
Name: past-tense verb.

```csharp
namespace <Project>.Persistence.Events.<Domain>;

public sealed record ItemCreated(Guid Id, string Name, DateTimeOffset Timestamp);
public sealed record ItemRemoved(Guid Id);
```

**Rules:**
- No `Event` or `Dto` suffix on record names
- Events are extend-only (never remove fields, only add)
- Events may reference types from the Messages project but never from domain projects

## Checklist

- [ ] Pattern A: file in `<Project>.Messages/<Domain>/`, one file per command/query
- [ ] Pattern B: nested inside owning actor class
- [ ] All records `sealed`
- [ ] `Failed` variant includes `Exception Cause`
- [ ] No cross-domain response base types
- [ ] Events in `<Project>.Persistence/Events/<Domain>/`, past-tense name, no suffix
